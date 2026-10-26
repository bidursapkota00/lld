# Design Snake & Ladder – Low-Level Design Guide

![Bidur Sapkota](https://www.bidursapkota.com.np/images/gravatar.webp "Bidur Sapkota - Developer")&nbsp;[Bidur Sapkota](https://www.bidursapkota.com.np/)

## Table of Contents

1. [Requirements](#requirements)
2. [Key Design Patterns](#key-design-patterns)
3. [Core Enums & Constants](#core-enums--constants)
4. [Dice (Strategy Pattern)](#dice-strategy-pattern)
5. [Board Entity – Snakes & Ladders](#board-entity--snakes--ladders)
6. [Board (Builder Pattern)](#board-builder-pattern)
7. [Player](#player)
8. [Game Event System (Observer Pattern)](#game-event-system-observer-pattern)
9. [Game Engine](#game-engine)
10. [Putting It All Together](#putting-it-all-together)
11. [Class Diagram](#class-diagram)
12. [How to Run](#how-to-run)

---

## Requirements

### Functional Requirements

| #  | Requirement                                                                     |
| -- | ------------------------------------------------------------------------------- |
| 1  | Configurable board size (default 100 squares, numbered 1 to N)                  |
| 2  | Support multiple snakes – landing on a snake's head moves the player to its tail|
| 3  | Support multiple ladders – landing on a ladder's bottom moves the player to top |
| 4  | 2 or more players take turns in order                                           |
| 5  | Dice roll determines how many squares a player advances                         |
| 6  | Configurable dice – single die, double dice, or custom strategies               |
| 7  | A player wins by reaching exactly the last square                               |
| 8  | If a roll would overshoot the last square, the player does not move             |
| 9  | Consecutive sixes grant an extra turn (optional, configurable)                  |
| 10 | Game tracks and prints a full move log                                          |

### Non-Functional Requirements

- Clean OOP with SOLID principles.
- Extensible – adding new board entities (e.g. power-ups, traps) should not require modifying the game engine.
- Configurable – board layout, dice type, and rules are injected, not hardcoded.

---

## Key Design Patterns

| Pattern  | Where It Is Used                          | Why                                                                                  |
| -------- | ----------------------------------------- | ------------------------------------------------------------------------------------ |
| Strategy | `Dice` abstraction with multiple impls    | Swap between single die, double dice, or rigged dice without changing the game engine |
| Builder  | `BoardBuilder` constructs the board       | Complex board setup with validation is separated from the Board class itself          |
| Observer | `GameEventListener` notified on events    | Logging, UI updates, and analytics can be added without modifying game logic          |
| Template Method | `BoardEntity.apply()` in base class | Common pre/post logic around entity effects, subclasses define the transformation     |

---

## Core Enums & Constants

```python
from enum import Enum


class GameStatus(Enum):
    NOT_STARTED = "not_started"
    IN_PROGRESS = "in_progress"
    FINISHED = "finished"


class EventType(Enum):
    GAME_STARTED = "game_started"
    TURN_STARTED = "turn_started"
    DICE_ROLLED = "dice_rolled"
    PLAYER_MOVED = "player_moved"
    HIT_SNAKE = "hit_snake"
    HIT_LADDER = "hit_ladder"
    OVERSHOOT = "overshoot"
    PLAYER_WON = "player_won"
    EXTRA_TURN = "extra_turn"


DEFAULT_BOARD_SIZE = 100
```

`GameStatus` drives the state machine for the game engine. `EventType` enumerates every meaningful event so observers can react selectively. `DEFAULT_BOARD_SIZE` avoids magic numbers — a standard board has 100 squares.

---

## Dice (Strategy Pattern)

Different games use different dice rules. The Strategy pattern lets the game engine depend on an abstraction, and concrete dice implementations can be swapped at configuration time.

### Abstract Dice

```python
from abc import ABC, abstractmethod
import random


class Dice(ABC):
    """Abstract dice strategy."""

    @abstractmethod
    def roll(self) -> int:
        """Return the total value of a single roll."""
        ...

    @abstractmethod
    def max_value(self) -> int:
        """Maximum possible value from a single roll."""
        ...
```

`roll` returns the total face value. `max_value` is needed by the game engine to determine if a roll could cause an overshoot. Keeping them abstract forces every implementation to provide both.

### Single Die

```python
class SingleDie(Dice):
    """Standard single six-sided die."""

    def __init__(self, faces: int = 6) -> None:
        self._faces = faces

    def roll(self) -> int:
        return random.randint(1, self._faces)

    def max_value(self) -> int:
        return self._faces
```

`faces` defaults to 6 but can be configured for non-standard dice. `random.randint(1, self._faces)` is inclusive on both ends, giving values 1 through 6.

### Double Dice

```python
class DoubleDice(Dice):
    """Two six-sided dice rolled together."""

    def __init__(self, faces: int = 6) -> None:
        self._faces = faces

    def roll(self) -> int:
        return random.randint(1, self._faces) + random.randint(1, self._faces)

    def max_value(self) -> int:
        return self._faces * 2
```

The minimum roll with double dice is 2, maximum is 12 with standard six-sided dice. The game engine does not need to know how many physical dice exist — it only calls `roll()` and gets a number.

### Crooked Die (for testing)

```python
class CrookedDie(Dice):
    """Always rolls even numbers. Useful for testing specific scenarios."""

    def __init__(self, faces: int = 6) -> None:
        self._faces = faces

    def roll(self) -> int:
        even_values = [i for i in range(2, self._faces + 1, 2)]
        return random.choice(even_values)

    def max_value(self) -> int:
        return self._faces


class FixedDice(Dice):
    """Rolls a predetermined sequence. Essential for unit testing."""

    def __init__(self, sequence: list[int]) -> None:
        self._sequence = sequence
        self._index = 0

    def roll(self) -> int:
        if self._index >= len(self._sequence):
            raise RuntimeError("FixedDice exhausted — no more values in sequence")
        value = self._sequence[self._index]
        self._index += 1
        return value

    def max_value(self) -> int:
        return max(self._sequence) if self._sequence else 6
```

`CrookedDie` demonstrates how easy it is to add a new dice variant — just implement two methods. `FixedDice` is invaluable for testing. By supplying a known sequence of rolls, you can write deterministic tests for specific scenarios like hitting a snake, climbing a ladder, or winning the game.

---

## Board Entity – Snakes & Ladders

Snakes and ladders are both "board entities" — objects that sit on a square and transport a player to a different square when landed on. The base class uses the Template Method pattern.

```python
class BoardEntity(ABC):
    """Something on the board that alters a player's position."""

    def __init__(self, start: int, end: int) -> None:
        self._start = start
        self._end = end

    @property
    def start(self) -> int:
        """The square the player lands on to trigger this entity."""
        return self._start

    @property
    def end(self) -> int:
        """The square the player is moved to."""
        return self._end

    @abstractmethod
    def entity_type(self) -> str:
        """Human-readable name for event reporting."""
        ...

    def apply(self, current_position: int) -> int:
        """Template Method: apply the entity's effect."""
        if current_position != self._start:
            raise ValueError(
                f"Entity at {self._start} cannot apply to position {current_position}"
            )
        return self._end


class Snake(BoardEntity):
    """Moves a player downward (start > end)."""

    def __init__(self, head: int, tail: int) -> None:
        if tail >= head:
            raise ValueError(f"Snake head ({head}) must be above tail ({tail})")
        super().__init__(start=head, end=tail)

    def entity_type(self) -> str:
        return "snake"


class Ladder(BoardEntity):
    """Moves a player upward (start < end)."""

    def __init__(self, bottom: int, top: int) -> None:
        if top <= bottom:
            raise ValueError(f"Ladder top ({top}) must be above bottom ({bottom})")
        super().__init__(start=bottom, end=top)

    def entity_type(self) -> str:
        return "ladder"
```

`BoardEntity` is the abstraction. `Snake` and `Ladder` are concrete implementations that validate their invariants in the constructor — a snake must go downward, a ladder must go upward. `apply` is the Template Method; right now it simply returns `self._end`, but the base class could be extended to add logging or cooldown logic without touching the subclasses. Using `@property` for `start` and `end` makes them read-only after construction.

This design makes it trivial to add new entity types in the future. For example, a `Portal` that teleports to a random square, or a `Trap` that freezes a player for one turn — just extend `BoardEntity`.

---

## Board (Builder Pattern)

The board has complex construction logic: validation of snake/ladder positions, overlap detection, and bounds checking. The Builder pattern separates this from the Board class itself.

### Board Class

```python
class Board:
    """The game board: a linear track of numbered squares with entities."""

    def __init__(self, size: int, entities: dict[int, BoardEntity]) -> None:
        self._size = size
        self._entities = entities  # position -> entity

    @property
    def size(self) -> int:
        return self._size

    def get_entity(self, position: int) -> BoardEntity | None:
        """Return the entity at a position, or None if empty."""
        return self._entities.get(position)

    def get_final_position(self, position: int) -> tuple[int, BoardEntity | None]:
        """
        Given a landing position, return the final position after
        applying any entity, and the entity that was triggered.
        """
        entity = self.get_entity(position)
        if entity is not None:
            return entity.apply(position), entity
        return position, None

    def is_valid_position(self, position: int) -> bool:
        return 1 <= position <= self._size

    def display(self) -> None:
        """Print the board layout showing all snakes and ladders."""
        print(f"\n{'='*50}")
        print(f"  BOARD ({self._size} squares)")
        print(f"{'='*50}")
        snakes = []
        ladders = []
        for pos, entity in sorted(self._entities.items()):
            if isinstance(entity, Snake):
                snakes.append(f"  {entity.start} → {entity.end}")
            elif isinstance(entity, Ladder):
                ladders.append(f"  {entity.start} → {entity.end}")
        print(f"\n  🐍 Snakes ({len(snakes)}):")
        for s in snakes:
            print(s)
        print(f"\n  🪜 Ladders ({len(ladders)}):")
        for l in ladders:
            print(l)
        print(f"{'='*50}\n")
```

`_entities` is a dictionary mapping square numbers to `BoardEntity` objects. This gives O(1) lookup when a player lands on a square. `get_final_position` is the key method — it checks if there is an entity on the square and applies it, returning both the final position and the entity (for event reporting). The board itself has no game logic; it is a pure data structure with query methods.

### Board Builder

```python
class BoardBuilder:
    """Fluent builder for constructing a validated Board."""

    def __init__(self, size: int = DEFAULT_BOARD_SIZE) -> None:
        self._size = size
        self._entities: dict[int, BoardEntity] = {}
        self._occupied: set[int] = set()

    def _validate_position(self, position: int, label: str) -> None:
        """Ensure a position is within bounds and not on the first or last square."""
        if position < 2 or position >= self._size:
            raise ValueError(
                f"{label} position {position} is out of bounds [2, {self._size - 1}]"
            )

    def _check_overlap(self, position: int) -> None:
        """Ensure no two entities share the same starting square."""
        if position in self._occupied:
            raise ValueError(
                f"Position {position} is already occupied by another entity"
            )

    def add_snake(self, head: int, tail: int) -> "BoardBuilder":
        """Add a snake from head to tail."""
        self._validate_position(head, "Snake head")
        self._validate_position(tail, "Snake tail")
        self._check_overlap(head)
        snake = Snake(head, tail)
        self._entities[head] = snake
        self._occupied.add(head)
        return self

    def add_ladder(self, bottom: int, top: int) -> "BoardBuilder":
        """Add a ladder from bottom to top."""
        self._validate_position(bottom, "Ladder bottom")
        self._validate_position(top, "Ladder top")
        self._check_overlap(bottom)
        ladder = Ladder(bottom, top)
        self._entities[bottom] = ladder
        self._occupied.add(bottom)
        return self

    def build(self) -> Board:
        """Validate and return the constructed Board."""
        # Check for cycles: entity end should not be another entity start
        # that leads back (simple one-level check)
        for pos, entity in self._entities.items():
            next_entity = self._entities.get(entity.end)
            if next_entity is not None:
                print(f"  ⚠ Warning: {entity.entity_type()} at {pos} "
                      f"leads to {entity.end}, which has a "
                      f"{next_entity.entity_type()}")
        return Board(self._size, dict(self._entities))

    @staticmethod
    def standard_board() -> Board:
        """Create a classic 100-square board with a preset layout."""
        return (
            BoardBuilder(100)
            # Snakes
            .add_snake(16, 6)
            .add_snake(47, 26)
            .add_snake(49, 11)
            .add_snake(56, 53)
            .add_snake(62, 19)
            .add_snake(64, 60)
            .add_snake(87, 24)
            .add_snake(93, 73)
            .add_snake(95, 75)
            .add_snake(98, 78)
            # Ladders
            .add_ladder(2, 38)
            .add_ladder(7, 14)
            .add_ladder(8, 31)
            .add_ladder(15, 26)
            .add_ladder(21, 42)
            .add_ladder(28, 84)
            .add_ladder(36, 44)
            .add_ladder(51, 67)
            .add_ladder(71, 91)
            .add_ladder(78, 98)
            .build()
        )
```

The builder uses method chaining (`return self`) for a fluent API. `_validate_position` ensures entities are not placed on square 1 (start) or square N (finish). `_check_overlap` prevents two entities from sharing the same trigger square. `build` emits warnings for chain situations (e.g. a ladder that leads to a snake head) but allows them — they are valid in some game variants. `standard_board` is a static factory method that returns a ready-to-play board with a classic layout.

---

## Player

```python
class Player:
    """Represents a player in the game."""

    def __init__(self, name: str) -> None:
        self.name = name
        self.position = 0  # 0 means not yet on the board

    def __repr__(self) -> str:
        return f"Player({self.name}, pos={self.position})"
```

Position 0 means the player has not entered the board yet. On the first roll, they move from 0 to the rolled value. This is cleaner than starting at position 1 because the first move is calculated the same way as every subsequent move: `new_position = current_position + dice_roll`.

---

## Game Event System (Observer Pattern)

The Observer pattern decouples game logic from side effects like logging, UI updates, and statistics. The game engine fires events; listeners react to them independently.

### Event & Listener

```python
from dataclasses import dataclass, field
from typing import Any


@dataclass
class GameEvent:
    """An event emitted by the game engine."""
    event_type: EventType
    data: dict[str, Any] = field(default_factory=dict)


class GameEventListener(ABC):
    """Abstract observer that reacts to game events."""

    @abstractmethod
    def on_event(self, event: GameEvent) -> None:
        ...
```

`GameEvent` carries the event type and an arbitrary data dictionary. Using a dict instead of typed fields keeps the event system generic — adding a new event type does not require a new class. `GameEventListener` is the observer interface with a single `on_event` method.

### Console Logger

```python
class ConsoleLogger(GameEventListener):
    """Prints game events to the console in a human-readable format."""

    def on_event(self, event: GameEvent) -> None:
        d = event.data
        match event.event_type:
            case EventType.GAME_STARTED:
                players = ", ".join(d["players"])
                print(f"\n🎲 Game started with players: {players}")
            case EventType.TURN_STARTED:
                print(f"\n--- {d['player']}'s turn (at square {d['position']}) ---")
            case EventType.DICE_ROLLED:
                print(f"  🎲 Rolled a {d['value']}")
            case EventType.PLAYER_MOVED:
                print(f"  📍 Moved from {d['from']} to {d['to']}")
            case EventType.HIT_SNAKE:
                print(f"  🐍 Snake! Slid down from {d['from']} to {d['to']}")
            case EventType.HIT_LADDER:
                print(f"  🪜 Ladder! Climbed from {d['from']} to {d['to']}")
            case EventType.OVERSHOOT:
                print(f"  ⛔ Overshoot! Needed ≤{d['needed']}, rolled {d['rolled']}. "
                      f"Stays at {d['position']}")
            case EventType.EXTRA_TURN:
                print(f"  ⭐ Rolled max ({d['value']})! Gets an extra turn.")
            case EventType.PLAYER_WON:
                print(f"\n🏆 {d['player']} wins in {d['turns']} turns! 🏆")
```

The `match` statement (Python 3.10+) provides clean event dispatching. Each case formats and prints the relevant data. Because this is a listener, removing it or swapping it for a GUI listener has zero impact on the game engine.

### Statistics Collector

```python
class StatsCollector(GameEventListener):
    """Collects game statistics for analysis."""

    def __init__(self) -> None:
        self.total_rolls = 0
        self.snakes_hit: dict[str, int] = {}
        self.ladders_hit: dict[str, int] = {}
        self.rolls_per_player: dict[str, int] = {}

    def on_event(self, event: GameEvent) -> None:
        d = event.data
        match event.event_type:
            case EventType.DICE_ROLLED:
                self.total_rolls += 1
                player = d.get("player", "unknown")
                self.rolls_per_player[player] = (
                    self.rolls_per_player.get(player, 0) + 1
                )
            case EventType.HIT_SNAKE:
                player = d.get("player", "unknown")
                self.snakes_hit[player] = self.snakes_hit.get(player, 0) + 1
            case EventType.HIT_LADDER:
                player = d.get("player", "unknown")
                self.ladders_hit[player] = self.ladders_hit.get(player, 0) + 1

    def display(self) -> None:
        print(f"\n{'='*40}")
        print("  GAME STATISTICS")
        print(f"{'='*40}")
        print(f"  Total rolls: {self.total_rolls}")
        for player, rolls in self.rolls_per_player.items():
            snakes = self.snakes_hit.get(player, 0)
            ladders = self.ladders_hit.get(player, 0)
            print(f"  {player}: {rolls} rolls, "
                  f"{snakes} snakes, {ladders} ladders")
        print(f"{'='*40}\n")
```

`StatsCollector` is a second observer that collects metrics without interfering with the game flow. After the game ends, call `display()` to print a summary. This demonstrates the power of the Observer pattern — you can attach any number of independent behaviors to the same event stream.

---

## Game Engine

The game engine is the core orchestrator. It owns the board, manages turn order, enforces rules, and emits events. All game logic lives here.

```python
class GameEngine:
    """Orchestrates a Snake & Ladder game."""

    def __init__(
        self,
        board: Board,
        players: list[Player],
        dice: Dice,
        allow_extra_turn_on_max: bool = True,
    ) -> None:
        if len(players) < 2:
            raise ValueError("Need at least 2 players")

        self._board = board
        self._players = players
        self._dice = dice
        self._allow_extra_turn = allow_extra_turn_on_max
        self._current_index = 0
        self._status = GameStatus.NOT_STARTED
        self._winner: Player | None = None
        self._turn_count = 0
        self._listeners: list[GameEventListener] = []

    # ------------------------------------------------------------------
    # Observer management
    # ------------------------------------------------------------------

    def add_listener(self, listener: GameEventListener) -> None:
        self._listeners.append(listener)

    def remove_listener(self, listener: GameEventListener) -> None:
        self._listeners.remove(listener)

    def _emit(self, event_type: EventType, **data: Any) -> None:
        event = GameEvent(event_type, data)
        for listener in self._listeners:
            listener.on_event(event)

    # ------------------------------------------------------------------
    # Properties
    # ------------------------------------------------------------------

    @property
    def status(self) -> GameStatus:
        return self._status

    @property
    def winner(self) -> Player | None:
        return self._winner

    @property
    def current_player(self) -> Player:
        return self._players[self._current_index]

    # ------------------------------------------------------------------
    # Game flow
    # ------------------------------------------------------------------

    def start(self) -> None:
        """Initialize and start the game."""
        if self._status != GameStatus.NOT_STARTED:
            raise RuntimeError("Game has already been started")
        self._status = GameStatus.IN_PROGRESS
        self._emit(EventType.GAME_STARTED,
                    players=[p.name for p in self._players])
        self._board.display()

    def play_turn(self) -> bool:
        """
        Execute one complete turn for the current player.
        Returns True if the game is still in progress, False if someone won.
        """
        if self._status != GameStatus.IN_PROGRESS:
            return False

        player = self.current_player
        self._turn_count += 1
        self._emit(EventType.TURN_STARTED,
                    player=player.name, position=player.position)

        gets_extra_turn = self._execute_single_roll(player)

        # Check for win
        if player.position == self._board.size:
            self._status = GameStatus.FINISHED
            self._winner = player
            self._emit(EventType.PLAYER_WON,
                        player=player.name, turns=self._turn_count)
            return False

        # Handle extra turn or advance to next player
        if gets_extra_turn:
            self._emit(EventType.EXTRA_TURN,
                        player=player.name, value=self._dice.max_value())
            # Do NOT advance _current_index — same player goes again
        else:
            self._current_index = (
                (self._current_index + 1) % len(self._players)
            )

        return True

    def play_full_game(self) -> Player:
        """Run the entire game until a winner is determined."""
        self.start()
        while self.play_turn():
            pass
        return self._winner  # type: ignore[return-value]

    # ------------------------------------------------------------------
    # Movement logic
    # ------------------------------------------------------------------

    def _execute_single_roll(self, player: Player) -> bool:
        """
        Roll the dice, move the player, handle entities.
        Returns True if the player earns an extra turn.
        """
        roll = self._dice.roll()
        self._emit(EventType.DICE_ROLLED, player=player.name, value=roll)

        new_position = player.position + roll

        # Overshoot check — must land exactly on the last square
        if new_position > self._board.size:
            needed = self._board.size - player.position
            self._emit(EventType.OVERSHOOT,
                        player=player.name, needed=needed,
                        rolled=roll, position=player.position)
            return False

        old_position = player.position
        player.position = new_position
        self._emit(EventType.PLAYER_MOVED,
                    player=player.name,
                    **{"from": old_position, "to": new_position})

        # Check for snake or ladder
        final_position, entity = self._board.get_final_position(new_position)
        if entity is not None:
            player.position = final_position
            if isinstance(entity, Snake):
                self._emit(EventType.HIT_SNAKE,
                            player=player.name,
                            **{"from": new_position, "to": final_position})
            elif isinstance(entity, Ladder):
                self._emit(EventType.HIT_LADDER,
                            player=player.name,
                            **{"from": new_position, "to": final_position})

        # Extra turn on max roll
        return (self._allow_extra_turn
                and roll == self._dice.max_value()
                and player.position != self._board.size)
```

`__init__` takes all dependencies through constructor injection — board, players, dice, and the extra-turn rule. This makes the engine fully configurable and testable. `_emit` is the event dispatcher; it creates a `GameEvent` and notifies all listeners. `play_turn` is the public method for one turn: roll, move, check entities, check win, handle extra turns, advance to next player. `play_full_game` is a convenience that loops until someone wins. `_execute_single_roll` contains the core movement logic: roll the dice, check for overshoot, move the player, check for snakes/ladders, and determine if an extra turn is earned. The `**{"from": ...}` syntax is used because `from` is a reserved word in Python.

---

## Putting It All Together

```python
def main() -> None:
    # Build the board
    board = BoardBuilder.standard_board()

    # Create players
    players = [
        Player("Alice"),
        Player("Bob"),
    ]

    # Choose a dice strategy
    dice = SingleDie(faces=6)

    # Create the game engine
    game = GameEngine(
        board=board,
        players=players,
        dice=dice,
        allow_extra_turn_on_max=True,
    )

    # Attach observers
    logger = ConsoleLogger()
    stats = StatsCollector()
    game.add_listener(logger)
    game.add_listener(stats)

    # Run the game
    winner = game.play_full_game()

    # Print statistics
    stats.display()
    print(f"🎉 {winner.name} is the champion!")


if __name__ == "__main__":
    main()
```

The `main` function wires everything together. The board is built using the static factory `standard_board()`. A `SingleDie` with 6 faces is the dice strategy. Two observers — a `ConsoleLogger` for real-time output and a `StatsCollector` for post-game analysis — are attached to the engine. `play_full_game` runs the complete match and returns the winner.

### Interactive Mode

For a turn-by-turn interactive game where each player presses Enter to roll:

```python
def main_interactive() -> None:
    board = BoardBuilder.standard_board()

    # Get player names
    num_players = int(input("Number of players: "))
    players = []
    for i in range(num_players):
        name = input(f"Player {i + 1} name: ").strip()
        players.append(Player(name))

    dice = SingleDie(faces=6)
    game = GameEngine(board=board, players=players, dice=dice)

    logger = ConsoleLogger()
    stats = StatsCollector()
    game.add_listener(logger)
    game.add_listener(stats)

    game.start()

    while game.status == GameStatus.IN_PROGRESS:
        player = game.current_player
        input(f"\n  Press Enter for {player.name} to roll...")
        game.play_turn()

    stats.display()


if __name__ == "__main__":
    main_interactive()
```

`main_interactive` prompts for player names and waits for a keypress before each roll. The game engine does not care whether it is driven by `play_full_game` or `play_turn` — the API supports both automatic and manual modes.

### Custom Board

```python
def main_custom() -> None:
    """Example: small board with double dice."""
    board = (
        BoardBuilder(size=30)
        .add_snake(27, 5)
        .add_snake(21, 9)
        .add_snake(17, 7)
        .add_ladder(3, 22)
        .add_ladder(5, 8)
        .add_ladder(11, 26)
        .add_ladder(20, 29)
        .build()
    )

    players = [Player("Alice"), Player("Bob"), Player("Charlie")]
    dice = DoubleDice(faces=6)

    game = GameEngine(board=board, players=players, dice=dice)
    game.add_listener(ConsoleLogger())

    winner = game.play_full_game()
    print(f"\n{winner.name} wins!")
```

This shows the flexibility of the Builder pattern — a 30-square board with double dice and three players is trivial to set up. The game engine works identically regardless of board size, dice type, or player count.

### Deterministic Testing

```python
def test_snake_hit() -> None:
    """Test that landing on a snake slides the player down."""
    board = (
        BoardBuilder(size=20)
        .add_snake(10, 3)
        .build()
    )
    players = [Player("Test1"), Player("Test2")]
    dice = FixedDice([5, 1, 5, 1])  # Test1: 5, Test2: 1, Test1: 5 -> 10 -> snake -> 3

    game = GameEngine(board=board, players=players, dice=dice,
                      allow_extra_turn_on_max=False)
    logger = ConsoleLogger()
    game.add_listener(logger)

    game.start()
    game.play_turn()  # Test1 rolls 5, moves to 5
    game.play_turn()  # Test2 rolls 1, moves to 1
    game.play_turn()  # Test1 rolls 5, moves to 10 -> snake -> 3

    assert players[0].position == 3, f"Expected 3, got {players[0].position}"
    print("\n✅ test_snake_hit passed!")


def test_overshoot() -> None:
    """Test that overshooting the last square keeps the player in place."""
    board = BoardBuilder(size=10).build()
    players = [Player("Test1"), Player("Test2")]
    dice = FixedDice([5, 1, 5, 1, 6, 1])
    # Test1: 5, Test2: 1, Test1: 5 -> at 10 (should NOT happen, 10 is exact)
    # Wait — 5+5=10 which IS the board size, so Test1 wins

    # Let's use a different sequence:
    # Board size 10. Test1: 4, Test2: 1, Test1: 4 -> at 8, Test2: 1 -> at 2
    # Test1: 6 -> 8+6=14 > 10, overshoot!
    dice2 = FixedDice([4, 1, 4, 1, 6])
    game = GameEngine(board=board, players=players, dice=dice2,
                      allow_extra_turn_on_max=False)
    game.add_listener(ConsoleLogger())

    game.start()
    game.play_turn()  # Test1 -> 4
    game.play_turn()  # Test2 -> 1
    game.play_turn()  # Test1 -> 8
    game.play_turn()  # Test2 -> 2
    game.play_turn()  # Test1 rolls 6, 8+6=14 > 10, stays at 8

    assert players[0].position == 8, f"Expected 8, got {players[0].position}"
    print("\n✅ test_overshoot passed!")
```

`FixedDice` makes testing deterministic. Each test sets up a minimal board, injects a known dice sequence, plays the required number of turns, and asserts the expected position. These tests verify snake mechanics and the overshoot rule without randomness.

---

## Class Diagram

```
┌──────────────────────────────────────────────────────┐
│                     GameEngine                        │
│──────────────────────────────────────────────────────│
│ - _board: Board                                      │
│ - _players: list[Player]                             │
│ - _dice: Dice                                        │
│ - _allow_extra_turn: bool                            │
│ - _current_index: int                                │
│ - _status: GameStatus                                │
│ - _winner: Player | None                             │
│ - _listeners: list[GameEventListener]                │
│──────────────────────────────────────────────────────│
│ + add_listener(listener) -> None                     │
│ + remove_listener(listener) -> None                  │
│ + start() -> None                                    │
│ + play_turn() -> bool                                │
│ + play_full_game() -> Player                         │
│ - _emit(event_type, **data) -> None                  │
│ - _execute_single_roll(player) -> bool               │
└───────┬──────────┬────────────┬───────────────────────┘
        │          │            │
   owns ▼     uses ▼       uses ▼
┌─────────────┐ ┌──────────┐ ┌────────────────────────┐
│    Board    │ │  Player  │ │   «abstract» Dice      │
│─────────────│ │──────────│ │────────────────────────│
│ - _size     │ │ + name   │ │ + roll() -> int        │
│ - _entities │ │ + position│ │ + max_value() -> int   │
│─────────────│ └──────────┘ └───────────┬────────────┘
│ + get_entity│                          │ implemented by
│ + get_final │              ┌───────────┼────────────┐
│   _position │              ▼           ▼            ▼
│ + display() │        SingleDie    DoubleDice    FixedDice
└──────┬──────┘
       │ contains
       ▼
┌──────────────────────────┐
│ «abstract» BoardEntity   │
│──────────────────────────│
│ - _start: int            │
│ - _end: int              │
│──────────────────────────│
│ + apply(pos) -> int      │
│ + entity_type() -> str   │
└──────────┬───────────────┘
           │ extended by
     ┌─────┴─────┐
     ▼           ▼
  Snake       Ladder


┌────────────────────────────┐       ┌───────────────────────────┐
│ «abstract» GameEventListener│       │        GameEvent           │
│────────────────────────────│       │───────────────────────────│
│ + on_event(event) -> None  │       │ + event_type: EventType   │
└──────────┬─────────────────┘       │ + data: dict[str, Any]    │
           │ implemented by          └───────────────────────────┘
     ┌─────┴──────────┐
     ▼                ▼
ConsoleLogger    StatsCollector


┌──────────────────────────────┐
│         BoardBuilder         │
│──────────────────────────────│
│ - _size: int                 │
│ - _entities: dict            │
│ - _occupied: set             │
│──────────────────────────────│
│ + add_snake(head, tail)      │
│ + add_ladder(bottom, top)    │
│ + build() -> Board           │
│ + standard_board() -> Board  │
└──────────────────────────────┘

┌──────────────────┐     ┌───────────────────┐
│ GameStatus(Enum) │     │  EventType(Enum)  │
│──────────────────│     │───────────────────│
│ NOT_STARTED      │     │ GAME_STARTED      │
│ IN_PROGRESS      │     │ TURN_STARTED      │
│ FINISHED         │     │ DICE_ROLLED       │
└──────────────────┘     │ PLAYER_MOVED      │
                         │ HIT_SNAKE         │
                         │ HIT_LADDER        │
                         │ OVERSHOOT         │
                         │ PLAYER_WON        │
                         │ EXTRA_TURN        │
                         └───────────────────┘
```

---

## How to Run

Save all code into a single file `snake_ladder.py` and run:

```bash
python snake_ladder.py
```

### Example Output

```
==================================================
  BOARD (100 squares)
==================================================

  🐍 Snakes (10):
  16 → 6
  47 → 26
  49 → 11
  56 → 53
  62 → 19
  64 → 60
  87 → 24
  93 → 73
  95 → 75
  98 → 78

  🪜 Ladders (10):
  2 → 38
  7 → 14
  8 → 31
  15 → 26
  21 → 42
  28 → 84
  36 → 44
  51 → 67
  71 → 91
  78 → 98
==================================================

🎲 Game started with players: Alice, Bob

--- Alice's turn (at square 0) ---
  🎲 Rolled a 4
  📍 Moved from 0 to 4

--- Bob's turn (at square 0) ---
  🎲 Rolled a 2
  📍 Moved from 0 to 2
  🪜 Ladder! Climbed from 2 to 38

--- Alice's turn (at square 4) ---
  🎲 Rolled a 3
  📍 Moved from 4 to 7
  🪜 Ladder! Climbed from 7 to 14

...

🏆 Alice wins in 42 turns! 🏆

========================================
  GAME STATISTICS
========================================
  Total rolls: 42
  Alice: 22 rolls, 1 snakes, 3 ladders
  Bob: 20 rolls, 2 snakes, 2 ladders
========================================

🎉 Alice is the champion!
```
