# Design Chess – Low-Level Design Guide

![Bidur Sapkota](https://www.bidursapkota.com.np/images/gravatar.webp "Bidur Sapkota - Developer")&nbsp;[Bidur Sapkota](https://www.bidursapkota.com.np/)

## Table of Contents

1. [Requirements](#requirements)
2. [Key Design Patterns](#key-design-patterns)
3. [Core Enums & Constants](#core-enums--constants)
4. [Piece Hierarchy (Strategy + Template Method)](#piece-hierarchy-strategy--template-method)
5. [Board](#board)
6. [Move & Move History (Command Pattern)](#move--move-history-command-pattern)
7. [Game Engine (State Pattern)](#game-engine-state-pattern)
8. [Check, Checkmate & Stalemate Detection](#check-checkmate--stalemate-detection)
9. [Special Moves – Castling, En Passant, Promotion](#special-moves--castling-en-passant-promotion)
10. [Player & Human Input](#player--human-input)
11. [Putting It All Together](#putting-it-all-together)
12. [Class Diagram](#class-diagram)
13. [How to Run](#how-to-run)

---

## Requirements

Before writing any code, pin down exactly what the system must support. Every interviewer expects you to clarify scope first.

### Functional Requirements

| #  | Requirement                                                                 |
| -- | --------------------------------------------------------------------------- |
| 1  | Standard 8×8 board with all 32 pieces in the initial position               |
| 2  | Two players alternate turns (White moves first)                             |
| 3  | Each piece type has its own movement and capture rules                      |
| 4  | Detect check – a king is under attack                                       |
| 5  | Detect checkmate – a king is in check with no legal escape                  |
| 6  | Detect stalemate – no legal moves but not in check                          |
| 7  | Special moves: castling (king-side, queen-side), en passant, pawn promotion |
| 8  | A move is illegal if it leaves the player's own king in check               |
| 9  | Move history with undo support                                              |
| 10 | Resign and draw offer                                                       |

### Non-Functional Requirements

- Clean OOP with SOLID principles.
- Extensible – adding a new piece type or rule should not require modifying existing logic.
- Testable – each component can be unit-tested independently.

---

## Key Design Patterns

The design uses four patterns. Each earns points in an interview because it shows you are not just coding, you are architecting.

| Pattern         | Where It Is Used                            | Why                                                                                         |
| --------------- | ------------------------------------------- | ------------------------------------------------------------------------------------------- |
| Strategy        | Each `Piece` subclass defines its own moves | New piece types can be added without changing the board or game engine                       |
| Template Method | `Piece.get_legal_moves()` calls helpers     | Common filtering logic (stay on board, don't capture own piece) is written once in the base |
| Command         | `Move` objects encapsulate a turn           | Each move can be executed and undone, enabling move history and replay                       |
| State           | `GameStatus` enum drives game flow          | The game engine's behaviour changes cleanly based on the current state                      |

---

## Core Enums & Constants

Start with the value types that every other class depends on.

```python
from enum import Enum


class Color(Enum):
    WHITE = "white"
    BLACK = "black"

    def opposite(self) -> "Color":
        return Color.BLACK if self == Color.WHITE else Color.WHITE


class GameStatus(Enum):
    ACTIVE = "active"
    WHITE_WINS = "white_wins"
    BLACK_WINS = "black_wins"
    STALEMATE = "stalemate"
    RESIGNED = "resigned"


BOARD_SIZE = 8
```

`Color` represents the two sides. `opposite()` is a convenience method so you never have to write ternary checks elsewhere. `GameStatus` is the state machine that the game engine will transition through. `BOARD_SIZE` is a constant to avoid magic numbers.

---

## Piece Hierarchy (Strategy + Template Method)

Every piece shares common data (color, position, whether it has moved) but differs in how it generates candidate squares. The base class provides the template; subclasses fill in the strategy.

### Base Class

```python
from __future__ import annotations
from abc import ABC, abstractmethod
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from board import Board


class Piece(ABC):
    """Abstract base for all chess pieces."""

    def __init__(self, color: Color, row: int, col: int) -> None:
        self.color = color
        self.row = row
        self.col = col
        self.has_moved = False

    @abstractmethod
    def symbol(self) -> str:
        """Single-character symbol for display (e.g. K, Q, R, B, N, P)."""
        ...

    @abstractmethod
    def _candidate_moves(self, board: Board) -> list[tuple[int, int]]:
        """Return all squares this piece can reach, ignoring check rules."""
        ...

    def get_legal_moves(self, board: Board) -> list[tuple[int, int]]:
        """Filter candidates: stay in bounds, don't capture own piece."""
        moves: list[tuple[int, int]] = []
        for r, c in self._candidate_moves(board):
            if 0 <= r < BOARD_SIZE and 0 <= c < BOARD_SIZE:
                target = board.get_piece(r, c)
                if target is None or target.color != self.color:
                    moves.append((r, c))
        return moves

    def __repr__(self) -> str:
        color_char = "w" if self.color == Color.WHITE else "b"
        return f"{color_char}{self.symbol()}"
```

`_candidate_moves` is the abstract hook (Strategy). `get_legal_moves` is the Template Method — it calls the hook and applies shared filtering. `TYPE_CHECKING` avoids circular imports; the `Board` type hint is only needed by the type checker, not at runtime. `has_moved` tracks whether a piece has ever moved, needed for castling and pawn double-step rules.

### Sliding Pieces Helper

Rook, Bishop, and Queen all slide along directions until they hit the edge or another piece. Factor this out to avoid duplication.

```python
def _slide(piece: Piece, board: Board, directions: list[tuple[int, int]]) -> list[tuple[int, int]]:
    """Generate squares along each direction until blocked."""
    result: list[tuple[int, int]] = []
    for dr, dc in directions:
        r, c = piece.row + dr, piece.col + dc
        while 0 <= r < BOARD_SIZE and 0 <= c < BOARD_SIZE:
            target = board.get_piece(r, c)
            if target is None:
                result.append((r, c))
            else:
                if target.color != piece.color:
                    result.append((r, c))  # can capture
                break  # blocked either way
            r += dr
            c += dc
    return result
```

The function walks one step at a time in each direction. If the square is empty, it adds it and keeps going. If the square has a piece, it adds it only if it is an opponent (capture), then stops. This single function powers three piece types.

### King

```python
class King(Piece):
    def symbol(self) -> str:
        return "K"

    def _candidate_moves(self, board: Board) -> list[tuple[int, int]]:
        moves: list[tuple[int, int]] = []
        for dr in (-1, 0, 1):
            for dc in (-1, 0, 1):
                if dr == 0 and dc == 0:
                    continue
                moves.append((self.row + dr, self.col + dc))
        return moves
```

The king moves one square in any direction — the 8 surrounding squares. Castling is handled separately in the game engine because it involves two pieces and multiple conditions.

### Queen

```python
class Queen(Piece):
    def symbol(self) -> str:
        return "Q"

    def _candidate_moves(self, board: Board) -> list[tuple[int, int]]:
        directions = [
            (-1, 0), (1, 0), (0, -1), (0, 1),   # rook directions
            (-1, -1), (-1, 1), (1, -1), (1, 1),  # bishop directions
        ]
        return _slide(self, board, directions)
```

The queen is simply a combination of rook and bishop directions.

### Rook

```python
class Rook(Piece):
    def symbol(self) -> str:
        return "R"

    def _candidate_moves(self, board: Board) -> list[tuple[int, int]]:
        directions = [(-1, 0), (1, 0), (0, -1), (0, 1)]
        return _slide(self, board, directions)
```

### Bishop

```python
class Bishop(Piece):
    def symbol(self) -> str:
        return "B"

    def _candidate_moves(self, board: Board) -> list[tuple[int, int]]:
        directions = [(-1, -1), (-1, 1), (1, -1), (1, 1)]
        return _slide(self, board, directions)
```

### Knight

```python
class Knight(Piece):
    def symbol(self) -> str:
        return "N"

    def _candidate_moves(self, board: Board) -> list[tuple[int, int]]:
        offsets = [
            (-2, -1), (-2, 1), (-1, -2), (-1, 2),
            (1, -2), (1, 2), (2, -1), (2, 1),
        ]
        return [(self.row + dr, self.col + dc) for dr, dc in offsets]
```

The knight moves in an "L" shape. It can jump over other pieces, so there is no sliding logic — just the 8 possible destination squares. Bounds and friendly-piece checks are handled by `get_legal_moves` in the base class.

### Pawn

```python
class Pawn(Piece):
    def symbol(self) -> str:
        return "P"

    def _candidate_moves(self, board: Board) -> list[tuple[int, int]]:
        moves: list[tuple[int, int]] = []
        direction = -1 if self.color == Color.WHITE else 1
        start_row = 6 if self.color == Color.WHITE else 1

        # One square forward
        r = self.row + direction
        if 0 <= r < BOARD_SIZE and board.get_piece(r, self.col) is None:
            moves.append((r, self.col))

            # Two squares forward from starting position
            r2 = self.row + 2 * direction
            if self.row == start_row and board.get_piece(r2, self.col) is None:
                moves.append((r2, self.col))

        # Diagonal captures
        for dc in (-1, 1):
            nr, nc = self.row + direction, self.col + dc
            if 0 <= nr < BOARD_SIZE and 0 <= nc < BOARD_SIZE:
                target = board.get_piece(nr, nc)
                if target is not None and target.color != self.color:
                    moves.append((nr, nc))

        # En passant captures
        if board.en_passant_target is not None:
            ep_r, ep_c = board.en_passant_target
            if ep_r == self.row + direction and abs(ep_c - self.col) == 1:
                moves.append((ep_r, ep_c))

        return moves
```

White pawns move upward (row decreases), black pawns move downward (row increases). `start_row` determines the initial rank for the two-square advance. Diagonal captures only happen when an enemy piece occupies that square. En passant is detected by checking the board's `en_passant_target`, which is set by the game engine whenever a pawn makes a two-square advance.

> **Note on Pawn vs Template Method:** The pawn overrides `_candidate_moves` but its capture logic is baked in (it cannot capture forward). This is intentional — the base class's `get_legal_moves` filter still applies for bounds checking, but the pawn's `_candidate_moves` already handles its unique forward-only, capture-diagonally semantics.

---

## Board

The board holds an 8×8 grid of pieces and exposes query methods. It knows nothing about game rules or turns.

```python
class Board:
    """8×8 chess board with piece placement and query methods."""

    def __init__(self) -> None:
        self.grid: list[list[Piece | None]] = [
            [None] * BOARD_SIZE for _ in range(BOARD_SIZE)
        ]
        self.en_passant_target: tuple[int, int] | None = None
        self._setup_pieces()

    def _setup_pieces(self) -> None:
        """Place all 32 pieces in the standard starting position."""
        # Row 0 = top of the board = Black's back rank
        # Row 7 = bottom of the board = White's back rank
        piece_order = [Rook, Knight, Bishop, Queen, King, Bishop, Knight, Rook]

        for col, PieceClass in enumerate(piece_order):
            self.grid[0][col] = PieceClass(Color.BLACK, 0, col)
            self.grid[7][col] = PieceClass(Color.WHITE, 7, col)

        for col in range(BOARD_SIZE):
            self.grid[1][col] = Pawn(Color.BLACK, 1, col)
            self.grid[6][col] = Pawn(Color.WHITE, 6, col)

    def get_piece(self, row: int, col: int) -> Piece | None:
        return self.grid[row][col]

    def set_piece(self, row: int, col: int, piece: Piece | None) -> None:
        self.grid[row][col] = piece
        if piece is not None:
            piece.row = row
            piece.col = col

    def find_king(self, color: Color) -> King:
        """Locate the king of the given color."""
        for row in self.grid:
            for piece in row:
                if isinstance(piece, King) and piece.color == color:
                    return piece
        raise ValueError(f"No {color.value} king found on the board")

    def get_all_pieces(self, color: Color) -> list[Piece]:
        """Return every piece of the given color still on the board."""
        pieces: list[Piece] = []
        for row in self.grid:
            for piece in row:
                if piece is not None and piece.color == color:
                    pieces.append(piece)
        return pieces

    def is_square_attacked(self, row: int, col: int, by_color: Color) -> bool:
        """Check whether any piece of `by_color` can reach (row, col)."""
        for piece in self.get_all_pieces(by_color):
            if (row, col) in piece.get_legal_moves(self):
                return True
        return False

    def display(self) -> None:
        """Print the board to the console."""
        print("   a  b  c  d  e  f  g  h")
        print("  +" + "--+" * 8)
        for r in range(BOARD_SIZE):
            row_str = f"{8 - r} |"
            for c in range(BOARD_SIZE):
                piece = self.grid[r][c]
                cell = str(piece) if piece else "  "
                row_str += f"{cell}|"
            print(row_str + f" {8 - r}")
            print("  +" + "--+" * 8)
        print("   a  b  c  d  e  f  g  h\n")
```

`_setup_pieces` uses the `piece_order` list to place the back rank. This is cleaner than eight separate assignments. `set_piece` always updates the piece's internal row and col to keep them in sync with the grid. `is_square_attacked` is the foundation for check detection — it asks whether any enemy piece can reach a given square. `en_passant_target` stores the square behind a pawn that just moved two squares, used by `Pawn._candidate_moves`.

---

## Move & Move History (Command Pattern)

Each move is an object that knows how to execute itself and how to undo itself. This is the Command pattern.

```python
from dataclasses import dataclass, field


@dataclass
class Move:
    """Encapsulates a single chess move with full undo information."""

    piece: Piece
    from_row: int
    from_col: int
    to_row: int
    to_col: int
    captured_piece: Piece | None = None
    is_castling: bool = False
    castling_rook: Piece | None = None
    castling_rook_from: tuple[int, int] | None = None
    castling_rook_to: tuple[int, int] | None = None
    is_en_passant: bool = False
    en_passant_captured_pos: tuple[int, int] | None = None
    promotion_piece: Piece | None = None
    previous_en_passant_target: tuple[int, int] | None = None
    piece_had_moved: bool = False
    rook_had_moved: bool = False

    def execute(self, board: Board) -> None:
        """Apply this move to the board."""
        self.piece_had_moved = self.piece.has_moved
        self.previous_en_passant_target = board.en_passant_target

        # Remove piece from origin
        board.set_piece(self.from_row, self.from_col, None)

        # Handle en passant capture
        if self.is_en_passant and self.en_passant_captured_pos:
            ep_r, ep_c = self.en_passant_captured_pos
            self.captured_piece = board.get_piece(ep_r, ep_c)
            board.set_piece(ep_r, ep_c, None)

        # Handle promotion
        if self.promotion_piece is not None:
            board.set_piece(self.to_row, self.to_col, self.promotion_piece)
        else:
            board.set_piece(self.to_row, self.to_col, self.piece)

        self.piece.has_moved = True

        # Handle castling – move the rook as well
        if self.is_castling and self.castling_rook:
            self.rook_had_moved = self.castling_rook.has_moved
            cr_from = self.castling_rook_from
            cr_to = self.castling_rook_to
            if cr_from and cr_to:
                board.set_piece(cr_from[0], cr_from[1], None)
                board.set_piece(cr_to[0], cr_to[1], self.castling_rook)
                self.castling_rook.has_moved = True

        # Set en passant target if pawn moved two squares
        if isinstance(self.piece, Pawn) and abs(self.to_row - self.from_row) == 2:
            mid_row = (self.from_row + self.to_row) // 2
            board.en_passant_target = (mid_row, self.to_col)
        else:
            board.en_passant_target = None

    def undo(self, board: Board) -> None:
        """Reverse this move, restoring the board to its previous state."""
        # Restore en passant target
        board.en_passant_target = self.previous_en_passant_target

        # Undo castling rook
        if self.is_castling and self.castling_rook:
            cr_from = self.castling_rook_from
            cr_to = self.castling_rook_to
            if cr_from and cr_to:
                board.set_piece(cr_to[0], cr_to[1], None)
                board.set_piece(cr_from[0], cr_from[1], self.castling_rook)
                self.castling_rook.has_moved = self.rook_had_moved

        # Undo promotion – put the original pawn back
        if self.promotion_piece is not None:
            board.set_piece(self.to_row, self.to_col, None)
            board.set_piece(self.from_row, self.from_col, self.piece)
        else:
            board.set_piece(self.to_row, self.to_col, None)
            board.set_piece(self.from_row, self.from_col, self.piece)

        self.piece.has_moved = self.piece_had_moved

        # Restore captured piece
        if self.is_en_passant and self.en_passant_captured_pos:
            ep_r, ep_c = self.en_passant_captured_pos
            board.set_piece(ep_r, ep_c, self.captured_piece)
        elif self.captured_piece is not None:
            board.set_piece(self.to_row, self.to_col, self.captured_piece)

    def to_algebraic(self) -> str:
        """Human-readable representation like 'e2e4' or 'Nf3'."""
        files = "abcdefgh"
        src = f"{files[self.from_col]}{8 - self.from_row}"
        dst = f"{files[self.to_col]}{8 - self.to_row}"
        return f"{self.piece.symbol()}{src}{dst}"


class MoveHistory:
    """Stack of moves supporting undo."""

    def __init__(self) -> None:
        self._moves: list[Move] = []

    def push(self, move: Move) -> None:
        self._moves.append(move)

    def pop(self) -> Move | None:
        return self._moves.pop() if self._moves else None

    def get_all(self) -> list[Move]:
        return list(self._moves)

    def __len__(self) -> int:
        return len(self._moves)
```

`Move` stores everything needed to fully undo itself: the captured piece, the previous en passant target, whether the piece had previously moved, and castling rook information. `execute` applies the move and updates all board state. `undo` reverses it exactly. `MoveHistory` is a simple stack; `pop` returns the last move so the game engine can undo it. `to_algebraic` converts the move to a human-readable string for display.

---

## Game Engine (State Pattern)

The game engine owns the board, enforces turn order, validates moves, and transitions between game states.

```python
class ChessGame:
    """Main game engine that orchestrates a chess match."""

    def __init__(self) -> None:
        self.board = Board()
        self.current_turn = Color.WHITE
        self.status = GameStatus.ACTIVE
        self.history = MoveHistory()

    # ------------------------------------------------------------------
    # Public API
    # ------------------------------------------------------------------

    def make_move(self, from_row: int, from_col: int,
                  to_row: int, to_col: int,
                  promotion_choice: str = "Q") -> bool:
        """
        Attempt to move a piece from (from_row, from_col) to (to_row, to_col).
        Returns True if the move was successful, False otherwise.
        """
        if self.status != GameStatus.ACTIVE:
            print("Game is over.")
            return False

        piece = self.board.get_piece(from_row, from_col)

        # Validate: piece exists and belongs to the current player
        if piece is None:
            print("No piece at the source square.")
            return False
        if piece.color != self.current_turn:
            print("Not your piece.")
            return False

        # Check if destination is in the piece's legal moves
        legal_moves = piece.get_legal_moves(self.board)

        # Add castling moves for the king
        if isinstance(piece, King):
            legal_moves += self._get_castling_moves(piece)

        if (to_row, to_col) not in legal_moves:
            print("Illegal move for this piece.")
            return False

        # Build the Move object
        move = self._build_move(piece, from_row, from_col, to_row, to_col,
                                promotion_choice)

        # Execute and verify the king is not left in check
        move.execute(self.board)

        if self._is_in_check(self.current_turn):
            move.undo(self.board)
            print("Move leaves your king in check.")
            return False

        # Move is valid – commit it
        self.history.push(move)
        self._switch_turn()
        self._update_game_status()
        return True

    def undo_move(self) -> bool:
        """Undo the last move."""
        move = self.history.pop()
        if move is None:
            print("No moves to undo.")
            return False
        move.undo(self.board)
        self._switch_turn()
        self.status = GameStatus.ACTIVE
        return True

    def resign(self) -> None:
        """Current player resigns."""
        self.status = GameStatus.RESIGNED
        winner = self.current_turn.opposite()
        print(f"{self.current_turn.value} resigns. {winner.value} wins!")

    # ------------------------------------------------------------------
    # Internal helpers
    # ------------------------------------------------------------------

    def _switch_turn(self) -> None:
        self.current_turn = self.current_turn.opposite()

    def _is_in_check(self, color: Color) -> bool:
        """Is the king of the given color currently in check?"""
        king = self.board.find_king(color)
        return self.board.is_square_attacked(king.row, king.col,
                                             color.opposite())

    def _has_any_legal_move(self, color: Color) -> bool:
        """Can the player of the given color make at least one legal move?"""
        for piece in self.board.get_all_pieces(color):
            legal = piece.get_legal_moves(self.board)
            if isinstance(piece, King):
                legal += self._get_castling_moves(piece)
            for to_r, to_c in legal:
                move = self._build_move(piece, piece.row, piece.col,
                                        to_r, to_c)
                move.execute(self.board)
                still_in_check = self._is_in_check(color)
                move.undo(self.board)
                if not still_in_check:
                    return True
        return False

    def _update_game_status(self) -> None:
        """After a turn switch, check for checkmate or stalemate."""
        if not self._has_any_legal_move(self.current_turn):
            if self._is_in_check(self.current_turn):
                # Checkmate
                winner = self.current_turn.opposite()
                self.status = (GameStatus.WHITE_WINS
                               if winner == Color.WHITE
                               else GameStatus.BLACK_WINS)
                print(f"Checkmate! {winner.value} wins!")
            else:
                # Stalemate
                self.status = GameStatus.STALEMATE
                print("Stalemate! The game is a draw.")
        elif self._is_in_check(self.current_turn):
            print(f"{self.current_turn.value} is in check!")

    # ------------------------------------------------------------------
    # Move construction
    # ------------------------------------------------------------------

    def _build_move(self, piece: Piece, fr: int, fc: int,
                    tr: int, tc: int,
                    promotion_choice: str = "Q") -> Move:
        """Create a Move object with all special-move metadata."""
        captured = self.board.get_piece(tr, tc)
        move = Move(piece=piece, from_row=fr, from_col=fc,
                    to_row=tr, to_col=tc, captured_piece=captured)

        # Castling detection
        if isinstance(piece, King) and abs(tc - fc) == 2:
            move.is_castling = True
            if tc > fc:  # King-side
                rook = self.board.get_piece(fr, 7)
                move.castling_rook = rook
                move.castling_rook_from = (fr, 7)
                move.castling_rook_to = (fr, 5)
            else:  # Queen-side
                rook = self.board.get_piece(fr, 0)
                move.castling_rook = rook
                move.castling_rook_from = (fr, 0)
                move.castling_rook_to = (fr, 3)

        # En passant detection
        if (isinstance(piece, Pawn)
                and self.board.en_passant_target == (tr, tc)):
            move.is_en_passant = True
            move.en_passant_captured_pos = (fr, tc)

        # Promotion detection
        promotion_row = 0 if piece.color == Color.WHITE else 7
        if isinstance(piece, Pawn) and tr == promotion_row:
            promo_map: dict[str, type[Piece]] = {
                "Q": Queen, "R": Rook, "B": Bishop, "N": Knight,
            }
            PieceClass = promo_map.get(promotion_choice.upper(), Queen)
            move.promotion_piece = PieceClass(piece.color, tr, tc)

        return move

    # ------------------------------------------------------------------
    # Castling
    # ------------------------------------------------------------------

    def _get_castling_moves(self, king: King) -> list[tuple[int, int]]:
        """Return castling destination squares for the given king."""
        moves: list[tuple[int, int]] = []
        if king.has_moved:
            return moves

        row = king.row
        enemy = king.color.opposite()

        # Cannot castle out of check
        if self.board.is_square_attacked(row, king.col, enemy):
            return moves

        # King-side castling
        rook_ks = self.board.get_piece(row, 7)
        if (isinstance(rook_ks, Rook)
                and not rook_ks.has_moved
                and self.board.get_piece(row, 5) is None
                and self.board.get_piece(row, 6) is None
                and not self.board.is_square_attacked(row, 5, enemy)
                and not self.board.is_square_attacked(row, 6, enemy)):
            moves.append((row, 6))

        # Queen-side castling
        rook_qs = self.board.get_piece(row, 0)
        if (isinstance(rook_qs, Rook)
                and not rook_qs.has_moved
                and self.board.get_piece(row, 1) is None
                and self.board.get_piece(row, 2) is None
                and self.board.get_piece(row, 3) is None
                and not self.board.is_square_attacked(row, 2, enemy)
                and not self.board.is_square_attacked(row, 3, enemy)):
            moves.append((row, 2))

        return moves
```

`make_move` is the single entry point for all moves. It validates the piece, checks legality, builds a `Move` object, executes it, verifies the king is safe, and only then commits it. If the move leaves the king in check, it undoes it immediately. `_has_any_legal_move` tries every possible move for a player by executing and undoing each one — this is the standard way to detect checkmate and stalemate. `_get_castling_moves` enforces all five castling requirements: king and rook haven't moved, squares between are empty, king doesn't pass through or land in check, and king is not currently in check.

---

## Check, Checkmate & Stalemate Detection

These are handled by three methods in `ChessGame`:

| Method               | What It Does                                                   |
| -------------------- | -------------------------------------------------------------- |
| `_is_in_check`       | Checks if any enemy piece attacks the king's square            |
| `_has_any_legal_move`| Tries every move for a player, returns True if one is legal    |
| `_update_game_status`| Combines the above: no legal moves + check = checkmate, etc.  |

The flow after every move:

```
make_move() succeeds
  → switch turn
  → _update_game_status()
      → _has_any_legal_move(current_turn)?
          No  → _is_in_check(current_turn)?
                  Yes → CHECKMATE (opponent wins)
                  No  → STALEMATE (draw)
          Yes → _is_in_check(current_turn)?
                  Yes → print "in check" (game continues)
                  No  → normal (game continues)
```

This approach is correct by construction. Every possible response move is tried (execute + undo), so there are no false negatives. The only tradeoff is performance — in the worst case this evaluates hundreds of moves — but for a standard chess game on an 8×8 board this is negligible.

---

## Special Moves – Castling, En Passant, Promotion

### Castling

Castling moves the king two squares toward a rook, and the rook jumps to the other side. Five conditions must all be true:

1. The king has never moved (`king.has_moved == False`).
2. The chosen rook has never moved (`rook.has_moved == False`).
3. All squares between king and rook are empty.
4. The king is not currently in check.
5. The king does not pass through or land on a square attacked by an enemy piece.

Castling is detected in `_build_move` when a king moves two columns. The rook's move is bundled into the same `Move` object so it can be undone atomically.

### En Passant

When a pawn advances two squares from its starting position, an opponent pawn on an adjacent file can capture it "in passing" on the very next move, as if the first pawn had only moved one square.

Implementation: When a pawn moves two squares, the board's `en_passant_target` is set to the square behind it. On the next move, if an opponent pawn captures onto that target square, the `Move` object is flagged with `is_en_passant = True` and the captured pawn's actual position is recorded. The `en_passant_target` is cleared after every move that is not a two-square pawn advance.

### Pawn Promotion

When a pawn reaches the opponent's back rank (row 0 for white, row 7 for black), it must be promoted to a queen, rook, bishop, or knight. The `promotion_choice` parameter in `make_move` lets the caller choose. Default is queen.

---

## Player & Human Input

```python
class Player:
    """Represents a chess player."""

    def __init__(self, name: str, color: Color) -> None:
        self.name = name
        self.color = color


def parse_input(notation: str) -> tuple[int, int, int, int] | None:
    """
    Parse 'e2e4' or 'e2 e4' style input into board coordinates.
    Returns (from_row, from_col, to_row, to_col) or None if invalid.
    """
    notation = notation.strip().replace(" ", "")
    if len(notation) < 4:
        return None

    files = "abcdefgh"
    try:
        fc = files.index(notation[0])
        fr = 8 - int(notation[1])
        tc = files.index(notation[2])
        tr = 8 - int(notation[3])
    except (ValueError, IndexError):
        return None

    if not (0 <= fr < 8 and 0 <= fc < 8 and 0 <= tr < 8 and 0 <= tc < 8):
        return None

    return fr, fc, tr, tc
```

`parse_input` converts standard algebraic-style coordinate notation (`e2e4`) into zero-indexed row/col pairs. File `a` maps to column 0, rank `1` maps to row 7 (bottom of the board for white). This is deliberately simple — a production system would support full algebraic notation with disambiguation.

---

## Putting It All Together

```python
def main() -> None:
    game = ChessGame()
    white = Player("White", Color.WHITE)
    black = Player("Black", Color.BLACK)
    players = {Color.WHITE: white, Color.BLACK: black}

    print("=== CHESS ===")
    print("Enter moves as 'e2e4'. Type 'undo' to undo, 'resign' to resign, 'quit' to exit.\n")

    while game.status == GameStatus.ACTIVE:
        game.board.display()
        player = players[game.current_turn]
        raw = input(f"{player.name}'s turn ({player.color.value}): ").strip().lower()

        if raw == "quit":
            print("Game ended by player.")
            break
        elif raw == "undo":
            game.undo_move()
            continue
        elif raw == "resign":
            game.resign()
            break

        coords = parse_input(raw)
        if coords is None:
            print("Invalid input. Use format like 'e2e4'.\n")
            continue

        fr, fc, tr, tc = coords

        # Ask for promotion choice if applicable
        promo = "Q"
        piece = game.board.get_piece(fr, fc)
        promotion_row = 0 if game.current_turn == Color.WHITE else 7
        if isinstance(piece, Pawn) and tr == promotion_row:
            promo = input("Promote to (Q/R/B/N): ").strip().upper()
            if promo not in ("Q", "R", "B", "N"):
                promo = "Q"

        success = game.make_move(fr, fc, tr, tc, promo)
        if not success:
            print("Try again.\n")

    # Final board state
    game.board.display()
    print(f"Final status: {game.status.value}")


if __name__ == "__main__":
    main()
```

The game loop is straightforward: display the board, read input, attempt the move, repeat. All intelligence lives in `ChessGame.make_move`. The loop only handles I/O and the quit/undo/resign commands.

---

## Class Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                          ChessGame                              │
│─────────────────────────────────────────────────────────────────│
│ - board: Board                                                  │
│ - current_turn: Color                                           │
│ - status: GameStatus                                            │
│ - history: MoveHistory                                          │
│─────────────────────────────────────────────────────────────────│
│ + make_move(fr, fc, tr, tc, promo) -> bool                      │
│ + undo_move() -> bool                                           │
│ + resign() -> None                                              │
│ - _is_in_check(color) -> bool                                   │
│ - _has_any_legal_move(color) -> bool                            │
│ - _update_game_status() -> None                                 │
│ - _build_move(...) -> Move                                      │
│ - _get_castling_moves(king) -> list[tuple]                      │
└────────────────────────┬────────────────────────────────────────┘
                         │ owns
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                            Board                                │
│─────────────────────────────────────────────────────────────────│
│ - grid: list[list[Piece | None]]                                │
│ - en_passant_target: tuple | None                               │
│─────────────────────────────────────────────────────────────────│
│ + get_piece(r, c) -> Piece | None                               │
│ + set_piece(r, c, piece) -> None                                │
│ + find_king(color) -> King                                      │
│ + get_all_pieces(color) -> list[Piece]                          │
│ + is_square_attacked(r, c, by_color) -> bool                    │
│ + display() -> None                                             │
└────────────────────────┬────────────────────────────────────────┘
                         │ contains
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                       «abstract» Piece                          │
│─────────────────────────────────────────────────────────────────│
│ - color: Color                                                  │
│ - row: int                                                      │
│ - col: int                                                      │
│ - has_moved: bool                                               │
│─────────────────────────────────────────────────────────────────│
│ + symbol() -> str              «abstract»                       │
│ + _candidate_moves(board) -> list[tuple]  «abstract»            │
│ + get_legal_moves(board) -> list[tuple]   «template method»     │
└────────────────────────┬────────────────────────────────────────┘
                         │ extended by
         ┌───────┬───────┼───────┬───────┬───────┐
         ▼       ▼       ▼       ▼       ▼       ▼
       King    Queen    Rook   Bishop  Knight   Pawn


┌─────────────────────────────────────────────────────────────────┐
│                           Move                                  │
│─────────────────────────────────────────────────────────────────│
│ - piece, from/to coords, captured_piece                         │
│ - is_castling, castling_rook, rook positions                    │
│ - is_en_passant, en_passant_captured_pos                        │
│ - promotion_piece, previous state flags                         │
│─────────────────────────────────────────────────────────────────│
│ + execute(board) -> None                                        │
│ + undo(board) -> None                                           │
│ + to_algebraic() -> str                                         │
└─────────────────────────────────────────────────────────────────┘

┌────────────────────┐       ┌──────────────┐
│    MoveHistory     │       │    Player     │
│────────────────────│       │──────────────│
│ - _moves: list     │       │ - name: str   │
│────────────────────│       │ - color: Color │
│ + push(move)       │       └──────────────┘
│ + pop() -> Move    │
│ + get_all() -> list│
└────────────────────┘

┌──────────────┐    ┌────────────────┐
│  Color(Enum) │    │ GameStatus(Enum)│
│──────────────│    │────────────────│
│ WHITE        │    │ ACTIVE         │
│ BLACK        │    │ WHITE_WINS     │
│──────────────│    │ BLACK_WINS     │
│ + opposite() │    │ STALEMATE      │
└──────────────┘    │ RESIGNED       │
                    └────────────────┘
```

---

## How to Run

Save all code into a single file `chess.py` (or split into modules) and run:

```bash
python chess.py
```

### Example Session

```
=== CHESS ===
Enter moves as 'e2e4'. Type 'undo' to undo, 'resign' to resign, 'quit' to exit.

   a  b  c  d  e  f  g  h
  +--+--+--+--+--+--+--+--+
8 |bR|bN|bB|bQ|bK|bB|bN|bR| 8
  +--+--+--+--+--+--+--+--+
7 |bP|bP|bP|bP|bP|bP|bP|bP| 7
  +--+--+--+--+--+--+--+--+
6 |  |  |  |  |  |  |  |  | 6
  +--+--+--+--+--+--+--+--+
5 |  |  |  |  |  |  |  |  | 5
  +--+--+--+--+--+--+--+--+
4 |  |  |  |  |  |  |  |  | 4
  +--+--+--+--+--+--+--+--+
3 |  |  |  |  |  |  |  |  | 3
  +--+--+--+--+--+--+--+--+
2 |wP|wP|wP|wP|wP|wP|wP|wP| 2
  +--+--+--+--+--+--+--+--+
1 |wR|wN|wB|wQ|wK|wB|wN|wR| 1
  +--+--+--+--+--+--+--+--+
   a  b  c  d  e  f  g  h

White's turn (white): e2e4
Black's turn (black): e7e5
White's turn (white): g1f3
...
```

### Scholar's Mate (4-Move Checkmate)

Test the checkmate detection with the classic scholar's mate:

```
White: e2e4
Black: e7e5
White: f1c4
Black: b8c6
White: d1h5
Black: g8f6
White: h5f7
→ Checkmate! white wins!
```
