# Design Tic-Tac-Toe – Low-Level Design Guide

![Bidur Sapkota](https://www.bidursapkota.com.np/images/gravatar.webp "Bidur Sapkota - Developer")&nbsp;[Bidur Sapkota](https://www.bidursapkota.com.np/)

## Table of Contents

1. [Requirements](#requirements)
2. [Key Design Patterns](#key-design-patterns)
3. [Core Enums & Constants](#core-enums--constants)
4. [Board & Win Detection](#board--win-detection)
5. [Move & Undo (Command Pattern)](#move--undo-command-pattern)
6. [Player Hierarchy (Strategy Pattern)](#player-hierarchy-strategy-pattern)
7. [Bot Strategies – Easy, Medium, Unbeatable](#bot-strategies--easy-medium-unbeatable)
8. [Game Engine (State Pattern)](#game-engine-state-pattern)
9. [Game Factory (Factory Pattern)](#game-factory-factory-pattern)
10. [Putting It All Together](#putting-it-all-together)
11. [Class Diagram](#class-diagram)
12. [How to Run](#how-to-run)

---

## Requirements

### Functional Requirements

| #  | Requirement                                                                 |
| -- | --------------------------------------------------------------------------- |
| 1  | Configurable N×N board (default 3×3)                                        |
| 2  | Two players alternate turns; X always goes first                            |
| 3  | A player wins by completing a full row, column, or diagonal with their mark |
| 4  | The game is a draw if the board is full with no winner                      |
| 5  | A move is invalid if the cell is already occupied or out of bounds          |
| 6  | Support human vs human, human vs bot, and bot vs bot modes                  |
| 7  | Bot difficulty levels: easy (random), medium (heuristic), unbeatable (minimax) |
| 8  | Move history with undo support                                              |
| 9  | Configurable win condition (e.g. 4-in-a-row on a 5×5 board)                |
| 10 | Game reset without creating a new instance                                  |

### Non-Functional Requirements

- Clean OOP with SOLID principles.
- Extensible – adding a new bot strategy should not require changing the game engine.
- Testable – deterministic bot and board state allow unit testing without randomness.

---

## Key Design Patterns

| Pattern  | Where It Is Used                          | Why                                                                                       |
| -------- | ----------------------------------------- | ----------------------------------------------------------------------------------------- |
| Strategy | Bot strategies (`BotStrategy` interface)  | Swap between random, heuristic, and minimax without changing the `BotPlayer` class         |
| Command  | `Move` objects with execute/undo          | Each move can be applied and reversed, enabling undo and replay                            |
| State    | `GameStatus` drives game flow             | The engine's behaviour changes cleanly based on whether the game is active, won, or drawn  |
| Factory  | `GameFactory` creates preconfigured games | Complex wiring of board, players, and strategies is centralised in one place               |

---

## Core Enums & Constants

```python
from enum import Enum


class Mark(Enum):
    X = "X"
    O = "O"
    EMPTY = " "

    def opposite(self) -> "Mark":
        if self == Mark.X:
            return Mark.O
        if self == Mark.O:
            return Mark.X
        raise ValueError("EMPTY has no opposite")

    def __str__(self) -> str:
        return self.value


class GameStatus(Enum):
    ACTIVE = "active"
    X_WINS = "x_wins"
    O_WINS = "o_wins"
    DRAW = "draw"


DEFAULT_BOARD_SIZE = 3
```

`Mark` represents what occupies a cell — `X`, `O`, or `EMPTY`. `opposite()` is used by the minimax algorithm and turn switching. `__str__` returns just the value so the board prints cleanly. `GameStatus` is the state machine with four terminal states. `DEFAULT_BOARD_SIZE` avoids magic numbers.

---

## Board & Win Detection

The board is the core data structure. It owns the grid, validates positions, checks for wins, and renders itself. All win detection lives here because it depends only on board state.

```python
from __future__ import annotations


class Board:
    """N×N Tic-Tac-Toe board with win detection."""

    def __init__(self, size: int = DEFAULT_BOARD_SIZE,
                 win_length: int | None = None) -> None:
        self._size = size
        self._win_length = win_length if win_length else size
        self._grid: list[list[Mark]] = [
            [Mark.EMPTY] * size for _ in range(size)
        ]
        self._move_count = 0

    @property
    def size(self) -> int:
        return self._size

    @property
    def win_length(self) -> int:
        return self._win_length

    @property
    def move_count(self) -> int:
        return self._move_count

    # ------------------------------------------------------------------
    # Cell operations
    # ------------------------------------------------------------------

    def get_cell(self, row: int, col: int) -> Mark:
        self._validate_bounds(row, col)
        return self._grid[row][col]

    def set_cell(self, row: int, col: int, mark: Mark) -> None:
        self._validate_bounds(row, col)
        if self._grid[row][col] != Mark.EMPTY:
            raise ValueError(f"Cell ({row}, {col}) is already occupied")
        self._grid[row][col] = mark
        self._move_count += 1

    def clear_cell(self, row: int, col: int) -> None:
        """Used by undo — clears a cell without validation."""
        self._validate_bounds(row, col)
        if self._grid[row][col] != Mark.EMPTY:
            self._grid[row][col] = Mark.EMPTY
            self._move_count -= 1

    def is_empty(self, row: int, col: int) -> bool:
        return self._grid[row][col] == Mark.EMPTY

    def is_full(self) -> bool:
        return self._move_count == self._size * self._size

    def get_empty_cells(self) -> list[tuple[int, int]]:
        """Return all unoccupied positions."""
        return [
            (r, c)
            for r in range(self._size)
            for c in range(self._size)
            if self._grid[r][c] == Mark.EMPTY
        ]

    # ------------------------------------------------------------------
    # Win detection
    # ------------------------------------------------------------------

    def check_winner(self) -> Mark | None:
        """
        Return the winning mark, or None if no winner yet.
        Checks all rows, columns, and diagonals for `win_length` in a row.
        """
        # Check rows
        for r in range(self._size):
            winner = self._check_line(
                [(r, c) for c in range(self._size)]
            )
            if winner:
                return winner

        # Check columns
        for c in range(self._size):
            winner = self._check_line(
                [(r, c) for r in range(self._size)]
            )
            if winner:
                return winner

        # Check main diagonals (top-left to bottom-right)
        for r in range(self._size - self._win_length + 1):
            for c in range(self._size - self._win_length + 1):
                winner = self._check_line(
                    [(r + i, c + i) for i in range(self._win_length)]
                )
                if winner:
                    return winner

        # Check anti-diagonals (top-right to bottom-left)
        for r in range(self._size - self._win_length + 1):
            for c in range(self._win_length - 1, self._size):
                winner = self._check_line(
                    [(r + i, c - i) for i in range(self._win_length)]
                )
                if winner:
                    return winner

        return None

    def _check_line(self, positions: list[tuple[int, int]]) -> Mark | None:
        """
        Check if any contiguous subsequence of `win_length` in the
        given positions contains the same non-empty mark.
        """
        marks = [self._grid[r][c] for r, c in positions]
        for start in range(len(marks) - self._win_length + 1):
            window = marks[start:start + self._win_length]
            if window[0] != Mark.EMPTY and all(m == window[0] for m in window):
                return window[0]
        return None

    # ------------------------------------------------------------------
    # Validation & display
    # ------------------------------------------------------------------

    def _validate_bounds(self, row: int, col: int) -> None:
        if not (0 <= row < self._size and 0 <= col < self._size):
            raise ValueError(
                f"Position ({row}, {col}) is out of bounds "
                f"for a {self._size}×{self._size} board"
            )

    def display(self) -> None:
        """Print the board with grid lines and coordinates."""
        # Column headers
        header = "   " + "   ".join(str(c) for c in range(self._size))
        print(header)
        separator = "  +" + "---+" * self._size

        print(separator)
        for r in range(self._size):
            row_str = f"{r} |"
            for c in range(self._size):
                row_str += f" {self._grid[r][c]} |"
            print(row_str)
            print(separator)
        print()

    def reset(self) -> None:
        """Clear the board for a new game."""
        self._grid = [
            [Mark.EMPTY] * self._size for _ in range(self._size)
        ]
        self._move_count = 0
```

`_size` is the board dimension and `_win_length` is how many marks in a row are needed to win. On a standard 3×3 board they are both 3, but on a 5×5 board you might want 4-in-a-row. `_move_count` tracks the number of marks placed, so `is_full()` runs in O(1) instead of scanning the entire grid. `set_cell` validates that the cell is empty before placing a mark. `clear_cell` is the inverse, used by the undo system. `check_winner` scans all rows, columns, and both diagonal directions. `_check_line` uses a sliding window of size `win_length` to support boards larger than the win condition. `get_empty_cells` is used by bot strategies to find available moves.

---

## Move & Undo (Command Pattern)

Each move is an object that can execute and undo itself. This is the Command pattern — the same approach used in the Chess design.

```python
from dataclasses import dataclass


@dataclass
class Move:
    """Encapsulates a single Tic-Tac-Toe move with undo capability."""

    row: int
    col: int
    mark: Mark

    def execute(self, board: Board) -> None:
        """Place the mark on the board."""
        board.set_cell(self.row, self.col, self.mark)

    def undo(self, board: Board) -> None:
        """Remove the mark from the board."""
        board.clear_cell(self.row, self.col)

    def __repr__(self) -> str:
        return f"{self.mark.value}({self.row},{self.col})"


class MoveHistory:
    """Stack of moves supporting undo and replay."""

    def __init__(self) -> None:
        self._moves: list[Move] = []

    def push(self, move: Move) -> None:
        self._moves.append(move)

    def pop(self) -> Move | None:
        return self._moves.pop() if self._moves else None

    def get_all(self) -> list[Move]:
        return list(self._moves)

    def clear(self) -> None:
        self._moves.clear()

    def __len__(self) -> int:
        return len(self._moves)

    def __repr__(self) -> str:
        return " → ".join(str(m) for m in self._moves)
```

The `Move` dataclass is lightweight — just row, col, and mark. `execute` calls `board.set_cell`, and `undo` calls `board.clear_cell`. `MoveHistory` is a stack. `__repr__` on both classes makes the move log human-readable: `X(1,1) → O(0,0) → X(2,2)`.

---

## Player Hierarchy (Strategy Pattern)

Players come in two types: human (reads from input) and bot (uses a strategy). The Strategy pattern lets the bot's decision-making algorithm be swapped independently.

### Abstract Player

```python
from abc import ABC, abstractmethod


class Player(ABC):
    """Abstract base for all player types."""

    def __init__(self, name: str, mark: Mark) -> None:
        self.name = name
        self.mark = mark

    @abstractmethod
    def get_move(self, board: Board) -> tuple[int, int]:
        """Return the (row, col) for this player's next move."""
        ...

    def __repr__(self) -> str:
        return f"{self.name}({self.mark.value})"
```

`get_move` is the abstract method that every player type must implement. The game engine calls this without knowing whether it is talking to a human or a bot.

### Human Player

```python
class HumanPlayer(Player):
    """Reads moves from console input."""

    def get_move(self, board: Board) -> tuple[int, int]:
        while True:
            try:
                raw = input(f"  {self.name} ({self.mark}), enter row col: ")
                parts = raw.strip().replace(",", " ").split()
                if len(parts) != 2:
                    print("  ✗ Enter two numbers separated by space (e.g. '1 2')")
                    continue
                row, col = int(parts[0]), int(parts[1])
                if not (0 <= row < board.size and 0 <= col < board.size):
                    print(f"  ✗ Position out of bounds. Use 0-{board.size - 1}")
                    continue
                if not board.is_empty(row, col):
                    print("  ✗ Cell is already occupied")
                    continue
                return row, col
            except ValueError:
                print("  ✗ Invalid input. Enter numbers only.")
```

The human player loops until valid input is received. It accepts comma-separated or space-separated numbers. All validation (bounds, occupancy) happens here so the game engine never receives an invalid move from a human.

### Bot Player

```python
class BotPlayer(Player):
    """Uses a pluggable strategy to choose moves."""

    def __init__(self, name: str, mark: Mark,
                 strategy: "BotStrategy") -> None:
        super().__init__(name, mark)
        self._strategy = strategy

    def get_move(self, board: Board) -> tuple[int, int]:
        move = self._strategy.choose_move(board, self.mark)
        print(f"  🤖 {self.name} ({self.mark}) plays ({move[0]}, {move[1]})")
        return move
```

`BotPlayer` delegates to a `BotStrategy`. The strategy is injected through the constructor, so changing the bot's difficulty is just a matter of passing a different strategy object. The bot prints its move so the human can follow the game.

---

## Bot Strategies – Easy, Medium, Unbeatable

### Abstract Strategy

```python
class BotStrategy(ABC):
    """Abstract strategy for bot move selection."""

    @abstractmethod
    def choose_move(self, board: Board, mark: Mark) -> tuple[int, int]:
        """Given the board state and the bot's mark, return (row, col)."""
        ...

    @abstractmethod
    def difficulty(self) -> str:
        """Human-readable difficulty name."""
        ...
```

### Easy – Random

```python
import random


class EasyStrategy(BotStrategy):
    """Picks a random empty cell. No intelligence."""

    def choose_move(self, board: Board, mark: Mark) -> tuple[int, int]:
        empty = board.get_empty_cells()
        if not empty:
            raise RuntimeError("No empty cells available")
        return random.choice(empty)

    def difficulty(self) -> str:
        return "Easy (Random)"
```

`EasyStrategy` is the simplest possible bot — it picks uniformly at random from available cells. It serves as a baseline and is useful for testing.

### Medium – Heuristic

```python
class MediumStrategy(BotStrategy):
    """
    Uses a priority-based heuristic:
    1. Win if possible (complete a row/col/diagonal).
    2. Block opponent from winning.
    3. Take the center.
    4. Take a corner.
    5. Take any available cell.
    """

    def choose_move(self, board: Board, mark: Mark) -> tuple[int, int]:
        opponent = mark.opposite()
        empty = board.get_empty_cells()

        # 1. Try to win
        for r, c in empty:
            board.set_cell(r, c, mark)
            if board.check_winner() == mark:
                board.clear_cell(r, c)
                return r, c
            board.clear_cell(r, c)

        # 2. Block opponent from winning
        for r, c in empty:
            board.set_cell(r, c, opponent)
            if board.check_winner() == opponent:
                board.clear_cell(r, c)
                return r, c
            board.clear_cell(r, c)

        # 3. Take center if available
        center = board.size // 2
        if board.is_empty(center, center):
            return center, center

        # 4. Take a corner
        corners = [
            (0, 0), (0, board.size - 1),
            (board.size - 1, 0), (board.size - 1, board.size - 1),
        ]
        available_corners = [(r, c) for r, c in corners if board.is_empty(r, c)]
        if available_corners:
            return random.choice(available_corners)

        # 5. Take any available cell
        return random.choice(empty)

    def difficulty(self) -> str:
        return "Medium (Heuristic)"
```

The medium strategy follows a classic priority list. Step 1 checks if the bot can win immediately by trying each empty cell — place the mark, check for a winner, then undo. Step 2 does the same for the opponent to find blocking moves. Steps 3-5 are positional heuristics. This bot is competent but beatable by a perfect player.

### Unbeatable – Minimax with Alpha-Beta Pruning

```python
class MinimaxStrategy(BotStrategy):
    """
    Perfect play using the Minimax algorithm with alpha-beta pruning.
    Guaranteed to never lose on a standard 3×3 board.
    """

    def __init__(self, max_depth: int | None = None) -> None:
        self._max_depth = max_depth  # None = unlimited

    def choose_move(self, board: Board, mark: Mark) -> tuple[int, int]:
        best_score = float("-inf")
        best_move: tuple[int, int] | None = None

        for r, c in board.get_empty_cells():
            board.set_cell(r, c, mark)
            score = self._minimax(
                board, mark.opposite(), mark,
                is_maximizing=False,
                alpha=float("-inf"),
                beta=float("inf"),
                depth=0,
            )
            board.clear_cell(r, c)

            if score > best_score:
                best_score = score
                best_move = (r, c)

        if best_move is None:
            raise RuntimeError("No moves available")
        return best_move

    def _minimax(
        self,
        board: Board,
        current_mark: Mark,
        bot_mark: Mark,
        is_maximizing: bool,
        alpha: float,
        beta: float,
        depth: int,
    ) -> float:
        """
        Recursively evaluate board positions.
        Returns +10 if the bot wins, -10 if opponent wins, 0 for draw.
        Subtracts depth to prefer faster wins and slower losses.
        """
        winner = board.check_winner()
        if winner == bot_mark:
            return 10 - depth       # prefer faster wins
        if winner == bot_mark.opposite():
            return depth - 10       # prefer slower losses
        if board.is_full():
            return 0                # draw

        # Depth limit for larger boards
        if self._max_depth is not None and depth >= self._max_depth:
            return 0  # treat as neutral

        if is_maximizing:
            max_eval = float("-inf")
            for r, c in board.get_empty_cells():
                board.set_cell(r, c, current_mark)
                eval_score = self._minimax(
                    board, current_mark.opposite(), bot_mark,
                    False, alpha, beta, depth + 1,
                )
                board.clear_cell(r, c)
                max_eval = max(max_eval, eval_score)
                alpha = max(alpha, eval_score)
                if beta <= alpha:
                    break  # prune
            return max_eval
        else:
            min_eval = float("inf")
            for r, c in board.get_empty_cells():
                board.set_cell(r, c, current_mark)
                eval_score = self._minimax(
                    board, current_mark.opposite(), bot_mark,
                    True, alpha, beta, depth + 1,
                )
                board.clear_cell(r, c)
                min_eval = min(min_eval, eval_score)
                beta = min(beta, eval_score)
                if beta <= alpha:
                    break  # prune
            return min_eval

    def difficulty(self) -> str:
        return "Unbeatable (Minimax)"
```

The minimax algorithm explores every possible game tree. The maximizing player tries to maximize the score; the minimizing player tries to minimize it. The scoring uses `10 - depth` to prefer faster wins and `depth - 10` to prefer slower losses. Alpha-beta pruning eliminates branches that cannot affect the final decision, dramatically reducing the search space. `_max_depth` is an optional limit for larger boards where full minimax is too slow — on a standard 3×3 board with 9! = 362,880 maximum positions, alpha-beta prunes this to a few hundred evaluations.

---

## Game Engine (State Pattern)

The game engine orchestrates the match: manages turns, validates moves, detects wins and draws, and supports undo and reset.

```python
class TicTacToeGame:
    """Main game engine that orchestrates a Tic-Tac-Toe match."""

    def __init__(self, board: Board,
                 player_x: Player, player_o: Player) -> None:
        if player_x.mark != Mark.X or player_o.mark != Mark.O:
            raise ValueError("Player marks must be X and O respectively")

        self._board = board
        self._players = {Mark.X: player_x, Mark.O: player_o}
        self._current_mark = Mark.X
        self._status = GameStatus.ACTIVE
        self._history = MoveHistory()

    # ------------------------------------------------------------------
    # Properties
    # ------------------------------------------------------------------

    @property
    def board(self) -> Board:
        return self._board

    @property
    def status(self) -> GameStatus:
        return self._status

    @property
    def current_player(self) -> Player:
        return self._players[self._current_mark]

    @property
    def history(self) -> MoveHistory:
        return self._history

    # ------------------------------------------------------------------
    # Game flow
    # ------------------------------------------------------------------

    def play_turn(self) -> GameStatus:
        """
        Execute one complete turn: get move from player, apply it,
        check for win/draw, switch turns.
        Returns the updated game status.
        """
        if self._status != GameStatus.ACTIVE:
            print(f"Game is over: {self._status.value}")
            return self._status

        player = self.current_player
        row, col = player.get_move(self._board)

        # Build and execute the move
        move = Move(row=row, col=col, mark=self._current_mark)
        move.execute(self._board)
        self._history.push(move)

        # Check for winner
        winner = self._board.check_winner()
        if winner is not None:
            self._status = (GameStatus.X_WINS
                            if winner == Mark.X
                            else GameStatus.O_WINS)
            return self._status

        # Check for draw
        if self._board.is_full():
            self._status = GameStatus.DRAW
            return self._status

        # Switch turns
        self._current_mark = self._current_mark.opposite()
        return self._status

    def undo_move(self) -> bool:
        """Undo the last move and switch back to the previous player."""
        move = self._history.pop()
        if move is None:
            print("No moves to undo.")
            return False
        move.undo(self._board)
        self._current_mark = move.mark  # revert to the player who made it
        self._status = GameStatus.ACTIVE
        return True

    def reset(self) -> None:
        """Reset the game to the initial state."""
        self._board.reset()
        self._history.clear()
        self._current_mark = Mark.X
        self._status = GameStatus.ACTIVE

    def play_full_game(self) -> GameStatus:
        """Run the entire game until it ends."""
        self._board.display()
        while self._status == GameStatus.ACTIVE:
            self.play_turn()
            self._board.display()
        self._print_result()
        return self._status

    def _print_result(self) -> None:
        match self._status:
            case GameStatus.X_WINS:
                player = self._players[Mark.X]
                print(f"🏆 {player.name} (X) wins!")
            case GameStatus.O_WINS:
                player = self._players[Mark.O]
                print(f"🏆 {player.name} (O) wins!")
            case GameStatus.DRAW:
                print("🤝 It's a draw!")

        print(f"\nMove log: {self._history}")
```

`play_turn` is the single entry point for one turn. It asks the current player for a move, executes it, checks for win or draw, and switches turns. The player's `get_move` handles all input validation, so the engine trusts the returned coordinates. `undo_move` pops the last move from history, reverses it on the board, and reverts the current turn to the player who made that move. `reset` clears everything for a new game without creating a new `TicTacToeGame` instance. `play_full_game` loops until the game ends, displaying the board after each turn.

---

## Game Factory (Factory Pattern)

The factory encapsulates the complex wiring of board, players, and strategies. Callers pick a game mode and get a ready-to-play instance.

```python
class GameFactory:
    """Factory for creating preconfigured game instances."""

    @staticmethod
    def human_vs_human(
        name_x: str = "Player X",
        name_o: str = "Player O",
        board_size: int = DEFAULT_BOARD_SIZE,
        win_length: int | None = None,
    ) -> TicTacToeGame:
        board = Board(board_size, win_length)
        px = HumanPlayer(name_x, Mark.X)
        po = HumanPlayer(name_o, Mark.O)
        return TicTacToeGame(board, px, po)

    @staticmethod
    def human_vs_bot(
        human_name: str = "You",
        bot_name: str = "Bot",
        difficulty: str = "medium",
        human_mark: Mark = Mark.X,
        board_size: int = DEFAULT_BOARD_SIZE,
        win_length: int | None = None,
    ) -> TicTacToeGame:
        strategy = GameFactory._get_strategy(difficulty, board_size)
        board = Board(board_size, win_length)

        if human_mark == Mark.X:
            px = HumanPlayer(human_name, Mark.X)
            po = BotPlayer(bot_name, Mark.O, strategy)
        else:
            px = BotPlayer(bot_name, Mark.X, strategy)
            po = HumanPlayer(human_name, Mark.O)

        return TicTacToeGame(board, px, po)

    @staticmethod
    def bot_vs_bot(
        name_x: str = "Bot X",
        name_o: str = "Bot O",
        difficulty_x: str = "minimax",
        difficulty_o: str = "easy",
        board_size: int = DEFAULT_BOARD_SIZE,
        win_length: int | None = None,
    ) -> TicTacToeGame:
        board = Board(board_size, win_length)
        sx = GameFactory._get_strategy(difficulty_x, board_size)
        so = GameFactory._get_strategy(difficulty_o, board_size)
        px = BotPlayer(name_x, Mark.X, sx)
        po = BotPlayer(name_o, Mark.O, so)
        return TicTacToeGame(board, px, po)

    @staticmethod
    def _get_strategy(difficulty: str, board_size: int) -> BotStrategy:
        match difficulty.lower():
            case "easy":
                return EasyStrategy()
            case "medium":
                return MediumStrategy()
            case "minimax" | "hard" | "unbeatable":
                # Limit depth on larger boards for performance
                max_depth = None if board_size <= 3 else board_size * 2
                return MinimaxStrategy(max_depth=max_depth)
            case _:
                raise ValueError(
                    f"Unknown difficulty: {difficulty}. "
                    f"Choose from: easy, medium, minimax"
                )
```

`human_vs_human` creates the simplest game — two humans with console input. `human_vs_bot` lets the human choose their mark and the bot's difficulty. `bot_vs_bot` is useful for testing strategies against each other. `_get_strategy` maps difficulty strings to strategy objects and applies sensible depth limits for larger boards.

---

## Putting It All Together

### Human vs Human

```python
def main_pvp() -> None:
    print("=== TIC-TAC-TOE: Human vs Human ===\n")
    game = GameFactory.human_vs_human("Alice", "Bob")
    game.play_full_game()


if __name__ == "__main__":
    main_pvp()
```

### Human vs Bot

```python
def main_pvb() -> None:
    print("=== TIC-TAC-TOE: Human vs Bot ===\n")

    name = input("Your name: ").strip() or "Player"
    difficulty = input("Difficulty (easy/medium/minimax): ").strip() or "medium"

    game = GameFactory.human_vs_bot(
        human_name=name,
        bot_name="Skynet",
        difficulty=difficulty,
    )
    game.play_full_game()


if __name__ == "__main__":
    main_pvb()
```

### Bot vs Bot (Strategy Comparison)

```python
def main_bvb() -> None:
    """Run multiple games to compare bot strategies."""
    print("=== TIC-TAC-TOE: Bot vs Bot ===\n")

    results = {GameStatus.X_WINS: 0, GameStatus.O_WINS: 0, GameStatus.DRAW: 0}
    num_games = 100

    for i in range(num_games):
        game = GameFactory.bot_vs_bot(
            name_x="Minimax",
            name_o="Random",
            difficulty_x="minimax",
            difficulty_o="easy",
        )
        game.board  # suppress display
        # Silently play the game
        while game.status == GameStatus.ACTIVE:
            game.play_turn()
        results[game.status] += 1

    print(f"Results after {num_games} games:")
    print(f"  Minimax (X) wins: {results[GameStatus.X_WINS]}")
    print(f"  Random  (O) wins: {results[GameStatus.O_WINS]}")
    print(f"  Draws:            {results[GameStatus.DRAW]}")
```

This batch runner pits minimax against random over 100 games. Minimax should never lose — the results will show only wins and draws. This is a great way to validate that the minimax implementation is correct.

### Interactive Menu

```python
def main() -> None:
    print("╔═══════════════════════════════════╗")
    print("║       TIC-TAC-TOE                 ║")
    print("╠═══════════════════════════════════╣")
    print("║  1. Human vs Human                ║")
    print("║  2. Human vs Bot                  ║")
    print("║  3. Bot vs Bot (single game)      ║")
    print("║  4. Bot vs Bot (100 games)        ║")
    print("║  5. Quit                          ║")
    print("╚═══════════════════════════════════╝")

    while True:
        choice = input("\nSelect mode (1-5): ").strip()

        match choice:
            case "1":
                game = GameFactory.human_vs_human()
                game.play_full_game()
            case "2":
                difficulty = input("Difficulty (easy/medium/minimax): ").strip()
                game = GameFactory.human_vs_bot(difficulty=difficulty or "medium")
                game.play_full_game()
            case "3":
                game = GameFactory.bot_vs_bot(
                    difficulty_x="minimax", difficulty_o="medium"
                )
                game.play_full_game()
            case "4":
                main_bvb()
            case "5":
                print("Goodbye!")
                break
            case _:
                print("Invalid choice.")
                continue

        again = input("\nPlay again? (y/n): ").strip().lower()
        if again != "y":
            break


if __name__ == "__main__":
    main()
```

The interactive menu lets the user pick a game mode, play, and replay. All the complexity is hidden behind `GameFactory` — the main function is just UI wiring.

### Unit Tests

```python
def test_x_wins_row() -> None:
    """Test that X wins by completing a row."""
    board = Board(3)
    board.set_cell(0, 0, Mark.X)
    board.set_cell(0, 1, Mark.X)
    board.set_cell(0, 2, Mark.X)
    assert board.check_winner() == Mark.X
    print("✅ test_x_wins_row passed")


def test_o_wins_column() -> None:
    """Test that O wins by completing a column."""
    board = Board(3)
    board.set_cell(0, 2, Mark.O)
    board.set_cell(1, 2, Mark.O)
    board.set_cell(2, 2, Mark.O)
    assert board.check_winner() == Mark.O
    print("✅ test_o_wins_column passed")


def test_diagonal_win() -> None:
    """Test diagonal win detection."""
    board = Board(3)
    board.set_cell(0, 0, Mark.X)
    board.set_cell(1, 1, Mark.X)
    board.set_cell(2, 2, Mark.X)
    assert board.check_winner() == Mark.X
    print("✅ test_diagonal_win passed")


def test_draw() -> None:
    """Test draw detection on a full board with no winner."""
    board = Board(3)
    # X O X
    # X X O
    # O X O
    moves = [
        (0, 0, Mark.X), (0, 1, Mark.O), (0, 2, Mark.X),
        (1, 0, Mark.X), (1, 1, Mark.X), (1, 2, Mark.O),
        (2, 0, Mark.O), (2, 1, Mark.X), (2, 2, Mark.O),
    ]
    for r, c, m in moves:
        board.set_cell(r, c, m)
    assert board.check_winner() is None
    assert board.is_full()
    print("✅ test_draw passed")


def test_undo() -> None:
    """Test that undo restores the board state."""
    board = Board(3)
    move = Move(1, 1, Mark.X)
    move.execute(board)
    assert board.get_cell(1, 1) == Mark.X
    move.undo(board)
    assert board.get_cell(1, 1) == Mark.EMPTY
    assert board.move_count == 0
    print("✅ test_undo passed")


def test_minimax_never_loses() -> None:
    """Test that minimax never loses against random over 50 games."""
    losses = 0
    for _ in range(50):
        game = GameFactory.bot_vs_bot(
            difficulty_x="minimax", difficulty_o="easy"
        )
        while game.status == GameStatus.ACTIVE:
            game.play_turn()
        if game.status == GameStatus.O_WINS:
            losses += 1
    assert losses == 0, f"Minimax lost {losses} times!"
    print("✅ test_minimax_never_loses passed (50 games)")


def run_all_tests() -> None:
    print("\n" + "=" * 40)
    print("  RUNNING TESTS")
    print("=" * 40 + "\n")
    test_x_wins_row()
    test_o_wins_column()
    test_diagonal_win()
    test_draw()
    test_undo()
    test_minimax_never_loses()
    print("\n✅ All tests passed!\n")
```

Each test is self-contained: create a board, set up the scenario, assert the result. `test_minimax_never_loses` runs 50 bot-vs-bot games to statistically verify that minimax is unbeatable. These tests run without user input and can be integrated into a CI pipeline.

---

## Class Diagram

```
┌──────────────────────────────────────────────────────┐
│                   TicTacToeGame                       │
│──────────────────────────────────────────────────────│
│ - _board: Board                                      │
│ - _players: dict[Mark, Player]                       │
│ - _current_mark: Mark                                │
│ - _status: GameStatus                                │
│ - _history: MoveHistory                              │
│──────────────────────────────────────────────────────│
│ + play_turn() -> GameStatus                          │
│ + undo_move() -> bool                                │
│ + reset() -> None                                    │
│ + play_full_game() -> GameStatus                     │
└──────┬──────────────┬────────────────────────────────┘
       │              │
  owns ▼         uses ▼
┌─────────────┐  ┌─────────────────────────────────────┐
│    Board    │  │         «abstract» Player            │
│─────────────│  │─────────────────────────────────────│
│ - _size     │  │ + name: str                          │
│ - _win_len  │  │ + mark: Mark                         │
│ - _grid     │  │─────────────────────────────────────│
│ - _move_cnt │  │ + get_move(board) -> (row, col)      │
│─────────────│  └──────────┬──────────────────────────┘
│ + get_cell  │             │ extended by
│ + set_cell  │       ┌─────┴──────┐
│ + clear_cell│       ▼            ▼
│ + check_    │  HumanPlayer   BotPlayer
│   winner()  │                    │ uses
│ + is_full() │                    ▼
│ + get_empty │  ┌──────────────────────────────────────┐
│   _cells()  │  │       «abstract» BotStrategy         │
│ + display() │  │──────────────────────────────────────│
│ + reset()   │  │ + choose_move(board, mark) -> (r, c) │
└─────────────┘  │ + difficulty() -> str                 │
                 └──────────┬───────────────────────────┘
                            │ implemented by
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
        EasyStrategy  MediumStrategy  MinimaxStrategy
        (random)      (heuristic)     (alpha-beta)


┌─────────────┐  ┌─────────────────┐
│    Move     │  │   MoveHistory   │
│─────────────│  │─────────────────│
│ + row: int  │  │ - _moves: list  │
│ + col: int  │  │─────────────────│
│ + mark: Mark│  │ + push(move)    │
│─────────────│  │ + pop() -> Move │
│ + execute() │  │ + clear()       │
│ + undo()    │  │ + get_all()     │
└─────────────┘  └─────────────────┘


┌──────────────────────────────────────┐
│            GameFactory               │
│──────────────────────────────────────│
│ + human_vs_human() -> TicTacToeGame  │
│ + human_vs_bot() -> TicTacToeGame    │
│ + bot_vs_bot() -> TicTacToeGame      │
└──────────────────────────────────────┘

┌─────────────┐    ┌─────────────────┐
│  Mark(Enum) │    │ GameStatus(Enum)│
│─────────────│    │─────────────────│
│ X           │    │ ACTIVE          │
│ O           │    │ X_WINS          │
│ EMPTY       │    │ O_WINS          │
│─────────────│    │ DRAW            │
│ + opposite()│    └─────────────────┘
└─────────────┘
```

---

## How to Run

Save all code into a single file `tic_tac_toe.py` and run:

```bash
python tic_tac_toe.py
```

### Example Session – Human vs Bot

```
=== TIC-TAC-TOE: Human vs Bot ===

Your name: Alice
Difficulty (easy/medium/minimax): minimax

   0   1   2
  +---+---+---+
0 |   |   |   |
  +---+---+---+
1 |   |   |   |
  +---+---+---+
2 |   |   |   |
  +---+---+---+

  Alice (X), enter row col: 1 1

   0   1   2
  +---+---+---+
0 |   |   |   |
  +---+---+---+
1 |   | X |   |
  +---+---+---+
2 |   |   |   |
  +---+---+---+

  🤖 Skynet (O) plays (0, 0)

   0   1   2
  +---+---+---+
0 | O |   |   |
  +---+---+---+
1 |   | X |   |
  +---+---+---+
2 |   |   |   |
  +---+---+---+

  Alice (X), enter row col: 2 2

   0   1   2
  +---+---+---+
0 | O |   |   |
  +---+---+---+
1 |   | X |   |
  +---+---+---+
2 |   |   | X |
  +---+---+---+

  🤖 Skynet (O) plays (0, 2)
  ...

🤝 It's a draw!

Move log: X(1,1) → O(0,0) → X(2,2) → O(0,2) → X(0,1) → O(2,1) → X(2,0) → O(1,0) → X(1,2)
```

Against the minimax bot, perfect play by both sides always results in a draw. Try the easy bot for a winnable game.
