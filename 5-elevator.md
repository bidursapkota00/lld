# Design Elevator System – Low-Level Design Guide

![Bidur Sapkota](https://www.bidursapkota.com.np/images/gravatar.webp "Bidur Sapkota - Developer")&nbsp;[Bidur Sapkota](https://www.bidursapkota.com.np/)

## Table of Contents

1. [Requirements](#requirements)
2. [Key Design Patterns](#key-design-patterns)
3. [Core Enums & Constants](#core-enums--constants)
4. [Request (Command Pattern)](#request-command-pattern)
5. [Elevator Car (State Pattern)](#elevator-car-state-pattern)
6. [Dispatch Strategy (Strategy Pattern)](#dispatch-strategy-strategy-pattern)
7. [Floor Panel & Display (Observer Pattern)](#floor-panel--display-observer-pattern)
8. [Elevator System (Facade)](#elevator-system-facade)
9. [Simulation Engine](#simulation-engine)
10. [Putting It All Together](#putting-it-all-together)
11. [Class Diagram](#class-diagram)
12. [How to Run](#how-to-run)

---

## Requirements

### Functional Requirements

| #  | Requirement                                                                           |
| -- | ------------------------------------------------------------------------------------- |
| 1  | Support a building with N floors and M elevator cars                                  |
| 2  | External requests: a person on a floor presses Up or Down                             |
| 3  | Internal requests: a person inside an elevator presses a destination floor button     |
| 4  | Each elevator tracks its current floor, direction, and state                          |
| 5  | Elevator moves one floor per time step, stops to pick up / drop off passengers        |
| 6  | Configurable dispatch strategy to decide which elevator serves an external request    |
| 7  | SCAN (elevator) algorithm: serve requests in the current direction before reversing   |
| 8  | Door opens on arrival at a requested floor, then closes before moving                 |
| 9  | Elevator becomes idle when it has no pending requests                                 |
| 10 | Display on each floor shows each elevator's current floor and direction               |
| 11 | Capacity limit — elevator rejects internal requests when at max occupancy             |
| 12 | Emergency stop support                                                                |

### Non-Functional Requirements

- Clean OOP with SOLID principles.
- Extensible — new dispatch strategies and scheduling algorithms can be added without modifying elevator logic.
- Simulatable — the system runs in discrete time steps for testing and demonstration.

---

## Key Design Patterns

| Pattern  | Where It Is Used                              | Why                                                                                  |
| -------- | --------------------------------------------- | ------------------------------------------------------------------------------------ |
| Strategy | `DispatchStrategy` selects which car to send  | Swap between nearest-car, zone-based, or round-robin dispatch without changing core  |
| State    | `ElevatorState` drives car behaviour          | Moving-up, moving-down, idle, and door-open states have distinct transition rules    |
| Command  | `Request` objects represent ride requests     | Requests are queued, prioritised, and can be cancelled independently                 |
| Observer | `FloorDisplay` observes elevator position     | Displays update automatically whenever an elevator moves                             |
| Facade   | `ElevatorSystem` exposes a simple API         | Hides the complexity of dispatch, car management, and request routing                |

---

## Core Enums & Constants

```python
from enum import Enum


class Direction(Enum):
    UP = "UP"
    DOWN = "DOWN"
    IDLE = "IDLE"

    def opposite(self) -> "Direction":
        if self == Direction.UP:
            return Direction.DOWN
        if self == Direction.DOWN:
            return Direction.UP
        return Direction.IDLE


class ElevatorState(Enum):
    IDLE = "idle"
    MOVING_UP = "moving_up"
    MOVING_DOWN = "moving_down"
    DOOR_OPEN = "door_open"
    EMERGENCY_STOP = "emergency_stop"


class RequestType(Enum):
    EXTERNAL = "external"   # from a floor button
    INTERNAL = "internal"   # from inside the elevator


DEFAULT_NUM_FLOORS = 10
DEFAULT_NUM_ELEVATORS = 3
DEFAULT_CAPACITY = 10
DOOR_OPEN_TICKS = 2  # time steps the door stays open
```

`Direction` represents the elevator's travel direction. `IDLE` means the car has no pending requests. `opposite()` is used when the elevator reverses at the end of a sweep. `ElevatorState` is the state machine — `DOOR_OPEN` is a temporary state that transitions back to moving or idle. `DOOR_OPEN_TICKS` controls how long the door stays open before closing, measured in simulation time steps.

---

## Request (Command Pattern)

A request is an immutable data object that represents either an external hall call or an internal floor button press.

```python
from dataclasses import dataclass, field
from typing import Optional


@dataclass(frozen=True)
class Request:
    """An elevator ride request."""

    floor: int
    direction: Direction           # desired direction (external) or IDLE (internal)
    request_type: RequestType
    timestamp: int = 0             # simulation tick when request was made
    source_elevator_id: int = -1   # which elevator (for internal requests)

    def __repr__(self) -> str:
        if self.request_type == RequestType.EXTERNAL:
            return f"EXT(floor={self.floor}, dir={self.direction.value})"
        return f"INT(floor={self.floor}, elev={self.source_elevator_id})"


class RequestQueue:
    """Manages pending requests for an elevator with deduplication."""

    def __init__(self) -> None:
        self._up_stops: set[int] = set()      # floors to stop at going up
        self._down_stops: set[int] = set()    # floors to stop at going down

    def add_stop(self, floor: int, direction: Direction) -> None:
        """Add a floor to the stop set for the given direction."""
        if direction == Direction.UP:
            self._up_stops.add(floor)
        elif direction == Direction.DOWN:
            self._down_stops.add(floor)
        else:
            # IDLE — add to whichever set based on context (handled by elevator)
            self._up_stops.add(floor)

    def remove_stop(self, floor: int, direction: Direction) -> None:
        """Remove a floor from the stop set after servicing."""
        self._up_stops.discard(floor)
        self._down_stops.discard(floor)

    def has_stop(self, floor: int) -> bool:
        """Check if the elevator needs to stop at this floor."""
        return floor in self._up_stops or floor in self._down_stops

    def has_stops_above(self, current: int) -> bool:
        return any(f > current for f in self._up_stops | self._down_stops)

    def has_stops_below(self, current: int) -> bool:
        return any(f < current for f in self._up_stops | self._down_stops)

    def has_any_stops(self) -> bool:
        return bool(self._up_stops or self._down_stops)

    def get_next_up(self, current: int) -> int | None:
        """Get the nearest floor above current in the up set."""
        above = sorted(f for f in self._up_stops if f > current)
        return above[0] if above else None

    def get_next_down(self, current: int) -> int | None:
        """Get the nearest floor below current in the down set."""
        below = sorted((f for f in self._down_stops if f < current), reverse=True)
        return below[0] if below else None

    @property
    def up_stops(self) -> set[int]:
        return set(self._up_stops)

    @property
    def down_stops(self) -> set[int]:
        return set(self._down_stops)

    def clear(self) -> None:
        self._up_stops.clear()
        self._down_stops.clear()

    def __repr__(self) -> str:
        return f"Up={sorted(self._up_stops)} Down={sorted(self._down_stops)}"
```

`Request` is a frozen dataclass — immutable after creation. External requests carry a direction (the hall button pressed); internal requests carry the source elevator ID. `RequestQueue` uses two sets — `_up_stops` and `_down_stops` — to separate floors by intended travel direction. This is the foundation of the SCAN algorithm: serve all up-stops while going up, then all down-stops while going down. `has_stops_above` and `has_stops_below` let the elevator decide when to reverse direction. Sets provide O(1) deduplication — pressing the same floor button twice does not create two stops.

---

## Elevator Car (State Pattern)

The elevator car is the most complex component. It manages its own state transitions, movement, door control, and stop servicing.

```python
from abc import ABC, abstractmethod
from typing import Callable


class ElevatorCar:
    """A single elevator car with state management and request handling."""

    def __init__(self, elevator_id: int, min_floor: int,
                 max_floor: int, capacity: int = DEFAULT_CAPACITY) -> None:
        self._id = elevator_id
        self._min_floor = min_floor
        self._max_floor = max_floor
        self._capacity = capacity
        self._current_floor = min_floor
        self._direction = Direction.IDLE
        self._state = ElevatorState.IDLE
        self._queue = RequestQueue()
        self._passengers = 0
        self._door_open_remaining = 0
        self._observers: list["ElevatorObserver"] = []

    # ------------------------------------------------------------------
    # Properties
    # ------------------------------------------------------------------

    @property
    def id(self) -> int:
        return self._id

    @property
    def current_floor(self) -> int:
        return self._current_floor

    @property
    def direction(self) -> Direction:
        return self._direction

    @property
    def state(self) -> ElevatorState:
        return self._state

    @property
    def passengers(self) -> int:
        return self._passengers

    @property
    def capacity(self) -> int:
        return self._capacity

    @property
    def queue(self) -> RequestQueue:
        return self._queue

    def is_idle(self) -> bool:
        return self._state == ElevatorState.IDLE

    def is_moving(self) -> bool:
        return self._state in (ElevatorState.MOVING_UP, ElevatorState.MOVING_DOWN)

    def has_capacity(self) -> bool:
        return self._passengers < self._capacity

    # ------------------------------------------------------------------
    # Observer management
    # ------------------------------------------------------------------

    def add_observer(self, observer: "ElevatorObserver") -> None:
        self._observers.append(observer)

    def _notify_observers(self) -> None:
        for obs in self._observers:
            obs.on_elevator_update(self)

    # ------------------------------------------------------------------
    # Request handling
    # ------------------------------------------------------------------

    def add_request(self, floor: int, direction: Direction = Direction.IDLE) -> bool:
        """
        Add a floor to this elevator's stop list.
        Returns False if the floor is out of range.
        """
        if floor < self._min_floor or floor > self._max_floor:
            return False

        # Determine which direction set to add to
        if floor > self._current_floor:
            self._queue.add_stop(floor, Direction.UP)
        elif floor < self._current_floor:
            self._queue.add_stop(floor, Direction.DOWN)
        else:
            # Already at this floor — open doors
            self._open_doors()
            return True

        # Wake up if idle
        if self._state == ElevatorState.IDLE:
            if floor > self._current_floor:
                self._direction = Direction.UP
                self._state = ElevatorState.MOVING_UP
            else:
                self._direction = Direction.DOWN
                self._state = ElevatorState.MOVING_DOWN

        return True

    # ------------------------------------------------------------------
    # Simulation step — SCAN algorithm
    # ------------------------------------------------------------------

    def step(self) -> None:
        """
        Execute one simulation time step. This is the core SCAN logic.
        
        The SCAN algorithm works like a disk arm:
        1. Move in the current direction, stopping at requested floors.
        2. When there are no more requests in the current direction, reverse.
        3. When there are no requests at all, become idle.
        """
        if self._state == ElevatorState.EMERGENCY_STOP:
            return

        if self._state == ElevatorState.DOOR_OPEN:
            self._handle_door_open()
            return

        if self._state == ElevatorState.IDLE:
            if self._queue.has_any_stops():
                # Wake up and choose direction
                if self._queue.has_stops_above(self._current_floor):
                    self._direction = Direction.UP
                    self._state = ElevatorState.MOVING_UP
                elif self._queue.has_stops_below(self._current_floor):
                    self._direction = Direction.DOWN
                    self._state = ElevatorState.MOVING_DOWN
            return

        # MOVING_UP or MOVING_DOWN
        self._move_one_floor()

        # Check if we should stop at this floor
        if self._queue.has_stop(self._current_floor):
            self._open_doors()
            return

        # Check if we need to reverse or go idle
        self._check_direction()

    def _move_one_floor(self) -> None:
        """Move one floor in the current direction."""
        if self._direction == Direction.UP:
            if self._current_floor < self._max_floor:
                self._current_floor += 1
        elif self._direction == Direction.DOWN:
            if self._current_floor > self._min_floor:
                self._current_floor -= 1
        self._notify_observers()

    def _open_doors(self) -> None:
        """Transition to DOOR_OPEN state."""
        self._state = ElevatorState.DOOR_OPEN
        self._door_open_remaining = DOOR_OPEN_TICKS
        self._queue.remove_stop(self._current_floor, self._direction)
        # Simulate passenger exchange
        self._passengers = max(0, self._passengers - 1)  # someone exits
        self._notify_observers()

    def _handle_door_open(self) -> None:
        """Count down door open time, then close and decide next action."""
        self._door_open_remaining -= 1
        if self._door_open_remaining <= 0:
            # Doors close
            self._check_direction()

    def _check_direction(self) -> None:
        """Decide whether to continue, reverse, or go idle."""
        if self._direction == Direction.UP:
            if self._queue.has_stops_above(self._current_floor):
                self._state = ElevatorState.MOVING_UP
            elif self._queue.has_stops_below(self._current_floor):
                self._direction = Direction.DOWN
                self._state = ElevatorState.MOVING_DOWN
            else:
                self._go_idle()
        elif self._direction == Direction.DOWN:
            if self._queue.has_stops_below(self._current_floor):
                self._state = ElevatorState.MOVING_DOWN
            elif self._queue.has_stops_above(self._current_floor):
                self._direction = Direction.UP
                self._state = ElevatorState.MOVING_UP
            else:
                self._go_idle()
        else:
            self._go_idle()

    def _go_idle(self) -> None:
        self._direction = Direction.IDLE
        self._state = ElevatorState.IDLE
        self._notify_observers()

    # ------------------------------------------------------------------
    # Passenger management
    # ------------------------------------------------------------------

    def board_passenger(self) -> bool:
        """A passenger boards the elevator. Returns False if at capacity."""
        if self._passengers >= self._capacity:
            return False
        self._passengers += 1
        return True

    def unboard_passenger(self) -> None:
        """A passenger exits the elevator."""
        self._passengers = max(0, self._passengers - 1)

    # ------------------------------------------------------------------
    # Emergency
    # ------------------------------------------------------------------

    def emergency_stop(self) -> None:
        """Immediately stop the elevator."""
        self._state = ElevatorState.EMERGENCY_STOP
        self._direction = Direction.IDLE
        self._notify_observers()

    def reset_emergency(self) -> None:
        """Clear emergency state and go idle."""
        self._state = ElevatorState.IDLE
        self._notify_observers()

    # ------------------------------------------------------------------
    # Display
    # ------------------------------------------------------------------

    def __repr__(self) -> str:
        dir_symbol = {"UP": "▲", "DOWN": "▼", "IDLE": "●"}
        return (
            f"Elevator {self._id}: "
            f"Floor {self._current_floor} "
            f"{dir_symbol.get(self._direction.value, '?')} "
            f"[{self._state.value}] "
            f"({self._passengers}/{self._capacity} pax) "
            f"Queue: {self._queue}"
        )
```

`step()` is the heart of the SCAN algorithm. Each call represents one time step. The state machine flows: `IDLE → MOVING → DOOR_OPEN → MOVING (or IDLE)`. When moving, the car advances one floor and checks if it should stop. When the door is open, it counts down `DOOR_OPEN_TICKS` before closing. `_check_direction` is called after every stop to decide whether to keep going, reverse, or go idle. The SCAN pattern serves all requests in the current direction before reversing — this minimises total travel distance.

`add_request` intelligently places the floor into the up or down stop set based on the floor's position relative to the car. If the car is idle, it wakes up and starts moving toward the request. `_open_doors` removes the floor from the stop set and simulates a passenger exchange.

---

## Dispatch Strategy (Strategy Pattern)

When a person presses a hall button (external request), the system must decide which elevator to send. Different strategies optimise for different goals.

### Abstract Strategy

```python
class DispatchStrategy(ABC):
    """Decides which elevator car should service an external request."""

    @abstractmethod
    def select_elevator(self, request: Request,
                        elevators: list[ElevatorCar]) -> ElevatorCar | None:
        """Return the best elevator for this request, or None if none available."""
        ...

    @abstractmethod
    def name(self) -> str:
        ...
```

### Nearest Car (LOOK-based)

```python
class NearestCarStrategy(DispatchStrategy):
    """
    Selects the elevator that can reach the requested floor fastest.
    
    Priority:
    1. An elevator already moving toward the floor in the right direction.
    2. An idle elevator closest to the floor.
    3. Any elevator (least loaded).
    """

    def select_elevator(self, request: Request,
                        elevators: list[ElevatorCar]) -> ElevatorCar | None:
        floor = request.floor
        direction = request.direction
        best: ElevatorCar | None = None
        best_score = float("inf")

        for car in elevators:
            if car.state == ElevatorState.EMERGENCY_STOP:
                continue

            score = self._calculate_score(car, floor, direction)
            if score < best_score:
                best_score = score
                best = car

        return best

    def _calculate_score(self, car: ElevatorCar, floor: int,
                         direction: Direction) -> float:
        """
        Lower score = better fit.
        
        Scoring:
        - Elevator moving toward the floor in the same direction: distance × 1
        - Idle elevator: distance × 2
        - Elevator moving away or opposite direction: distance × 3
        """
        distance = abs(car.current_floor - floor)

        if car.is_idle():
            return distance * 2

        if car.direction == Direction.UP and direction == Direction.UP:
            if car.current_floor <= floor:
                return distance * 1  # already heading there
            return distance * 3  # passed the floor, will need to reverse

        if car.direction == Direction.DOWN and direction == Direction.DOWN:
            if car.current_floor >= floor:
                return distance * 1
            return distance * 3

        return distance * 3  # moving in the wrong direction

    def name(self) -> str:
        return "Nearest Car"
```

The nearest-car strategy uses a scoring function. An elevator already heading toward the floor in the matching direction gets the lowest score (distance × 1). An idle elevator gets distance × 2. An elevator heading away gets distance × 3. This naturally prefers elevators that can serve the request without changing direction.

### Zone-Based

```python
class ZoneStrategy(DispatchStrategy):
    """
    Divides floors into zones, each served by dedicated elevators.
    Falls back to nearest-car if the zone elevator is busy.
    """

    def __init__(self, num_floors: int, num_elevators: int) -> None:
        self._zones: dict[int, list[int]] = {}  # elevator_id -> [floors]
        floors_per_zone = max(1, num_floors // num_elevators)

        for i in range(num_elevators):
            start = i * floors_per_zone
            end = start + floors_per_zone if i < num_elevators - 1 else num_floors
            self._zones[i] = list(range(start, end + 1))

        self._fallback = NearestCarStrategy()

    def select_elevator(self, request: Request,
                        elevators: list[ElevatorCar]) -> ElevatorCar | None:
        # Find the zone elevator for this floor
        for car in elevators:
            if car.id in self._zones:
                if request.floor in self._zones[car.id]:
                    if car.state != ElevatorState.EMERGENCY_STOP:
                        return car

        # Fallback to nearest car
        return self._fallback.select_elevator(request, elevators)

    def name(self) -> str:
        return "Zone-Based"
```

`ZoneStrategy` divides the building into zones and assigns each elevator to a zone. Elevator 0 handles floors 0-3, elevator 1 handles floors 4-7, etc. This reduces contention in busy buildings. If the zone elevator is in emergency stop, it falls back to `NearestCarStrategy`.

### Round Robin

```python
class RoundRobinStrategy(DispatchStrategy):
    """Distributes requests evenly across all elevators in rotation."""

    def __init__(self) -> None:
        self._next_index = 0

    def select_elevator(self, request: Request,
                        elevators: list[ElevatorCar]) -> ElevatorCar | None:
        available = [c for c in elevators
                     if c.state != ElevatorState.EMERGENCY_STOP]
        if not available:
            return None

        car = available[self._next_index % len(available)]
        self._next_index = (self._next_index + 1) % len(available)
        return car

    def name(self) -> str:
        return "Round Robin"
```

`RoundRobinStrategy` is the simplest dispatch — it cycles through elevators regardless of position. It guarantees equal load distribution but does not optimise for travel time. Useful as a baseline or in low-traffic buildings.

---

## Floor Panel & Display (Observer Pattern)

### Observer Interface

```python
class ElevatorObserver(ABC):
    """Observer interface for elevator state changes."""

    @abstractmethod
    def on_elevator_update(self, elevator: ElevatorCar) -> None:
        ...
```

### Floor Display

```python
class FloorDisplay(ElevatorObserver):
    """Shows the current position and direction of each elevator on a floor."""

    def __init__(self, floor_number: int) -> None:
        self._floor = floor_number
        self._elevator_info: dict[int, tuple[int, Direction, ElevatorState]] = {}

    def on_elevator_update(self, elevator: ElevatorCar) -> None:
        self._elevator_info[elevator.id] = (
            elevator.current_floor,
            elevator.direction,
            elevator.state,
        )

    def render(self) -> str:
        """Return a formatted string for this floor's display."""
        parts = []
        for eid in sorted(self._elevator_info):
            floor, direction, state = self._elevator_info[eid]
            dir_sym = {"UP": "▲", "DOWN": "▼", "IDLE": "●"}
            symbol = dir_sym.get(direction.value, "?")
            here = " ◄" if floor == self._floor else ""
            parts.append(f"E{eid}:{floor}{symbol}{here}")
        return " | ".join(parts)
```

`FloorDisplay` is registered as an observer on each elevator. Every time an elevator moves, changes state, or opens doors, it calls `on_elevator_update`, and the display records the new position. `render` formats all elevator positions into a single line. The `◄` marker indicates an elevator is currently on this floor.

### Event Logger

```python
class EventLogger(ElevatorObserver):
    """Logs all elevator state changes for analysis."""

    def __init__(self) -> None:
        self._log: list[str] = []

    def on_elevator_update(self, elevator: ElevatorCar) -> None:
        entry = (
            f"  [E{elevator.id}] Floor {elevator.current_floor} "
            f"| {elevator.state.value} | Dir: {elevator.direction.value} "
            f"| Pax: {elevator.passengers}"
        )
        self._log.append(entry)

    def print_log(self, last_n: int = 20) -> None:
        print(f"\n  Event Log (last {last_n}):")
        for entry in self._log[-last_n:]:
            print(entry)

    def clear(self) -> None:
        self._log.clear()
```

`EventLogger` is a second observer that records every state change. This is invaluable for debugging dispatch decisions and analysing elevator behaviour. `print_log` shows the most recent entries.

---

## Elevator System (Facade)

The `ElevatorSystem` ties everything together. It manages all elevator cars, dispatches external requests, and runs the simulation.

```python
class ElevatorSystem:
    """
    Central controller for all elevators in a building.
    Acts as a Facade over cars, dispatch, and floor displays.
    """

    def __init__(self, num_floors: int, num_elevators: int,
                 capacity: int = DEFAULT_CAPACITY,
                 strategy: DispatchStrategy | None = None) -> None:
        self._num_floors = num_floors
        self._min_floor = 0
        self._max_floor = num_floors - 1
        self._tick = 0

        # Create elevator cars
        self._elevators: list[ElevatorCar] = []
        for i in range(num_elevators):
            car = ElevatorCar(i, self._min_floor, self._max_floor, capacity)
            self._elevators.append(car)

        # Dispatch strategy
        self._strategy = strategy or NearestCarStrategy()

        # Floor displays
        self._floor_displays: list[FloorDisplay] = []
        for f in range(num_floors):
            display = FloorDisplay(f)
            self._floor_displays.append(display)

        # Register displays as observers on all elevators
        for car in self._elevators:
            for display in self._floor_displays:
                car.add_observer(display)

        # Pending external requests (for tracking)
        self._pending_requests: list[Request] = []

    # ------------------------------------------------------------------
    # Properties
    # ------------------------------------------------------------------

    @property
    def tick(self) -> int:
        return self._tick

    @property
    def elevators(self) -> list[ElevatorCar]:
        return list(self._elevators)

    @property
    def num_floors(self) -> int:
        return self._num_floors

    # ------------------------------------------------------------------
    # Request handling
    # ------------------------------------------------------------------

    def request_elevator(self, floor: int, direction: Direction) -> ElevatorCar | None:
        """
        External request: someone on a floor presses Up or Down.
        Dispatches the request to the best elevator.
        """
        if floor < self._min_floor or floor > self._max_floor:
            print(f"  ✗ Floor {floor} does not exist")
            return None

        request = Request(
            floor=floor,
            direction=direction,
            request_type=RequestType.EXTERNAL,
            timestamp=self._tick,
        )

        car = self._strategy.select_elevator(request, self._elevators)
        if car is None:
            print(f"  ✗ No elevator available for floor {floor}")
            self._pending_requests.append(request)
            return None

        car.add_request(floor, direction)
        print(f"  📡 Dispatched E{car.id} to floor {floor} ({direction.value})")
        return car

    def request_floor(self, elevator_id: int, floor: int) -> bool:
        """
        Internal request: someone inside elevator presses a floor button.
        """
        if elevator_id < 0 or elevator_id >= len(self._elevators):
            print(f"  ✗ Elevator {elevator_id} does not exist")
            return False

        car = self._elevators[elevator_id]

        if not car.has_capacity():
            print(f"  ✗ Elevator {elevator_id} is at capacity")
            return False

        success = car.add_request(floor)
        if success:
            car.board_passenger()
            print(f"  🔘 E{elevator_id}: floor {floor} button pressed")
        return success

    # ------------------------------------------------------------------
    # Simulation
    # ------------------------------------------------------------------

    def step(self) -> None:
        """Advance the simulation by one time step."""
        self._tick += 1
        for car in self._elevators:
            car.step()

        # Retry pending requests
        remaining: list[Request] = []
        for req in self._pending_requests:
            car = self._strategy.select_elevator(req, self._elevators)
            if car:
                car.add_request(req.floor, req.direction)
            else:
                remaining.append(req)
        self._pending_requests = remaining

    def run(self, steps: int) -> None:
        """Run the simulation for a given number of steps."""
        for _ in range(steps):
            self.step()

    def run_until_idle(self, max_steps: int = 200) -> int:
        """Run until all elevators are idle or max steps reached."""
        for i in range(max_steps):
            self.step()
            if all(car.is_idle() for car in self._elevators):
                return i + 1
        return max_steps

    # ------------------------------------------------------------------
    # Display
    # ------------------------------------------------------------------

    def display_status(self) -> None:
        """Print the current state of all elevators."""
        print(f"\n  {'='*60}")
        print(f"  ELEVATOR SYSTEM — Tick {self._tick}")
        print(f"  Strategy: {self._strategy.name()}")
        print(f"  {'='*60}")
        for car in self._elevators:
            print(f"  {car}")
        print()

    def display_building(self) -> None:
        """Print a visual representation of the building."""
        print(f"\n  {'='*60}")
        print(f"  BUILDING VIEW — Tick {self._tick}")
        print(f"  {'='*60}")

        for floor in range(self._max_floor, self._min_floor - 1, -1):
            # Floor label
            floor_str = f"  F{floor:>2d} |"

            # Show elevators on this floor
            for car in self._elevators:
                if car.current_floor == floor:
                    state_char = {
                        ElevatorState.IDLE: "■",
                        ElevatorState.MOVING_UP: "▲",
                        ElevatorState.MOVING_DOWN: "▼",
                        ElevatorState.DOOR_OPEN: "◆",
                        ElevatorState.EMERGENCY_STOP: "✗",
                    }
                    char = state_char.get(car.state, "?")
                    floor_str += f" [{char}E{car.id}]"
                else:
                    floor_str += "      "

            # Floor display panel
            display_info = self._floor_displays[floor].render()
            floor_str += f"  │ {display_info}"

            print(floor_str)

        print(f"  {'─'*60}")
        print()
```

`request_elevator` is for external hall calls. It creates a `Request`, uses the dispatch strategy to select the best car, and adds the floor to that car's stop list. `request_floor` is for internal button presses — it validates capacity and adds the destination floor. `step` advances all elevators by one time step and retries any pending requests that previously had no available car. `run_until_idle` runs the simulation until all elevators have no more stops — essential for testing. `display_building` renders a vertical view of the building with elevator positions.

---

## Simulation Engine

A helper that scripts realistic scenarios for demonstration.

```python
class Scenario:
    """A sequence of timed events for simulation."""

    def __init__(self, system: ElevatorSystem) -> None:
        self._system = system
        self._events: list[tuple[int, Callable]] = []

    def at_tick(self, tick: int, action: Callable) -> "Scenario":
        """Schedule an action at a specific tick."""
        self._events.append((tick, action))
        return self

    def run(self, display_interval: int = 5, max_ticks: int = 100) -> None:
        """Execute the scenario with periodic status display."""
        # Sort events by tick
        self._events.sort(key=lambda e: e[0])
        event_idx = 0

        print("\n" + "═" * 62)
        print("  SIMULATION START")
        print("═" * 62)

        for tick in range(1, max_ticks + 1):
            # Execute scheduled events for this tick
            while (event_idx < len(self._events)
                   and self._events[event_idx][0] == tick):
                print(f"\n  ⏱ Tick {tick}: Executing event...")
                self._events[event_idx][1]()
                event_idx += 1

            # Advance simulation
            self._system.step()

            # Display at intervals
            if tick % display_interval == 0:
                self._system.display_building()

            # Check if all done
            if (event_idx >= len(self._events)
                    and all(car.is_idle() for car in self._system.elevators)):
                print(f"\n  ✅ All elevators idle at tick {tick}")
                self._system.display_building()
                break

        print("\n" + "═" * 62)
        print("  SIMULATION END")
        print("═" * 62)
```

`Scenario` lets you script events at specific ticks: "at tick 3, someone on floor 7 presses Down". `run` executes the simulation, firing events at the right ticks and displaying the building state at configurable intervals. It stops early when all elevators are idle and all events have been processed.

---

## Putting It All Together

### Basic Simulation

```python
def main() -> None:
    # Create a 10-floor building with 3 elevators
    system = ElevatorSystem(
        num_floors=10,
        num_elevators=3,
        capacity=8,
        strategy=NearestCarStrategy(),
    )

    # Attach an event logger
    logger = EventLogger()
    for car in system.elevators:
        car.add_observer(logger)

    print("╔═══════════════════════════════════════╗")
    print("║       ELEVATOR SYSTEM                 ║")
    print("║  10 Floors | 3 Elevators              ║")
    print("║  Strategy: Nearest Car                ║")
    print("╚═══════════════════════════════════════╝")

    # People press hall buttons
    system.request_elevator(5, Direction.UP)    # Someone on floor 5 going up
    system.request_elevator(8, Direction.DOWN)  # Someone on floor 8 going down
    system.request_elevator(3, Direction.UP)    # Someone on floor 3 going up

    system.display_status()

    # Run until all served
    steps = system.run_until_idle()
    print(f"  All requests served in {steps} steps")

    system.display_building()
    logger.print_log(last_n=15)


if __name__ == "__main__":
    main()
```

### Scripted Scenario

```python
def main_scenario() -> None:
    system = ElevatorSystem(
        num_floors=15,
        num_elevators=3,
        capacity=10,
        strategy=NearestCarStrategy(),
    )

    scenario = Scenario(system)

    # Morning rush: people going up from the lobby
    scenario.at_tick(1, lambda: system.request_elevator(0, Direction.UP))
    scenario.at_tick(1, lambda: system.request_floor(0, 5))

    scenario.at_tick(2, lambda: system.request_elevator(0, Direction.UP))
    scenario.at_tick(2, lambda: system.request_floor(1, 10))

    scenario.at_tick(3, lambda: system.request_elevator(0, Direction.UP))
    scenario.at_tick(3, lambda: system.request_floor(2, 14))

    # Mid-simulation: someone on floor 7 wants to go down
    scenario.at_tick(8, lambda: system.request_elevator(7, Direction.DOWN))

    # Someone on floor 12 wants to go down
    scenario.at_tick(10, lambda: system.request_elevator(12, Direction.DOWN))

    scenario.run(display_interval=5, max_ticks=50)


if __name__ == "__main__":
    main_scenario()
```

### Strategy Comparison

```python
def compare_strategies() -> None:
    """Compare dispatch strategies on the same workload."""
    import random
    random.seed(42)

    strategies: list[DispatchStrategy] = [
        NearestCarStrategy(),
        RoundRobinStrategy(),
        ZoneStrategy(num_floors=20, num_elevators=4),
    ]

    # Generate a random workload
    requests: list[tuple[int, Direction]] = []
    for _ in range(15):
        floor = random.randint(0, 19)
        direction = random.choice([Direction.UP, Direction.DOWN])
        requests.append((floor, direction))

    print("\n" + "=" * 55)
    print("  STRATEGY COMPARISON")
    print("=" * 55)

    for strategy in strategies:
        system = ElevatorSystem(
            num_floors=20,
            num_elevators=4,
            strategy=strategy,
        )

        # Submit all requests
        for floor, direction in requests:
            system.request_elevator(floor, direction)

        # Run until idle
        steps = system.run_until_idle(max_steps=200)
        print(f"  {strategy.name():>20s}: completed in {steps:>3d} steps")

    print("=" * 55)
```

This test generates 15 random requests and runs each strategy to completion, comparing how many time steps each one takes. `NearestCarStrategy` typically wins because it minimises total travel distance.

### Unit Tests

```python
def test_basic_movement() -> None:
    """Test that an elevator moves to the requested floor."""
    system = ElevatorSystem(num_floors=10, num_elevators=1)
    system.request_elevator(5, Direction.UP)
    steps = system.run_until_idle()
    assert system.elevators[0].current_floor == 5
    print(f"✅ test_basic_movement passed ({steps} steps)")


def test_scan_direction() -> None:
    """Test that SCAN serves requests in the current direction first."""
    system = ElevatorSystem(num_floors=10, num_elevators=1)
    car = system.elevators[0]

    # Request floor 8, then floor 3 — should go to 8 first (going up)
    car.add_request(8, Direction.UP)
    car.add_request(3, Direction.DOWN)

    # Run until floor 8 is reached
    for _ in range(20):
        system.step()
        if car.current_floor == 8 and car.queue.has_stop(3):
            # Reached 8 while 3 is still pending — correct SCAN order
            break

    # Now it should reverse to floor 3
    system.run_until_idle()
    assert car.current_floor == 3
    print("✅ test_scan_direction passed")


def test_dispatch_nearest() -> None:
    """Test that nearest-car dispatch selects the closest elevator."""
    system = ElevatorSystem(num_floors=10, num_elevators=3,
                            strategy=NearestCarStrategy())
    # Move elevator 1 to floor 7
    system.elevators[1].add_request(7)
    system.run_until_idle()

    # Request from floor 8 — elevator 1 (at floor 7) should be selected
    car = system.request_elevator(8, Direction.UP)
    assert car is not None
    assert car.id == 1
    print("✅ test_dispatch_nearest passed")


def test_capacity_limit() -> None:
    """Test that an elevator rejects requests when at capacity."""
    system = ElevatorSystem(num_floors=10, num_elevators=1, capacity=2)

    # Board 2 passengers
    system.elevators[0].board_passenger()
    system.elevators[0].board_passenger()

    # Third should fail
    assert not system.elevators[0].has_capacity()
    print("✅ test_capacity_limit passed")


def test_emergency_stop() -> None:
    """Test that emergency stop halts the elevator."""
    system = ElevatorSystem(num_floors=10, num_elevators=1)
    car = system.elevators[0]

    car.add_request(9)
    system.step()
    system.step()

    car.emergency_stop()
    floor_at_stop = car.current_floor

    # Further steps should not move the car
    system.step()
    system.step()
    assert car.current_floor == floor_at_stop
    assert car.state == ElevatorState.EMERGENCY_STOP

    # Reset and verify it can move again
    car.reset_emergency()
    assert car.state == ElevatorState.IDLE
    print("✅ test_emergency_stop passed")


def run_all_tests() -> None:
    print("\n" + "=" * 40)
    print("  RUNNING TESTS")
    print("=" * 40 + "\n")
    test_basic_movement()
    test_scan_direction()
    test_dispatch_nearest()
    test_capacity_limit()
    test_emergency_stop()
    print("\n✅ All tests passed!\n")
```

Each test creates a minimal system, performs a specific action, and asserts the outcome. `test_scan_direction` verifies the SCAN algorithm processes floors in the correct order. `test_dispatch_nearest` checks that the dispatch strategy selects the closest idle elevator. `test_emergency_stop` confirms that the emergency stop halts movement and that the elevator can be reset.

---

## Class Diagram

```
┌──────────────────────────────────────────────────────────────┐
│                      ElevatorSystem                           │
│──────────────────────────────────────────────────────────────│
│ - _num_floors: int                                            │
│ - _elevators: list[ElevatorCar]                               │
│ - _strategy: DispatchStrategy                                 │
│ - _floor_displays: list[FloorDisplay]                         │
│ - _pending_requests: list[Request]                            │
│ - _tick: int                                                  │
│──────────────────────────────────────────────────────────────│
│ + request_elevator(floor, direction) -> ElevatorCar | None    │
│ + request_floor(elevator_id, floor) -> bool                   │
│ + step() -> None                                              │
│ + run(steps) -> None                                          │
│ + run_until_idle(max) -> int                                  │
│ + display_building() -> None                                  │
└──────┬───────────┬──────────────┬────────────────────────────┘
       │           │              │
  owns ▼      uses ▼        owns ▼
┌──────────────┐ ┌──────────────────────────────────────────┐
│ ElevatorCar  │ │     «abstract» DispatchStrategy          │
│──────────────│ │──────────────────────────────────────────│
│ - _id        │ │ + select_elevator(req, cars) -> Car|None │
│ - _current_  │ │ + name() -> str                          │
│   floor      │ └──────────┬───────────────────────────────┘
│ - _direction │            │ implemented by
│ - _state     │   ┌────────┼──────────┐
│ - _queue     │   ▼        ▼          ▼
│ - _capacity  │ Nearest  ZoneBased  RoundRobin
│ - _passengers│ CarStrat  Strategy   Strategy
│──────────────│
│ + add_request│
│ + step()     │  ┌────────────────────────────────────────┐
│ + board_     │  │    «abstract» ElevatorObserver         │
│   passenger  │  │────────────────────────────────────────│
│ + emergency_ │  │ + on_elevator_update(car) -> None      │
│   stop()     │  └──────────┬─────────────────────────────┘
└──────┬───────┘             │ implemented by
       │ owns          ┌─────┴──────────┐
       ▼               ▼                ▼
┌──────────────┐  FloorDisplay     EventLogger
│ RequestQueue │
│──────────────│
│ - _up_stops  │  ┌────────────────────────────────────────┐
│ - _down_stops│  │            Scenario                    │
│──────────────│  │────────────────────────────────────────│
│ + add_stop   │  │ - _system: ElevatorSystem              │
│ + remove_stop│  │ - _events: list[(tick, Callable)]      │
│ + has_stops_ │  │────────────────────────────────────────│
│   above()    │  │ + at_tick(tick, action) -> self         │
│ + has_stops_ │  │ + run(interval, max_ticks) -> None     │
│   below()    │  └────────────────────────────────────────┘
└──────────────┘


┌───────────────────────┐
│  Request (frozen)     │
│───────────────────────│
│ + floor: int          │
│ + direction: Direction│
│ + request_type        │
│ + timestamp: int      │
│ + source_elevator_id  │
└───────────────────────┘

┌────────────────┐  ┌──────────────────┐  ┌───────────────────┐
│ Direction(Enum)│  │ElevatorState(Enum)│  │ RequestType(Enum) │
│────────────────│  │──────────────────│  │───────────────────│
│ UP             │  │ IDLE             │  │ EXTERNAL          │
│ DOWN           │  │ MOVING_UP        │  │ INTERNAL          │
│ IDLE           │  │ MOVING_DOWN      │  └───────────────────┘
│────────────────│  │ DOOR_OPEN        │
│ + opposite()   │  │ EMERGENCY_STOP   │
└────────────────┘  └──────────────────┘
```

---

## How to Run

Save all code into a single file `elevator_system.py` and run:

```bash
python elevator_system.py
```

### Example Output

```
╔═══════════════════════════════════════╗
║       ELEVATOR SYSTEM                 ║
║  10 Floors | 3 Elevators              ║
║  Strategy: Nearest Car                ║
╚═══════════════════════════════════════╝

  📡 Dispatched E0 to floor 5 (UP)
  📡 Dispatched E1 to floor 8 (DOWN)
  📡 Dispatched E2 to floor 3 (UP)

  ============================================================
  BUILDING VIEW — Tick 5
  ============================================================
  F 9 |                          │ E0:5▲ | E1:5▲ | E2:3●
  F 8 |                          │
  F 7 |                          │
  F 6 |                          │
  F 5 | [▲E0]                    │
  F 4 |              [▲E1]       │
  F 3 |                    [●E2] │
  F 2 |                          │
  F 1 |                          │
  F 0 |                          │
  ────────────────────────────────────────────────────────

  ✅ All elevators idle at tick 12
  All requests served in 12 steps
```
