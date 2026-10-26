# Design Ride-Sharing System – Low-Level Design Guide

![Bidur Sapkota](https://www.bidursapkota.com.np/images/gravatar.webp "Bidur Sapkota - Developer")&nbsp;[Bidur Sapkota](https://www.bidursapkota.com.np/)

## Table of Contents

1. [Requirements](#requirements)
2. [Key Design Patterns](#key-design-patterns)
3. [Core Enums & Constants](#core-enums--constants)
4. [Location & Geography](#location--geography)
5. [Vehicle & Vehicle Types](#vehicle--vehicle-types)
6. [User, Rider & Driver](#user-rider--driver)
7. [Pricing Engine (Strategy Pattern)](#pricing-engine-strategy-pattern)
8. [Driver Matching (Strategy Pattern)](#driver-matching-strategy-pattern)
9. [Trip (State Pattern)](#trip-state-pattern)
10. [Payment Processing (Strategy Pattern)](#payment-processing-strategy-pattern)
11. [Rating System](#rating-system)
12. [Notification System (Observer Pattern)](#notification-system-observer-pattern)
13. [Trip History & Analytics](#trip-history--analytics)
14. [Ride-Sharing System (Facade)](#ride-sharing-system-facade)
15. [Simulation Engine](#simulation-engine)
16. [Putting It All Together](#putting-it-all-together)
17. [Class Diagram](#class-diagram)
18. [How to Run](#how-to-run)

---

## Requirements

### Functional Requirements

| #  | Requirement                                                                              |
| -- | ---------------------------------------------------------------------------------------- |
| 1  | Riders can request a ride by specifying pickup location, destination, and vehicle type    |
| 2  | System estimates fare before the ride is confirmed (based on distance, time, surge)       |
| 3  | Multiple vehicle types supported (economy, comfort, premium, XL)                         |
| 4  | Drivers are matched to ride requests using a pluggable dispatch strategy                  |
| 5  | Drivers can accept or decline ride requests; declined requests are re-dispatched          |
| 6  | Trip lifecycle: REQUESTED → DRIVER_ASSIGNED → EN_ROUTE → ARRIVED → IN_PROGRESS → COMPLETED (or CANCELLED) |
| 7  | Surge pricing — fare multiplier increases when demand exceeds supply                     |
| 8  | Real-time trip tracking with estimated time of arrival (ETA)                              |
| 9  | Multiple payment methods (credit card, wallet, cash)                                     |
| 10 | Two-way rating system — riders rate drivers and drivers rate riders after completion      |
| 11 | Observer-based notifications — riders and drivers are notified on trip state changes      |
| 12 | Trip history and analytics maintained for riders, drivers, and the platform               |

### Non-Functional Requirements

- Clean OOP with SOLID principles.
- Extensible — new vehicle types, pricing models, matching algorithms, and payment methods can be added without modifying core logic.
- Simulatable — the system runs step-by-step for testing and demonstration.

---

## Key Design Patterns

| Pattern  | Where It Is Used                                     | Why                                                                                        |
| -------- | ---------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| State    | `Trip` transitions through lifecycle states          | Each state encapsulates valid transitions and actions; adding a new state is isolated       |
| Strategy | `PricingStrategy` computes fares                     | Swap between standard, surge, subscription, or promo pricing without changing core logic    |
| Strategy | `MatchingStrategy` selects drivers for ride requests | Switch between nearest, highest-rated, or load-balanced dispatch at runtime                |
| Strategy | `PaymentStrategy` processes payments                 | Add new payment methods without modifying the checkout flow                                |
| Observer | `TripEventListener` observes trip state changes      | Riders, drivers, loggers, and analytics react independently to the same events              |
| Facade   | `RideSharingSystem` exposes a simple API             | Hides the complexity of matching, pricing, trips, payments, and notifications               |

---

## Core Enums & Constants

```python
from enum import Enum


class TripStatus(Enum):
    REQUESTED = "requested"
    DRIVER_ASSIGNED = "driver_assigned"
    EN_ROUTE = "en_route"          # driver heading to pickup
    ARRIVED = "arrived"            # driver at pickup location
    IN_PROGRESS = "in_progress"    # rider in car, driving to destination
    COMPLETED = "completed"
    CANCELLED = "cancelled"


class VehicleType(Enum):
    ECONOMY = "economy"
    COMFORT = "comfort"
    PREMIUM = "premium"
    XL = "xl"


class DriverStatus(Enum):
    AVAILABLE = "available"
    BUSY = "busy"
    OFFLINE = "offline"


class PaymentMethod(Enum):
    CREDIT_CARD = "credit_card"
    DEBIT_CARD = "debit_card"
    WALLET = "wallet"
    CASH = "cash"


class PaymentStatus(Enum):
    PENDING = "pending"
    COMPLETED = "completed"
    FAILED = "failed"
    REFUNDED = "refunded"


# ------------------------------------------------------------------
# Pricing constants
# ------------------------------------------------------------------

BASE_FARE: dict[VehicleType, float] = {
    VehicleType.ECONOMY: 2.50,
    VehicleType.COMFORT: 4.00,
    VehicleType.PREMIUM: 7.00,
    VehicleType.XL: 5.50,
}

PER_KM_RATE: dict[VehicleType, float] = {
    VehicleType.ECONOMY: 1.20,
    VehicleType.COMFORT: 1.80,
    VehicleType.PREMIUM: 3.00,
    VehicleType.XL: 2.20,
}

PER_MINUTE_RATE: dict[VehicleType, float] = {
    VehicleType.ECONOMY: 0.15,
    VehicleType.COMFORT: 0.25,
    VehicleType.PREMIUM: 0.50,
    VehicleType.XL: 0.30,
}

MINIMUM_FARE: dict[VehicleType, float] = {
    VehicleType.ECONOMY: 5.00,
    VehicleType.COMFORT: 8.00,
    VehicleType.PREMIUM: 15.00,
    VehicleType.XL: 10.00,
}

BOOKING_FEE = 1.50                  # flat fee on every trip
PLATFORM_FEE_RATE = 0.20            # 20% platform commission
CANCELLATION_FEE = 5.00             # fee for late cancellation
MAX_PICKUP_RADIUS_KM = 10.0         # max distance to find drivers
SURGE_THRESHOLD = 0.7               # surge triggers when demand/supply > 0.7
AVERAGE_SPEED_KPH = 30.0            # for ETA estimation
```

`TripStatus` models the full lifecycle of a ride — from the moment a rider requests it through driver assignment, pickup, the ride itself, and completion. `VehicleType` distinguishes vehicle tiers with different pricing. The pricing constants use per-vehicle-type dictionaries so each tier has its own base fare, distance rate, time rate, and minimum fare. `PLATFORM_FEE_RATE` at 20% is what the platform retains from each trip — the rest goes to the driver. `SURGE_THRESHOLD` triggers dynamic pricing when the ratio of active requests to available drivers exceeds 0.7.

---

## Location & Geography

```python
from dataclasses import dataclass
import math


@dataclass(frozen=True)
class Location:
    """A geographic coordinate with distance computation."""
    latitude: float
    longitude: float

    def distance_to(self, other: "Location") -> float:
        """Approximate distance in km using the Haversine formula."""
        R = 6371  # Earth radius in km
        lat1, lat2 = math.radians(self.latitude), math.radians(other.latitude)
        dlat = math.radians(other.latitude - self.latitude)
        dlon = math.radians(other.longitude - self.longitude)
        a = (math.sin(dlat / 2) ** 2 +
             math.cos(lat1) * math.cos(lat2) * math.sin(dlon / 2) ** 2)
        return R * 2 * math.atan2(math.sqrt(a), math.sqrt(1 - a))

    def estimated_travel_time(self, other: "Location") -> float:
        """Estimated travel time in minutes at average speed."""
        dist = self.distance_to(other)
        return (dist / AVERAGE_SPEED_KPH) * 60

    def __repr__(self) -> str:
        return f"({self.latitude:.4f}, {self.longitude:.4f})"


@dataclass(frozen=True)
class Route:
    """A route between two locations with computed metrics."""
    pickup: Location
    destination: Location

    @property
    def distance_km(self) -> float:
        return self.pickup.distance_to(self.destination)

    @property
    def estimated_minutes(self) -> float:
        return self.pickup.estimated_travel_time(self.destination)

    def __repr__(self) -> str:
        return (
            f"Route({self.pickup} → {self.destination} "
            f"{self.distance_km:.1f}km ~{self.estimated_minutes:.0f}min)"
        )
```

`Location` uses the Haversine formula to compute real-world distances between GPS coordinates — this feeds into fare calculation, ETA estimation, and driver matching. `estimated_travel_time` divides distance by an average speed constant to get a rough ETA in minutes. `Route` bundles a pickup-destination pair with precomputed distance and time, which simplifies fare calculation and trip display. Both are frozen dataclasses — routes and coordinates are immutable facts about the trip.

---

## Vehicle & Vehicle Types

```python
@dataclass
class Vehicle:
    """A driver's vehicle with type, capacity, and identification."""
    vehicle_id: str
    vehicle_type: VehicleType
    make: str          # e.g., "Toyota"
    model: str         # e.g., "Camry"
    year: int
    license_plate: str
    color: str
    capacity: int      # max passengers

    def __repr__(self) -> str:
        return (
            f"{self.color} {self.year} {self.make} {self.model} "
            f"({self.vehicle_type.value}) [{self.license_plate}]"
        )
```

`Vehicle` is a data container that holds everything a rider sees when a driver is assigned — make, model, color, license plate. `vehicle_type` determines which ride requests this vehicle can serve and which pricing tier applies. `capacity` limits the number of passengers for XL requests.

---

## User, Rider & Driver

```python
class User:
    """Base class for all users in the ride-sharing platform."""

    def __init__(self, user_id: str, name: str, phone: str,
                 email: str) -> None:
        self._user_id = user_id
        self._name = name
        self._phone = phone
        self._email = email

    @property
    def user_id(self) -> str:
        return self._user_id

    @property
    def name(self) -> str:
        return self._name

    @property
    def phone(self) -> str:
        return self._phone

    @property
    def email(self) -> str:
        return self._email


class Rider(User):
    """A rider who requests and takes trips."""

    def __init__(self, user_id: str, name: str, phone: str,
                 email: str, location: Location,
                 home_address: str = "",
                 work_address: str = "") -> None:
        super().__init__(user_id, name, phone, email)
        self._location = location
        self._home_address = home_address
        self._work_address = work_address
        self._trip_history: list[str] = []   # trip IDs
        self._rating: float = 5.0
        self._total_ratings: int = 0
        self._wallet_balance: float = 0.0

    # ------------------------------------------------------------------
    # Properties
    # ------------------------------------------------------------------

    @property
    def location(self) -> Location:
        return self._location

    @location.setter
    def location(self, loc: Location) -> None:
        self._location = loc

    @property
    def home_address(self) -> str:
        return self._home_address

    @property
    def work_address(self) -> str:
        return self._work_address

    @property
    def trip_history(self) -> list[str]:
        return list(self._trip_history)

    @property
    def rating(self) -> float:
        return self._rating

    @property
    def wallet_balance(self) -> float:
        return self._wallet_balance

    # ------------------------------------------------------------------
    # Operations
    # ------------------------------------------------------------------

    def add_to_history(self, trip_id: str) -> None:
        self._trip_history.append(trip_id)

    def add_rating(self, score: float) -> None:
        score = max(1.0, min(5.0, score))
        self._total_ratings += 1
        self._rating += (score - self._rating) / self._total_ratings

    def top_up_wallet(self, amount: float) -> None:
        self._wallet_balance += amount

    def debit_wallet(self, amount: float) -> bool:
        if amount > self._wallet_balance:
            return False
        self._wallet_balance -= amount
        return True

    def credit_wallet(self, amount: float) -> None:
        self._wallet_balance += amount

    def __repr__(self) -> str:
        return (
            f"Rider({self._name} ⭐{self._rating:.1f} "
            f"Trips: {len(self._trip_history)} "
            f"Wallet: ${self._wallet_balance:.2f})"
        )


class Driver(User):
    """A driver who accepts and completes trips."""

    def __init__(self, user_id: str, name: str, phone: str,
                 email: str, location: Location,
                 vehicle: Vehicle) -> None:
        super().__init__(user_id, name, phone, email)
        self._location = location
        self._vehicle = vehicle
        self._status = DriverStatus.AVAILABLE
        self._current_trip_id: str | None = None
        self._trip_history: list[str] = []
        self._rating: float = 4.8
        self._total_ratings: int = 0
        self._total_earnings: float = 0.0
        self._total_distance_km: float = 0.0

    # ------------------------------------------------------------------
    # Properties
    # ------------------------------------------------------------------

    @property
    def location(self) -> Location:
        return self._location

    @location.setter
    def location(self, loc: Location) -> None:
        self._location = loc

    @property
    def vehicle(self) -> Vehicle:
        return self._vehicle

    @property
    def status(self) -> DriverStatus:
        return self._status

    @property
    def current_trip_id(self) -> str | None:
        return self._current_trip_id

    @property
    def trip_history(self) -> list[str]:
        return list(self._trip_history)

    @property
    def rating(self) -> float:
        return self._rating

    @property
    def total_earnings(self) -> float:
        return self._total_earnings

    @property
    def total_distance_km(self) -> float:
        return self._total_distance_km

    @property
    def vehicle_type(self) -> VehicleType:
        return self._vehicle.vehicle_type

    # ------------------------------------------------------------------
    # Operations
    # ------------------------------------------------------------------

    def assign_trip(self, trip_id: str) -> None:
        self._current_trip_id = trip_id
        self._status = DriverStatus.BUSY

    def complete_trip(self, earnings: float, distance_km: float) -> None:
        self._trip_history.append(self._current_trip_id)
        self._current_trip_id = None
        self._status = DriverStatus.AVAILABLE
        self._total_earnings += earnings
        self._total_distance_km += distance_km

    def release_trip(self) -> None:
        """Release current trip (declined or cancelled)."""
        self._current_trip_id = None
        self._status = DriverStatus.AVAILABLE

    def go_offline(self) -> None:
        self._status = DriverStatus.OFFLINE

    def go_online(self) -> None:
        if self._current_trip_id is None:
            self._status = DriverStatus.AVAILABLE

    def add_rating(self, score: float) -> None:
        score = max(1.0, min(5.0, score))
        self._total_ratings += 1
        self._rating += (score - self._rating) / self._total_ratings

    def __repr__(self) -> str:
        status_emoji = {
            DriverStatus.AVAILABLE: "🟢",
            DriverStatus.BUSY: "🔴",
            DriverStatus.OFFLINE: "⚫",
        }
        return (
            f"Driver({self._name} {status_emoji[self._status]} "
            f"⭐{self._rating:.1f} 🚗{self._vehicle.vehicle_type.value} "
            f"Trips: {len(self._trip_history)} "
            f"💰${self._total_earnings:.2f})"
        )
```

`User` provides shared identity fields. `Rider` adds a wallet for in-app payments, a trip history, and a rating — riders are also rated in most platforms, which helps drivers decide whether to accept a request. `Driver` tracks status (`AVAILABLE`, `BUSY`, `OFFLINE`), current trip assignment, cumulative earnings, and total distance driven. `complete_trip` credits earnings and updates stats atomically. `add_rating` uses the incremental mean formula for both riders and drivers — this avoids storing every individual rating while maintaining an accurate running average.

---

## Pricing Engine (Strategy Pattern)

The pricing engine computes fare estimates before a ride and final fares after completion. Different strategies handle standard pricing, surge pricing, and promotional discounts.

### Fare Breakdown

```python
@dataclass
class FareBreakdown:
    """Detailed breakdown of a trip fare."""
    base_fare: float
    distance_fare: float
    time_fare: float
    booking_fee: float
    surge_multiplier: float
    surge_amount: float
    discount: float
    subtotal: float
    total: float
    driver_payout: float     # what the driver receives
    platform_fee: float      # what the platform keeps

    def __repr__(self) -> str:
        surge_str = (f" (×{self.surge_multiplier:.1f} surge)"
                     if self.surge_multiplier > 1.0 else "")
        discount_str = (f" -${self.discount:.2f} discount"
                        if self.discount > 0 else "")
        return (
            f"Fare: ${self.total:.2f}{surge_str}{discount_str} "
            f"(Driver: ${self.driver_payout:.2f})"
        )

    def display(self) -> None:
        print(f"\n  {'─'*45}")
        print(f"  FARE BREAKDOWN")
        print(f"  {'─'*45}")
        print(f"  Base fare:       ${self.base_fare:.2f}")
        print(f"  Distance fare:   ${self.distance_fare:.2f}")
        print(f"  Time fare:       ${self.time_fare:.2f}")
        print(f"  Booking fee:     ${self.booking_fee:.2f}")
        if self.surge_multiplier > 1.0:
            print(f"  Surge (×{self.surge_multiplier:.1f}):    "
                  f"+${self.surge_amount:.2f}")
        if self.discount > 0:
            print(f"  Discount:       -${self.discount:.2f}")
        print(f"  {'─'*45}")
        print(f"  TOTAL:           ${self.total:.2f}")
        print(f"  Driver payout:   ${self.driver_payout:.2f}")
        print(f"  Platform fee:    ${self.platform_fee:.2f}")
        print(f"  {'─'*45}\n")
```

### Abstract Strategy

```python
from abc import ABC, abstractmethod


class PricingStrategy(ABC):
    """Defines how ride fares are calculated."""

    @abstractmethod
    def calculate_fare(self, route: Route,
                       vehicle_type: VehicleType,
                       surge_multiplier: float = 1.0) -> FareBreakdown:
        """Calculate fare for a trip."""
        ...

    @abstractmethod
    def name(self) -> str:
        ...
```

### Standard Pricing

```python
class StandardPricing(PricingStrategy):
    """
    Standard fare calculation:
    fare = base + (distance × per_km) + (time × per_min)
    Then apply surge multiplier, booking fee, and minimum fare.
    """

    def calculate_fare(self, route: Route,
                       vehicle_type: VehicleType,
                       surge_multiplier: float = 1.0) -> FareBreakdown:
        base = BASE_FARE[vehicle_type]
        distance_fare = route.distance_km * PER_KM_RATE[vehicle_type]
        time_fare = route.estimated_minutes * PER_MINUTE_RATE[vehicle_type]

        subtotal = base + distance_fare + time_fare
        minimum = MINIMUM_FARE[vehicle_type]
        subtotal = max(subtotal, minimum)

        # Apply surge
        surge_amount = 0.0
        if surge_multiplier > 1.0:
            surge_amount = subtotal * (surge_multiplier - 1.0)

        total_before_fee = subtotal + surge_amount
        total = total_before_fee + BOOKING_FEE

        # Platform takes its cut from pre-fee amount
        platform_fee = total_before_fee * PLATFORM_FEE_RATE
        driver_payout = total_before_fee - platform_fee

        return FareBreakdown(
            base_fare=base,
            distance_fare=distance_fare,
            time_fare=time_fare,
            booking_fee=BOOKING_FEE,
            surge_multiplier=surge_multiplier,
            surge_amount=surge_amount,
            discount=0.0,
            subtotal=subtotal,
            total=total,
            driver_payout=driver_payout,
            platform_fee=platform_fee,
        )

    def name(self) -> str:
        return "Standard Pricing"
```

### Promotional Pricing

```python
class PromotionalPricing(PricingStrategy):
    """
    Applies a percentage discount on the standard fare.
    Used for first-ride promos, loyalty discounts, or campaigns.
    """

    def __init__(self, discount_percent: float = 0.20,
                 max_discount: float = 10.00) -> None:
        self._discount_percent = discount_percent
        self._max_discount = max_discount

    def calculate_fare(self, route: Route,
                       vehicle_type: VehicleType,
                       surge_multiplier: float = 1.0) -> FareBreakdown:
        # Start with standard calculation
        base = BASE_FARE[vehicle_type]
        distance_fare = route.distance_km * PER_KM_RATE[vehicle_type]
        time_fare = route.estimated_minutes * PER_MINUTE_RATE[vehicle_type]

        subtotal = base + distance_fare + time_fare
        minimum = MINIMUM_FARE[vehicle_type]
        subtotal = max(subtotal, minimum)

        surge_amount = 0.0
        if surge_multiplier > 1.0:
            surge_amount = subtotal * (surge_multiplier - 1.0)

        total_before_discount = subtotal + surge_amount

        # Apply discount (capped at max)
        discount = min(
            total_before_discount * self._discount_percent,
            self._max_discount
        )
        total_after_discount = total_before_discount - discount
        total = total_after_discount + BOOKING_FEE

        # Platform absorbs the discount cost
        platform_fee = total_after_discount * PLATFORM_FEE_RATE
        driver_payout = total_before_discount * (1 - PLATFORM_FEE_RATE)

        return FareBreakdown(
            base_fare=base,
            distance_fare=distance_fare,
            time_fare=time_fare,
            booking_fee=BOOKING_FEE,
            surge_multiplier=surge_multiplier,
            surge_amount=surge_amount,
            discount=discount,
            subtotal=subtotal,
            total=total,
            driver_payout=driver_payout,
            platform_fee=platform_fee - discount,
        )

    def name(self) -> str:
        return f"Promo ({self._discount_percent*100:.0f}% off)"
```

### Surge Calculator

```python
class SurgeCalculator:
    """
    Computes the surge multiplier based on supply and demand.

    Surge = 1.0 when demand/supply ratio is below threshold.
    Above threshold, surge grows linearly up to a configurable cap.
    """

    def __init__(self, max_surge: float = 3.0) -> None:
        self._max_surge = max_surge

    def compute_surge(self, active_requests: int,
                      available_drivers: int) -> float:
        if available_drivers == 0:
            return self._max_surge

        ratio = active_requests / available_drivers

        if ratio <= SURGE_THRESHOLD:
            return 1.0

        # Linear surge between threshold and max
        surge = 1.0 + (ratio - SURGE_THRESHOLD) * 2.0
        return min(surge, self._max_surge)

    def __repr__(self) -> str:
        return f"SurgeCalculator(max={self._max_surge}×)"
```

`FareBreakdown` gives full transparency — riders see exactly what they are paying for, and the system tracks what goes to the driver versus the platform. `StandardPricing` follows the formula used by most ride-sharing apps: base fare + distance rate + time rate, floored at the minimum fare for that vehicle type. The surge multiplier scales the pre-booking-fee subtotal — a 1.5× surge on a $20 fare adds $10. `PromotionalPricing` applies a capped percentage discount and importantly, the driver payout stays unchanged — the platform absorbs the discount cost, which is how real promotional pricing works. `SurgeCalculator` models dynamic pricing: when demand (active ride requests) exceeds supply (available drivers) beyond the threshold, surge grows linearly up to a configurable cap.

---

## Driver Matching (Strategy Pattern)

Different matching algorithms optimise for different goals — minimise pickup time, maximise rider satisfaction, or balance driver workload.

### Abstract Strategy

```python
class MatchingStrategy(ABC):
    """Defines how a driver is selected for a ride request."""

    @abstractmethod
    def find_driver(self, rider: Rider, pickup: Location,
                    vehicle_type: VehicleType,
                    drivers: list[Driver]) -> Driver | None:
        """
        Select the best available driver for the request.
        Returns None if no suitable driver is found.
        """
        ...

    @abstractmethod
    def name(self) -> str:
        ...
```

### Nearest Driver Strategy

```python
class NearestDriverStrategy(MatchingStrategy):
    """
    Assign the closest available driver with the correct vehicle type.

    Minimises pickup time, which is the primary factor in rider
    satisfaction according to most ride-sharing studies.
    """

    def find_driver(self, rider: Rider, pickup: Location,
                    vehicle_type: VehicleType,
                    drivers: list[Driver]) -> Driver | None:
        compatible = [
            d for d in drivers
            if (d.status == DriverStatus.AVAILABLE
                and d.vehicle_type == vehicle_type
                and d.location.distance_to(pickup) <= MAX_PICKUP_RADIUS_KM)
        ]
        if not compatible:
            return None

        return min(
            compatible,
            key=lambda d: d.location.distance_to(pickup)
        )

    def name(self) -> str:
        return "Nearest Driver"
```

### Best Rated Driver Strategy

```python
class BestRatedDriverStrategy(MatchingStrategy):
    """
    Assign the highest-rated available driver within range.

    Prioritises service quality for premium ride requests
    or high-value riders.
    """

    def find_driver(self, rider: Rider, pickup: Location,
                    vehicle_type: VehicleType,
                    drivers: list[Driver]) -> Driver | None:
        compatible = [
            d for d in drivers
            if (d.status == DriverStatus.AVAILABLE
                and d.vehicle_type == vehicle_type
                and d.location.distance_to(pickup) <= MAX_PICKUP_RADIUS_KM)
        ]
        if not compatible:
            return None

        return max(compatible, key=lambda d: d.rating)

    def name(self) -> str:
        return "Best Rated Driver"
```

### Balanced Load Strategy

```python
class BalancedLoadStrategy(MatchingStrategy):
    """
    Assign the driver with the fewest completed trips.

    Ensures fair distribution of rides among drivers,
    particularly useful for newer drivers who need experience.
    """

    def find_driver(self, rider: Rider, pickup: Location,
                    vehicle_type: VehicleType,
                    drivers: list[Driver]) -> Driver | None:
        compatible = [
            d for d in drivers
            if (d.status == DriverStatus.AVAILABLE
                and d.vehicle_type == vehicle_type
                and d.location.distance_to(pickup) <= MAX_PICKUP_RADIUS_KM)
        ]
        if not compatible:
            return None

        return min(compatible, key=lambda d: len(d.trip_history))

    def name(self) -> str:
        return "Balanced Load"
```

All three strategies filter for available drivers with the correct vehicle type within the pickup radius. `NearestDriverStrategy` minimises the time between request and pickup — the most common algorithm used by Uber and Lyft. `BestRatedDriverStrategy` selects for quality, useful for premium tiers where riders pay more and expect better service. `BalancedLoadStrategy` ensures fairness among drivers — new or underutilised drivers get priority, preventing a few high-volume drivers from monopolising all the rides.

---

## Trip (State Pattern)

A trip transitions through a strict lifecycle. The state pattern encapsulates what actions are valid in each state and what the next valid state is.

### Trip State Interface

```python
import time


class TripState(ABC):
    """Abstract state in the trip lifecycle."""

    @abstractmethod
    def status(self) -> TripStatus:
        ...

    @abstractmethod
    def next_states(self) -> list[TripStatus]:
        """Valid transitions from this state."""
        ...

    def can_cancel(self) -> bool:
        return False

    def __repr__(self) -> str:
        return self.status().value


class RequestedState(TripState):
    def status(self) -> TripStatus:
        return TripStatus.REQUESTED

    def next_states(self) -> list[TripStatus]:
        return [TripStatus.DRIVER_ASSIGNED, TripStatus.CANCELLED]

    def can_cancel(self) -> bool:
        return True  # free cancellation


class DriverAssignedState(TripState):
    def status(self) -> TripStatus:
        return TripStatus.DRIVER_ASSIGNED

    def next_states(self) -> list[TripStatus]:
        return [TripStatus.EN_ROUTE, TripStatus.CANCELLED]

    def can_cancel(self) -> bool:
        return True  # free cancellation


class EnRouteState(TripState):
    def status(self) -> TripStatus:
        return TripStatus.EN_ROUTE

    def next_states(self) -> list[TripStatus]:
        return [TripStatus.ARRIVED, TripStatus.CANCELLED]

    def can_cancel(self) -> bool:
        return True  # cancellation with fee


class ArrivedState(TripState):
    def status(self) -> TripStatus:
        return TripStatus.ARRIVED

    def next_states(self) -> list[TripStatus]:
        return [TripStatus.IN_PROGRESS, TripStatus.CANCELLED]

    def can_cancel(self) -> bool:
        return True  # cancellation with fee


class InProgressState(TripState):
    def status(self) -> TripStatus:
        return TripStatus.IN_PROGRESS

    def next_states(self) -> list[TripStatus]:
        return [TripStatus.COMPLETED]

    def can_cancel(self) -> bool:
        return False  # cannot cancel during ride


class CompletedState(TripState):
    def status(self) -> TripStatus:
        return TripStatus.COMPLETED

    def next_states(self) -> list[TripStatus]:
        return []  # terminal state


class CancelledState(TripState):
    def status(self) -> TripStatus:
        return TripStatus.CANCELLED

    def next_states(self) -> list[TripStatus]:
        return []  # terminal state
```

### State Factory

```python
_TRIP_STATE_MAP: dict[TripStatus, type[TripState]] = {
    TripStatus.REQUESTED: RequestedState,
    TripStatus.DRIVER_ASSIGNED: DriverAssignedState,
    TripStatus.EN_ROUTE: EnRouteState,
    TripStatus.ARRIVED: ArrivedState,
    TripStatus.IN_PROGRESS: InProgressState,
    TripStatus.COMPLETED: CompletedState,
    TripStatus.CANCELLED: CancelledState,
}


def get_trip_state(status: TripStatus) -> TripState:
    return _TRIP_STATE_MAP[status]()
```

### Trip

```python
class Trip:
    """A ride-sharing trip with state-machine lifecycle."""

    _next_id: int = 0

    def __init__(self, rider_id: str, pickup: Location,
                 destination: Location, vehicle_type: VehicleType,
                 fare: FareBreakdown,
                 payment_method: PaymentMethod) -> None:
        Trip._next_id += 1
        self._id = f"TRIP-{Trip._next_id}"
        self._rider_id = rider_id
        self._pickup = pickup
        self._destination = destination
        self._route = Route(pickup, destination)
        self._vehicle_type = vehicle_type
        self._fare = fare
        self._payment_method = payment_method

        self._state: TripState = RequestedState()
        self._driver_id: str | None = None
        self._payment_status = PaymentStatus.PENDING
        self._created_at = int(time.time() * 1000)
        self._updated_at = self._created_at
        self._state_history: list[tuple[TripStatus, int]] = [
            (TripStatus.REQUESTED, self._created_at)
        ]

        self._rider_rating: float | None = None    # rider rates driver
        self._driver_rating: float | None = None   # driver rates rider
        self._cancellation_reason: str | None = None
        self._cancellation_fee: float = 0.0

    # ------------------------------------------------------------------
    # Properties
    # ------------------------------------------------------------------

    @property
    def id(self) -> str:
        return self._id

    @property
    def rider_id(self) -> str:
        return self._rider_id

    @property
    def driver_id(self) -> str | None:
        return self._driver_id

    @property
    def pickup(self) -> Location:
        return self._pickup

    @property
    def destination(self) -> Location:
        return self._destination

    @property
    def route(self) -> Route:
        return self._route

    @property
    def vehicle_type(self) -> VehicleType:
        return self._vehicle_type

    @property
    def fare(self) -> FareBreakdown:
        return self._fare

    @property
    def payment_method(self) -> PaymentMethod:
        return self._payment_method

    @property
    def status(self) -> TripStatus:
        return self._state.status()

    @property
    def payment_status(self) -> PaymentStatus:
        return self._payment_status

    @property
    def state_history(self) -> list[tuple[TripStatus, int]]:
        return list(self._state_history)

    @property
    def rider_rating(self) -> float | None:
        return self._rider_rating

    @property
    def driver_rating(self) -> float | None:
        return self._driver_rating

    @property
    def cancellation_fee(self) -> float:
        return self._cancellation_fee

    # ------------------------------------------------------------------
    # State transitions
    # ------------------------------------------------------------------

    def transition_to(self, new_status: TripStatus) -> bool:
        """
        Attempt to transition to a new state.
        Returns False if the transition is invalid.
        """
        if new_status not in self._state.next_states():
            print(
                f"  ✗ Invalid transition: {self._state.status().value} "
                f"→ {new_status.value}"
            )
            return False

        self._state = get_trip_state(new_status)
        self._updated_at = int(time.time() * 1000)
        self._state_history.append((new_status, self._updated_at))
        return True

    def cancel(self, reason: str = "Rider cancelled",
               apply_fee: bool = False) -> bool:
        """Cancel the trip if the current state allows it."""
        if not self._state.can_cancel():
            print(f"  ✗ Cannot cancel trip in "
                  f"{self._state.status().value} state")
            return False
        self._cancellation_reason = reason
        if apply_fee:
            self._cancellation_fee = CANCELLATION_FEE
        return self.transition_to(TripStatus.CANCELLED)

    def assign_driver(self, driver_id: str) -> None:
        self._driver_id = driver_id

    def set_payment_status(self, status: PaymentStatus) -> None:
        self._payment_status = status

    def rate_driver(self, score: float) -> None:
        """Rider rates the driver (1.0 – 5.0)."""
        self._rider_rating = max(1.0, min(5.0, score))

    def rate_rider(self, score: float) -> None:
        """Driver rates the rider (1.0 – 5.0)."""
        self._driver_rating = max(1.0, min(5.0, score))

    def is_active(self) -> bool:
        return self._state.status() not in (
            TripStatus.COMPLETED, TripStatus.CANCELLED
        )

    def has_cancellation_fee(self) -> bool:
        """Fee applies when cancelled after driver is en route."""
        return self._state.status() in (
            TripStatus.EN_ROUTE, TripStatus.ARRIVED
        )

    # ------------------------------------------------------------------
    # ETA
    # ------------------------------------------------------------------

    def pickup_eta(self, driver_location: Location) -> float:
        """ETA for driver to reach pickup in minutes."""
        return driver_location.estimated_travel_time(self._pickup)

    def destination_eta(self) -> float:
        """ETA for trip from pickup to destination in minutes."""
        return self._route.estimated_minutes

    # ------------------------------------------------------------------
    # Display
    # ------------------------------------------------------------------

    def __repr__(self) -> str:
        driver_str = f" 🚗{self._driver_id}" if self._driver_id else ""
        return (
            f"Trip#{self._id}({self._rider_id} "
            f"{self._vehicle_type.value} "
            f"${self._fare.total:.2f} "
            f"[{self._state.status().value}]{driver_str})"
        )

    def display(self) -> None:
        print(f"\n  {'═'*55}")
        print(f"  TRIP — {self._id}")
        print(f"  {'═'*55}")
        print(f"  Rider:       {self._rider_id}")
        print(f"  Driver:      {self._driver_id or 'Unassigned'}")
        print(f"  Vehicle:     {self._vehicle_type.value}")
        print(f"  Status:      {self._state.status().value}")
        print(f"  Pickup:      {self._pickup}")
        print(f"  Destination: {self._destination}")
        print(f"  Distance:    {self._route.distance_km:.1f} km")
        print(f"  Est. Time:   {self._route.estimated_minutes:.0f} min")
        print(f"  Payment:     {self._payment_method.value} "
              f"({self._payment_status.value})")
        self._fare.display()
```

Each `TripState` subclass defines which transitions are legal and whether cancellation is allowed. `RequestedState` and `DriverAssignedState` allow free cancellation (no fee). `EnRouteState` and `ArrivedState` allow cancellation but with a fee — the driver has already started moving. `InProgressState` blocks cancellation entirely — the rider is in the car. `transition_to` checks the current state's `next_states()` before allowing any change. The trip stores both rider-rates-driver and driver-rates-rider ratings — the two-way rating system used by Uber and Lyft. `pickup_eta` and `destination_eta` compute ETAs from the driver's current location and the route respectively.

---

## Payment Processing (Strategy Pattern)

```python
class PaymentStrategy(ABC):
    """Defines how a payment is processed."""

    @abstractmethod
    def charge(self, amount: float, rider: Rider) -> tuple[bool, str]:
        """
        Process the payment.
        Returns (success, transaction_message).
        """
        ...

    @abstractmethod
    def refund(self, amount: float, rider: Rider) -> tuple[bool, str]:
        """Process a refund."""
        ...

    @abstractmethod
    def method(self) -> PaymentMethod:
        ...


class CreditCardPayment(PaymentStrategy):
    """Simulates credit card payment processing."""

    def charge(self, amount: float, rider: Rider) -> tuple[bool, str]:
        print(f"  💳 Charging ${amount:.2f} to credit card "
              f"for {rider.name}")
        return True, f"CC-{int(time.time())}"

    def refund(self, amount: float, rider: Rider) -> tuple[bool, str]:
        print(f"  💳 Refunding ${amount:.2f} to credit card "
              f"for {rider.name}")
        return True, f"CC-REF-{int(time.time())}"

    def method(self) -> PaymentMethod:
        return PaymentMethod.CREDIT_CARD


class WalletPayment(PaymentStrategy):
    """Processes payment from in-app wallet balance."""

    def charge(self, amount: float, rider: Rider) -> tuple[bool, str]:
        if rider.debit_wallet(amount):
            print(f"  👛 Debited ${amount:.2f} from wallet "
                  f"for {rider.name}")
            return True, f"WALLET-{int(time.time())}"
        print(f"  ✗ Insufficient wallet balance. "
              f"Available: ${rider.wallet_balance:.2f}")
        return False, "Insufficient balance"

    def refund(self, amount: float, rider: Rider) -> tuple[bool, str]:
        rider.credit_wallet(amount)
        print(f"  👛 Refunded ${amount:.2f} to wallet "
              f"for {rider.name}")
        return True, f"WALLET-REF-{int(time.time())}"

    def method(self) -> PaymentMethod:
        return PaymentMethod.WALLET


class CashPayment(PaymentStrategy):
    """Cash payment — collected by the driver at the end."""

    def charge(self, amount: float, rider: Rider) -> tuple[bool, str]:
        print(f"  💵 Cash payment: ${amount:.2f} to be collected "
              f"from {rider.name}")
        return True, f"CASH-{int(time.time())}"

    def refund(self, amount: float, rider: Rider) -> tuple[bool, str]:
        rider.credit_wallet(amount)
        print(f"  💵 Cash refund: ${amount:.2f} credited to wallet "
              f"for {rider.name}")
        return True, f"CASH-REF-{int(time.time())}"

    def method(self) -> PaymentMethod:
        return PaymentMethod.CASH


# Payment strategy factory
_PAYMENT_MAP: dict[PaymentMethod, type[PaymentStrategy]] = {
    PaymentMethod.CREDIT_CARD: CreditCardPayment,
    PaymentMethod.WALLET: WalletPayment,
    PaymentMethod.CASH: CashPayment,
}


def get_payment_strategy(method: PaymentMethod) -> PaymentStrategy:
    strategy_cls = _PAYMENT_MAP.get(method)
    if strategy_cls is None:
        raise ValueError(f"Unsupported payment method: {method.value}")
    return strategy_cls()
```

Each `PaymentStrategy` handles `charge` and `refund` for its specific method. `CreditCardPayment` simulates a gateway call. `WalletPayment` checks and debits the rider's in-app balance — the only strategy that can fail due to insufficient funds. `CashPayment` defers collection to trip completion — the driver collects cash directly. The factory `get_payment_strategy` maps enum values to strategy instances, making new payment methods a one-line addition.

---

## Rating System

```python
class RatingSystem:
    """
    Two-way rating system for completed trips.

    Riders rate drivers and drivers rate riders.
    Ratings update the running average on each user.
    """

    def __init__(self) -> None:
        self._ratings: list[dict] = []

    def submit_rider_rating(self, trip: Trip, score: float,
                            driver: Driver) -> None:
        """Rider rates the driver."""
        trip.rate_driver(score)
        driver.add_rating(score)
        self._ratings.append({
            "trip_id": trip.id,
            "from": trip.rider_id,
            "to": trip.driver_id,
            "type": "rider_rates_driver",
            "score": score,
        })
        print(f"  ⭐ {trip.rider_id} rated driver {driver.name}: "
              f"{score:.1f}/5.0")

    def submit_driver_rating(self, trip: Trip, score: float,
                             rider: Rider) -> None:
        """Driver rates the rider."""
        trip.rate_rider(score)
        rider.add_rating(score)
        self._ratings.append({
            "trip_id": trip.id,
            "from": trip.driver_id,
            "to": trip.rider_id,
            "type": "driver_rates_rider",
            "score": score,
        })
        print(f"  ⭐ Driver rated {rider.name}: "
              f"{score:.1f}/5.0")

    def average_rating_for(self, user_id: str) -> float | None:
        """Compute average rating received by a user."""
        scores = [r["score"] for r in self._ratings
                  if r["to"] == user_id]
        if not scores:
            return None
        return sum(scores) / len(scores)

    def display(self) -> None:
        print(f"\n  {'═'*45}")
        print(f"  RATING SUMMARY — {len(self._ratings)} ratings")
        print(f"  {'═'*45}")
        for r in self._ratings:
            print(f"  {r['trip_id']}: {r['from']} → {r['to']} "
                  f"({r['type']}) ⭐{r['score']:.1f}")
        print()
```

`RatingSystem` maintains a log of all ratings and delegates score application to the trip and user objects. `submit_rider_rating` updates both the trip record (for history) and the driver's running average (for matching). The two-way design means both parties have accountability — low-rated riders may find it harder to get drivers to accept their requests, while low-rated drivers get fewer assignments in quality-based matching strategies.

---

## Notification System (Observer Pattern)

### Listener Interface

```python
class TripEventListener(ABC):
    """Observer interface for trip lifecycle events."""

    @abstractmethod
    def on_trip_requested(self, trip: Trip) -> None:
        ...

    @abstractmethod
    def on_driver_assigned(self, trip: Trip, driver: Driver) -> None:
        ...

    @abstractmethod
    def on_driver_en_route(self, trip: Trip) -> None:
        ...

    @abstractmethod
    def on_driver_arrived(self, trip: Trip) -> None:
        ...

    @abstractmethod
    def on_trip_started(self, trip: Trip) -> None:
        ...

    @abstractmethod
    def on_trip_completed(self, trip: Trip) -> None:
        ...

    @abstractmethod
    def on_trip_cancelled(self, trip: Trip, reason: str) -> None:
        ...
```

### Trip Logger

```python
class TripLogger(TripEventListener):
    """Logs all trip events to console."""

    def __init__(self) -> None:
        self._log: list[str] = []

    def _log_event(self, entry: str) -> None:
        self._log.append(entry)
        print(entry)

    def on_trip_requested(self, trip: Trip) -> None:
        self._log_event(
            f"  📝 TRIP REQUESTED: {trip.id} by {trip.rider_id} "
            f"({trip.vehicle_type.value}) — ${trip.fare.total:.2f}"
        )

    def on_driver_assigned(self, trip: Trip, driver: Driver) -> None:
        dist = driver.location.distance_to(trip.pickup)
        self._log_event(
            f"  🚗 DRIVER ASSIGNED: {driver.name} → {trip.id} "
            f"({dist:.1f}km away)"
        )

    def on_driver_en_route(self, trip: Trip) -> None:
        self._log_event(
            f"  🚗 EN ROUTE: {trip.driver_id} heading to pickup "
            f"for {trip.id}"
        )

    def on_driver_arrived(self, trip: Trip) -> None:
        self._log_event(
            f"  📍 ARRIVED: {trip.driver_id} at pickup "
            f"for {trip.id}"
        )

    def on_trip_started(self, trip: Trip) -> None:
        self._log_event(
            f"  🚀 TRIP STARTED: {trip.id} — "
            f"{trip.route.distance_km:.1f}km, "
            f"~{trip.route.estimated_minutes:.0f}min"
        )

    def on_trip_completed(self, trip: Trip) -> None:
        self._log_event(
            f"  ✅ TRIP COMPLETED: {trip.id} — "
            f"${trip.fare.total:.2f} "
            f"(Driver: ${trip.fare.driver_payout:.2f})"
        )

    def on_trip_cancelled(self, trip: Trip, reason: str) -> None:
        fee_str = (f" (fee: ${trip.cancellation_fee:.2f})"
                   if trip.cancellation_fee > 0 else "")
        self._log_event(
            f"  ❌ CANCELLED: {trip.id} — {reason}{fee_str}"
        )

    def print_log(self, last_n: int = 20) -> None:
        print(f"\n  Event Log (last {last_n}):")
        for entry in self._log[-last_n:]:
            print(entry)

    def clear(self) -> None:
        self._log.clear()
```

### Rider Notifier

```python
class RiderNotifier(TripEventListener):
    """Sends notifications to riders (simulated)."""

    def __init__(self) -> None:
        self._notifications: list[tuple[str, str]] = []

    def _notify(self, rider_id: str, message: str) -> None:
        self._notifications.append((rider_id, message))
        print(f"  📱 [{rider_id}]: {message}")

    def on_trip_requested(self, trip: Trip) -> None:
        self._notify(trip.rider_id,
                     f"Looking for a {trip.vehicle_type.value} "
                     f"driver near you...")

    def on_driver_assigned(self, trip: Trip, driver: Driver) -> None:
        self._notify(
            trip.rider_id,
            f"Driver {driver.name} is assigned! "
            f"Vehicle: {driver.vehicle} "
            f"⭐{driver.rating:.1f}"
        )

    def on_driver_en_route(self, trip: Trip) -> None:
        self._notify(trip.rider_id,
                     f"Your driver is on the way!")

    def on_driver_arrived(self, trip: Trip) -> None:
        self._notify(trip.rider_id,
                     f"Your driver has arrived at the pickup point!")

    def on_trip_started(self, trip: Trip) -> None:
        self._notify(
            trip.rider_id,
            f"Trip started! ETA: "
            f"~{trip.destination_eta():.0f} min"
        )

    def on_trip_completed(self, trip: Trip) -> None:
        self._notify(trip.rider_id,
                     f"You've arrived! Fare: ${trip.fare.total:.2f}. "
                     f"Please rate your driver.")

    def on_trip_cancelled(self, trip: Trip, reason: str) -> None:
        fee_msg = ""
        if trip.cancellation_fee > 0:
            fee_msg = f" Cancellation fee: ${trip.cancellation_fee:.2f}"
        self._notify(trip.rider_id,
                     f"Trip cancelled: {reason}.{fee_msg}")
```

### Driver Notifier

```python
class DriverNotifier(TripEventListener):
    """Sends notifications to drivers (simulated)."""

    def __init__(self) -> None:
        self._notifications: list[tuple[str, str]] = []

    def _notify(self, driver_id: str, message: str) -> None:
        self._notifications.append((driver_id, message))
        print(f"  📟 [{driver_id}]: {message}")

    def on_trip_requested(self, trip: Trip) -> None:
        pass  # drivers don't see raw requests

    def on_driver_assigned(self, trip: Trip, driver: Driver) -> None:
        self._notify(
            driver.user_id,
            f"New ride! Pickup: {trip.pickup}, "
            f"Destination: {trip.destination} "
            f"(${trip.fare.driver_payout:.2f} payout)"
        )

    def on_driver_en_route(self, trip: Trip) -> None:
        pass  # driver initiated this

    def on_driver_arrived(self, trip: Trip) -> None:
        pass  # driver initiated this

    def on_trip_started(self, trip: Trip) -> None:
        self._notify(
            trip.driver_id,
            f"Trip started! Navigate to {trip.destination}"
        )

    def on_trip_completed(self, trip: Trip) -> None:
        self._notify(
            trip.driver_id,
            f"Trip complete! Earned: "
            f"${trip.fare.driver_payout:.2f}")

    def on_trip_cancelled(self, trip: Trip, reason: str) -> None:
        if trip.driver_id:
            self._notify(trip.driver_id,
                         f"Trip cancelled by rider: {reason}")
```

`TripEventListener` defines seven hooks — one for each lifecycle event. `TripLogger` provides an audit trail with emoji markers. `RiderNotifier` and `DriverNotifier` simulate push notifications tailored to each party. Drivers do not see raw ride requests (that is handled by the matching engine), but they receive assignment notifications with payout information. The observer pattern means adding a new notification channel (SMS, email, analytics tracker) requires zero changes to existing code.

---

## Trip History & Analytics

```python
class TripAnalytics:
    """
    Tracks all trips and computes platform-wide analytics.
    """

    def __init__(self) -> None:
        self._trips: dict[str, Trip] = {}

    def record_trip(self, trip: Trip) -> None:
        self._trips[trip.id] = trip

    def get_trip(self, trip_id: str) -> Trip | None:
        return self._trips.get(trip_id)

    @property
    def all_trips(self) -> list[Trip]:
        return list(self._trips.values())

    @property
    def active_trips(self) -> list[Trip]:
        return [t for t in self._trips.values() if t.is_active()]

    @property
    def completed_trips(self) -> list[Trip]:
        return [t for t in self._trips.values()
                if t.status == TripStatus.COMPLETED]

    @property
    def cancelled_trips(self) -> list[Trip]:
        return [t for t in self._trips.values()
                if t.status == TripStatus.CANCELLED]

    # ------------------------------------------------------------------
    # Analytics
    # ------------------------------------------------------------------

    @property
    def total_revenue(self) -> float:
        return sum(t.fare.total for t in self.completed_trips)

    @property
    def total_platform_fees(self) -> float:
        return sum(t.fare.platform_fee for t in self.completed_trips)

    @property
    def total_driver_payouts(self) -> float:
        return sum(t.fare.driver_payout for t in self.completed_trips)

    @property
    def total_distance_km(self) -> float:
        return sum(t.route.distance_km for t in self.completed_trips)

    @property
    def average_fare(self) -> float:
        completed = self.completed_trips
        if not completed:
            return 0.0
        return self.total_revenue / len(completed)

    @property
    def cancellation_rate(self) -> float:
        total = len(self._trips)
        if total == 0:
            return 0.0
        return len(self.cancelled_trips) / total

    def trips_for_rider(self, rider_id: str) -> list[Trip]:
        return [t for t in self._trips.values()
                if t.rider_id == rider_id]

    def trips_for_driver(self, driver_id: str) -> list[Trip]:
        return [t for t in self._trips.values()
                if t.driver_id == driver_id]

    # ------------------------------------------------------------------
    # Display
    # ------------------------------------------------------------------

    def display_active(self) -> None:
        active = self.active_trips
        print(f"\n  {'═'*55}")
        print(f"  ACTIVE TRIPS — {len(active)}")
        print(f"  {'═'*55}")
        for trip in active:
            print(f"  {trip}")
        if not active:
            print("  No active trips")
        print()

    def display_summary(self) -> None:
        print(f"\n  {'═'*55}")
        print(f"  PLATFORM ANALYTICS")
        print(f"  {'═'*55}")
        print(f"  Total trips:        {len(self._trips)}")
        print(f"  Completed:          {len(self.completed_trips)}")
        print(f"  Cancelled:          {len(self.cancelled_trips)} "
              f"({self.cancellation_rate*100:.0f}%)")
        print(f"  Active:             {len(self.active_trips)}")
        print(f"  {'─'*55}")
        print(f"  Total revenue:      ${self.total_revenue:,.2f}")
        print(f"  Platform fees:      ${self.total_platform_fees:,.2f}")
        print(f"  Driver payouts:     ${self.total_driver_payouts:,.2f}")
        print(f"  Total distance:     {self.total_distance_km:,.1f} km")
        print(f"  Avg fare:           ${self.average_fare:.2f}")
        print(f"  {'═'*55}\n")

    def display_history(self, last_n: int = 10) -> None:
        trips = self.all_trips[-last_n:]
        print(f"\n  {'═'*55}")
        print(f"  TRIP HISTORY — last {last_n}")
        print(f"  {'═'*55}")
        for trip in trips:
            print(f"  {trip}")
        print()
```

`TripAnalytics` serves as both a trip registry and a platform analytics engine. It computes revenue split (total revenue vs. platform fees vs. driver payouts), tracks cancellation rates, and aggregates total distance. `trips_for_rider` and `trips_for_driver` enable per-user history queries. These analytics drive business decisions — a high cancellation rate might prompt surge adjustments, while average fare trends inform pricing strategy changes.

---

## Ride-Sharing System (Facade)

The `RideSharingSystem` ties everything together. It manages riders, drivers, trips, pricing, matching, payments, ratings, and notifications through a single unified API.

```python
class RideSharingSystem:
    """
    Central ride-sharing platform.
    Acts as a Facade over riders, drivers, trips, pricing,
    matching, payments, ratings, and notifications.
    """

    def __init__(self,
                 matching_strategy: MatchingStrategy | None = None,
                 pricing_strategy: PricingStrategy | None = None
                 ) -> None:
        self._riders: dict[str, Rider] = {}
        self._drivers: dict[str, Driver] = {}
        self._matching = matching_strategy or NearestDriverStrategy()
        self._pricing = pricing_strategy or StandardPricing()
        self._surge = SurgeCalculator()
        self._analytics = TripAnalytics()
        self._ratings = RatingSystem()
        self._listeners: list[TripEventListener] = []
        self._tick = 0
        self._active_requests: int = 0  # for surge calculation

    # ------------------------------------------------------------------
    # Properties
    # ------------------------------------------------------------------

    @property
    def tick(self) -> int:
        return self._tick

    @property
    def analytics(self) -> TripAnalytics:
        return self._analytics

    @property
    def rating_system(self) -> RatingSystem:
        return self._ratings

    # ------------------------------------------------------------------
    # Registration
    # ------------------------------------------------------------------

    def register_rider(self, rider: Rider) -> None:
        self._riders[rider.user_id] = rider

    def register_driver(self, driver: Driver) -> None:
        self._drivers[driver.user_id] = driver

    def add_listener(self, listener: TripEventListener) -> None:
        self._listeners.append(listener)

    # ------------------------------------------------------------------
    # Getters
    # ------------------------------------------------------------------

    def get_rider(self, rider_id: str) -> Rider | None:
        return self._riders.get(rider_id)

    def get_driver(self, driver_id: str) -> Driver | None:
        return self._drivers.get(driver_id)

    def get_trip(self, trip_id: str) -> Trip | None:
        return self._analytics.get_trip(trip_id)

    def available_drivers(self,
                          vehicle_type: VehicleType | None = None
                          ) -> list[Driver]:
        drivers = [d for d in self._drivers.values()
                   if d.status == DriverStatus.AVAILABLE]
        if vehicle_type:
            drivers = [d for d in drivers
                       if d.vehicle_type == vehicle_type]
        return drivers

    # ------------------------------------------------------------------
    # Fare estimation
    # ------------------------------------------------------------------

    def estimate_fare(self, pickup: Location, destination: Location,
                      vehicle_type: VehicleType) -> FareBreakdown:
        """Get a fare estimate without creating a trip."""
        route = Route(pickup, destination)
        surge = self._compute_surge()
        return self._pricing.calculate_fare(route, vehicle_type, surge)

    def _compute_surge(self) -> float:
        available = len(self.available_drivers())
        return self._surge.compute_surge(
            self._active_requests, available
        )

    # ------------------------------------------------------------------
    # Trip request
    # ------------------------------------------------------------------

    def request_trip(self, rider_id: str, pickup: Location,
                     destination: Location,
                     vehicle_type: VehicleType,
                     payment_method: PaymentMethod
                     ) -> Trip | None:
        """
        Request a new trip. Estimates fare, finds driver,
        processes payment hold, and creates the trip.
        Returns the trip or None if no driver is found.
        """
        self._tick += 1

        # --- Validate rider ---
        rider = self._riders.get(rider_id)
        if rider is None:
            print(f"  ✗ Rider '{rider_id}' not found")
            return None

        # --- Validate distance ---
        route = Route(pickup, destination)
        if route.distance_km < 0.1:
            print(f"  ✗ Pickup and destination are too close")
            return None

        # --- Compute fare ---
        self._active_requests += 1
        surge = self._compute_surge()
        fare = self._pricing.calculate_fare(
            route, vehicle_type, surge
        )

        # --- Create trip ---
        trip = Trip(
            rider_id=rider_id,
            pickup=pickup,
            destination=destination,
            vehicle_type=vehicle_type,
            fare=fare,
            payment_method=payment_method,
        )

        # --- Process payment hold (except cash) ---
        if payment_method != PaymentMethod.CASH:
            strategy = get_payment_strategy(payment_method)
            success, txn = strategy.charge(fare.total, rider)
            if not success:
                self._active_requests -= 1
                print(f"  ✗ Payment failed: {txn}")
                return None
            trip.set_payment_status(PaymentStatus.COMPLETED)

        # --- Register trip ---
        self._analytics.record_trip(trip)
        rider.add_to_history(trip.id)
        self._notify_requested(trip)

        # --- Find and assign driver ---
        self._assign_driver(trip)

        return trip

    # ------------------------------------------------------------------
    # Driver assignment
    # ------------------------------------------------------------------

    def _assign_driver(self, trip: Trip) -> bool:
        """Find and assign a driver to the trip."""
        rider = self._riders.get(trip.rider_id)
        if rider is None:
            return False

        drivers = list(self._drivers.values())
        driver = self._matching.find_driver(
            rider, trip.pickup, trip.vehicle_type, drivers
        )

        if driver is None:
            print(f"  ⚠ No {trip.vehicle_type.value} drivers "
                  f"available for {trip.id}")
            return False

        driver.assign_trip(trip.id)
        trip.assign_driver(driver.user_id)
        trip.transition_to(TripStatus.DRIVER_ASSIGNED)
        self._active_requests = max(0, self._active_requests - 1)
        self._notify_driver_assigned(trip, driver)
        return True

    # ------------------------------------------------------------------
    # Trip lifecycle
    # ------------------------------------------------------------------

    def driver_en_route(self, trip_id: str) -> bool:
        """Driver starts heading to the pickup location."""
        trip = self._analytics.get_trip(trip_id)
        if trip is None:
            return False

        if trip.transition_to(TripStatus.EN_ROUTE):
            self._notify_en_route(trip)
            return True
        return False

    def driver_arrived(self, trip_id: str) -> bool:
        """Driver has arrived at the pickup location."""
        trip = self._analytics.get_trip(trip_id)
        if trip is None:
            return False

        if trip.transition_to(TripStatus.ARRIVED):
            self._notify_arrived(trip)
            return True
        return False

    def start_trip(self, trip_id: str) -> bool:
        """Rider is in the car, trip begins."""
        trip = self._analytics.get_trip(trip_id)
        if trip is None:
            return False

        if trip.transition_to(TripStatus.IN_PROGRESS):
            self._notify_started(trip)
            return True
        return False

    def complete_trip(self, trip_id: str) -> bool:
        """Trip has reached the destination."""
        trip = self._analytics.get_trip(trip_id)
        if trip is None:
            return False

        if not trip.transition_to(TripStatus.COMPLETED):
            return False

        # Complete driver stats
        if trip.driver_id:
            driver = self._drivers.get(trip.driver_id)
            if driver:
                driver.complete_trip(
                    trip.fare.driver_payout,
                    trip.route.distance_km
                )

        # Process cash payment
        if trip.payment_method == PaymentMethod.CASH:
            trip.set_payment_status(PaymentStatus.COMPLETED)

        self._notify_completed(trip)
        return True

    def cancel_trip(self, trip_id: str,
                    reason: str = "Rider cancelled") -> bool:
        """Cancel a trip. May incur a cancellation fee."""
        trip = self._analytics.get_trip(trip_id)
        if trip is None:
            print(f"  ✗ Trip '{trip_id}' not found")
            return False

        # Determine if fee applies
        apply_fee = trip.has_cancellation_fee()

        if not trip.cancel(reason, apply_fee):
            return False

        # Release driver
        if trip.driver_id:
            driver = self._drivers.get(trip.driver_id)
            if driver:
                driver.release_trip()

        # Update active requests
        self._active_requests = max(0, self._active_requests - 1)

        # Refund payment (minus cancellation fee)
        self._process_refund(trip)

        self._notify_cancelled(trip, reason)
        return True

    # ------------------------------------------------------------------
    # Ratings
    # ------------------------------------------------------------------

    def rate_driver(self, trip_id: str, score: float) -> bool:
        """Rider rates the driver after a completed trip."""
        trip = self._analytics.get_trip(trip_id)
        if trip is None or trip.status != TripStatus.COMPLETED:
            print(f"  ✗ Can only rate completed trips")
            return False

        driver = self._drivers.get(trip.driver_id)
        if driver is None:
            return False

        self._ratings.submit_rider_rating(trip, score, driver)
        return True

    def rate_rider(self, trip_id: str, score: float) -> bool:
        """Driver rates the rider after a completed trip."""
        trip = self._analytics.get_trip(trip_id)
        if trip is None or trip.status != TripStatus.COMPLETED:
            return False

        rider = self._riders.get(trip.rider_id)
        if rider is None:
            return False

        self._ratings.submit_driver_rating(trip, score, rider)
        return True

    # ------------------------------------------------------------------
    # ETA
    # ------------------------------------------------------------------

    def get_pickup_eta(self, trip_id: str) -> float | None:
        """Get ETA for driver to reach the pickup in minutes."""
        trip = self._analytics.get_trip(trip_id)
        if trip is None or not trip.driver_id:
            return None

        driver = self._drivers.get(trip.driver_id)
        if driver is None:
            return None

        return trip.pickup_eta(driver.location)

    def get_destination_eta(self, trip_id: str) -> float | None:
        """Get ETA for trip from pickup to destination in minutes."""
        trip = self._analytics.get_trip(trip_id)
        if trip is None:
            return None
        return trip.destination_eta()

    # ------------------------------------------------------------------
    # Payment
    # ------------------------------------------------------------------

    def _process_refund(self, trip: Trip) -> None:
        """Refund a cancelled trip's payment (minus cancellation fee)."""
        if trip.payment_status != PaymentStatus.COMPLETED:
            return

        rider = self._riders.get(trip.rider_id)
        if rider is None:
            return

        refund_amount = trip.fare.total - trip.cancellation_fee
        if refund_amount <= 0:
            return

        strategy = get_payment_strategy(trip.payment_method)
        success, _ = strategy.refund(refund_amount, rider)
        if success:
            trip.set_payment_status(PaymentStatus.REFUNDED)

    # ------------------------------------------------------------------
    # Notifications
    # ------------------------------------------------------------------

    def _notify_requested(self, trip: Trip) -> None:
        for listener in self._listeners:
            listener.on_trip_requested(trip)

    def _notify_driver_assigned(self, trip: Trip,
                                driver: Driver) -> None:
        for listener in self._listeners:
            listener.on_driver_assigned(trip, driver)

    def _notify_en_route(self, trip: Trip) -> None:
        for listener in self._listeners:
            listener.on_driver_en_route(trip)

    def _notify_arrived(self, trip: Trip) -> None:
        for listener in self._listeners:
            listener.on_driver_arrived(trip)

    def _notify_started(self, trip: Trip) -> None:
        for listener in self._listeners:
            listener.on_trip_started(trip)

    def _notify_completed(self, trip: Trip) -> None:
        for listener in self._listeners:
            listener.on_trip_completed(trip)

    def _notify_cancelled(self, trip: Trip, reason: str) -> None:
        for listener in self._listeners:
            listener.on_trip_cancelled(trip, reason)

    # ------------------------------------------------------------------
    # Display
    # ------------------------------------------------------------------

    def display_status(self) -> None:
        surge = self._compute_surge()
        print(f"\n  {'═'*60}")
        print(f"  RIDE-SHARING SYSTEM — Tick {self._tick}")
        print(f"  Matching: {self._matching.name()} | "
              f"Pricing: {self._pricing.name()}")
        print(f"  Surge: ×{surge:.1f} | "
              f"Active Requests: {self._active_requests} | "
              f"Available Drivers: {len(self.available_drivers())}")
        print(f"  {'═'*60}")

        print(f"\n  DRIVERS:")
        print(f"  {'─'*55}")
        for d in self._drivers.values():
            print(f"  {d}")

        print(f"\n  RIDERS:")
        print(f"  {'─'*55}")
        for r in self._riders.values():
            print(f"  {r}")

        self._analytics.display_active()
```

`request_trip` is the primary entry point. It follows a strict pipeline: validate rider → compute route and fare (with surge) → create trip → process payment hold → register trip → find and assign driver. Payment is processed before driver matching — if payment fails, no trip is created. `_assign_driver` delegates to the matching strategy and transitions the trip from `REQUESTED` to `DRIVER_ASSIGNED`. The lifecycle methods (`driver_en_route`, `driver_arrived`, `start_trip`, `complete_trip`) each delegate to the trip's state machine. `cancel_trip` applies a cancellation fee when the driver has already started driving to the pickup — the `has_cancellation_fee` method on the trip checks the current state. `_process_refund` returns the fare minus the cancellation fee to the rider. The surge calculator updates dynamically as `_active_requests` changes with each new request and assignment.

---

## Simulation Engine

A helper that scripts realistic ride-sharing scenarios for demonstration.

```python
from typing import Callable


class RideScenario:
    """A sequence of timed ride-sharing actions for simulation."""

    def __init__(self, system: RideSharingSystem) -> None:
        self._system = system
        self._events: list[tuple[int, str, Callable]] = []

    def at_tick(self, tick: int, description: str,
                action: Callable) -> "RideScenario":
        """Schedule an action at a specific tick."""
        self._events.append((tick, description, action))
        return self

    def run(self, display_interval: int = 5) -> None:
        """Execute the scenario with periodic status display."""
        self._events.sort(key=lambda e: e[0])
        event_idx = 0
        max_tick = (max(e[0] for e in self._events) + 5
                    if self._events else 10)

        print("\n" + "═" * 62)
        print("  RIDE-SHARING SIMULATION START")
        print("═" * 62)

        for tick in range(1, max_tick + 1):
            while (event_idx < len(self._events)
                   and self._events[event_idx][0] == tick):
                _, desc, action = self._events[event_idx]
                print(f"\n  ⏱ Tick {tick}: {desc}")
                action()
                event_idx += 1

            if tick % display_interval == 0 or tick == max_tick:
                self._system.display_status()

            if (event_idx >= len(self._events)
                    and tick > self._events[-1][0] + 2):
                break

        print("\n" + "═" * 62)
        print("  RIDE-SHARING SIMULATION END")
        print("═" * 62)
```

`RideScenario` lets you script events at specific ticks: "at tick 3, Alice requests a ride". `run` executes the simulation in order, firing events at the right ticks and displaying the system state at configurable intervals.

---

## Putting It All Together

### Basic Simulation

```python
def main() -> None:
    # Reset counters for clean start
    Trip._next_id = 0

    # Create the system
    system = RideSharingSystem(
        matching_strategy=NearestDriverStrategy(),
        pricing_strategy=StandardPricing(),
    )

    # Attach observers
    logger = TripLogger()
    rider_notifier = RiderNotifier()
    driver_notifier = DriverNotifier()
    system.add_listener(logger)
    system.add_listener(rider_notifier)
    system.add_listener(driver_notifier)

    # --- Register riders ---
    alice = Rider(
        "alice", "Alice Johnson", "555-0101",
        "alice@email.com", Location(27.7172, 85.3240),
        home_address="123 Main St",
        work_address="Tech Park, Ring Road"
    )
    alice.top_up_wallet(200.0)
    system.register_rider(alice)

    bob = Rider(
        "bob", "Bob Smith", "555-0102",
        "bob@email.com", Location(27.7050, 85.3400),
        home_address="456 Oak Avenue"
    )
    system.register_rider(bob)

    # --- Register drivers ---
    driver_raj = Driver(
        "driver-raj", "Raj Kumar", "555-0201",
        "raj@drive.com", Location(27.7180, 85.3220),
        Vehicle("V-001", VehicleType.ECONOMY,
                "Toyota", "Yaris", 2022, "BA-1-1234",
                "White", 4)
    )
    driver_priya = Driver(
        "driver-priya", "Priya Sharma", "555-0202",
        "priya@drive.com", Location(27.7100, 85.3350),
        Vehicle("V-002", VehicleType.COMFORT,
                "Honda", "City", 2023, "BA-2-5678",
                "Silver", 4)
    )
    driver_arjun = Driver(
        "driver-arjun", "Arjun Patel", "555-0203",
        "arjun@drive.com", Location(27.7200, 85.3150),
        Vehicle("V-003", VehicleType.PREMIUM,
                "Mercedes", "E-Class", 2024, "BA-3-9012",
                "Black", 4)
    )
    system.register_driver(driver_raj)
    system.register_driver(driver_priya)
    system.register_driver(driver_arjun)

    # ╔══════════════════════════════════════════╗
    # ║       RIDE-SHARING SYSTEM                ║
    # ╚══════════════════════════════════════════╝

    print("╔══════════════════════════════════════════╗")
    print("║       RIDE-SHARING SYSTEM                ║")
    print("║  Matching: Nearest Driver                ║")
    print("║  Pricing:  Standard                      ║")
    print("╚══════════════════════════════════════════╝")

    # --- Alice requests an economy ride ---
    print("\n  --- Alice estimates fare ---")
    estimate = system.estimate_fare(
        Location(27.7172, 85.3240),  # Alice's location
        Location(27.6900, 85.3500),  # destination
        VehicleType.ECONOMY
    )
    estimate.display()

    print("\n  --- Alice requests a ride ---")
    trip1 = system.request_trip(
        "alice",
        Location(27.7172, 85.3240),
        Location(27.6900, 85.3500),
        VehicleType.ECONOMY,
        PaymentMethod.WALLET
    )

    if trip1:
        pickup_eta = system.get_pickup_eta(trip1.id)
        dest_eta = system.get_destination_eta(trip1.id)
        print(f"\n  ⏱ Pickup ETA: {pickup_eta:.0f} min")
        print(f"  ⏱ Destination ETA: {dest_eta:.0f} min")

        system.driver_en_route(trip1.id)
        system.driver_arrived(trip1.id)
        system.start_trip(trip1.id)
        system.complete_trip(trip1.id)

        # Ratings
        system.rate_driver(trip1.id, 4.5)
        system.rate_rider(trip1.id, 5.0)

    # --- Bob requests a comfort ride ---
    print("\n  --- Bob requests a comfort ride ---")
    trip2 = system.request_trip(
        "bob",
        Location(27.7050, 85.3400),
        Location(27.7300, 85.3100),
        VehicleType.COMFORT,
        PaymentMethod.CREDIT_CARD
    )

    if trip2:
        system.driver_en_route(trip2.id)

        # Bob cancels after driver is en route (fee applies)
        print("\n  --- Bob cancels (driver already en route) ---")
        system.cancel_trip(trip2.id, "Changed my mind")

    # --- Display final state ---
    system.display_status()
    system.analytics.display_summary()
    system.rating_system.display()


if __name__ == "__main__":
    main()
```

### Scripted Scenario

```python
def main_scenario() -> None:
    Trip._next_id = 0

    system = RideSharingSystem(
        matching_strategy=NearestDriverStrategy(),
        pricing_strategy=StandardPricing(),
    )
    logger = TripLogger()
    rider_notifier = RiderNotifier()
    system.add_listener(logger)
    system.add_listener(rider_notifier)

    # Setup drivers
    drivers_data = [
        ("d1", "Raj", Location(27.7180, 85.3220),
         Vehicle("V1", VehicleType.ECONOMY, "Toyota", "Yaris",
                 2022, "BA-1-1234", "White", 4)),
        ("d2", "Priya", Location(27.7100, 85.3350),
         Vehicle("V2", VehicleType.ECONOMY, "Suzuki", "Swift",
                 2023, "BA-2-5678", "Blue", 4)),
        ("d3", "Arjun", Location(27.7200, 85.3150),
         Vehicle("V3", VehicleType.COMFORT, "Honda", "City",
                 2024, "BA-3-9012", "Silver", 4)),
    ]
    for uid, name, loc, vehicle in drivers_data:
        system.register_driver(
            Driver(uid, name, "555-0000",
                   f"{uid}@drive.com", loc, vehicle)
        )

    # Setup riders
    for uid, name, loc in [
        ("alice", "Alice", Location(27.7172, 85.3240)),
        ("bob", "Bob", Location(27.7050, 85.3400)),
        ("charlie", "Charlie", Location(27.7220, 85.3180)),
    ]:
        r = Rider(uid, name, "555-0000",
                  f"{uid}@email.com", loc, f"{name}'s Home")
        r.top_up_wallet(500.0)
        system.register_rider(r)

    scenario = RideScenario(system)

    # Alice requests a ride
    scenario.at_tick(1, "Alice requests economy ride",
                     lambda: system.request_trip(
                         "alice",
                         Location(27.7172, 85.3240),
                         Location(27.6900, 85.3500),
                         VehicleType.ECONOMY,
                         PaymentMethod.WALLET))

    scenario.at_tick(2, "Alice's driver en route",
                     lambda: system.driver_en_route("TRIP-1"))

    scenario.at_tick(3, "Alice's driver arrived",
                     lambda: system.driver_arrived("TRIP-1"))

    # Bob requests at the same time — tests driver availability
    scenario.at_tick(3, "Bob requests economy ride",
                     lambda: system.request_trip(
                         "bob",
                         Location(27.7050, 85.3400),
                         Location(27.7300, 85.3100),
                         VehicleType.ECONOMY,
                         PaymentMethod.WALLET))

    scenario.at_tick(4, "Alice's trip starts",
                     lambda: system.start_trip("TRIP-1"))

    scenario.at_tick(4, "Bob's driver en route",
                     lambda: system.driver_en_route("TRIP-2"))

    # Charlie requests comfort ride
    scenario.at_tick(5, "Charlie requests comfort ride",
                     lambda: system.request_trip(
                         "charlie",
                         Location(27.7220, 85.3180),
                         Location(27.7000, 85.3400),
                         VehicleType.COMFORT,
                         PaymentMethod.WALLET))

    # Charlie cancels before driver departs
    scenario.at_tick(6, "Charlie cancels (free cancellation)",
                     lambda: system.cancel_trip(
                         "TRIP-3", "Changed plans"))

    scenario.at_tick(6, "Alice's trip completed",
                     lambda: system.complete_trip("TRIP-1"))

    scenario.at_tick(7, "Alice rates driver 5 stars",
                     lambda: system.rate_driver("TRIP-1", 5.0))

    scenario.at_tick(7, "Bob's driver arrived",
                     lambda: system.driver_arrived("TRIP-2"))

    scenario.at_tick(8, "Bob's trip starts",
                     lambda: system.start_trip("TRIP-2"))

    scenario.at_tick(9, "Bob's trip completed",
                     lambda: system.complete_trip("TRIP-2"))

    scenario.at_tick(10, "Bob rates driver 4 stars",
                     lambda: system.rate_driver("TRIP-2", 4.0))

    scenario.run(display_interval=5)
    system.analytics.display_summary()


if __name__ == "__main__":
    main_scenario()
```

### Strategy Comparison

```python
def compare_strategies() -> None:
    """Compare matching and pricing strategies on the same workload."""

    # --- Matching strategy comparison ---
    matching_strategies: list[MatchingStrategy] = [
        NearestDriverStrategy(),
        BestRatedDriverStrategy(),
        BalancedLoadStrategy(),
    ]

    print("\n" + "=" * 55)
    print("  MATCHING STRATEGY COMPARISON")
    print("=" * 55)

    for strategy in matching_strategies:
        Trip._next_id = 0

        system = RideSharingSystem(
            matching_strategy=strategy,
            pricing_strategy=StandardPricing()
        )

        # Setup rider
        r = Rider("rider", "TestRider", "555", "r@test.com",
                   Location(27.7172, 85.3240), "Home")
        r.top_up_wallet(500.0)
        system.register_rider(r)

        # Setup drivers at different distances with different stats
        drivers_info = [
            ("D1", "NearDriver", Location(27.7175, 85.3245), 20, 4.2),
            ("D2", "MidDriver", Location(27.7160, 85.3260), 8, 4.9),
            ("D3", "FarDriver", Location(27.7130, 85.3300), 3, 4.5),
        ]
        for uid, name, loc, trips, rating in drivers_info:
            d = Driver(uid, name, "555", f"{uid}@test.com", loc,
                       Vehicle(f"V-{uid}", VehicleType.ECONOMY,
                               "Toyota", "Yaris", 2022,
                               f"XX-{uid}", "White", 4))
            d._rating = rating
            d._trip_history = ["x"] * trips  # simulate trip count
            system.register_driver(d)

        trip = system.request_trip(
            "rider",
            Location(27.7172, 85.3240),
            Location(27.6900, 85.3500),
            VehicleType.ECONOMY,
            PaymentMethod.WALLET
        )

        if trip and trip.driver_id:
            driver = system.get_driver(trip.driver_id)
            dist = driver.location.distance_to(trip.pickup)
            print(f"\n  Strategy: {strategy.name()}")
            print(f"    Selected: {driver.name}")
            print(f"    Distance: {dist:.2f} km")
            print(f"    Rating: {driver.rating:.1f}")
            print(f"    Completed trips: {len(driver.trip_history)}")

    # --- Pricing strategy comparison ---
    print("\n" + "=" * 55)
    print("  PRICING STRATEGY COMPARISON")
    print("=" * 55)

    route = Route(
        Location(27.7172, 85.3240),
        Location(27.6900, 85.3500)
    )

    pricing_strategies: list[tuple[PricingStrategy, float]] = [
        (StandardPricing(), 1.0),
        (StandardPricing(), 1.5),
        (StandardPricing(), 2.0),
        (PromotionalPricing(0.20, 10.00), 1.0),
    ]

    for strategy, surge in pricing_strategies:
        fare = strategy.calculate_fare(
            route, VehicleType.ECONOMY, surge
        )
        surge_str = f" (surge ×{surge})" if surge > 1.0 else ""
        print(f"\n  {strategy.name()}{surge_str}:")
        print(f"    Total:          ${fare.total:.2f}")
        print(f"    Driver payout:  ${fare.driver_payout:.2f}")
        print(f"    Platform fee:   ${fare.platform_fee:.2f}")
        if fare.discount > 0:
            print(f"    Discount:      -${fare.discount:.2f}")

    print("\n" + "=" * 55)
```

This comparison highlights strategy differences. For matching: `NearestDriverStrategy` picks the driver closest to the pickup, `BestRatedDriverStrategy` picks the highest-rated driver regardless of distance, and `BalancedLoadStrategy` picks the driver with fewest trips. For pricing: standard fares scale linearly with surge, while promotional pricing applies a capped discount and absorbs the cost on the platform side.

### Unit Tests

```python
def test_basic_trip_flow() -> None:
    """Test that a complete trip flow works."""
    Trip._next_id = 0

    system = RideSharingSystem()
    r = Rider("rider", "Test", "555", "r@test.com",
              Location(27.7172, 85.3240), "Home")
    r.top_up_wallet(100.0)
    system.register_rider(r)

    d = Driver("driver", "TestDriver", "555", "d@test.com",
               Location(27.7180, 85.3220),
               Vehicle("V1", VehicleType.ECONOMY, "Toyota", "Yaris",
                       2022, "XX-1234", "White", 4))
    system.register_driver(d)

    trip = system.request_trip(
        "rider",
        Location(27.7172, 85.3240),
        Location(27.6900, 85.3500),
        VehicleType.ECONOMY,
        PaymentMethod.WALLET
    )

    assert trip is not None
    assert trip.status == TripStatus.DRIVER_ASSIGNED
    assert trip.driver_id == "driver"
    print("✅ test_basic_trip_flow passed")


def test_trip_lifecycle() -> None:
    """Test full trip state transitions."""
    Trip._next_id = 0

    system = RideSharingSystem()
    r = Rider("rider", "Test", "555", "r@test.com",
              Location(27.7172, 85.3240), "Home")
    r.top_up_wallet(100.0)
    system.register_rider(r)

    d = Driver("driver", "TestDriver", "555", "d@test.com",
               Location(27.7180, 85.3220),
               Vehicle("V1", VehicleType.ECONOMY, "Toyota", "Yaris",
                       2022, "XX-1234", "White", 4))
    system.register_driver(d)

    trip = system.request_trip(
        "rider",
        Location(27.7172, 85.3240),
        Location(27.6900, 85.3500),
        VehicleType.ECONOMY,
        PaymentMethod.WALLET
    )
    assert trip is not None

    assert system.driver_en_route(trip.id)
    assert trip.status == TripStatus.EN_ROUTE

    assert system.driver_arrived(trip.id)
    assert trip.status == TripStatus.ARRIVED

    assert system.start_trip(trip.id)
    assert trip.status == TripStatus.IN_PROGRESS

    assert system.complete_trip(trip.id)
    assert trip.status == TripStatus.COMPLETED

    assert d.status == DriverStatus.AVAILABLE  # driver freed
    assert d.total_earnings > 0
    print("✅ test_trip_lifecycle passed")


def test_free_cancellation() -> None:
    """Test that cancellation before en_route is free."""
    Trip._next_id = 0

    system = RideSharingSystem()
    r = Rider("rider", "Test", "555", "r@test.com",
              Location(27.7172, 85.3240), "Home")
    r.top_up_wallet(100.0)
    system.register_rider(r)

    d = Driver("driver", "TestDriver", "555", "d@test.com",
               Location(27.7180, 85.3220),
               Vehicle("V1", VehicleType.ECONOMY, "Toyota", "Yaris",
                       2022, "XX-1234", "White", 4))
    system.register_driver(d)

    trip = system.request_trip(
        "rider",
        Location(27.7172, 85.3240),
        Location(27.6900, 85.3500),
        VehicleType.ECONOMY,
        PaymentMethod.WALLET
    )
    assert trip is not None

    # Cancel while still in DRIVER_ASSIGNED (free)
    assert system.cancel_trip(trip.id, "Changed mind")
    assert trip.status == TripStatus.CANCELLED
    assert trip.cancellation_fee == 0.0
    assert d.status == DriverStatus.AVAILABLE  # driver freed
    print("✅ test_free_cancellation passed")


def test_paid_cancellation() -> None:
    """Test that cancellation after en_route incurs a fee."""
    Trip._next_id = 0

    system = RideSharingSystem()
    r = Rider("rider", "Test", "555", "r@test.com",
              Location(27.7172, 85.3240), "Home")
    r.top_up_wallet(100.0)
    system.register_rider(r)

    d = Driver("driver", "TestDriver", "555", "d@test.com",
               Location(27.7180, 85.3220),
               Vehicle("V1", VehicleType.ECONOMY, "Toyota", "Yaris",
                       2022, "XX-1234", "White", 4))
    system.register_driver(d)

    trip = system.request_trip(
        "rider",
        Location(27.7172, 85.3240),
        Location(27.6900, 85.3500),
        VehicleType.ECONOMY,
        PaymentMethod.WALLET
    )
    assert trip is not None

    system.driver_en_route(trip.id)

    # Cancel while EN_ROUTE (fee applies)
    assert system.cancel_trip(trip.id, "Changed mind")
    assert trip.status == TripStatus.CANCELLED
    assert trip.cancellation_fee == CANCELLATION_FEE
    print("✅ test_paid_cancellation passed")


def test_cannot_cancel_in_progress() -> None:
    """Test that cancellation fails during an active ride."""
    Trip._next_id = 0

    system = RideSharingSystem()
    r = Rider("rider", "Test", "555", "r@test.com",
              Location(27.7172, 85.3240), "Home")
    r.top_up_wallet(100.0)
    system.register_rider(r)

    d = Driver("driver", "TestDriver", "555", "d@test.com",
               Location(27.7180, 85.3220),
               Vehicle("V1", VehicleType.ECONOMY, "Toyota", "Yaris",
                       2022, "XX-1234", "White", 4))
    system.register_driver(d)

    trip = system.request_trip(
        "rider",
        Location(27.7172, 85.3240),
        Location(27.6900, 85.3500),
        VehicleType.ECONOMY,
        PaymentMethod.WALLET
    )
    assert trip is not None

    system.driver_en_route(trip.id)
    system.driver_arrived(trip.id)
    system.start_trip(trip.id)

    result = system.cancel_trip(trip.id, "Too late")
    assert result is False
    assert trip.status == TripStatus.IN_PROGRESS
    print("✅ test_cannot_cancel_in_progress passed")


def test_invalid_state_transition() -> None:
    """Test that skipping states is rejected."""
    Trip._next_id = 0

    system = RideSharingSystem()
    r = Rider("rider", "Test", "555", "r@test.com",
              Location(27.7172, 85.3240), "Home")
    r.top_up_wallet(100.0)
    system.register_rider(r)

    d = Driver("driver", "TestDriver", "555", "d@test.com",
               Location(27.7180, 85.3220),
               Vehicle("V1", VehicleType.ECONOMY, "Toyota", "Yaris",
                       2022, "XX-1234", "White", 4))
    system.register_driver(d)

    trip = system.request_trip(
        "rider",
        Location(27.7172, 85.3240),
        Location(27.6900, 85.3500),
        VehicleType.ECONOMY,
        PaymentMethod.WALLET
    )
    assert trip is not None

    # Cannot skip from DRIVER_ASSIGNED to IN_PROGRESS
    result = system.start_trip(trip.id)
    assert result is False
    assert trip.status == TripStatus.DRIVER_ASSIGNED

    # Cannot go directly to COMPLETED
    result = system.complete_trip(trip.id)
    assert result is False
    print("✅ test_invalid_state_transition passed")


def test_vehicle_type_matching() -> None:
    """Test that drivers only match their vehicle type."""
    Trip._next_id = 0

    system = RideSharingSystem()
    r = Rider("rider", "Test", "555", "r@test.com",
              Location(27.7172, 85.3240), "Home")
    r.top_up_wallet(100.0)
    system.register_rider(r)

    # Only register a COMFORT driver
    d = Driver("driver", "ComfortDriver", "555", "d@test.com",
               Location(27.7180, 85.3220),
               Vehicle("V1", VehicleType.COMFORT, "Honda", "City",
                       2023, "XX-5678", "Silver", 4))
    system.register_driver(d)

    # Request ECONOMY — should fail (no economy drivers)
    trip = system.request_trip(
        "rider",
        Location(27.7172, 85.3240),
        Location(27.6900, 85.3500),
        VehicleType.ECONOMY,
        PaymentMethod.WALLET
    )
    assert trip is not None
    assert trip.driver_id is None  # no match
    assert trip.status == TripStatus.REQUESTED
    print("✅ test_vehicle_type_matching passed")


def test_surge_pricing() -> None:
    """Test that surge multiplier increases fares."""
    route = Route(
        Location(27.7172, 85.3240),
        Location(27.6900, 85.3500)
    )
    pricing = StandardPricing()

    normal = pricing.calculate_fare(route, VehicleType.ECONOMY, 1.0)
    surged = pricing.calculate_fare(route, VehicleType.ECONOMY, 2.0)

    assert surged.total > normal.total
    assert surged.surge_multiplier == 2.0
    assert surged.surge_amount > 0
    # Driver payout should also be higher during surge
    assert surged.driver_payout > normal.driver_payout
    print("✅ test_surge_pricing passed")


def test_two_way_ratings() -> None:
    """Test the two-way rating system."""
    Trip._next_id = 0

    system = RideSharingSystem()
    r = Rider("rider", "TestRider", "555", "r@test.com",
              Location(27.7172, 85.3240), "Home")
    r.top_up_wallet(100.0)
    system.register_rider(r)

    d = Driver("driver", "TestDriver", "555", "d@test.com",
               Location(27.7180, 85.3220),
               Vehicle("V1", VehicleType.ECONOMY, "Toyota", "Yaris",
                       2022, "XX-1234", "White", 4))
    system.register_driver(d)

    trip = system.request_trip(
        "rider",
        Location(27.7172, 85.3240),
        Location(27.6900, 85.3500),
        VehicleType.ECONOMY,
        PaymentMethod.WALLET
    )
    assert trip is not None

    system.driver_en_route(trip.id)
    system.driver_arrived(trip.id)
    system.start_trip(trip.id)
    system.complete_trip(trip.id)

    # Rate both ways
    assert system.rate_driver(trip.id, 4.5)
    assert system.rate_rider(trip.id, 5.0)
    assert trip.rider_rating == 4.5
    assert trip.driver_rating == 5.0
    print("✅ test_two_way_ratings passed")


def test_wallet_insufficient_balance() -> None:
    """Test that wallet payment fails with insufficient balance."""
    Trip._next_id = 0

    system = RideSharingSystem()
    r = Rider("rider", "Test", "555", "r@test.com",
              Location(27.7172, 85.3240), "Home")
    r.top_up_wallet(1.0)  # only $1
    system.register_rider(r)

    d = Driver("driver", "TestDriver", "555", "d@test.com",
               Location(27.7180, 85.3220),
               Vehicle("V1", VehicleType.ECONOMY, "Toyota", "Yaris",
                       2022, "XX-1234", "White", 4))
    system.register_driver(d)

    trip = system.request_trip(
        "rider",
        Location(27.7172, 85.3240),
        Location(27.6900, 85.3500),
        VehicleType.ECONOMY,
        PaymentMethod.WALLET
    )
    assert trip is None  # payment fails
    print("✅ test_wallet_insufficient_balance passed")


def run_all_tests() -> None:
    print("\n" + "=" * 40)
    print("  RUNNING TESTS")
    print("=" * 40 + "\n")
    test_basic_trip_flow()
    test_trip_lifecycle()
    test_free_cancellation()
    test_paid_cancellation()
    test_cannot_cancel_in_progress()
    test_invalid_state_transition()
    test_vehicle_type_matching()
    test_surge_pricing()
    test_two_way_ratings()
    test_wallet_insufficient_balance()
    print("\n✅ All tests passed!\n")
```

Each test resets the trip ID counter for isolation. `test_basic_trip_flow` verifies the request-to-assignment pipeline. `test_trip_lifecycle` walks through every state from `REQUESTED` to `COMPLETED` and checks that the driver is freed and credited. `test_free_cancellation` confirms no fee for early cancellation. `test_paid_cancellation` verifies the fee when the driver is already en route. `test_cannot_cancel_in_progress` ensures the state machine blocks mid-ride cancellation. `test_vehicle_type_matching` checks that an economy request fails when only a comfort driver is available. `test_surge_pricing` validates that surge multipliers increase both the total fare and the driver payout. `test_two_way_ratings` exercises the full two-way rating flow.

---

## Class Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                    RideSharingSystem                              │
│──────────────────────────────────────────────────────────────────│
│ - _riders: dict[str, Rider]                                       │
│ - _drivers: dict[str, Driver]                                     │
│ - _matching: MatchingStrategy                                     │
│ - _pricing: PricingStrategy                                       │
│ - _surge: SurgeCalculator                                         │
│ - _analytics: TripAnalytics                                       │
│ - _ratings: RatingSystem                                          │
│ - _listeners: list[TripEventListener]                             │
│ - _tick: int                                                      │
│──────────────────────────────────────────────────────────────────│
│ + request_trip(rider, pickup, dest, type, pay) -> Trip            │
│ + driver_en_route(trip_id) -> bool                                │
│ + driver_arrived(trip_id) -> bool                                 │
│ + start_trip(trip_id) -> bool                                     │
│ + complete_trip(trip_id) -> bool                                  │
│ + cancel_trip(trip_id, reason) -> bool                            │
│ + estimate_fare(pickup, dest, type) -> FareBreakdown              │
│ + rate_driver(trip_id, score) -> bool                             │
│ + rate_rider(trip_id, score) -> bool                              │
│ + display_status() -> None                                        │
└──────┬──────────┬──────────┬──────────┬───────────┬──────────────┘
       │          │          │          │           │
  uses ▼     uses ▼     uses ▼    uses ▼      uses ▼
┌──────────────────────────────────────────────────────────────────┐
│  «abstract» MatchingStrategy                                      │
│──────────────────────────────────────────────────────────────────│
│ + find_driver(rider, pickup, type, drivers) -> Driver | None      │
│ + name() -> str                                                   │
└──────────┬────────────────────────────────────────────────────────┘
           │ implemented by
  ┌────────┼──────────────────┐
  ▼        ▼                  ▼
Nearest   BestRated        Balanced
Driver    Driver           Load
Strategy  Strategy         Strategy

┌──────────────────────────────────────────────────────────────────┐
│  «abstract» PricingStrategy                                       │
│──────────────────────────────────────────────────────────────────│
│ + calculate_fare(route, type, surge) -> FareBreakdown             │
│ + name() -> str                                                   │
└──────────┬────────────────────────────────────────────────────────┘
           │ implemented by
  ┌────────┴──────────┐
  ▼                   ▼
Standard           Promotional
Pricing            Pricing

┌──────────────────────────────────────────────────────────────────┐
│  «abstract» PaymentStrategy                                       │
│──────────────────────────────────────────────────────────────────│
│ + charge(amount, rider) -> (bool, str)                            │
│ + refund(amount, rider) -> (bool, str)                            │
│ + method() -> PaymentMethod                                       │
└──────────┬──────────────────────────────────────────────────────┘
           │ implemented by
  ┌────────┼──────────────────┐
  ▼        ▼                  ▼
CreditCard  Wallet          Cash
Payment     Payment         Payment

┌──────────────────────────────────────────────────────────────────┐
│  «abstract» TripEventListener                                     │
│──────────────────────────────────────────────────────────────────│
│ + on_trip_requested(trip)                                         │
│ + on_driver_assigned(trip, driver)                                │
│ + on_driver_en_route(trip)                                        │
│ + on_driver_arrived(trip)                                         │
│ + on_trip_started(trip)                                           │
│ + on_trip_completed(trip)                                         │
│ + on_trip_cancelled(trip, reason)                                 │
└──────────┬──────────────────────────────────────────────────────┘
           │ implemented by
  ┌────────┼──────────────────┐
  ▼        ▼                  ▼
TripLogger  RiderNotifier  DriverNotifier

┌──────────────┐ ┌──────────────┐
│    Rider     │ │   Driver     │
│──────────────│ │──────────────│
│ - _user_id   │ │ - _user_id   │
│ - _name      │ │ - _name      │
│ - _location  │ │ - _location  │
│ - _rating    │ │ - _vehicle   │
│ - _wallet    │ │ - _status    │
│ - _trip_     │ │ - _current_  │
│   history    │ │   trip_id    │
│──────────────│ │ - _rating    │
│ + top_up_    │ │ - _earnings  │
│   wallet()   │ │──────────────│
│ + debit_     │ │ + assign_    │
│   wallet()   │ │   trip()     │
│ + add_rating │ │ + complete_  │
└──────────────┘ │   trip()     │
                 │ + release_   │
┌──────────────┐ │   trip()     │
│  Vehicle     │ │ + go_offline │
│──────────────│ │ + go_online  │
│ + vehicle_id │ │ + add_rating │
│ + type       │ └──────────────┘
│ + make/model │
│ + license    │
│ + color      │
│ + capacity   │
└──────────────┘

┌──────────────────────────────────┐
│  Trip                             │
│──────────────────────────────────│
│ - _id: str                        │
│ - _rider_id: str                  │
│ - _driver_id: str | None         │
│ - _pickup: Location               │
│ - _destination: Location          │
│ - _route: Route                   │
│ - _vehicle_type: VehicleType      │
│ - _fare: FareBreakdown            │
│ - _state: TripState               │
│ - _payment_method                 │
│ - _payment_status                 │
│ - _rider_rating: float | None    │
│ - _driver_rating: float | None   │
│ - _cancellation_fee: float        │
│──────────────────────────────────│
│ + transition_to(status) -> bool   │
│ + cancel(reason, fee) -> bool     │
│ + assign_driver(id) -> None       │
│ + rate_driver(score) -> None      │
│ + rate_rider(score) -> None       │
│ + pickup_eta(loc) -> float        │
│ + destination_eta() -> float      │
│ + is_active() -> bool             │
└──────────────────────────────────┘

┌──────────────────────────────────┐
│ «abstract» TripState              │
│──────────────────────────────────│
│ + status() -> TripStatus          │
│ + next_states() -> list           │
│ + can_cancel() -> bool            │
└──────────┬───────────────────────┘
           │ implemented by
  ┌────────┼─────────┬──────────┐
  ▼        ▼         ▼          ▼
Requested DriverAss  EnRoute  Arrived
State     ignedState State    State
           ┌─────────┼──────────┐
           ▼         ▼          ▼
        InProgress Completed Cancelled
        State      State     State

┌──────────────────────────────────┐
│  FareBreakdown                    │
│──────────────────────────────────│
│ + base_fare: float                │
│ + distance_fare: float            │
│ + time_fare: float                │
│ + booking_fee: float              │
│ + surge_multiplier: float         │
│ + surge_amount: float             │
│ + discount: float                 │
│ + subtotal: float                 │
│ + total: float                    │
│ + driver_payout: float            │
│ + platform_fee: float             │
│ + display() -> None               │
└──────────────────────────────────┘

┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│ Location        │ │ Route           │ │ SurgeCalculator │
│ (frozen)        │ │ (frozen)        │ │─────────────────│
│─────────────────│ │─────────────────│ │ - _max_surge    │
│ + latitude      │ │ + pickup        │ │─────────────────│
│ + longitude     │ │ + destination   │ │ + compute_surge │
│─────────────────│ │ + distance_km   │ │   (requests,    │
│ + distance_to() │ │ + estimated_    │ │    available)   │
│ + estimated_    │ │   minutes       │ │   -> float      │
│   travel_time() │ └─────────────────┘ └─────────────────┘
└─────────────────┘

┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│ TripAnalytics    │ │ RatingSystem     │ │ RideScenario     │
│──────────────────│ │──────────────────│ │──────────────────│
│ - _trips         │ │ - _ratings       │ │ - _system        │
│──────────────────│ │──────────────────│ │ - _events        │
│ + record_trip    │ │ + submit_rider_  │ │──────────────────│
│ + active_trips   │ │   rating()       │ │ + at_tick()      │
│ + completed_trips│ │ + submit_driver_ │ │ + run()          │
│ + total_revenue  │ │   rating()       │ └──────────────────┘
│ + cancellation_  │ └──────────────────┘
│   rate           │
│ + display_       │
│   summary()      │
└──────────────────┘

┌────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│ TripStatus     │ │ VehicleType      │ │ DriverStatus     │
│ (Enum)         │ │ (Enum)           │ │ (Enum)           │
│────────────────│ │──────────────────│ │──────────────────│
│ REQUESTED      │ │ ECONOMY          │ │ AVAILABLE        │
│ DRIVER_ASSIGNED│ │ COMFORT          │ │ BUSY             │
│ EN_ROUTE       │ │ PREMIUM          │ │ OFFLINE          │
│ ARRIVED        │ │ XL               │ └──────────────────┘
│ IN_PROGRESS    │ └──────────────────┘
│ COMPLETED      │ ┌──────────────────┐ ┌──────────────────┐
│ CANCELLED      │ │ PaymentMethod    │ │ PaymentStatus    │
└────────────────┘ │ (Enum)           │ │ (Enum)           │
                   │──────────────────│ │──────────────────│
                   │ CREDIT_CARD      │ │ PENDING          │
                   │ DEBIT_CARD       │ │ COMPLETED        │
                   │ WALLET           │ │ FAILED           │
                   │ CASH             │ │ REFUNDED         │
                   └──────────────────┘ └──────────────────┘
```

---

## How to Run

Save all code into a single file `ride_sharing.py` and run:

```bash
python ride_sharing.py
```

### Example Output

```
╔══════════════════════════════════════════╗
║       RIDE-SHARING SYSTEM                ║
║  Matching: Nearest Driver                ║
║  Pricing:  Standard                      ║
╚══════════════════════════════════════════╝

  --- Alice estimates fare ---

  ─────────────────────────────────────────────
  FARE BREAKDOWN
  ─────────────────────────────────────────────
  Base fare:       $2.50
  Distance fare:   $4.22
  Time fare:       $1.06
  Booking fee:     $1.50
  ─────────────────────────────────────────────
  TOTAL:           $9.28
  Driver payout:   $6.22
  Platform fee:    $1.56
  ─────────────────────────────────────────────

  --- Alice requests a ride ---
  👛 Debited $9.28 from wallet for Alice Johnson
  📝 TRIP REQUESTED: TRIP-1 by alice (economy) — $9.28
  📱 [alice]: Looking for a economy driver near you...
  🚗 DRIVER ASSIGNED: Raj Kumar → TRIP-1 (0.2km away)
  📱 [alice]: Driver Raj Kumar is assigned! Vehicle: White 2022 Toyota Yaris (economy) [BA-1-1234] ⭐4.8
  📟 [driver-raj]: New ride! Pickup: (27.7172, 85.3240), Destination: (27.6900, 85.3500) ($6.22 payout)

  ⏱ Pickup ETA: 1 min
  ⏱ Destination ETA: 7 min

  🚗 EN ROUTE: driver-raj heading to pickup for TRIP-1
  📱 [alice]: Your driver is on the way!
  📍 ARRIVED: driver-raj at pickup for TRIP-1
  📱 [alice]: Your driver has arrived at the pickup point!
  🚀 TRIP STARTED: TRIP-1 — 3.5km, ~7min
  📱 [alice]: Trip started! ETA: ~7 min
  ✅ TRIP COMPLETED: TRIP-1 — $9.28 (Driver: $6.22)
  📱 [alice]: You've arrived! Fare: $9.28. Please rate your driver.
  📟 [driver-raj]: Trip complete! Earned: $6.22
  ⭐ alice rated driver Raj Kumar: 4.5/5.0
  ⭐ Driver rated Alice Johnson: 5.0/5.0

  --- Bob requests a comfort ride ---
  💳 Charging $14.56 to credit card for Bob Smith
  📝 TRIP REQUESTED: TRIP-2 by bob (comfort) — $14.56
  🚗 DRIVER ASSIGNED: Priya Sharma → TRIP-2 (0.6km away)
  🚗 EN ROUTE: driver-priya heading to pickup for TRIP-2

  --- Bob cancels (driver already en route) ---
  💳 Refunding $9.56 to credit card for Bob Smith
  ❌ CANCELLED: TRIP-2 — Changed my mind (fee: $5.00)

  ═══════════════════════════════════════════════════════════
  PLATFORM ANALYTICS
  ═══════════════════════════════════════════════════════════
  Total trips:        2
  Completed:          1
  Cancelled:          1 (50%)
  Active:             0
  ───────────────────────────────────────────────────────────
  Total revenue:      $9.28
  Platform fees:      $1.56
  Driver payouts:     $6.22
  Total distance:     3.5 km
  Avg fare:           $9.28
  ═══════════════════════════════════════════════════════════
```
