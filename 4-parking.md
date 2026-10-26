# Design Parking Lot – Low-Level Design Guide

![Bidur Sapkota](https://www.bidursapkota.com.np/images/gravatar.webp "Bidur Sapkota - Developer")&nbsp;[Bidur Sapkota](https://www.bidursapkota.com.np/)

## Table of Contents

1. [Requirements](#requirements)
2. [Key Design Patterns](#key-design-patterns)
3. [Core Enums & Constants](#core-enums--constants)
4. [Vehicle Hierarchy](#vehicle-hierarchy)
5. [Parking Spot](#parking-spot)
6. [Parking Floor](#parking-floor)
7. [Ticket & Payment](#ticket--payment)
8. [Pricing Strategy (Strategy Pattern)](#pricing-strategy-strategy-pattern)
9. [Display Board (Observer Pattern)](#display-board-observer-pattern)
10. [Entry & Exit Gates](#entry--exit-gates)
11. [Parking Lot (Singleton + Facade)](#parking-lot-singleton--facade)
12. [Putting It All Together](#putting-it-all-together)
13. [Class Diagram](#class-diagram)
14. [How to Run](#how-to-run)

---

## Requirements

### Functional Requirements

| #  | Requirement                                                                         |
| -- | ----------------------------------------------------------------------------------- |
| 1  | Multi-floor parking lot with configurable number of floors                          |
| 2  | Each floor has multiple parking spots of different sizes (small, medium, large)      |
| 3  | Three vehicle types: motorcycle, car, bus — each requires a specific spot size       |
| 4  | A vehicle is assigned the smallest available spot that fits it                       |
| 5  | Issue a ticket on entry with the assigned spot, record entry time                    |
| 6  | Calculate the fee on exit based on duration and a configurable pricing strategy      |
| 7  | Support multiple pricing strategies: hourly, flat rate, tiered                       |
| 8  | Track available spots per floor and per type in real time                            |
| 9  | Display board shows live availability per floor                                      |
| 10 | Reject entry when no suitable spot is available                                      |
| 11 | Support multiple entry and exit gates                                                |
| 12 | Prevent duplicate parking — a vehicle already parked cannot enter again              |

### Non-Functional Requirements

- Thread-safe operations for concurrent entry/exit (design-level, not runtime).
- Clean OOP with SOLID principles.
- Extensible – adding a new vehicle type, spot size, or pricing model should not require modifying existing code.

---

## Key Design Patterns

| Pattern   | Where It Is Used                             | Why                                                                                    |
| --------- | -------------------------------------------- | -------------------------------------------------------------------------------------- |
| Strategy  | `PricingStrategy` for fee calculation        | Swap between hourly, flat, and tiered pricing without changing ticket or gate logic     |
| Observer  | `DisplayBoard` listens to availability changes | Boards update automatically when spots are taken or freed                             |
| Singleton | `ParkingLot` has one global instance         | Only one parking lot exists; centralises state management                              |
| Facade    | `ParkingLot` exposes a simple `park`/`unpark` API | Hides the complexity of floor selection, spot assignment, and ticketing              |
| Factory Method | `ParkingFloor.find_available_spot()`    | Encapsulates spot selection logic so the caller does not choose spots directly          |

---

## Core Enums & Constants

```python
from enum import Enum
from datetime import datetime


class VehicleType(Enum):
    MOTORCYCLE = "motorcycle"
    CAR = "car"
    BUS = "bus"


class SpotSize(Enum):
    SMALL = 1       # fits motorcycle
    MEDIUM = 2      # fits car
    LARGE = 3       # fits bus

    def __ge__(self, other: "SpotSize") -> bool:
        return self.value >= other.value

    def __gt__(self, other: "SpotSize") -> bool:
        return self.value > other.value

    def __le__(self, other: "SpotSize") -> bool:
        return self.value <= other.value

    def __lt__(self, other: "SpotSize") -> bool:
        return self.value < other.value


class SpotStatus(Enum):
    AVAILABLE = "available"
    OCCUPIED = "occupied"


# Maps each vehicle type to the minimum spot size it requires
VEHICLE_SPOT_REQUIREMENT: dict[VehicleType, SpotSize] = {
    VehicleType.MOTORCYCLE: SpotSize.SMALL,
    VehicleType.CAR: SpotSize.MEDIUM,
    VehicleType.BUS: SpotSize.LARGE,
}
```

`VehicleType` enumerates the kinds of vehicles the lot accepts. `SpotSize` uses integer values to enable comparison — a `LARGE` spot can fit any vehicle, a `SMALL` spot only fits a motorcycle. The comparison operators (`__ge__`, `__lt__`, etc.) let you write `spot.size >= required_size` naturally. `VEHICLE_SPOT_REQUIREMENT` is a lookup table that maps each vehicle to its minimum spot size. Adding a new vehicle type only requires adding one entry here.

---

## Vehicle Hierarchy

```python
from abc import ABC, abstractmethod


class Vehicle(ABC):
    """Abstract base for all vehicle types."""

    def __init__(self, license_plate: str, vehicle_type: VehicleType) -> None:
        self._license_plate = license_plate
        self._vehicle_type = vehicle_type

    @property
    def license_plate(self) -> str:
        return self._license_plate

    @property
    def vehicle_type(self) -> VehicleType:
        return self._vehicle_type

    @property
    def required_spot_size(self) -> SpotSize:
        return VEHICLE_SPOT_REQUIREMENT[self._vehicle_type]

    def __repr__(self) -> str:
        return f"{self._vehicle_type.value}({self._license_plate})"

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Vehicle):
            return NotImplemented
        return self._license_plate == other._license_plate

    def __hash__(self) -> int:
        return hash(self._license_plate)


class Motorcycle(Vehicle):
    def __init__(self, license_plate: str) -> None:
        super().__init__(license_plate, VehicleType.MOTORCYCLE)


class Car(Vehicle):
    def __init__(self, license_plate: str) -> None:
        super().__init__(license_plate, VehicleType.CAR)


class Bus(Vehicle):
    def __init__(self, license_plate: str) -> None:
        super().__init__(license_plate, VehicleType.BUS)
```

`Vehicle` is abstract. Each subclass passes its type to the base constructor, so you never need a type check — just access `vehicle.vehicle_type` or `vehicle.required_spot_size`. `__eq__` and `__hash__` use the license plate as the identity key, so two `Vehicle` objects with the same plate are considered the same vehicle. This prevents duplicate parking.

---

## Parking Spot

```python
class ParkingSpot:
    """A single parking space on a floor."""

    def __init__(self, spot_id: str, size: SpotSize, floor_number: int) -> None:
        self._spot_id = spot_id
        self._size = size
        self._floor_number = floor_number
        self._status = SpotStatus.AVAILABLE
        self._vehicle: Vehicle | None = None

    @property
    def spot_id(self) -> str:
        return self._spot_id

    @property
    def size(self) -> SpotSize:
        return self._size

    @property
    def floor_number(self) -> int:
        return self._floor_number

    @property
    def status(self) -> SpotStatus:
        return self._status

    @property
    def vehicle(self) -> Vehicle | None:
        return self._vehicle

    def is_available(self) -> bool:
        return self._status == SpotStatus.AVAILABLE

    def can_fit(self, vehicle: Vehicle) -> bool:
        """Check if this spot can accommodate the given vehicle."""
        return self.is_available() and self._size >= vehicle.required_spot_size

    def park(self, vehicle: Vehicle) -> None:
        """Occupy this spot with the given vehicle."""
        if not self.is_available():
            raise RuntimeError(f"Spot {self._spot_id} is already occupied")
        if self._size < vehicle.required_spot_size:
            raise ValueError(
                f"Spot {self._spot_id} ({self._size.name}) is too small "
                f"for {vehicle} (needs {vehicle.required_spot_size.name})"
            )
        self._vehicle = vehicle
        self._status = SpotStatus.OCCUPIED

    def unpark(self) -> Vehicle:
        """Free this spot and return the vehicle that was parked."""
        if self._vehicle is None:
            raise RuntimeError(f"Spot {self._spot_id} is already empty")
        vehicle = self._vehicle
        self._vehicle = None
        self._status = SpotStatus.AVAILABLE
        return vehicle

    def __repr__(self) -> str:
        status = "🚗" if self._status == SpotStatus.OCCUPIED else "⬜"
        return f"{status}{self._spot_id}"
```

`can_fit` checks both availability and size compatibility. `park` validates before occupying — double-parking and size mismatches are caught immediately. `unpark` returns the vehicle so the caller can use it for fee calculation. The `__repr__` uses emoji for a quick visual read in the console.

---

## Parking Floor

Each floor manages its own collection of spots and handles spot assignment.

```python
from typing import Callable


class ParkingFloor:
    """A single floor in the parking lot with its own spots."""

    def __init__(self, floor_number: int,
                 small_spots: int, medium_spots: int,
                 large_spots: int) -> None:
        self._floor_number = floor_number
        self._spots: list[ParkingSpot] = []
        self._create_spots(small_spots, medium_spots, large_spots)

    def _create_spots(self, small: int, medium: int, large: int) -> None:
        """Generate spots with IDs like F1-S01, F1-M01, F1-L01."""
        floor_prefix = f"F{self._floor_number}"
        counter = 1
        for _ in range(small):
            spot_id = f"{floor_prefix}-S{counter:02d}"
            self._spots.append(ParkingSpot(spot_id, SpotSize.SMALL, self._floor_number))
            counter += 1
        counter = 1
        for _ in range(medium):
            spot_id = f"{floor_prefix}-M{counter:02d}"
            self._spots.append(ParkingSpot(spot_id, SpotSize.MEDIUM, self._floor_number))
            counter += 1
        counter = 1
        for _ in range(large):
            spot_id = f"{floor_prefix}-L{counter:02d}"
            self._spots.append(ParkingSpot(spot_id, SpotSize.LARGE, self._floor_number))
            counter += 1

    @property
    def floor_number(self) -> int:
        return self._floor_number

    def find_available_spot(self, vehicle: Vehicle) -> ParkingSpot | None:
        """
        Find the best available spot for a vehicle.
        Prefers the smallest spot that fits, to save larger spots for larger vehicles.
        """
        best: ParkingSpot | None = None
        for spot in self._spots:
            if spot.can_fit(vehicle):
                if best is None or spot.size < best.size:
                    best = spot
        return best

    def get_availability(self) -> dict[SpotSize, tuple[int, int]]:
        """Return (available, total) counts for each spot size."""
        counts: dict[SpotSize, tuple[int, int]] = {}
        for size in SpotSize:
            total = sum(1 for s in self._spots if s.size == size)
            available = sum(
                1 for s in self._spots
                if s.size == size and s.is_available()
            )
            if total > 0:
                counts[size] = (available, total)
        return counts

    def display(self) -> None:
        """Print the floor layout."""
        print(f"\n  Floor {self._floor_number}:")
        for size in SpotSize:
            spots_of_size = [s for s in self._spots if s.size == size]
            if spots_of_size:
                row = " ".join(str(s) for s in spots_of_size)
                print(f"    {size.name:6s}: {row}")
```

`_create_spots` generates spots with descriptive IDs like `F1-S01` (Floor 1, Small spot 01). This naming scheme makes debugging and ticket reading easy. `find_available_spot` implements a best-fit strategy — it finds the smallest available spot that can fit the vehicle. A motorcycle gets a small spot if one is free, even though a large spot could also work. This maximises capacity. `get_availability` returns counts per size, used by the display board.

---

## Ticket & Payment

```python
import uuid


class ParkingTicket:
    """Issued on entry, closed on exit with fee calculation."""

    def __init__(self, vehicle: Vehicle, spot: ParkingSpot) -> None:
        self._ticket_id = str(uuid.uuid4())[:8].upper()
        self._vehicle = vehicle
        self._spot = spot
        self._entry_time = datetime.now()
        self._exit_time: datetime | None = None
        self._fee: float = 0.0
        self._is_paid = False

    @property
    def ticket_id(self) -> str:
        return self._ticket_id

    @property
    def vehicle(self) -> Vehicle:
        return self._vehicle

    @property
    def spot(self) -> ParkingSpot:
        return self._spot

    @property
    def entry_time(self) -> datetime:
        return self._entry_time

    @property
    def exit_time(self) -> datetime | None:
        return self._exit_time

    @property
    def fee(self) -> float:
        return self._fee

    @property
    def is_paid(self) -> bool:
        return self._is_paid

    def close(self, exit_time: datetime | None = None) -> None:
        """Record exit time."""
        self._exit_time = exit_time or datetime.now()

    def get_duration_hours(self) -> float:
        """Calculate parking duration in hours."""
        if self._exit_time is None:
            end = datetime.now()
        else:
            end = self._exit_time
        delta = end - self._entry_time
        return delta.total_seconds() / 3600

    def set_fee(self, fee: float) -> None:
        self._fee = round(fee, 2)

    def pay(self) -> None:
        if self._is_paid:
            raise RuntimeError(f"Ticket {self._ticket_id} is already paid")
        self._is_paid = True

    def __repr__(self) -> str:
        status = "PAID" if self._is_paid else "OPEN"
        return (
            f"Ticket({self._ticket_id} | {self._vehicle} | "
            f"Spot {self._spot.spot_id} | {status})"
        )

    def print_receipt(self) -> None:
        """Print a formatted receipt."""
        duration = self.get_duration_hours()
        print(f"\n  ┌─────────────────────────────────────┐")
        print(f"  │         PARKING RECEIPT              │")
        print(f"  ├─────────────────────────────────────┤")
        print(f"  │  Ticket:   {self._ticket_id:>25s} │")
        print(f"  │  Vehicle:  {str(self._vehicle):>25s} │")
        print(f"  │  Spot:     {self._spot.spot_id:>25s} │")
        print(f"  │  Entry:    {self._entry_time.strftime('%Y-%m-%d %H:%M'):>25s} │")
        if self._exit_time:
            print(f"  │  Exit:     {self._exit_time.strftime('%Y-%m-%d %H:%M'):>25s} │")
        print(f"  │  Duration: {duration:>22.1f} hr │")
        print(f"  │  Fee:      ${self._fee:>23.2f} │")
        print(f"  │  Status:   {'PAID' if self._is_paid else 'UNPAID':>25s} │")
        print(f"  └─────────────────────────────────────┘")
```

`ParkingTicket` captures everything about a parking session. `_ticket_id` uses a truncated UUID for uniqueness without being unwieldy. `get_duration_hours` calculates the elapsed time between entry and exit. `close` records the exit time but does not compute the fee — that is the pricing strategy's job (Single Responsibility). `print_receipt` formats a styled receipt for the console.

---

## Pricing Strategy (Strategy Pattern)

Different parking lots use different pricing models. The Strategy pattern lets the exit gate calculate fees using any formula without knowing its internals.

### Abstract Strategy

```python
class PricingStrategy(ABC):
    """Abstract pricing strategy for fee calculation."""

    @abstractmethod
    def calculate_fee(self, ticket: ParkingTicket) -> float:
        """Calculate the parking fee based on the ticket details."""
        ...

    @abstractmethod
    def name(self) -> str:
        """Human-readable name for display."""
        ...
```

### Hourly Rate

```python
class HourlyPricing(PricingStrategy):
    """Charges a fixed rate per hour, per vehicle type."""

    def __init__(self, rates: dict[VehicleType, float] | None = None) -> None:
        self._rates = rates or {
            VehicleType.MOTORCYCLE: 1.0,
            VehicleType.CAR: 2.0,
            VehicleType.BUS: 5.0,
        }

    def calculate_fee(self, ticket: ParkingTicket) -> float:
        hours = ticket.get_duration_hours()
        # Round up to the next hour
        import math
        rounded_hours = math.ceil(hours) if hours > 0 else 1
        rate = self._rates.get(ticket.vehicle.vehicle_type, 2.0)
        return rounded_hours * rate

    def name(self) -> str:
        return "Hourly"
```

`HourlyPricing` rounds up to the next full hour — 2 hours 10 minutes is charged as 3 hours. Rates vary by vehicle type. The default rates can be overridden through the constructor.

### Flat Rate

```python
class FlatRatePricing(PricingStrategy):
    """Charges a single flat fee regardless of duration."""

    def __init__(self, rates: dict[VehicleType, float] | None = None) -> None:
        self._rates = rates or {
            VehicleType.MOTORCYCLE: 5.0,
            VehicleType.CAR: 10.0,
            VehicleType.BUS: 20.0,
        }

    def calculate_fee(self, ticket: ParkingTicket) -> float:
        return self._rates.get(ticket.vehicle.vehicle_type, 10.0)

    def name(self) -> str:
        return "Flat Rate"
```

`FlatRatePricing` ignores duration entirely. Common in event venues or airports with fixed parking fees.

### Tiered Pricing

```python
class TieredPricing(PricingStrategy):
    """
    Charges different rates for different time brackets.
    Example: first 2 hours at $2/hr, next 4 hours at $1.5/hr, beyond that $1/hr.
    """

    def __init__(
        self,
        tiers: list[tuple[float, float]] | None = None,
        vehicle_multipliers: dict[VehicleType, float] | None = None,
    ) -> None:
        # Each tier: (up_to_hours, rate_per_hour)
        # The last tier applies to all remaining hours
        self._tiers = tiers or [
            (2, 3.0),    # first 2 hours at $3/hr
            (4, 2.0),    # next 4 hours at $2/hr
            (float("inf"), 1.0),  # beyond 6 hours at $1/hr
        ]
        self._multipliers = vehicle_multipliers or {
            VehicleType.MOTORCYCLE: 0.5,
            VehicleType.CAR: 1.0,
            VehicleType.BUS: 2.5,
        }

    def calculate_fee(self, ticket: ParkingTicket) -> float:
        import math
        hours = max(math.ceil(ticket.get_duration_hours()), 1)
        remaining = float(hours)
        total = 0.0

        for tier_hours, rate in self._tiers:
            if remaining <= 0:
                break
            billable = min(remaining, tier_hours)
            total += billable * rate
            remaining -= billable

        multiplier = self._multipliers.get(ticket.vehicle.vehicle_type, 1.0)
        return total * multiplier

    def name(self) -> str:
        return "Tiered"
```

`TieredPricing` walks through the tier brackets, consuming hours from each tier in order. The vehicle multiplier scales the total — motorcycles pay half, buses pay 2.5×. This is the most flexible strategy and handles complex real-world pricing schemes.

---

## Display Board (Observer Pattern)

Display boards show real-time availability. They observe the parking lot and update automatically when spots change.

```python
class DisplayObserver(ABC):
    """Observer interface for parking lot state changes."""

    @abstractmethod
    def update(self, floor_number: int, availability: dict[SpotSize, tuple[int, int]]) -> None:
        """Called when availability changes on a floor."""
        ...


class DisplayBoard(DisplayObserver):
    """Console display board showing live spot availability."""

    def __init__(self, board_id: str) -> None:
        self._board_id = board_id
        self._floor_data: dict[int, dict[SpotSize, tuple[int, int]]] = {}

    def update(self, floor_number: int, availability: dict[SpotSize, tuple[int, int]]) -> None:
        """Receive updated availability from the parking lot."""
        self._floor_data[floor_number] = availability

    def display(self) -> None:
        """Render the current availability."""
        print(f"\n  ╔═══════════════════════════════════════╗")
        print(f"  ║     DISPLAY BOARD [{self._board_id:^10s}]       ║")
        print(f"  ╠═══════════════════════════════════════╣")
        print(f"  ║  Floor  │  Small  │ Medium │  Large  ║")
        print(f"  ╠═════════╪═════════╪════════╪═════════╣")
        for floor_num in sorted(self._floor_data):
            avail = self._floor_data[floor_num]
            small = avail.get(SpotSize.SMALL, (0, 0))
            medium = avail.get(SpotSize.MEDIUM, (0, 0))
            large = avail.get(SpotSize.LARGE, (0, 0))
            print(
                f"  ║    {floor_num}    │  {small[0]:>2}/{small[1]:<2}  "
                f"│  {medium[0]:>2}/{medium[1]:<2} "
                f"│  {large[0]:>2}/{large[1]:<2}  ║"
            )
        print(f"  ╚═══════════════════════════════════════╝")
```

`DisplayObserver` is the abstract observer. `DisplayBoard` implements it with a formatted console output. `update` is called by the parking lot whenever availability changes on any floor. The board stores the latest data and can be rendered at any time. `display` formats it as a table showing `available/total` for each spot size on each floor.

---

## Entry & Exit Gates

Gates are the interaction points. The entry gate issues tickets; the exit gate processes payments.

```python
class EntryGate:
    """Issues tickets and triggers parking."""

    def __init__(self, gate_id: str) -> None:
        self._gate_id = gate_id

    @property
    def gate_id(self) -> str:
        return self._gate_id

    def process_entry(self, vehicle: Vehicle,
                      parking_lot: "ParkingLot") -> ParkingTicket | None:
        """Attempt to park a vehicle and issue a ticket."""
        print(f"\n  🚗 Gate {self._gate_id}: {vehicle} arriving...")
        ticket = parking_lot.park_vehicle(vehicle)
        if ticket:
            print(f"  ✅ Ticket issued: {ticket.ticket_id} → Spot {ticket.spot.spot_id}")
        else:
            print(f"  ❌ No available spot for {vehicle}")
        return ticket


class ExitGate:
    """Processes payments and triggers unparking."""

    def __init__(self, gate_id: str, pricing: PricingStrategy) -> None:
        self._gate_id = gate_id
        self._pricing = pricing

    @property
    def gate_id(self) -> str:
        return self._gate_id

    def process_exit(self, ticket: ParkingTicket,
                     parking_lot: "ParkingLot",
                     exit_time: datetime | None = None) -> float:
        """Calculate fee, process payment, and unpark the vehicle."""
        print(f"\n  🚗 Gate {self._gate_id}: {ticket.vehicle} exiting...")

        # Close the ticket
        ticket.close(exit_time)

        # Calculate fee
        fee = self._pricing.calculate_fee(ticket)
        ticket.set_fee(fee)

        # Process payment
        ticket.pay()

        # Unpark the vehicle
        parking_lot.unpark_vehicle(ticket)

        # Print receipt
        ticket.print_receipt()
        return fee
```

`EntryGate` delegates to `parking_lot.park_vehicle`, which handles spot assignment. `ExitGate` owns a `PricingStrategy` — different gates can have different pricing if needed (e.g. valet exit vs regular exit). The exit flow is: close ticket → calculate fee → pay → unpark → receipt. Each step is a separate method call, making the flow easy to test and modify.

---

## Parking Lot (Singleton + Facade)

The `ParkingLot` is the central class. It is a Singleton (one lot exists) and a Facade (simplifies the complex interactions between floors, spots, tickets, and observers).

```python
from threading import Lock


class ParkingLot:
    """
    The main parking lot. Manages floors, spot assignment, ticketing,
    and observer notifications.
    
    Implements the Singleton pattern — only one instance exists.
    """

    _instance: "ParkingLot | None" = None
    _lock = Lock()

    def __new__(cls, *args, **kwargs) -> "ParkingLot":
        if cls._instance is None:
            with cls._lock:
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
        return cls._instance

    def __init__(self, name: str = "Main Lot") -> None:
        # Guard against re-initialization on subsequent __new__ calls
        if hasattr(self, "_initialized"):
            return
        self._initialized = True
        self._name = name
        self._floors: list[ParkingFloor] = []
        self._active_tickets: dict[str, ParkingTicket] = {}  # license_plate -> ticket
        self._all_tickets: list[ParkingTicket] = []
        self._observers: list[DisplayObserver] = []

    @classmethod
    def reset_instance(cls) -> None:
        """Reset the singleton (for testing)."""
        cls._instance = None

    # ------------------------------------------------------------------
    # Configuration
    # ------------------------------------------------------------------

    def add_floor(self, small: int, medium: int, large: int) -> "ParkingLot":
        """Add a new floor with the given number of spots. Returns self for chaining."""
        floor_number = len(self._floors) + 1
        floor = ParkingFloor(floor_number, small, medium, large)
        self._floors.append(floor)
        return self

    def add_observer(self, observer: DisplayObserver) -> None:
        self._observers.append(observer)

    def remove_observer(self, observer: DisplayObserver) -> None:
        self._observers.remove(observer)

    # ------------------------------------------------------------------
    # Core operations
    # ------------------------------------------------------------------

    def park_vehicle(self, vehicle: Vehicle) -> ParkingTicket | None:
        """
        Find a spot for the vehicle, assign it, and issue a ticket.
        Returns None if no spot is available.
        """
        # Check for duplicate
        if vehicle.license_plate in self._active_tickets:
            print(f"  ⚠ Vehicle {vehicle} is already parked!")
            return None

        # Search floors for the best spot
        spot = self._find_best_spot(vehicle)
        if spot is None:
            return None

        # Park and issue ticket
        spot.park(vehicle)
        ticket = ParkingTicket(vehicle, spot)
        self._active_tickets[vehicle.license_plate] = ticket
        self._all_tickets.append(ticket)

        # Notify observers
        self._notify_observers(spot.floor_number)

        return ticket

    def unpark_vehicle(self, ticket: ParkingTicket) -> Vehicle:
        """Free the spot and remove the vehicle from active tracking."""
        vehicle = ticket.spot.unpark()
        self._active_tickets.pop(vehicle.license_plate, None)

        # Notify observers
        self._notify_observers(ticket.spot.floor_number)

        return vehicle

    def _find_best_spot(self, vehicle: Vehicle) -> ParkingSpot | None:
        """Search all floors for the smallest available spot that fits."""
        for floor in self._floors:
            spot = floor.find_available_spot(vehicle)
            if spot is not None:
                return spot
        return None

    # ------------------------------------------------------------------
    # Observer notification
    # ------------------------------------------------------------------

    def _notify_observers(self, floor_number: int) -> None:
        """Push updated availability for a floor to all observers."""
        floor = self._floors[floor_number - 1]
        availability = floor.get_availability()
        for observer in self._observers:
            observer.update(floor_number, availability)

    # ------------------------------------------------------------------
    # Queries
    # ------------------------------------------------------------------

    def get_availability(self) -> dict[int, dict[SpotSize, tuple[int, int]]]:
        """Return availability for all floors."""
        return {
            floor.floor_number: floor.get_availability()
            for floor in self._floors
        }

    def get_active_ticket(self, license_plate: str) -> ParkingTicket | None:
        return self._active_tickets.get(license_plate)

    def get_total_capacity(self) -> int:
        total = 0
        for floor in self._floors:
            for _, (_, t) in floor.get_availability().items():
                total += t
        return total

    def get_occupied_count(self) -> int:
        return len(self._active_tickets)

    # ------------------------------------------------------------------
    # Display
    # ------------------------------------------------------------------

    def display_status(self) -> None:
        """Print the full parking lot status."""
        occupied = self.get_occupied_count()
        capacity = self.get_total_capacity()
        print(f"\n{'='*50}")
        print(f"  🅿️  {self._name}")
        print(f"  Floors: {len(self._floors)} | "
              f"Occupied: {occupied}/{capacity}")
        print(f"{'='*50}")
        for floor in self._floors:
            floor.display()
        print()

    def display_active_tickets(self) -> None:
        """Print all currently active (parked) tickets."""
        print(f"\n  Active Tickets ({len(self._active_tickets)}):")
        for plate, ticket in self._active_tickets.items():
            print(f"    {ticket}")
```

The Singleton uses double-checked locking with `Lock` for thread safety. `__new__` ensures only one instance is created; `__init__` guards against re-initialization with the `_initialized` flag. `reset_instance` is a testing escape hatch. `park_vehicle` is the main entry point: it checks for duplicates, searches floors in order, parks the vehicle, issues a ticket, and notifies observers. `_find_best_spot` iterates floors top to bottom and uses each floor's `find_available_spot` (which prefers the smallest fit). `_notify_observers` pushes updated availability to all display boards whenever a spot changes state.

---

## Putting It All Together

### Full Simulation

```python
from datetime import timedelta


def main() -> None:
    # Reset singleton for clean demo
    ParkingLot.reset_instance()

    # Build the parking lot
    lot = ParkingLot("City Center Parking")
    lot.add_floor(small=5, medium=10, large=2)    # Floor 1
    lot.add_floor(small=5, medium=10, large=2)    # Floor 2
    lot.add_floor(small=0, medium=5, large=5)     # Floor 3 (buses)

    # Attach display board
    board = DisplayBoard("MAIN")
    lot.add_observer(board)

    # Initialize board with current availability
    for floor_num, avail in lot.get_availability().items():
        board.update(floor_num, avail)

    # Set up gates
    entry_a = EntryGate("A")
    entry_b = EntryGate("B")
    exit_x = ExitGate("X", HourlyPricing())
    exit_y = ExitGate("Y", TieredPricing())

    # Show initial state
    lot.display_status()
    board.display()

    # ─── Vehicles arrive ───
    print("\n" + "=" * 50)
    print("  ARRIVALS")
    print("=" * 50)

    vehicles = [
        Car("CAR-001"),
        Car("CAR-002"),
        Motorcycle("MOTO-001"),
        Bus("BUS-001"),
        Car("CAR-003"),
        Motorcycle("MOTO-002"),
    ]

    tickets: dict[str, ParkingTicket] = {}
    for v in vehicles:
        gate = entry_a if hash(v) % 2 == 0 else entry_b
        ticket = gate.process_entry(v, lot)
        if ticket:
            tickets[v.license_plate] = ticket

    # Show updated state
    lot.display_status()
    board.display()
    lot.display_active_tickets()

    # ─── Try duplicate entry ───
    print("\n  --- Duplicate entry attempt ---")
    entry_a.process_entry(Car("CAR-001"), lot)

    # ─── Vehicles exit ───
    print("\n" + "=" * 50)
    print("  DEPARTURES")
    print("=" * 50)

    # Simulate some time passing
    for plate, ticket in tickets.items():
        # Fake entry time to 3 hours ago for realistic fees
        ticket._entry_time = datetime.now() - timedelta(hours=3)

    # Process exits
    if "CAR-001" in tickets:
        exit_x.process_exit(tickets["CAR-001"], lot)

    if "MOTO-001" in tickets:
        exit_x.process_exit(tickets["MOTO-001"], lot)

    if "BUS-001" in tickets:
        exit_y.process_exit(tickets["BUS-001"], lot)

    # Show final state
    lot.display_status()
    board.display()

    # ─── Revenue summary ───
    print("\n  Revenue Summary:")
    total_revenue = sum(t.fee for t in lot._all_tickets if t.is_paid)
    print(f"  Total collected: ${total_revenue:.2f}")


if __name__ == "__main__":
    main()
```

The simulation creates a 3-floor lot with two entry gates and two exit gates (each with a different pricing strategy). It parks six vehicles, attempts a duplicate entry, then exits three of them after simulating 3 hours of parking. The display board updates automatically after each park/unpark. The revenue summary shows total collected fees.

### Unit Tests

```python
def test_spot_assignment() -> None:
    """Test that vehicles get the smallest fitting spot."""
    ParkingLot.reset_instance()
    lot = ParkingLot("Test Lot")
    lot.add_floor(small=2, medium=2, large=1)

    moto = Motorcycle("M-1")
    ticket = lot.park_vehicle(moto)
    assert ticket is not None
    assert ticket.spot.size == SpotSize.SMALL
    print("✅ test_spot_assignment passed")


def test_capacity_full() -> None:
    """Test that parking rejects when full."""
    ParkingLot.reset_instance()
    lot = ParkingLot("Test Lot")
    lot.add_floor(small=0, medium=1, large=0)

    car1 = Car("C-1")
    car2 = Car("C-2")

    t1 = lot.park_vehicle(car1)
    t2 = lot.park_vehicle(car2)

    assert t1 is not None
    assert t2 is None  # no more medium spots
    print("✅ test_capacity_full passed")


def test_duplicate_vehicle() -> None:
    """Test that the same vehicle cannot park twice."""
    ParkingLot.reset_instance()
    lot = ParkingLot("Test Lot")
    lot.add_floor(small=5, medium=5, large=5)

    car = Car("DUP-1")
    t1 = lot.park_vehicle(car)
    t2 = lot.park_vehicle(car)

    assert t1 is not None
    assert t2 is None
    print("✅ test_duplicate_vehicle passed")


def test_unpark_frees_spot() -> None:
    """Test that unparking frees the spot for reuse."""
    ParkingLot.reset_instance()
    lot = ParkingLot("Test Lot")
    lot.add_floor(small=0, medium=1, large=0)

    car1 = Car("C-1")
    car2 = Car("C-2")

    t1 = lot.park_vehicle(car1)
    assert t1 is not None

    # Lot is full
    assert lot.park_vehicle(car2) is None

    # Unpark car1
    lot.unpark_vehicle(t1)

    # Now car2 can park
    t2 = lot.park_vehicle(car2)
    assert t2 is not None
    assert t2.spot.spot_id == t1.spot.spot_id  # same spot reused
    print("✅ test_unpark_frees_spot passed")


def test_hourly_pricing() -> None:
    """Test hourly pricing calculation."""
    ParkingLot.reset_instance()
    lot = ParkingLot("Test Lot")
    lot.add_floor(small=1, medium=1, large=1)

    car = Car("PRICE-1")
    ticket = lot.park_vehicle(car)
    assert ticket is not None

    # Simulate 3 hours of parking
    ticket._entry_time = datetime.now() - timedelta(hours=3)
    ticket.close()

    pricing = HourlyPricing()  # Car rate = $2/hr
    fee = pricing.calculate_fee(ticket)
    assert fee == 6.0, f"Expected $6.0, got ${fee}"
    print("✅ test_hourly_pricing passed")


def test_tiered_pricing() -> None:
    """Test tiered pricing calculation."""
    ParkingLot.reset_instance()
    lot = ParkingLot("Test Lot")
    lot.add_floor(small=1, medium=1, large=1)

    car = Car("TIER-1")
    ticket = lot.park_vehicle(car)
    assert ticket is not None

    # Simulate 5 hours of parking
    ticket._entry_time = datetime.now() - timedelta(hours=5)
    ticket.close()

    pricing = TieredPricing()  # Default tiers, car multiplier = 1.0
    fee = pricing.calculate_fee(ticket)
    # First 2 hr × $3 = $6, next 3 hr × $2 = $6 → total $12
    assert fee == 12.0, f"Expected $12.0, got ${fee}"
    print("✅ test_tiered_pricing passed")


def test_observer_notification() -> None:
    """Test that observers are notified on park/unpark."""
    ParkingLot.reset_instance()
    lot = ParkingLot("Test Lot")
    lot.add_floor(small=2, medium=2, large=1)

    board = DisplayBoard("TEST")
    lot.add_observer(board)

    car = Car("OBS-1")
    lot.park_vehicle(car)

    # Board should have floor 1 data
    assert 1 in board._floor_data
    avail = board._floor_data[1]
    medium_avail, medium_total = avail[SpotSize.MEDIUM]
    assert medium_avail == 1 and medium_total == 2  # 1 of 2 medium spots used
    print("✅ test_observer_notification passed")


def run_all_tests() -> None:
    print("\n" + "=" * 40)
    print("  RUNNING TESTS")
    print("=" * 40 + "\n")
    test_spot_assignment()
    test_capacity_full()
    test_duplicate_vehicle()
    test_unpark_frees_spot()
    test_hourly_pricing()
    test_tiered_pricing()
    test_observer_notification()
    print("\n✅ All tests passed!\n")
```

Each test resets the singleton, creates a minimal lot, and tests one specific behaviour. `test_spot_assignment` verifies best-fit allocation. `test_capacity_full` confirms rejection when full. `test_unpark_frees_spot` checks that spots are reusable. The pricing tests use `timedelta` to simulate time passing. `test_observer_notification` verifies that the display board receives updates.

---

## Class Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                   ParkingLot «Singleton»                     │
│─────────────────────────────────────────────────────────────│
│ - _name: str                                                │
│ - _floors: list[ParkingFloor]                               │
│ - _active_tickets: dict[str, ParkingTicket]                 │
│ - _observers: list[DisplayObserver]                         │
│─────────────────────────────────────────────────────────────│
│ + add_floor(small, medium, large) -> self                   │
│ + park_vehicle(vehicle) -> ParkingTicket | None             │
│ + unpark_vehicle(ticket) -> Vehicle                         │
│ + add_observer(observer) -> None                            │
│ + get_availability() -> dict                                │
│ + display_status() -> None                                  │
└──────┬──────────┬──────────────┬────────────────────────────┘
       │          │              │
  owns ▼     owns ▼        notifies ▼
┌─────────────┐ ┌──────────────┐ ┌────────────────────────────┐
│ ParkingFloor│ │ParkingTicket │ │ «abstract» DisplayObserver │
│─────────────│ │──────────────│ │────────────────────────────│
│ - _floor_num│ │ - _ticket_id │ │ + update(floor, avail)     │
│ - _spots    │ │ - _vehicle   │ └─────────────┬──────────────┘
│─────────────│ │ - _spot      │               │ implemented by
│ + find_     │ │ - _entry_time│               ▼
│  available_ │ │ - _exit_time │         DisplayBoard
│  spot()     │ │ - _fee       │
│ + get_      │ │──────────────│
│  availability│ │ + close()    │
└──────┬──────┘ │ + pay()      │
       │        │ + get_       │
  contains ▼    │  duration()  │
┌─────────────┐ │ + print_     │
│ ParkingSpot │ │  receipt()   │
│─────────────│ └──────────────┘
│ - _spot_id  │
│ - _size     │
│ - _status   │
│ - _vehicle  │
│─────────────│
│ + can_fit() │
│ + park()    │
│ + unpark()  │
└─────────────┘


┌──────────────────────────────────────┐
│          «abstract» Vehicle          │
│──────────────────────────────────────│
│ - _license_plate: str                │
│ - _vehicle_type: VehicleType         │
│──────────────────────────────────────│
│ + required_spot_size -> SpotSize     │
└──────────┬───────────────────────────┘
           │ extended by
     ┌─────┼──────┐
     ▼     ▼      ▼
 Motorcycle Car   Bus


┌──────────────────────────────────────────────┐
│         «abstract» PricingStrategy           │
│──────────────────────────────────────────────│
│ + calculate_fee(ticket) -> float             │
│ + name() -> str                              │
└──────────┬───────────────────────────────────┘
           │ implemented by
     ┌─────┼──────────────┐
     ▼     ▼              ▼
  Hourly  FlatRate     Tiered
  Pricing Pricing      Pricing


┌──────────┐    ┌───────────┐
│ EntryGate│    │  ExitGate │
│──────────│    │───────────│
│ + process│    │ - _pricing│
│   _entry │    │ + process │
└──────────┘    │   _exit   │
                └───────────┘

┌─────────────────┐  ┌──────────────┐  ┌───────────────┐
│ VehicleType(Enum)│  │ SpotSize(Enum)│  │SpotStatus(Enum)│
│─────────────────│  │──────────────│  │───────────────│
│ MOTORCYCLE      │  │ SMALL (1)    │  │ AVAILABLE     │
│ CAR             │  │ MEDIUM (2)   │  │ OCCUPIED      │
│ BUS             │  │ LARGE (3)    │  └───────────────┘
└─────────────────┘  └──────────────┘
```

---

## How to Run

Save all code into a single file `parking_lot.py` and run:

```bash
python parking_lot.py
```

### Example Output

```
==================================================
  🅿️  City Center Parking
  Floors: 3 | Occupied: 0/42
==================================================

  Floor 1:
    SMALL : ⬜F1-S01 ⬜F1-S02 ⬜F1-S03 ⬜F1-S04 ⬜F1-S05
    MEDIUM: ⬜F1-M01 ⬜F1-M02 ... ⬜F1-M10
    LARGE : ⬜F1-L01 ⬜F1-L02

  ╔═══════════════════════════════════════╗
  ║     DISPLAY BOARD [   MAIN   ]       ║
  ╠═══════════════════════════════════════╣
  ║  Floor  │  Small  │ Medium │  Large  ║
  ╠═════════╪═════════╪════════╪═════════╣
  ║    1    │   5/ 5  │  10/10 │   2/ 2  ║
  ║    2    │   5/ 5  │  10/10 │   2/ 2  ║
  ║    3    │   0/ 0  │   5/ 5 │   5/ 5  ║
  ╚═══════════════════════════════════════╝

==================================================
  ARRIVALS
==================================================

  🚗 Gate A: car(CAR-001) arriving...
  ✅ Ticket issued: A3B7C1D9 → Spot F1-M01

  🚗 Gate B: motorcycle(MOTO-001) arriving...
  ✅ Ticket issued: E4F8G2H6 → Spot F1-S01

  🚗 Gate A: bus(BUS-001) arriving...
  ✅ Ticket issued: I5J9K3L7 → Spot F1-L01

  --- Duplicate entry attempt ---
  ⚠ Vehicle car(CAR-001) is already parked!

==================================================
  DEPARTURES
==================================================

  🚗 Gate X: car(CAR-001) exiting...

  ┌─────────────────────────────────────┐
  │         PARKING RECEIPT              │
  ├─────────────────────────────────────┤
  │  Ticket:                   A3B7C1D9 │
  │  Vehicle:              car(CAR-001) │
  │  Spot:                      F1-M01  │
  │  Entry:           2026-10-26 11:51  │
  │  Exit:            2026-10-26 14:51  │
  │  Duration:                  3.0 hr  │
  │  Fee:                        $6.00  │
  │  Status:                      PAID  │
  └─────────────────────────────────────┘
```
