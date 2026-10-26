# Design Food Delivery System – Low-Level Design Guide

![Bidur Sapkota](https://www.bidursapkota.com.np/images/gravatar.webp "Bidur Sapkota - Developer")&nbsp;[Bidur Sapkota](https://www.bidursapkota.com.np/)

## Table of Contents

1. [Requirements](#requirements)
2. [Key Design Patterns](#key-design-patterns)
3. [Core Enums & Constants](#core-enums--constants)
4. [Menu & MenuItem](#menu--menuitem)
5. [Restaurant (Subject for Observer)](#restaurant-subject-for-observer)
6. [Cart & CartItem](#cart--cartitem)
7. [Order (State Pattern)](#order-state-pattern)
8. [User, Customer & DeliveryAgent](#user-customer--deliveryagent)
9. [Delivery Assignment (Strategy Pattern)](#delivery-assignment-strategy-pattern)
10. [Payment Processing (Strategy Pattern)](#payment-processing-strategy-pattern)
11. [Notification System (Observer Pattern)](#notification-system-observer-pattern)
12. [Order Tracker](#order-tracker)
13. [Food Delivery System (Facade)](#food-delivery-system-facade)
14. [Simulation Engine](#simulation-engine)
15. [Putting It All Together](#putting-it-all-together)
16. [Class Diagram](#class-diagram)
17. [How to Run](#how-to-run)

---

## Requirements

### Functional Requirements

| #  | Requirement                                                                              |
| -- | ---------------------------------------------------------------------------------------- |
| 1  | Multiple restaurants, each with its own menu of items (name, price, category, prep time) |
| 2  | Customers can browse restaurants and their menus                                         |
| 3  | Customers can add menu items to a cart (single restaurant per cart)                       |
| 4  | Customers place orders from the cart; each order has a delivery address                   |
| 5  | Order lifecycle: PLACED → CONFIRMED → PREPARING → READY → PICKED_UP → DELIVERED (or CANCELLED) |
| 6  | Restaurants confirm or reject incoming orders                                            |
| 7  | Delivery agents are assigned to orders using a pluggable dispatch strategy               |
| 8  | Agents can accept or decline assignments; declined orders are re-dispatched              |
| 9  | Multiple payment methods (credit card, wallet, cash on delivery)                         |
| 10 | Real-time order tracking with estimated delivery time                                    |
| 11 | Observer-based notifications — customers, restaurants, and agents are notified on state changes |
| 12 | Order history is maintained for customers and restaurants                                 |

### Non-Functional Requirements

- Clean OOP with SOLID principles.
- Extensible — new payment methods, dispatch strategies, and notification channels can be added without modifying core logic.
- Simulatable — the system runs step-by-step for testing and demonstration.

---

## Key Design Patterns

| Pattern  | Where It Is Used                                      | Why                                                                                        |
| -------- | ----------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| State    | `Order` transitions through lifecycle states          | Each state encapsulates valid transitions; adding a new state does not touch existing ones  |
| Strategy | `DeliveryStrategy` selects how agents are dispatched  | Swap between nearest-first, least-busy, round-robin without changing core dispatch logic    |
| Strategy | `PaymentStrategy` processes different payment methods | Add new payment methods (UPI, crypto) without modifying the checkout flow                  |
| Observer | `OrderEventListener` observes order state changes     | Customers, restaurants, agents, and loggers react independently to the same events          |
| Facade   | `FoodDeliverySystem` exposes a simple API             | Hides the complexity of restaurants, carts, orders, payments, dispatch, and notifications   |
| Builder  | `Cart` builds an order step-by-step                   | Separates the complex order construction from its representation                           |

---

## Core Enums & Constants

```python
from enum import Enum


class OrderStatus(Enum):
    PLACED = "placed"
    CONFIRMED = "confirmed"
    PREPARING = "preparing"
    READY = "ready_for_pickup"
    PICKED_UP = "picked_up"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"


class PaymentMethod(Enum):
    CREDIT_CARD = "credit_card"
    DEBIT_CARD = "debit_card"
    WALLET = "wallet"
    CASH_ON_DELIVERY = "cash_on_delivery"


class PaymentStatus(Enum):
    PENDING = "pending"
    COMPLETED = "completed"
    FAILED = "failed"
    REFUNDED = "refunded"


class AgentStatus(Enum):
    AVAILABLE = "available"
    BUSY = "busy"
    OFFLINE = "offline"


class ItemCategory(Enum):
    APPETIZER = "appetizer"
    MAIN_COURSE = "main_course"
    DESSERT = "dessert"
    BEVERAGE = "beverage"
    SIDE = "side"


# Fees and limits
DELIVERY_FEE_BASE = 2.50        # base delivery fee
DELIVERY_FEE_PER_KM = 1.00     # per kilometer
SERVICE_FEE_RATE = 0.05         # 5% service fee
MIN_ORDER_AMOUNT = 10.00        # minimum order value
MAX_DELIVERY_RADIUS_KM = 15.0   # max delivery distance

# Time constants (in minutes)
BASE_DELIVERY_TIME = 10         # base travel time
TIME_PER_KM = 3                 # minutes per km
```

`OrderStatus` models the full lifecycle of a food delivery order — from placement through preparation, pickup, and delivery. The `CANCELLED` state can be reached from `PLACED` or `CONFIRMED` but not after preparation starts. `PaymentMethod` and `PaymentStatus` are kept separate because an order's payment can transition independently (a cash order starts `PENDING` and completes on delivery). `AgentStatus` tracks whether a delivery agent can accept new orders. Fee constants model a real-world pricing structure: a base delivery charge, a per-kilometer surcharge, and a platform service fee.

---

## Menu & MenuItem

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class MenuItem:
    """An immutable menu item offered by a restaurant."""
    item_id: str
    name: str
    price: float
    category: ItemCategory
    prep_time_minutes: int  # estimated preparation time
    is_vegetarian: bool = False
    is_available: bool = True

    def __repr__(self) -> str:
        veg = " 🌿" if self.is_vegetarian else ""
        avail = "" if self.is_available else " [UNAVAILABLE]"
        return (
            f"{self.name} — ${self.price:.2f} "
            f"({self.category.value}, ~{self.prep_time_minutes}min)"
            f"{veg}{avail}"
        )


class Menu:
    """A restaurant's menu — a collection of menu items."""

    def __init__(self) -> None:
        self._items: dict[str, MenuItem] = {}

    @property
    def items(self) -> list[MenuItem]:
        return list(self._items.values())

    @property
    def available_items(self) -> list[MenuItem]:
        return [item for item in self._items.values() if item.is_available]

    def add_item(self, item: MenuItem) -> None:
        self._items[item.item_id] = item

    def remove_item(self, item_id: str) -> MenuItem | None:
        return self._items.pop(item_id, None)

    def get_item(self, item_id: str) -> MenuItem | None:
        return self._items.get(item_id)

    def get_by_category(self, category: ItemCategory) -> list[MenuItem]:
        return [item for item in self._items.values()
                if item.category == category and item.is_available]

    def update_availability(self, item_id: str,
                            available: bool) -> bool:
        item = self._items.get(item_id)
        if item is None:
            return False
        # Replace with updated availability (frozen dataclass)
        self._items[item_id] = MenuItem(
            item_id=item.item_id,
            name=item.name,
            price=item.price,
            category=item.category,
            prep_time_minutes=item.prep_time_minutes,
            is_vegetarian=item.is_vegetarian,
            is_available=available,
        )
        return True

    def display(self) -> None:
        print(f"\n  {'─'*50}")
        print(f"  MENU ({len(self.available_items)} items available)")
        print(f"  {'─'*50}")
        for cat in ItemCategory:
            cat_items = self.get_by_category(cat)
            if cat_items:
                print(f"  {cat.value.upper()}:")
                for item in cat_items:
                    print(f"    {item}")
        print()
```

`MenuItem` is a frozen dataclass — immutable after creation, which prevents accidental price or name changes in a running system. `prep_time_minutes` feeds into the estimated delivery time calculation. Since `MenuItem` is frozen, `update_availability` replaces the entire object rather than mutating it — this is the standard pattern for immutable records. `Menu` groups items by category for browsing and filters out unavailable items so customers only see what they can order.

---

## Restaurant (Subject for Observer)

```python
import math


@dataclass(frozen=True)
class Location:
    """A geographic coordinate."""
    latitude: float
    longitude: float

    def distance_to(self, other: "Location") -> float:
        """Approximate distance in km using Haversine formula."""
        R = 6371  # Earth radius in km
        lat1, lat2 = math.radians(self.latitude), math.radians(other.latitude)
        dlat = math.radians(other.latitude - self.latitude)
        dlon = math.radians(other.longitude - self.longitude)
        a = (math.sin(dlat / 2) ** 2 +
             math.cos(lat1) * math.cos(lat2) * math.sin(dlon / 2) ** 2)
        return R * 2 * math.atan2(math.sqrt(a), math.sqrt(1 - a))

    def __repr__(self) -> str:
        return f"({self.latitude:.4f}, {self.longitude:.4f})"


class Restaurant:
    """A restaurant that receives and prepares food orders."""

    _next_id: int = 0

    def __init__(self, name: str, location: Location,
                 cuisine: str = "General") -> None:
        Restaurant._next_id += 1
        self._id = f"REST-{Restaurant._next_id}"
        self._name = name
        self._location = location
        self._cuisine = cuisine
        self._menu = Menu()
        self._is_open = True
        self._rating: float = 4.0
        self._total_ratings: int = 0
        self._active_orders: list[str] = []  # order IDs being prepared

    # ------------------------------------------------------------------
    # Properties
    # ------------------------------------------------------------------

    @property
    def id(self) -> str:
        return self._id

    @property
    def name(self) -> str:
        return self._name

    @property
    def location(self) -> Location:
        return self._location

    @property
    def cuisine(self) -> str:
        return self._cuisine

    @property
    def menu(self) -> Menu:
        return self._menu

    @property
    def is_open(self) -> bool:
        return self._is_open

    @property
    def rating(self) -> float:
        return self._rating

    @property
    def active_order_count(self) -> int:
        return len(self._active_orders)

    # ------------------------------------------------------------------
    # Operations
    # ------------------------------------------------------------------

    def open(self) -> None:
        self._is_open = True

    def close(self) -> None:
        self._is_open = False

    def add_rating(self, score: float) -> None:
        """Update running average rating (1.0 – 5.0)."""
        score = max(1.0, min(5.0, score))
        self._total_ratings += 1
        self._rating += (score - self._rating) / self._total_ratings

    def add_active_order(self, order_id: str) -> None:
        self._active_orders.append(order_id)

    def remove_active_order(self, order_id: str) -> None:
        if order_id in self._active_orders:
            self._active_orders.remove(order_id)

    def estimated_prep_time(self, item_ids: list[str]) -> int:
        """Estimate prep time: max item prep time + queue delay."""
        if not item_ids:
            return 0
        max_prep = max(
            (self._menu.get_item(iid).prep_time_minutes
             for iid in item_ids if self._menu.get_item(iid)),
            default=0
        )
        queue_delay = len(self._active_orders) * 5  # 5 min per queued order
        return max_prep + queue_delay

    def __repr__(self) -> str:
        status = "OPEN" if self._is_open else "CLOSED"
        return (
            f"Restaurant({self._name} [{self._cuisine}] "
            f"⭐{self._rating:.1f} {status} "
            f"📍{self._location})"
        )
```

`Restaurant` is the entity that owns a `Menu` and receives orders. `Location` uses the Haversine formula to compute real-world distances between coordinates — this feeds into delivery time estimates and agent dispatch. `estimated_prep_time` takes the maximum preparation time across all ordered items (since the kitchen prepares them in parallel) and adds a queue delay proportional to how many orders are already being prepared. `add_rating` uses an incremental mean formula — it does not store all ratings, just the running average and count.

---

## Cart & CartItem

```python
@dataclass
class CartItem:
    """An item in a customer's cart with a quantity."""
    menu_item: MenuItem
    quantity: int

    @property
    def subtotal(self) -> float:
        return self.menu_item.price * self.quantity

    def __repr__(self) -> str:
        return (
            f"{self.menu_item.name} x{self.quantity} "
            f"= ${self.subtotal:.2f}"
        )


class Cart:
    """
    A customer's shopping cart.

    Enforces single-restaurant constraint — all items must
    come from the same restaurant.
    """

    def __init__(self, customer_id: str) -> None:
        self._customer_id = customer_id
        self._restaurant_id: str | None = None
        self._items: dict[str, CartItem] = {}  # item_id -> CartItem

    # ------------------------------------------------------------------
    # Properties
    # ------------------------------------------------------------------

    @property
    def customer_id(self) -> str:
        return self._customer_id

    @property
    def restaurant_id(self) -> str | None:
        return self._restaurant_id

    @property
    def items(self) -> list[CartItem]:
        return list(self._items.values())

    @property
    def total(self) -> float:
        return sum(item.subtotal for item in self._items.values())

    @property
    def item_count(self) -> int:
        return sum(item.quantity for item in self._items.values())

    @property
    def is_empty(self) -> bool:
        return len(self._items) == 0

    # ------------------------------------------------------------------
    # Cart operations
    # ------------------------------------------------------------------

    def add_item(self, menu_item: MenuItem,
                 restaurant_id: str, quantity: int = 1) -> bool:
        """
        Add an item to the cart.
        Returns False if the item is from a different restaurant.
        """
        if self._restaurant_id and self._restaurant_id != restaurant_id:
            print(f"  ✗ Cart contains items from another restaurant. "
                  f"Clear cart first.")
            return False

        if not menu_item.is_available:
            print(f"  ✗ {menu_item.name} is currently unavailable.")
            return False

        self._restaurant_id = restaurant_id

        if menu_item.item_id in self._items:
            existing = self._items[menu_item.item_id]
            self._items[menu_item.item_id] = CartItem(
                menu_item=existing.menu_item,
                quantity=existing.quantity + quantity,
            )
        else:
            self._items[menu_item.item_id] = CartItem(
                menu_item=menu_item, quantity=quantity
            )
        return True

    def remove_item(self, item_id: str) -> bool:
        """Remove an item entirely from the cart."""
        if item_id in self._items:
            del self._items[item_id]
            if not self._items:
                self._restaurant_id = None
            return True
        return False

    def update_quantity(self, item_id: str, quantity: int) -> bool:
        """Update the quantity of an item. Removes if quantity <= 0."""
        if item_id not in self._items:
            return False
        if quantity <= 0:
            return self.remove_item(item_id)
        existing = self._items[item_id]
        self._items[item_id] = CartItem(
            menu_item=existing.menu_item, quantity=quantity
        )
        return True

    def clear(self) -> None:
        """Empty the cart."""
        self._items.clear()
        self._restaurant_id = None

    def get_item_ids(self) -> list[str]:
        """Return all menu item IDs in the cart."""
        return list(self._items.keys())

    # ------------------------------------------------------------------
    # Display
    # ------------------------------------------------------------------

    def display(self) -> None:
        if self.is_empty:
            print("  🛒 Cart is empty")
            return

        print(f"\n  {'─'*45}")
        print(f"  🛒 CART — {self._customer_id}")
        print(f"  Restaurant: {self._restaurant_id}")
        print(f"  {'─'*45}")
        for item in self._items.values():
            print(f"  {item}")
        print(f"  {'─'*45}")
        print(f"  Subtotal: ${self.total:.2f}")
        service_fee = self.total * SERVICE_FEE_RATE
        print(f"  Service Fee (5%): ${service_fee:.2f}")
        print(f"  Delivery Fee: ${DELIVERY_FEE_BASE:.2f}+")
        print(f"  {'─'*45}\n")

    def __repr__(self) -> str:
        return (
            f"Cart({self._customer_id} "
            f"{self.item_count} items "
            f"${self.total:.2f})"
        )
```

`Cart` enforces the single-restaurant constraint found in real delivery apps — you cannot order from two restaurants in one order. `add_item` checks this and rejects cross-restaurant additions. If an item is already in the cart, its quantity increments rather than creating a duplicate entry. The cart acts as a builder for orders — it accumulates items and computes subtotals before the order is finalized. `clear` resets the restaurant binding so the customer can start a new cart for a different restaurant.

---

## Order (State Pattern)

An order transitions through a strict lifecycle. The state pattern encapsulates what actions are valid in each state and what the next valid state is.

### Order State Interface

```python
from abc import ABC, abstractmethod
import time


class OrderState(ABC):
    """Abstract state in the order lifecycle."""

    @abstractmethod
    def status(self) -> OrderStatus:
        ...

    @abstractmethod
    def next_states(self) -> list[OrderStatus]:
        """Valid transitions from this state."""
        ...

    def can_cancel(self) -> bool:
        return False

    def __repr__(self) -> str:
        return self.status().value


class PlacedState(OrderState):
    def status(self) -> OrderStatus:
        return OrderStatus.PLACED

    def next_states(self) -> list[OrderStatus]:
        return [OrderStatus.CONFIRMED, OrderStatus.CANCELLED]

    def can_cancel(self) -> bool:
        return True


class ConfirmedState(OrderState):
    def status(self) -> OrderStatus:
        return OrderStatus.CONFIRMED

    def next_states(self) -> list[OrderStatus]:
        return [OrderStatus.PREPARING, OrderStatus.CANCELLED]

    def can_cancel(self) -> bool:
        return True


class PreparingState(OrderState):
    def status(self) -> OrderStatus:
        return OrderStatus.PREPARING

    def next_states(self) -> list[OrderStatus]:
        return [OrderStatus.READY]

    def can_cancel(self) -> bool:
        return False  # too late to cancel


class ReadyState(OrderState):
    def status(self) -> OrderStatus:
        return OrderStatus.READY

    def next_states(self) -> list[OrderStatus]:
        return [OrderStatus.PICKED_UP]

    def can_cancel(self) -> bool:
        return False


class PickedUpState(OrderState):
    def status(self) -> OrderStatus:
        return OrderStatus.PICKED_UP

    def next_states(self) -> list[OrderStatus]:
        return [OrderStatus.DELIVERED]

    def can_cancel(self) -> bool:
        return False


class DeliveredState(OrderState):
    def status(self) -> OrderStatus:
        return OrderStatus.DELIVERED

    def next_states(self) -> list[OrderStatus]:
        return []  # terminal state

    def can_cancel(self) -> bool:
        return False


class CancelledState(OrderState):
    def status(self) -> OrderStatus:
        return OrderStatus.CANCELLED

    def next_states(self) -> list[OrderStatus]:
        return []  # terminal state

    def can_cancel(self) -> bool:
        return False
```

### State Factory

```python
_STATE_MAP: dict[OrderStatus, type[OrderState]] = {
    OrderStatus.PLACED: PlacedState,
    OrderStatus.CONFIRMED: ConfirmedState,
    OrderStatus.PREPARING: PreparingState,
    OrderStatus.READY: ReadyState,
    OrderStatus.PICKED_UP: PickedUpState,
    OrderStatus.DELIVERED: DeliveredState,
    OrderStatus.CANCELLED: CancelledState,
}


def get_state(status: OrderStatus) -> OrderState:
    """Factory function to get a state instance."""
    return _STATE_MAP[status]()
```

### Order

```python
@dataclass
class OrderItem:
    """A line item in an order (snapshot of menu item at order time)."""
    item_id: str
    name: str
    price: float
    quantity: int

    @property
    def subtotal(self) -> float:
        return self.price * self.quantity


class Order:
    """A food delivery order with state-machine lifecycle."""

    _next_id: int = 0

    def __init__(self, customer_id: str, restaurant_id: str,
                 items: list[OrderItem], delivery_address: str,
                 delivery_location: Location,
                 payment_method: PaymentMethod) -> None:
        Order._next_id += 1
        self._id = f"ORD-{Order._next_id}"
        self._customer_id = customer_id
        self._restaurant_id = restaurant_id
        self._items = list(items)
        self._delivery_address = delivery_address
        self._delivery_location = delivery_location
        self._payment_method = payment_method

        self._state: OrderState = PlacedState()
        self._agent_id: str | None = None
        self._payment_status = PaymentStatus.PENDING
        self._created_at = int(time.time() * 1000)
        self._updated_at = self._created_at
        self._state_history: list[tuple[OrderStatus, int]] = [
            (OrderStatus.PLACED, self._created_at)
        ]

        # Pricing
        self._subtotal = sum(item.subtotal for item in items)
        self._service_fee = self._subtotal * SERVICE_FEE_RATE
        self._delivery_fee = DELIVERY_FEE_BASE
        self._total = self._subtotal + self._service_fee + self._delivery_fee
        self._cancellation_reason: str | None = None

    # ------------------------------------------------------------------
    # Properties
    # ------------------------------------------------------------------

    @property
    def id(self) -> str:
        return self._id

    @property
    def customer_id(self) -> str:
        return self._customer_id

    @property
    def restaurant_id(self) -> str:
        return self._restaurant_id

    @property
    def items(self) -> list[OrderItem]:
        return list(self._items)

    @property
    def delivery_address(self) -> str:
        return self._delivery_address

    @property
    def delivery_location(self) -> Location:
        return self._delivery_location

    @property
    def payment_method(self) -> PaymentMethod:
        return self._payment_method

    @property
    def status(self) -> OrderStatus:
        return self._state.status()

    @property
    def agent_id(self) -> str | None:
        return self._agent_id

    @property
    def subtotal(self) -> float:
        return self._subtotal

    @property
    def service_fee(self) -> float:
        return self._service_fee

    @property
    def delivery_fee(self) -> float:
        return self._delivery_fee

    @property
    def total(self) -> float:
        return self._total

    @property
    def payment_status(self) -> PaymentStatus:
        return self._payment_status

    @property
    def state_history(self) -> list[tuple[OrderStatus, int]]:
        return list(self._state_history)

    @property
    def cancellation_reason(self) -> str | None:
        return self._cancellation_reason

    # ------------------------------------------------------------------
    # State transitions
    # ------------------------------------------------------------------

    def transition_to(self, new_status: OrderStatus) -> bool:
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

        self._state = get_state(new_status)
        self._updated_at = int(time.time() * 1000)
        self._state_history.append((new_status, self._updated_at))
        return True

    def cancel(self, reason: str = "Customer requested") -> bool:
        """Cancel the order if the current state allows it."""
        if not self._state.can_cancel():
            print(f"  ✗ Cannot cancel order in {self._state.status().value} state")
            return False
        self._cancellation_reason = reason
        return self.transition_to(OrderStatus.CANCELLED)

    def assign_agent(self, agent_id: str) -> None:
        self._agent_id = agent_id

    def set_delivery_fee(self, fee: float) -> None:
        """Update delivery fee (based on actual distance)."""
        self._delivery_fee = fee
        self._total = self._subtotal + self._service_fee + self._delivery_fee

    def set_payment_status(self, status: PaymentStatus) -> None:
        self._payment_status = status

    def is_active(self) -> bool:
        return self._state.status() not in (
            OrderStatus.DELIVERED, OrderStatus.CANCELLED
        )

    # ------------------------------------------------------------------
    # Display
    # ------------------------------------------------------------------

    def __repr__(self) -> str:
        agent_str = f" 🚴{self._agent_id}" if self._agent_id else ""
        return (
            f"Order#{self._id}({self._customer_id} from "
            f"{self._restaurant_id} ${self._total:.2f} "
            f"[{self._state.status().value}]{agent_str})"
        )

    def display(self) -> None:
        print(f"\n  {'═'*55}")
        print(f"  ORDER — {self._id}")
        print(f"  {'═'*55}")
        print(f"  Customer:    {self._customer_id}")
        print(f"  Restaurant:  {self._restaurant_id}")
        print(f"  Status:      {self._state.status().value}")
        print(f"  Agent:       {self._agent_id or 'Unassigned'}")
        print(f"  Payment:     {self._payment_method.value} "
              f"({self._payment_status.value})")
        print(f"  Address:     {self._delivery_address}")
        print(f"  {'─'*55}")
        for item in self._items:
            print(f"  {item.name} x{item.quantity} "
                  f"= ${item.subtotal:.2f}")
        print(f"  {'─'*55}")
        print(f"  Subtotal:      ${self._subtotal:.2f}")
        print(f"  Service Fee:   ${self._service_fee:.2f}")
        print(f"  Delivery Fee:  ${self._delivery_fee:.2f}")
        print(f"  TOTAL:         ${self._total:.2f}")
        print(f"  {'═'*55}\n")
```

Each `OrderState` subclass defines what transitions are legal and whether cancellation is allowed. `PlacedState` and `ConfirmedState` allow cancellation, but once the restaurant starts preparing (`PreparingState`), it is too late — food is already being made. `transition_to` checks the current state's `next_states()` before allowing a change, making invalid transitions impossible. The `_state_history` records every transition with a timestamp, enabling the order tracker to compute elapsed times. `OrderItem` is a snapshot of the menu item at order time — even if the restaurant changes its price later, this order keeps the original price. The pricing model mirrors real delivery apps: item subtotal + service fee (percentage) + delivery fee (flat + distance).

---

## User, Customer & DeliveryAgent

```python
class User:
    """Base class for all users in the system."""

    def __init__(self, user_id: str, name: str, phone: str,
                 email: str, location: Location) -> None:
        self._user_id = user_id
        self._name = name
        self._phone = phone
        self._email = email
        self._location = location

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

    @property
    def location(self) -> Location:
        return self._location

    @location.setter
    def location(self, loc: Location) -> None:
        self._location = loc


class Customer(User):
    """A customer who places food orders."""

    def __init__(self, user_id: str, name: str, phone: str,
                 email: str, location: Location,
                 default_address: str = "") -> None:
        super().__init__(user_id, name, phone, email, location)
        self._default_address = default_address
        self._order_history: list[str] = []  # order IDs
        self._cart = Cart(user_id)
        self._wallet_balance: float = 0.0

    @property
    def default_address(self) -> str:
        return self._default_address

    @property
    def cart(self) -> Cart:
        return self._cart

    @property
    def order_history(self) -> list[str]:
        return list(self._order_history)

    @property
    def wallet_balance(self) -> float:
        return self._wallet_balance

    def add_to_order_history(self, order_id: str) -> None:
        self._order_history.append(order_id)

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
            f"Customer({self._name} 📍{self._location} "
            f"Orders: {len(self._order_history)} "
            f"Wallet: ${self._wallet_balance:.2f})"
        )


class DeliveryAgent(User):
    """A delivery agent who picks up and delivers orders."""

    def __init__(self, user_id: str, name: str, phone: str,
                 email: str, location: Location,
                 vehicle_type: str = "bike") -> None:
        super().__init__(user_id, name, phone, email, location)
        self._vehicle_type = vehicle_type
        self._status = AgentStatus.AVAILABLE
        self._current_order_id: str | None = None
        self._total_deliveries: int = 0
        self._rating: float = 4.5
        self._total_ratings: int = 0
        self._earnings: float = 0.0

    # ------------------------------------------------------------------
    # Properties
    # ------------------------------------------------------------------

    @property
    def vehicle_type(self) -> str:
        return self._vehicle_type

    @property
    def status(self) -> AgentStatus:
        return self._status

    @property
    def current_order_id(self) -> str | None:
        return self._current_order_id

    @property
    def total_deliveries(self) -> int:
        return self._total_deliveries

    @property
    def rating(self) -> float:
        return self._rating

    @property
    def earnings(self) -> float:
        return self._earnings

    # ------------------------------------------------------------------
    # Operations
    # ------------------------------------------------------------------

    def assign_order(self, order_id: str) -> None:
        self._current_order_id = order_id
        self._status = AgentStatus.BUSY

    def complete_delivery(self, delivery_fee: float) -> None:
        self._current_order_id = None
        self._status = AgentStatus.AVAILABLE
        self._total_deliveries += 1
        self._earnings += delivery_fee

    def release_order(self) -> None:
        """Release current order (declined or cancelled)."""
        self._current_order_id = None
        self._status = AgentStatus.AVAILABLE

    def go_offline(self) -> None:
        self._status = AgentStatus.OFFLINE

    def go_online(self) -> None:
        if self._current_order_id is None:
            self._status = AgentStatus.AVAILABLE

    def add_rating(self, score: float) -> None:
        score = max(1.0, min(5.0, score))
        self._total_ratings += 1
        self._rating += (score - self._rating) / self._total_ratings

    def __repr__(self) -> str:
        status_emoji = {
            AgentStatus.AVAILABLE: "🟢",
            AgentStatus.BUSY: "🔴",
            AgentStatus.OFFLINE: "⚫",
        }
        return (
            f"Agent({self._name} {status_emoji[self._status]} "
            f"⭐{self._rating:.1f} "
            f"📦{self._total_deliveries} "
            f"💰${self._earnings:.2f})"
        )
```

`User` is the base class with shared identity and location data. `Customer` owns a `Cart` and maintains a wallet balance for in-app payments. `DeliveryAgent` tracks availability, current assignment, cumulative earnings, and a running average rating. The agent state machine is simple: `AVAILABLE` → `BUSY` (when assigned) → `AVAILABLE` (after delivery). `complete_delivery` credits the agent's earnings with the delivery fee. `release_order` handles declined or cancelled assignments without affecting delivery stats.

---

## Delivery Assignment (Strategy Pattern)

Different dispatch algorithms optimise for different goals — minimise delivery time, balance agent workload, or provide fair rotation.

### Abstract Strategy

```python
class DeliveryStrategy(ABC):
    """Defines how a delivery agent is selected for an order."""

    @abstractmethod
    def select_agent(self, order: Order,
                     restaurant: Restaurant,
                     agents: list[DeliveryAgent]) -> DeliveryAgent | None:
        """
        Select the best available agent for the order.
        Returns None if no agent is available.
        """
        ...

    @abstractmethod
    def name(self) -> str:
        ...
```

### Nearest Agent Strategy

```python
class NearestAgentStrategy(DeliveryStrategy):
    """
    Assign the available agent closest to the restaurant.

    Minimises pickup time, which reduces total delivery time.
    """

    def select_agent(self, order: Order,
                     restaurant: Restaurant,
                     agents: list[DeliveryAgent]) -> DeliveryAgent | None:
        available = [a for a in agents
                     if a.status == AgentStatus.AVAILABLE]
        if not available:
            return None

        # Sort by distance to restaurant
        return min(
            available,
            key=lambda a: a.location.distance_to(restaurant.location)
        )

    def name(self) -> str:
        return "Nearest Agent"
```

### Least Busy Strategy

```python
class LeastBusyStrategy(DeliveryStrategy):
    """
    Assign the available agent with the fewest total deliveries.

    Balances workload across agents.
    """

    def select_agent(self, order: Order,
                     restaurant: Restaurant,
                     agents: list[DeliveryAgent]) -> DeliveryAgent | None:
        available = [a for a in agents
                     if a.status == AgentStatus.AVAILABLE]
        if not available:
            return None

        return min(available, key=lambda a: a.total_deliveries)

    def name(self) -> str:
        return "Least Busy"
```

### Highest Rated Strategy

```python
class HighestRatedStrategy(DeliveryStrategy):
    """
    Assign the available agent with the highest rating.

    Prioritises service quality over speed or fairness.
    """

    def select_agent(self, order: Order,
                     restaurant: Restaurant,
                     agents: list[DeliveryAgent]) -> DeliveryAgent | None:
        available = [a for a in agents
                     if a.status == AgentStatus.AVAILABLE]
        if not available:
            return None

        return max(available, key=lambda a: a.rating)

    def name(self) -> str:
        return "Highest Rated"
```

`NearestAgentStrategy` minimises the time between order-ready and pickup — the agent closest to the restaurant reaches it fastest. `LeastBusyStrategy` ensures fair distribution of work; new agents or those who have done fewer deliveries get priority. `HighestRatedStrategy` optimises for customer satisfaction by always sending the best-reviewed agent. All three implement the same interface, so the system can switch strategies at runtime or even per-order based on customer tier.

---

## Payment Processing (Strategy Pattern)

```python
class PaymentStrategy(ABC):
    """Defines how a payment is processed."""

    @abstractmethod
    def pay(self, amount: float, customer: Customer) -> tuple[bool, str]:
        """
        Process the payment.
        Returns (success, transaction_message).
        """
        ...

    @abstractmethod
    def refund(self, amount: float, customer: Customer) -> tuple[bool, str]:
        """Process a refund."""
        ...

    @abstractmethod
    def method(self) -> PaymentMethod:
        ...


class CreditCardPayment(PaymentStrategy):
    """Simulates credit card payment processing."""

    def pay(self, amount: float, customer: Customer) -> tuple[bool, str]:
        # In production, this would call a payment gateway API
        print(f"  💳 Charging ${amount:.2f} to credit card "
              f"for {customer.name}")
        return True, f"CC-TXN-{int(time.time())}"

    def refund(self, amount: float, customer: Customer) -> tuple[bool, str]:
        print(f"  💳 Refunding ${amount:.2f} to credit card "
              f"for {customer.name}")
        return True, f"CC-REFUND-{int(time.time())}"

    def method(self) -> PaymentMethod:
        return PaymentMethod.CREDIT_CARD


class WalletPayment(PaymentStrategy):
    """Processes payment from in-app wallet balance."""

    def pay(self, amount: float, customer: Customer) -> tuple[bool, str]:
        if customer.debit_wallet(amount):
            print(f"  👛 Debited ${amount:.2f} from wallet "
                  f"for {customer.name}")
            return True, f"WALLET-TXN-{int(time.time())}"
        print(f"  ✗ Insufficient wallet balance. "
              f"Available: ${customer.wallet_balance:.2f}")
        return False, "Insufficient balance"

    def refund(self, amount: float, customer: Customer) -> tuple[bool, str]:
        customer.credit_wallet(amount)
        print(f"  👛 Refunded ${amount:.2f} to wallet "
              f"for {customer.name}")
        return True, f"WALLET-REFUND-{int(time.time())}"

    def method(self) -> PaymentMethod:
        return PaymentMethod.WALLET


class CashOnDeliveryPayment(PaymentStrategy):
    """Cash on delivery — payment is collected at the door."""

    def pay(self, amount: float, customer: Customer) -> tuple[bool, str]:
        print(f"  💵 Cash on delivery: ${amount:.2f} to be collected "
              f"from {customer.name}")
        return True, f"COD-{int(time.time())}"

    def refund(self, amount: float, customer: Customer) -> tuple[bool, str]:
        # COD refunds are complex in practice; here we credit wallet
        customer.credit_wallet(amount)
        print(f"  💵 COD refund: ${amount:.2f} credited to wallet "
              f"for {customer.name}")
        return True, f"COD-REFUND-{int(time.time())}"

    def method(self) -> PaymentMethod:
        return PaymentMethod.CASH_ON_DELIVERY


# Payment strategy factory
PAYMENT_STRATEGIES: dict[PaymentMethod, type[PaymentStrategy]] = {
    PaymentMethod.CREDIT_CARD: CreditCardPayment,
    PaymentMethod.WALLET: WalletPayment,
    PaymentMethod.CASH_ON_DELIVERY: CashOnDeliveryPayment,
}


def get_payment_strategy(method: PaymentMethod) -> PaymentStrategy:
    strategy_cls = PAYMENT_STRATEGIES.get(method)
    if strategy_cls is None:
        raise ValueError(f"Unsupported payment method: {method.value}")
    return strategy_cls()
```

Each `PaymentStrategy` handles `pay` and `refund` for its specific method. `CreditCardPayment` simulates a gateway call — in production, this would integrate with Stripe or a similar provider. `WalletPayment` checks and debits the customer's in-app balance, making it the only strategy that can fail due to insufficient funds. `CashOnDeliveryPayment` defers payment to delivery time — the transaction is recorded but no money moves until the agent collects cash. The factory `get_payment_strategy` maps enum values to strategy instances, making it easy to add new payment methods.

---

## Notification System (Observer Pattern)

### Listener Interface

```python
class OrderEventListener(ABC):
    """Observer interface for order lifecycle events."""

    @abstractmethod
    def on_order_placed(self, order: Order) -> None:
        ...

    @abstractmethod
    def on_order_confirmed(self, order: Order) -> None:
        ...

    @abstractmethod
    def on_order_preparing(self, order: Order) -> None:
        ...

    @abstractmethod
    def on_order_ready(self, order: Order) -> None:
        ...

    @abstractmethod
    def on_order_picked_up(self, order: Order) -> None:
        ...

    @abstractmethod
    def on_order_delivered(self, order: Order) -> None:
        ...

    @abstractmethod
    def on_order_cancelled(self, order: Order, reason: str) -> None:
        ...

    @abstractmethod
    def on_agent_assigned(self, order: Order,
                          agent: DeliveryAgent) -> None:
        ...
```

### Order Logger

```python
class OrderLogger(OrderEventListener):
    """Logs all order events to console."""

    def __init__(self) -> None:
        self._log: list[str] = []

    def _log_event(self, entry: str) -> None:
        self._log.append(entry)
        print(entry)

    def on_order_placed(self, order: Order) -> None:
        self._log_event(
            f"  📝 ORDER PLACED: {order.id} by {order.customer_id} "
            f"from {order.restaurant_id} — ${order.total:.2f}"
        )

    def on_order_confirmed(self, order: Order) -> None:
        self._log_event(
            f"  ✅ ORDER CONFIRMED: {order.id} — "
            f"restaurant accepted"
        )

    def on_order_preparing(self, order: Order) -> None:
        self._log_event(
            f"  👨‍🍳 PREPARING: {order.id} — "
            f"kitchen started"
        )

    def on_order_ready(self, order: Order) -> None:
        self._log_event(
            f"  📦 READY: {order.id} — "
            f"waiting for pickup"
        )

    def on_order_picked_up(self, order: Order) -> None:
        self._log_event(
            f"  🚴 PICKED UP: {order.id} by "
            f"{order.agent_id}"
        )

    def on_order_delivered(self, order: Order) -> None:
        self._log_event(
            f"  🏠 DELIVERED: {order.id} to "
            f"{order.customer_id}"
        )

    def on_order_cancelled(self, order: Order, reason: str) -> None:
        self._log_event(
            f"  ❌ CANCELLED: {order.id} — {reason}"
        )

    def on_agent_assigned(self, order: Order,
                          agent: DeliveryAgent) -> None:
        self._log_event(
            f"  🚴 AGENT ASSIGNED: {agent.name} → "
            f"{order.id}"
        )

    def print_log(self, last_n: int = 20) -> None:
        print(f"\n  Event Log (last {last_n}):")
        for entry in self._log[-last_n:]:
            print(entry)

    def clear(self) -> None:
        self._log.clear()
```

### Customer Notifier

```python
class CustomerNotifier(OrderEventListener):
    """Sends notifications to customers (simulated)."""

    def __init__(self) -> None:
        self._notifications: list[tuple[str, str]] = []

    def on_order_placed(self, order: Order) -> None:
        self._notify(order.customer_id,
                     f"Your order {order.id} has been placed!")

    def on_order_confirmed(self, order: Order) -> None:
        self._notify(order.customer_id,
                     f"Your order {order.id} is confirmed by the restaurant.")

    def on_order_preparing(self, order: Order) -> None:
        self._notify(order.customer_id,
                     f"Your order {order.id} is being prepared.")

    def on_order_ready(self, order: Order) -> None:
        self._notify(order.customer_id,
                     f"Your order {order.id} is ready for pickup!")

    def on_order_picked_up(self, order: Order) -> None:
        self._notify(order.customer_id,
                     f"Your order {order.id} is on the way! "
                     f"Agent: {order.agent_id}")

    def on_order_delivered(self, order: Order) -> None:
        self._notify(order.customer_id,
                     f"Your order {order.id} has been delivered. Enjoy!")

    def on_order_cancelled(self, order: Order, reason: str) -> None:
        self._notify(order.customer_id,
                     f"Your order {order.id} was cancelled: {reason}")

    def on_agent_assigned(self, order: Order,
                          agent: DeliveryAgent) -> None:
        self._notify(order.customer_id,
                     f"Agent {agent.name} is assigned to your order "
                     f"{order.id}.")

    def _notify(self, customer_id: str, message: str) -> None:
        self._notifications.append((customer_id, message))
        print(f"  📱 [{customer_id}]: {message}")
```

`OrderEventListener` defines eight hooks — one for each meaningful lifecycle event plus agent assignment. `OrderLogger` acts as an audit trail, recording every event with emoji markers. `CustomerNotifier` simulates push notifications — in production, each `_notify` call would trigger an SMS, push notification, or email. Both observers coexist independently because the observer pattern decouples event production from consumption — adding a new observer (e.g., an analytics tracker) requires zero changes to the existing code.

---

## Order Tracker

```python
class OrderTracker:
    """
    Tracks orders and computes estimated delivery times.

    Combines restaurant prep time, agent travel time to restaurant,
    and agent travel time to customer.
    """

    def __init__(self) -> None:
        self._orders: dict[str, Order] = {}

    def track_order(self, order: Order) -> None:
        self._orders[order.id] = order

    def get_order(self, order_id: str) -> Order | None:
        return self._orders.get(order_id)

    @property
    def active_orders(self) -> list[Order]:
        return [o for o in self._orders.values() if o.is_active()]

    @property
    def all_orders(self) -> list[Order]:
        return list(self._orders.values())

    def estimate_delivery_time(self, order: Order,
                               restaurant: Restaurant,
                               agent: DeliveryAgent | None) -> int:
        """
        Estimate total delivery time in minutes.

        Components:
        1. Prep time = max item prep + queue delay
        2. Pickup time = agent distance to restaurant
        3. Delivery time = restaurant distance to customer
        """
        # Prep time
        item_ids = [item.item_id for item in order.items]
        prep_time = restaurant.estimated_prep_time(item_ids)

        # Pickup travel time (agent → restaurant)
        pickup_time = 0
        if agent:
            dist_to_restaurant = agent.location.distance_to(
                restaurant.location
            )
            pickup_time = BASE_DELIVERY_TIME + int(dist_to_restaurant * TIME_PER_KM)

        # Delivery travel time (restaurant → customer)
        dist_to_customer = restaurant.location.distance_to(
            order.delivery_location
        )
        delivery_time = BASE_DELIVERY_TIME + int(dist_to_customer * TIME_PER_KM)

        return prep_time + pickup_time + delivery_time

    def compute_delivery_fee(self, restaurant: Restaurant,
                             delivery_location: Location) -> float:
        """Compute delivery fee based on distance."""
        distance = restaurant.location.distance_to(delivery_location)
        if distance > MAX_DELIVERY_RADIUS_KM:
            return -1.0  # out of range
        return DELIVERY_FEE_BASE + (distance * DELIVERY_FEE_PER_KM)

    def display_active(self) -> None:
        active = self.active_orders
        print(f"\n  {'═'*55}")
        print(f"  ORDER TRACKER — {len(active)} active orders")
        print(f"  {'═'*55}")
        for order in active:
            print(f"  {order}")
        if not active:
            print("  No active orders")
        print()

    def display_history(self, last_n: int = 10) -> None:
        orders = self.all_orders[-last_n:]
        print(f"\n  {'═'*55}")
        print(f"  ORDER HISTORY — last {last_n}")
        print(f"  {'═'*55}")
        for order in orders:
            print(f"  {order}")
        print()
```

`OrderTracker` is the central registry of all orders. `estimate_delivery_time` breaks the total wait into three components: kitchen preparation, agent travel to the restaurant, and agent travel to the customer. Each component is computed independently — prep time comes from the restaurant's current queue and menu items, while travel times use the `Location.distance_to` method. `compute_delivery_fee` calculates distance-based pricing and rejects orders beyond the delivery radius by returning -1.

---

## Food Delivery System (Facade)

The `FoodDeliverySystem` ties everything together. It manages restaurants, customers, agents, carts, orders, payments, dispatch, and event notifications through a single unified API.

```python
class FoodDeliverySystem:
    """
    Central food delivery platform.
    Acts as a Facade over restaurants, customers, agents,
    orders, payments, and dispatch.
    """

    def __init__(self,
                 delivery_strategy: DeliveryStrategy | None = None) -> None:
        self._restaurants: dict[str, Restaurant] = {}
        self._customers: dict[str, Customer] = {}
        self._agents: dict[str, DeliveryAgent] = {}
        self._delivery_strategy = delivery_strategy or NearestAgentStrategy()
        self._tracker = OrderTracker()
        self._listeners: list[OrderEventListener] = []
        self._tick = 0

    # ------------------------------------------------------------------
    # Properties
    # ------------------------------------------------------------------

    @property
    def tick(self) -> int:
        return self._tick

    @property
    def tracker(self) -> OrderTracker:
        return self._tracker

    # ------------------------------------------------------------------
    # Registration
    # ------------------------------------------------------------------

    def register_restaurant(self, restaurant: Restaurant) -> None:
        self._restaurants[restaurant.id] = restaurant

    def register_customer(self, customer: Customer) -> None:
        self._customers[customer.user_id] = customer

    def register_agent(self, agent: DeliveryAgent) -> None:
        self._agents[agent.user_id] = agent

    def add_listener(self, listener: OrderEventListener) -> None:
        self._listeners.append(listener)

    # ------------------------------------------------------------------
    # Getters
    # ------------------------------------------------------------------

    def get_restaurant(self, restaurant_id: str) -> Restaurant | None:
        return self._restaurants.get(restaurant_id)

    def get_customer(self, customer_id: str) -> Customer | None:
        return self._customers.get(customer_id)

    def get_agent(self, agent_id: str) -> DeliveryAgent | None:
        return self._agents.get(agent_id)

    def get_order(self, order_id: str) -> Order | None:
        return self._tracker.get_order(order_id)

    def list_restaurants(self, open_only: bool = True) -> list[Restaurant]:
        if open_only:
            return [r for r in self._restaurants.values() if r.is_open]
        return list(self._restaurants.values())

    def available_agents(self) -> list[DeliveryAgent]:
        return [a for a in self._agents.values()
                if a.status == AgentStatus.AVAILABLE]

    # ------------------------------------------------------------------
    # Cart operations
    # ------------------------------------------------------------------

    def add_to_cart(self, customer_id: str, restaurant_id: str,
                    item_id: str, quantity: int = 1) -> bool:
        """Add a menu item to the customer's cart."""
        customer = self._customers.get(customer_id)
        if customer is None:
            print(f"  ✗ Customer '{customer_id}' not found")
            return False

        restaurant = self._restaurants.get(restaurant_id)
        if restaurant is None:
            print(f"  ✗ Restaurant '{restaurant_id}' not found")
            return False

        if not restaurant.is_open:
            print(f"  ✗ {restaurant.name} is currently closed")
            return False

        menu_item = restaurant.menu.get_item(item_id)
        if menu_item is None:
            print(f"  ✗ Item '{item_id}' not found on menu")
            return False

        return customer.cart.add_item(menu_item, restaurant_id, quantity)

    def clear_cart(self, customer_id: str) -> None:
        customer = self._customers.get(customer_id)
        if customer:
            customer.cart.clear()

    # ------------------------------------------------------------------
    # Order placement
    # ------------------------------------------------------------------

    def place_order(self, customer_id: str,
                    payment_method: PaymentMethod,
                    delivery_address: str = "",
                    delivery_location: Location | None = None
                    ) -> Order | None:
        """
        Place an order from the customer's cart.
        Validates, processes payment, creates order, dispatches agent.
        Returns the order or None if failed.
        """
        self._tick += 1

        # --- Validate customer ---
        customer = self._customers.get(customer_id)
        if customer is None:
            print(f"  ✗ Customer '{customer_id}' not found")
            return None

        cart = customer.cart
        if cart.is_empty:
            print(f"  ✗ Cart is empty")
            return None

        # --- Validate restaurant ---
        restaurant = self._restaurants.get(cart.restaurant_id)
        if restaurant is None:
            print(f"  ✗ Restaurant not found")
            return None

        if not restaurant.is_open:
            print(f"  ✗ {restaurant.name} is currently closed")
            return None

        # --- Minimum order check ---
        if cart.total < MIN_ORDER_AMOUNT:
            print(f"  ✗ Minimum order is ${MIN_ORDER_AMOUNT:.2f}. "
                  f"Cart total: ${cart.total:.2f}")
            return None

        # --- Resolve delivery location ---
        addr = delivery_address or customer.default_address
        loc = delivery_location or customer.location
        if not addr:
            print(f"  ✗ No delivery address provided")
            return None

        # --- Compute delivery fee ---
        delivery_fee = self._tracker.compute_delivery_fee(
            restaurant, loc
        )
        if delivery_fee < 0:
            print(f"  ✗ Delivery location is out of range "
                  f"(max {MAX_DELIVERY_RADIUS_KM} km)")
            return None

        # --- Build order items (snapshot) ---
        order_items = [
            OrderItem(
                item_id=ci.menu_item.item_id,
                name=ci.menu_item.name,
                price=ci.menu_item.price,
                quantity=ci.quantity,
            )
            for ci in cart.items
        ]

        # --- Create order ---
        order = Order(
            customer_id=customer_id,
            restaurant_id=restaurant.id,
            items=order_items,
            delivery_address=addr,
            delivery_location=loc,
            payment_method=payment_method,
        )
        order.set_delivery_fee(delivery_fee)

        # --- Process payment ---
        if payment_method != PaymentMethod.CASH_ON_DELIVERY:
            strategy = get_payment_strategy(payment_method)
            success, txn_msg = strategy.pay(order.total, customer)
            if not success:
                print(f"  ✗ Payment failed: {txn_msg}")
                return None
            order.set_payment_status(PaymentStatus.COMPLETED)
        # COD stays PENDING until delivery

        # --- Register order ---
        self._tracker.track_order(order)
        customer.add_to_order_history(order.id)
        restaurant.add_active_order(order.id)
        cart.clear()

        # --- Notify ---
        self._notify_placed(order)

        return order

    # ------------------------------------------------------------------
    # Order lifecycle management
    # ------------------------------------------------------------------

    def confirm_order(self, order_id: str) -> bool:
        """Restaurant confirms the order."""
        order = self._tracker.get_order(order_id)
        if order is None:
            print(f"  ✗ Order '{order_id}' not found")
            return False

        if not order.transition_to(OrderStatus.CONFIRMED):
            return False

        self._notify_confirmed(order)

        # Dispatch agent
        self._dispatch_agent(order)
        return True

    def reject_order(self, order_id: str,
                     reason: str = "Restaurant busy") -> bool:
        """Restaurant rejects the order."""
        order = self._tracker.get_order(order_id)
        if order is None:
            return False

        restaurant = self._restaurants.get(order.restaurant_id)
        if restaurant:
            restaurant.remove_active_order(order_id)

        if order.cancel(reason):
            # Refund payment
            self._process_refund(order)
            self._notify_cancelled(order, reason)
            return True
        return False

    def start_preparing(self, order_id: str) -> bool:
        """Restaurant starts preparing the order."""
        order = self._tracker.get_order(order_id)
        if order is None:
            return False

        if order.transition_to(OrderStatus.PREPARING):
            self._notify_preparing(order)
            return True
        return False

    def mark_ready(self, order_id: str) -> bool:
        """Restaurant marks order as ready for pickup."""
        order = self._tracker.get_order(order_id)
        if order is None:
            return False

        if order.transition_to(OrderStatus.READY):
            self._notify_ready(order)
            return True
        return False

    def pickup_order(self, order_id: str) -> bool:
        """Agent picks up the order from the restaurant."""
        order = self._tracker.get_order(order_id)
        if order is None:
            return False

        if order.transition_to(OrderStatus.PICKED_UP):
            self._notify_picked_up(order)
            return True
        return False

    def deliver_order(self, order_id: str) -> bool:
        """Agent delivers the order to the customer."""
        order = self._tracker.get_order(order_id)
        if order is None:
            return False

        if not order.transition_to(OrderStatus.DELIVERED):
            return False

        # Complete agent delivery
        if order.agent_id:
            agent = self._agents.get(order.agent_id)
            if agent:
                agent.complete_delivery(order.delivery_fee)

        # Remove from restaurant active orders
        restaurant = self._restaurants.get(order.restaurant_id)
        if restaurant:
            restaurant.remove_active_order(order_id)

        # Process COD payment
        if order.payment_method == PaymentMethod.CASH_ON_DELIVERY:
            order.set_payment_status(PaymentStatus.COMPLETED)

        self._notify_delivered(order)
        return True

    def cancel_order(self, order_id: str,
                     reason: str = "Customer requested") -> bool:
        """Cancel an order if still cancellable."""
        order = self._tracker.get_order(order_id)
        if order is None:
            print(f"  ✗ Order '{order_id}' not found")
            return False

        if not order.cancel(reason):
            return False

        # Release agent
        if order.agent_id:
            agent = self._agents.get(order.agent_id)
            if agent:
                agent.release_order()

        # Remove from restaurant
        restaurant = self._restaurants.get(order.restaurant_id)
        if restaurant:
            restaurant.remove_active_order(order_id)

        # Refund
        self._process_refund(order)
        self._notify_cancelled(order, reason)
        return True

    # ------------------------------------------------------------------
    # Agent dispatch
    # ------------------------------------------------------------------

    def _dispatch_agent(self, order: Order) -> bool:
        """Dispatch a delivery agent for the order."""
        restaurant = self._restaurants.get(order.restaurant_id)
        if restaurant is None:
            return False

        agents = list(self._agents.values())
        agent = self._delivery_strategy.select_agent(
            order, restaurant, agents
        )

        if agent is None:
            print(f"  ⚠ No agents available for {order.id}")
            return False

        agent.assign_order(order.id)
        order.assign_agent(agent.user_id)
        self._notify_agent_assigned(order, agent)
        return True

    # ------------------------------------------------------------------
    # Estimated delivery time
    # ------------------------------------------------------------------

    def get_eta(self, order_id: str) -> int | None:
        """Get estimated delivery time in minutes."""
        order = self._tracker.get_order(order_id)
        if order is None or not order.is_active():
            return None

        restaurant = self._restaurants.get(order.restaurant_id)
        if restaurant is None:
            return None

        agent = None
        if order.agent_id:
            agent = self._agents.get(order.agent_id)

        return self._tracker.estimate_delivery_time(
            order, restaurant, agent
        )

    # ------------------------------------------------------------------
    # Payment
    # ------------------------------------------------------------------

    def _process_refund(self, order: Order) -> None:
        """Refund a cancelled order's payment."""
        if order.payment_status != PaymentStatus.COMPLETED:
            return  # nothing to refund

        customer = self._customers.get(order.customer_id)
        if customer is None:
            return

        strategy = get_payment_strategy(order.payment_method)
        success, _ = strategy.refund(order.total, customer)
        if success:
            order.set_payment_status(PaymentStatus.REFUNDED)

    # ------------------------------------------------------------------
    # Notifications
    # ------------------------------------------------------------------

    def _notify_placed(self, order: Order) -> None:
        for listener in self._listeners:
            listener.on_order_placed(order)

    def _notify_confirmed(self, order: Order) -> None:
        for listener in self._listeners:
            listener.on_order_confirmed(order)

    def _notify_preparing(self, order: Order) -> None:
        for listener in self._listeners:
            listener.on_order_preparing(order)

    def _notify_ready(self, order: Order) -> None:
        for listener in self._listeners:
            listener.on_order_ready(order)

    def _notify_picked_up(self, order: Order) -> None:
        for listener in self._listeners:
            listener.on_order_picked_up(order)

    def _notify_delivered(self, order: Order) -> None:
        for listener in self._listeners:
            listener.on_order_delivered(order)

    def _notify_cancelled(self, order: Order, reason: str) -> None:
        for listener in self._listeners:
            listener.on_order_cancelled(order, reason)

    def _notify_agent_assigned(self, order: Order,
                                agent: DeliveryAgent) -> None:
        for listener in self._listeners:
            listener.on_agent_assigned(order, agent)

    # ------------------------------------------------------------------
    # Display
    # ------------------------------------------------------------------

    def display_status(self) -> None:
        print(f"\n  {'═'*60}")
        print(f"  FOOD DELIVERY SYSTEM — Tick {self._tick}")
        print(f"  Dispatch: {self._delivery_strategy.name()}")
        print(f"  {'═'*60}")

        print(f"\n  RESTAURANTS:")
        print(f"  {'─'*55}")
        for r in self._restaurants.values():
            print(f"  {r}")

        print(f"\n  AGENTS:")
        print(f"  {'─'*55}")
        for a in self._agents.values():
            print(f"  {a}")

        print(f"\n  CUSTOMERS:")
        print(f"  {'─'*55}")
        for c in self._customers.values():
            print(f"  {c}")

        self._tracker.display_active()
```

`place_order` follows a strict pipeline: validate customer and cart → validate restaurant is open → check minimum order → compute delivery fee (distance-based) → snapshot cart items into order items → process payment → register order → notify. Payment is processed before the order is registered — if payment fails, no order is created. Cart is cleared after successful placement. `confirm_order` triggers agent dispatch — the restaurant first confirms it can fulfill the order, then the system finds a delivery agent. The lifecycle methods (`start_preparing`, `mark_ready`, `pickup_order`, `deliver_order`) each delegate to the order's state machine, which enforces valid transitions. `cancel_order` cleans up everything — releases the agent, removes the order from the restaurant's queue, and refunds the payment. `_process_refund` only acts on completed payments; COD orders with pending status have nothing to refund.

---

## Simulation Engine

A helper that scripts realistic delivery scenarios for demonstration.

```python
from typing import Callable


class DeliveryScenario:
    """A sequence of timed delivery actions for simulation."""

    def __init__(self, system: FoodDeliverySystem) -> None:
        self._system = system
        self._events: list[tuple[int, str, Callable]] = []

    def at_tick(self, tick: int, description: str,
                action: Callable) -> "DeliveryScenario":
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
        print("  FOOD DELIVERY SIMULATION START")
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
        print("  FOOD DELIVERY SIMULATION END")
        print("═" * 62)
```

`DeliveryScenario` lets you script events at specific ticks: "at tick 3, the restaurant confirms the order". `run` executes the simulation in order, firing events at the right ticks and displaying the system state at configurable intervals. The scenario extends a few ticks past the last event to allow final state changes to settle.

---

## Putting It All Together

### Basic Simulation

```python
def main() -> None:
    # Reset order counter for clean start
    Order._next_id = 0
    Restaurant._next_id = 0

    # Create the system
    system = FoodDeliverySystem(
        delivery_strategy=NearestAgentStrategy()
    )

    # Attach observers
    logger = OrderLogger()
    notifier = CustomerNotifier()
    system.add_listener(logger)
    system.add_listener(notifier)

    # --- Register restaurants ---
    burger_place = Restaurant(
        "Burger Palace",
        Location(27.7172, 85.3240),
        "American"
    )
    burger_place.menu.add_item(MenuItem(
        "BP-1", "Classic Burger", 8.99,
        ItemCategory.MAIN_COURSE, 12
    ))
    burger_place.menu.add_item(MenuItem(
        "BP-2", "Cheese Fries", 4.99,
        ItemCategory.SIDE, 8
    ))
    burger_place.menu.add_item(MenuItem(
        "BP-3", "Milkshake", 5.49,
        ItemCategory.BEVERAGE, 5
    ))
    burger_place.menu.add_item(MenuItem(
        "BP-4", "Veggie Burger", 9.49,
        ItemCategory.MAIN_COURSE, 14, is_vegetarian=True
    ))
    system.register_restaurant(burger_place)

    sushi_bar = Restaurant(
        "Sushi Master",
        Location(27.7100, 85.3300),
        "Japanese"
    )
    sushi_bar.menu.add_item(MenuItem(
        "SM-1", "Salmon Roll", 12.99,
        ItemCategory.MAIN_COURSE, 15
    ))
    sushi_bar.menu.add_item(MenuItem(
        "SM-2", "Miso Soup", 3.99,
        ItemCategory.APPETIZER, 5, is_vegetarian=True
    ))
    sushi_bar.menu.add_item(MenuItem(
        "SM-3", "Tempura", 8.49,
        ItemCategory.APPETIZER, 10
    ))
    sushi_bar.menu.add_item(MenuItem(
        "SM-4", "Green Tea", 2.49,
        ItemCategory.BEVERAGE, 3, is_vegetarian=True
    ))
    system.register_restaurant(sushi_bar)

    # --- Register customers ---
    alice = Customer(
        "alice", "Alice Johnson", "555-0101",
        "alice@email.com", Location(27.7200, 85.3180),
        "123 Main Street"
    )
    alice.top_up_wallet(200.0)
    system.register_customer(alice)

    bob = Customer(
        "bob", "Bob Smith", "555-0102",
        "bob@email.com", Location(27.7150, 85.3350),
        "456 Oak Avenue"
    )
    system.register_customer(bob)

    # --- Register delivery agents ---
    agent1 = DeliveryAgent(
        "agent-1", "Dave Rider", "555-0201",
        "dave@delivery.com", Location(27.7180, 85.3220),
        "bike"
    )
    agent2 = DeliveryAgent(
        "agent-2", "Emma Swift", "555-0202",
        "emma@delivery.com", Location(27.7120, 85.3280),
        "scooter"
    )
    system.register_agent(agent1)
    system.register_agent(agent2)

    # ╔══════════════════════════════════════════╗
    # ║       FOOD DELIVERY SYSTEM               ║
    # ╚══════════════════════════════════════════╝

    print("╔══════════════════════════════════════════╗")
    print("║       FOOD DELIVERY SYSTEM               ║")
    print("║  Restaurants: Burger Palace, Sushi Master ║")
    print("║  Strategy: Nearest Agent                  ║")
    print("╚══════════════════════════════════════════╝")

    # --- Alice orders from Burger Palace ---
    print("\n  --- Alice browses Burger Palace menu ---")
    burger_place.menu.display()

    system.add_to_cart("alice", burger_place.id, "BP-1", 2)  # 2 Classic Burgers
    system.add_to_cart("alice", burger_place.id, "BP-2", 1)  # 1 Cheese Fries
    system.add_to_cart("alice", burger_place.id, "BP-3", 2)  # 2 Milkshakes
    alice.cart.display()

    order1 = system.place_order(
        "alice", PaymentMethod.WALLET, "123 Main Street"
    )

    # Restaurant confirms and prepares
    if order1:
        eta = system.get_eta(order1.id)
        print(f"\n  ⏱ Estimated delivery time: {eta} minutes")

        system.confirm_order(order1.id)
        system.start_preparing(order1.id)
        system.mark_ready(order1.id)
        system.pickup_order(order1.id)
        system.deliver_order(order1.id)

    # --- Bob orders from Sushi Master ---
    print("\n  --- Bob orders from Sushi Master ---")
    system.add_to_cart("bob", sushi_bar.id, "SM-1", 1)  # Salmon Roll
    system.add_to_cart("bob", sushi_bar.id, "SM-2", 2)  # 2 Miso Soup
    system.add_to_cart("bob", sushi_bar.id, "SM-3", 1)  # Tempura

    order2 = system.place_order(
        "bob", PaymentMethod.CREDIT_CARD, "456 Oak Avenue"
    )

    if order2:
        system.confirm_order(order2.id)
        system.start_preparing(order2.id)

        # Bob cancels — but too late (already preparing)
        print("\n  --- Bob tries to cancel ---")
        cancelled = system.cancel_order(order2.id, "Changed my mind")
        print(f"  Cancel result: {'success' if cancelled else 'FAILED (too late)'}")

        # Continue delivery
        system.mark_ready(order2.id)
        system.pickup_order(order2.id)
        system.deliver_order(order2.id)

    # --- Final system status ---
    system.display_status()
    system.tracker.display_history()


if __name__ == "__main__":
    main()
```

### Scripted Scenario

```python
def main_scenario() -> None:
    Order._next_id = 0
    Restaurant._next_id = 0

    system = FoodDeliverySystem(
        delivery_strategy=LeastBusyStrategy()
    )
    logger = OrderLogger()
    system.add_listener(logger)

    # Setup restaurant
    pizza = Restaurant(
        "Pizza Paradise",
        Location(27.7172, 85.3240),
        "Italian"
    )
    pizza.menu.add_item(MenuItem(
        "PP-1", "Margherita", 10.99,
        ItemCategory.MAIN_COURSE, 18, is_vegetarian=True
    ))
    pizza.menu.add_item(MenuItem(
        "PP-2", "Pepperoni", 12.99,
        ItemCategory.MAIN_COURSE, 18
    ))
    pizza.menu.add_item(MenuItem(
        "PP-3", "Garlic Bread", 4.99,
        ItemCategory.SIDE, 8, is_vegetarian=True
    ))
    pizza.menu.add_item(MenuItem(
        "PP-4", "Cola", 2.49,
        ItemCategory.BEVERAGE, 1
    ))
    system.register_restaurant(pizza)

    # Setup customers
    for name, uid, loc, addr in [
        ("Alice", "alice", Location(27.7200, 85.3180), "123 Main St"),
        ("Bob", "bob", Location(27.7150, 85.3350), "456 Oak Ave"),
        ("Charlie", "charlie", Location(27.7220, 85.3150), "789 Pine Rd"),
    ]:
        c = Customer(uid, name, "555-0000", f"{uid}@email.com",
                     loc, addr)
        c.top_up_wallet(500.0)
        system.register_customer(c)

    # Setup agents
    for name, uid, loc in [
        ("Dave", "agent-1", Location(27.7180, 85.3220)),
        ("Emma", "agent-2", Location(27.7120, 85.3280)),
        ("Frank", "agent-3", Location(27.7190, 85.3200)),
    ]:
        system.register_agent(
            DeliveryAgent(uid, name, "555-0000",
                          f"{uid}@delivery.com", loc)
        )

    scenario = DeliveryScenario(system)

    # Alice orders pizza
    def alice_order():
        system.add_to_cart("alice", pizza.id, "PP-1", 1)
        system.add_to_cart("alice", pizza.id, "PP-3", 1)
        system.add_to_cart("alice", pizza.id, "PP-4", 2)
        system.place_order("alice", PaymentMethod.WALLET)

    scenario.at_tick(1, "Alice orders Margherita + sides",
                     alice_order)

    scenario.at_tick(2, "Restaurant confirms Alice's order",
                     lambda: system.confirm_order("ORD-1"))

    scenario.at_tick(3, "Kitchen starts preparing Alice's order",
                     lambda: system.start_preparing("ORD-1"))

    # Bob orders at the same time
    def bob_order():
        system.add_to_cart("bob", pizza.id, "PP-2", 2)
        system.add_to_cart("bob", pizza.id, "PP-4", 1)
        system.place_order("bob", PaymentMethod.WALLET)

    scenario.at_tick(3, "Bob orders 2 Pepperoni + Cola",
                     bob_order)

    scenario.at_tick(4, "Restaurant confirms Bob's order",
                     lambda: system.confirm_order("ORD-2"))

    scenario.at_tick(4, "Kitchen starts preparing Bob's order",
                     lambda: system.start_preparing("ORD-2"))

    # Charlie orders and cancels
    def charlie_order():
        system.add_to_cart("charlie", pizza.id, "PP-1", 2)
        system.place_order("charlie", PaymentMethod.WALLET)

    scenario.at_tick(5, "Charlie orders 2 Margherita",
                     charlie_order)

    scenario.at_tick(6, "Charlie cancels (changed mind)",
                     lambda: system.cancel_order(
                         "ORD-3", "Changed my mind"))

    # Alice's order progresses
    scenario.at_tick(6, "Alice's order ready for pickup",
                     lambda: system.mark_ready("ORD-1"))

    scenario.at_tick(7, "Agent picks up Alice's order",
                     lambda: system.pickup_order("ORD-1"))

    scenario.at_tick(8, "Alice's order delivered",
                     lambda: system.deliver_order("ORD-1"))

    # Bob's order progresses
    scenario.at_tick(7, "Bob's order ready for pickup",
                     lambda: system.mark_ready("ORD-2"))

    scenario.at_tick(8, "Agent picks up Bob's order",
                     lambda: system.pickup_order("ORD-2"))

    scenario.at_tick(9, "Bob's order delivered",
                     lambda: system.deliver_order("ORD-2"))

    scenario.run(display_interval=4)
    system.tracker.display_history()


if __name__ == "__main__":
    main_scenario()
```

### Strategy Comparison

```python
def compare_strategies() -> None:
    """Compare dispatch strategies on the same workload."""
    strategies: list[DeliveryStrategy] = [
        NearestAgentStrategy(),
        LeastBusyStrategy(),
        HighestRatedStrategy(),
    ]

    print("\n" + "=" * 55)
    print("  DISPATCH STRATEGY COMPARISON")
    print("=" * 55)

    for strategy in strategies:
        Order._next_id = 0
        Restaurant._next_id = 0

        system = FoodDeliverySystem(delivery_strategy=strategy)

        # Setup restaurant
        r = Restaurant("TestFood", Location(27.7172, 85.3240))
        r.menu.add_item(MenuItem(
            "T-1", "Test Item", 15.00,
            ItemCategory.MAIN_COURSE, 10
        ))
        system.register_restaurant(r)

        # Setup customer
        c = Customer("cust", "Customer", "555", "c@test.com",
                     Location(27.7200, 85.3180), "Test Address")
        c.top_up_wallet(500.0)
        system.register_customer(c)

        # Setup agents at different distances/stats
        agents_data = [
            ("A1", "Agent Near", Location(27.7175, 85.3245), 5, 4.2),
            ("A2", "Agent Mid", Location(27.7160, 85.3260), 10, 4.8),
            ("A3", "Agent Far", Location(27.7130, 85.3300), 2, 4.5),
        ]
        for uid, name, loc, deliveries, rating in agents_data:
            agent = DeliveryAgent(uid, name, "555",
                                  f"{uid}@test.com", loc)
            agent._total_deliveries = deliveries
            agent._rating = rating
            system.register_agent(agent)

        # Place and confirm order
        system.add_to_cart("cust", r.id, "T-1", 1)
        order = system.place_order("cust", PaymentMethod.WALLET)
        if order:
            system.confirm_order(order.id)

            assigned = system.get_agent(order.agent_id) if order.agent_id else None
            print(f"\n  Strategy: {strategy.name()}")
            if assigned:
                dist = assigned.location.distance_to(r.location)
                print(f"    Selected: {assigned.name}")
                print(f"    Distance to restaurant: {dist:.2f} km")
                print(f"    Total deliveries: {assigned.total_deliveries}")
                print(f"    Rating: {assigned.rating:.1f}")
            else:
                print(f"    No agent assigned")

    print("\n" + "=" * 55)
```

This comparison highlights the difference between dispatch strategies. With `NearestAgentStrategy`, Agent Near (closest to the restaurant) is selected. With `LeastBusyStrategy`, Agent Far (fewest deliveries) gets the order. With `HighestRatedStrategy`, Agent Mid (highest rating at 4.8) is assigned. Each strategy optimises for a different metric.

### Unit Tests

```python
def test_place_order() -> None:
    """Test that a complete order flow works."""
    Order._next_id = 0
    Restaurant._next_id = 0

    system = FoodDeliverySystem()
    r = Restaurant("TestRestaurant", Location(27.7172, 85.3240))
    r.menu.add_item(MenuItem(
        "T-1", "Burger", 10.00, ItemCategory.MAIN_COURSE, 10
    ))
    system.register_restaurant(r)

    c = Customer("cust", "Test", "555", "c@test.com",
                 Location(27.7200, 85.3180), "Test Address")
    c.top_up_wallet(100.0)
    system.register_customer(c)

    system.add_to_cart("cust", r.id, "T-1", 2)
    order = system.place_order("cust", PaymentMethod.WALLET)

    assert order is not None
    assert order.status == OrderStatus.PLACED
    assert len(order.items) == 1
    assert order.items[0].quantity == 2
    assert order.subtotal == 20.00
    print("✅ test_place_order passed")


def test_order_lifecycle() -> None:
    """Test full order state transitions."""
    Order._next_id = 0
    Restaurant._next_id = 0

    system = FoodDeliverySystem()
    r = Restaurant("TestRestaurant", Location(27.7172, 85.3240))
    r.menu.add_item(MenuItem(
        "T-1", "Pasta", 12.00, ItemCategory.MAIN_COURSE, 10
    ))
    system.register_restaurant(r)

    c = Customer("cust", "Test", "555", "c@test.com",
                 Location(27.7200, 85.3180), "Test Address")
    c.top_up_wallet(100.0)
    system.register_customer(c)

    agent = DeliveryAgent("agent", "TestAgent", "555",
                          "a@test.com", Location(27.7180, 85.3220))
    system.register_agent(agent)

    system.add_to_cart("cust", r.id, "T-1", 1)
    order = system.place_order("cust", PaymentMethod.WALLET)
    assert order is not None

    assert system.confirm_order(order.id)
    assert order.status == OrderStatus.CONFIRMED

    assert system.start_preparing(order.id)
    assert order.status == OrderStatus.PREPARING

    assert system.mark_ready(order.id)
    assert order.status == OrderStatus.READY

    assert system.pickup_order(order.id)
    assert order.status == OrderStatus.PICKED_UP

    assert system.deliver_order(order.id)
    assert order.status == OrderStatus.DELIVERED

    print("✅ test_order_lifecycle passed")


def test_cancel_order() -> None:
    """Test that cancellation works before preparation."""
    Order._next_id = 0
    Restaurant._next_id = 0

    system = FoodDeliverySystem()
    r = Restaurant("TestRestaurant", Location(27.7172, 85.3240))
    r.menu.add_item(MenuItem(
        "T-1", "Salad", 8.00, ItemCategory.MAIN_COURSE, 5
    ))
    system.register_restaurant(r)

    c = Customer("cust", "Test", "555", "c@test.com",
                 Location(27.7200, 85.3180), "Test Address")
    c.top_up_wallet(100.0)
    system.register_customer(c)

    initial_balance = c.wallet_balance
    system.add_to_cart("cust", r.id, "T-1", 1)
    order = system.place_order("cust", PaymentMethod.WALLET)
    assert order is not None

    paid_amount = order.total
    assert c.wallet_balance < initial_balance  # money deducted

    assert system.cancel_order(order.id, "Test cancel")
    assert order.status == OrderStatus.CANCELLED
    assert order.payment_status == PaymentStatus.REFUNDED
    print("✅ test_cancel_order passed")


def test_cancel_after_preparing_fails() -> None:
    """Test that cancellation fails after preparation starts."""
    Order._next_id = 0
    Restaurant._next_id = 0

    system = FoodDeliverySystem()
    r = Restaurant("TestRestaurant", Location(27.7172, 85.3240))
    r.menu.add_item(MenuItem(
        "T-1", "Steak", 25.00, ItemCategory.MAIN_COURSE, 20
    ))
    system.register_restaurant(r)

    c = Customer("cust", "Test", "555", "c@test.com",
                 Location(27.7200, 85.3180), "Test Address")
    c.top_up_wallet(100.0)
    system.register_customer(c)

    system.add_to_cart("cust", r.id, "T-1", 1)
    order = system.place_order("cust", PaymentMethod.WALLET)
    assert order is not None

    system.confirm_order(order.id)
    system.start_preparing(order.id)

    # Cannot cancel after preparing
    result = system.cancel_order(order.id, "Too late")
    assert result is False
    assert order.status == OrderStatus.PREPARING
    print("✅ test_cancel_after_preparing_fails passed")


def test_invalid_state_transition() -> None:
    """Test that invalid state transitions are rejected."""
    Order._next_id = 0
    Restaurant._next_id = 0

    system = FoodDeliverySystem()
    r = Restaurant("TestRestaurant", Location(27.7172, 85.3240))
    r.menu.add_item(MenuItem(
        "T-1", "Soup", 6.00, ItemCategory.APPETIZER, 5
    ))
    system.register_restaurant(r)

    c = Customer("cust", "Test", "555", "c@test.com",
                 Location(27.7200, 85.3180), "Test Address")
    c.top_up_wallet(100.0)
    system.register_customer(c)

    system.add_to_cart("cust", r.id, "T-1", 2)
    order = system.place_order("cust", PaymentMethod.WALLET)
    assert order is not None

    # Cannot skip from PLACED to PREPARING
    result = system.start_preparing(order.id)
    assert result is False
    assert order.status == OrderStatus.PLACED

    # Cannot go from PLACED to PICKED_UP
    result = system.pickup_order(order.id)
    assert result is False
    assert order.status == OrderStatus.PLACED
    print("✅ test_invalid_state_transition passed")


def test_single_restaurant_cart() -> None:
    """Test that cart rejects items from multiple restaurants."""
    Order._next_id = 0
    Restaurant._next_id = 0

    system = FoodDeliverySystem()
    r1 = Restaurant("Restaurant A", Location(27.7172, 85.3240))
    r1.menu.add_item(MenuItem(
        "A-1", "Item A", 10.00, ItemCategory.MAIN_COURSE, 10
    ))
    r2 = Restaurant("Restaurant B", Location(27.7100, 85.3300))
    r2.menu.add_item(MenuItem(
        "B-1", "Item B", 12.00, ItemCategory.MAIN_COURSE, 12
    ))
    system.register_restaurant(r1)
    system.register_restaurant(r2)

    c = Customer("cust", "Test", "555", "c@test.com",
                 Location(27.7200, 85.3180), "Test Address")
    system.register_customer(c)

    assert system.add_to_cart("cust", r1.id, "A-1", 1) is True
    assert system.add_to_cart("cust", r2.id, "B-1", 1) is False  # rejected
    print("✅ test_single_restaurant_cart passed")


def test_minimum_order_amount() -> None:
    """Test that orders below minimum are rejected."""
    Order._next_id = 0
    Restaurant._next_id = 0

    system = FoodDeliverySystem()
    r = Restaurant("TestRestaurant", Location(27.7172, 85.3240))
    r.menu.add_item(MenuItem(
        "T-1", "Small Snack", 3.00, ItemCategory.SIDE, 3
    ))
    system.register_restaurant(r)

    c = Customer("cust", "Test", "555", "c@test.com",
                 Location(27.7200, 85.3180), "Test Address")
    c.top_up_wallet(100.0)
    system.register_customer(c)

    system.add_to_cart("cust", r.id, "T-1", 1)  # $3.00 < $10.00 minimum
    order = system.place_order("cust", PaymentMethod.WALLET)
    assert order is None  # rejected
    print("✅ test_minimum_order_amount passed")


def test_wallet_insufficient_balance() -> None:
    """Test that wallet payment fails with insufficient balance."""
    Order._next_id = 0
    Restaurant._next_id = 0

    system = FoodDeliverySystem()
    r = Restaurant("TestRestaurant", Location(27.7172, 85.3240))
    r.menu.add_item(MenuItem(
        "T-1", "Expensive Dish", 50.00, ItemCategory.MAIN_COURSE, 15
    ))
    system.register_restaurant(r)

    c = Customer("cust", "Test", "555", "c@test.com",
                 Location(27.7200, 85.3180), "Test Address")
    c.top_up_wallet(5.0)  # only $5
    system.register_customer(c)

    system.add_to_cart("cust", r.id, "T-1", 1)  # $50 + fees
    order = system.place_order("cust", PaymentMethod.WALLET)
    assert order is None  # payment fails
    print("✅ test_wallet_insufficient_balance passed")


def run_all_tests() -> None:
    print("\n" + "=" * 40)
    print("  RUNNING TESTS")
    print("=" * 40 + "\n")
    test_place_order()
    test_order_lifecycle()
    test_cancel_order()
    test_cancel_after_preparing_fails()
    test_invalid_state_transition()
    test_single_restaurant_cart()
    test_minimum_order_amount()
    test_wallet_insufficient_balance()
    print("\n✅ All tests passed!\n")
```

Each test resets order and restaurant ID counters for isolation. `test_place_order` verifies the basic flow — cart to order with correct pricing. `test_order_lifecycle` walks through every state transition from `PLACED` to `DELIVERED`. `test_cancel_order` confirms that cancellation refunds the payment. `test_cancel_after_preparing_fails` ensures the state machine blocks cancellation once cooking starts. `test_invalid_state_transition` verifies that skipping states (e.g., PLACED → PREPARING) is rejected. `test_single_restaurant_cart` checks the multi-restaurant constraint. `test_minimum_order_amount` and `test_wallet_insufficient_balance` test edge cases in validation and payment.

---

## Class Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                    FoodDeliverySystem                             │
│──────────────────────────────────────────────────────────────────│
│ - _restaurants: dict[str, Restaurant]                             │
│ - _customers: dict[str, Customer]                                 │
│ - _agents: dict[str, DeliveryAgent]                               │
│ - _delivery_strategy: DeliveryStrategy                            │
│ - _tracker: OrderTracker                                          │
│ - _listeners: list[OrderEventListener]                            │
│ - _tick: int                                                      │
│──────────────────────────────────────────────────────────────────│
│ + place_order(cust, payment, addr, loc) -> Order                  │
│ + confirm_order(order_id) -> bool                                 │
│ + start_preparing(order_id) -> bool                               │
│ + mark_ready(order_id) -> bool                                    │
│ + pickup_order(order_id) -> bool                                  │
│ + deliver_order(order_id) -> bool                                 │
│ + cancel_order(order_id, reason) -> bool                          │
│ + add_to_cart(cust, rest, item, qty) -> bool                      │
│ + display_status() -> None                                        │
└──────┬──────────┬──────────┬──────────┬───────────┬──────────────┘
       │          │          │          │           │
  owns ▼     uses ▼     owns ▼    owns ▼      uses ▼
┌──────────────┐ ┌──────────────────────────────────────────┐
│  Restaurant  │ │  «abstract» DeliveryStrategy             │
│──────────────│ │──────────────────────────────────────────│
│ - _id        │ │ + select_agent(order, rest, agents)      │
│ - _name      │ │   -> DeliveryAgent | None                │
│ - _location  │ │ + name() -> str                          │
│ - _menu      │ └──────────┬───────────────────────────────┘
│ - _is_open   │            │ implemented by
│ - _rating    │   ┌────────┼──────────────────┐
│──────────────│   ▼        ▼                  ▼
│ + open()     │ Nearest   LeastBusy      HighestRated
│ + close()    │ Agent     Strategy       Strategy
│ + add_rating │ Strategy
│ + estimated_ │
│   prep_time  │ ┌──────────────────────────────────────────┐
└──────┬───────┘ │  «abstract» PaymentStrategy              │
       │         │──────────────────────────────────────────│
  owns ▼         │ + pay(amount, customer) -> (bool, str)   │
┌──────────────┐ │ + refund(amount, customer) -> (bool, str)│
│    Menu      │ │ + method() -> PaymentMethod              │
│──────────────│ └──────────┬──────────────────────────────┘
│ - _items     │            │ implemented by
│──────────────│   ┌────────┼──────────────────┐
│ + add_item   │   ▼        ▼                  ▼
│ + get_item   │ CreditCard  Wallet      CashOnDelivery
│ + get_by_    │ Payment     Payment     Payment
│   category   │
└──────────────┘ ┌──────────────────────────────────────────┐
                 │  «abstract» OrderEventListener            │
┌──────────────┐ │──────────────────────────────────────────│
│   Customer   │ │ + on_order_placed(order)                  │
│──────────────│ │ + on_order_confirmed(order)               │
│ - _user_id   │ │ + on_order_preparing(order)               │
│ - _name      │ │ + on_order_ready(order)                   │
│ - _cart      │ │ + on_order_picked_up(order)               │
│ - _wallet    │ │ + on_order_delivered(order)                │
│ - _order_    │ │ + on_order_cancelled(order, reason)       │
│   history    │ │ + on_agent_assigned(order, agent)         │
│──────────────│ └──────────┬──────────────────────────────┘
│ + top_up_    │            │ implemented by
│   wallet()   │      ┌─────┴──────────┐
│ + debit_     │      ▼                ▼
│   wallet()   │  OrderLogger   CustomerNotifier
└──────────────┘

┌──────────────┐ ┌──────────────────────────────────────────┐
│DeliveryAgent │ │         OrderTracker                      │
│──────────────│ │──────────────────────────────────────────│
│ - _user_id   │ │ - _orders: dict[str, Order]               │
│ - _status    │ │──────────────────────────────────────────│
│ - _current_  │ │ + track_order(order) -> None               │
│   order_id   │ │ + estimate_delivery_time(...) -> int      │
│ - _total_    │ │ + compute_delivery_fee(...) -> float      │
│   deliveries │ │ + active_orders -> list[Order]            │
│ - _earnings  │ │ + display_active() -> None                │
│ - _rating    │ └──────────────────────────────────────────┘
│──────────────│
│ + assign_    │ ┌──────────────────────────────────────────┐
│   order()    │ │       DeliveryScenario                    │
│ + complete_  │ │──────────────────────────────────────────│
│   delivery() │ │ - _system: FoodDeliverySystem             │
│ + release_   │ │ - _events: list[(tick, desc, Callable)]  │
│   order()    │ │──────────────────────────────────────────│
│ + go_offline │ │ + at_tick(tick, desc, action) -> self      │
│ + go_online  │ │ + run(interval) -> None                  │
└──────────────┘ └──────────────────────────────────────────┘

┌──────────────────────────────┐ ┌──────────────────────────────┐
│  Order                        │ │  «abstract» OrderState        │
│──────────────────────────────│ │──────────────────────────────│
│ - _id: str                    │ │ + status() -> OrderStatus     │
│ - _customer_id: str           │ │ + next_states() -> list       │
│ - _restaurant_id: str         │ │ + can_cancel() -> bool        │
│ - _items: list[OrderItem]     │ └──────────┬─────────────────┘
│ - _state: OrderState          │            │ implemented by
│ - _agent_id: str | None       │  ┌─────────┼──────────┐
│ - _payment_method             │  ▼         ▼          ▼
│ - _payment_status             │ Placed  Confirmed  Preparing
│ - _delivery_fee               │ State   State      State
│ - _total: float               │
│──────────────────────────────│  ┌─────────┼──────────┐
│ + transition_to(status) -> bool│  ▼         ▼          ▼
│ + cancel(reason) -> bool      │ Ready    PickedUp   Delivered
│ + assign_agent(id) -> None    │ State    State      State
│ + is_active() -> bool         │
│ + display() -> None           │          ▼
└──────────────────────────────┘       Cancelled
                                       State

┌──────────────┐  ┌─────────────┐ ┌────────────────────┐
│ Cart         │  │  CartItem   │ │ MenuItem (frozen)   │
│──────────────│  │─────────────│ │────────────────────│
│ - _customer_ │  │ + menu_item │ │ + item_id: str      │
│   id         │  │ + quantity  │ │ + name: str         │
│ - _restaurant│  │ + subtotal  │ │ + price: float      │
│   _id        │  └─────────────┘ │ + category          │
│ - _items     │                  │ + prep_time_minutes │
│──────────────│                  │ + is_vegetarian     │
│ + add_item() │                  │ + is_available      │
│ + remove_    │                  └────────────────────┘
│   item()     │
│ + clear()    │  ┌─────────────┐
│ + total      │  │ OrderItem   │
└──────────────┘  │─────────────│
                  │ + item_id   │
┌──────────────┐  │ + name      │
│ Location     │  │ + price     │
│ (frozen)     │  │ + quantity  │
│──────────────│  │ + subtotal  │
│ + latitude   │  └─────────────┘
│ + longitude  │
│──────────────│
│ + distance_  │
│   to()       │
└──────────────┘

┌─────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│ OrderStatus     │ │ PaymentMethod    │ │ AgentStatus      │
│ (Enum)          │ │ (Enum)           │ │ (Enum)           │
│─────────────────│ │──────────────────│ │──────────────────│
│ PLACED          │ │ CREDIT_CARD      │ │ AVAILABLE        │
│ CONFIRMED       │ │ DEBIT_CARD       │ │ BUSY             │
│ PREPARING       │ │ WALLET           │ │ OFFLINE          │
│ READY           │ │ CASH_ON_DELIVERY │ └──────────────────┘
│ PICKED_UP       │ └──────────────────┘
│ DELIVERED       │ ┌──────────────────┐
│ CANCELLED       │ │ PaymentStatus    │
└─────────────────┘ │ (Enum)           │
                    │──────────────────│
┌─────────────────┐ │ PENDING          │
│ ItemCategory    │ │ COMPLETED        │
│ (Enum)          │ │ FAILED           │
│─────────────────│ │ REFUNDED         │
│ APPETIZER       │ └──────────────────┘
│ MAIN_COURSE     │
│ DESSERT         │
│ BEVERAGE        │
│ SIDE            │
└─────────────────┘
```

---

## How to Run

Save all code into a single file `food_delivery.py` and run:

```bash
python food_delivery.py
```

### Example Output

```
╔══════════════════════════════════════════╗
║       FOOD DELIVERY SYSTEM               ║
║  Restaurants: Burger Palace, Sushi Master ║
║  Strategy: Nearest Agent                  ║
╚══════════════════════════════════════════╝

  --- Alice browses Burger Palace menu ---

  ──────────────────────────────────────────────────
  MENU (4 items available)
  ──────────────────────────────────────────────────
  MAIN_COURSE:
    Classic Burger — $8.99 (main_course, ~12min)
    Veggie Burger — $9.49 (main_course, ~14min) 🌿
  SIDE:
    Cheese Fries — $4.99 (side, ~8min)
  BEVERAGE:
    Milkshake — $5.49 (beverage, ~5min)

  ─────────────────────────────────────────────
  🛒 CART — alice
  Restaurant: REST-1
  ─────────────────────────────────────────────
  Classic Burger x2 = $17.98
  Cheese Fries x1 = $4.99
  Milkshake x2 = $10.98
  ─────────────────────────────────────────────
  Subtotal: $33.95
  Service Fee (5%): $1.70
  Delivery Fee: $2.50+
  ─────────────────────────────────────────────

  👛 Debited $38.64 from wallet for Alice Johnson
  📝 ORDER PLACED: ORD-1 by alice from REST-1 — $38.64
  📱 [alice]: Your order ORD-1 has been placed!

  ⏱ Estimated delivery time: 35 minutes

  ✅ ORDER CONFIRMED: ORD-1 — restaurant accepted
  📱 [alice]: Your order ORD-1 is confirmed by the restaurant.
  🚴 AGENT ASSIGNED: Dave Rider → ORD-1
  📱 [alice]: Agent Dave Rider is assigned to your order ORD-1.
  👨‍🍳 PREPARING: ORD-1 — kitchen started
  📱 [alice]: Your order ORD-1 is being prepared.
  📦 READY: ORD-1 — waiting for pickup
  📱 [alice]: Your order ORD-1 is ready for pickup!
  🚴 PICKED UP: ORD-1 by agent-1
  📱 [alice]: Your order ORD-1 is on the way! Agent: agent-1
  🏠 DELIVERED: ORD-1 to alice
  📱 [alice]: Your order ORD-1 has been delivered. Enjoy!

  --- Bob orders from Sushi Master ---
  💳 Charging $32.95 to credit card for Bob Smith
  📝 ORDER PLACED: ORD-2 by bob from REST-2 — $32.95
  📱 [bob]: Your order ORD-2 has been placed!
  ✅ ORDER CONFIRMED: ORD-2 — restaurant accepted
  🚴 AGENT ASSIGNED: Emma Swift → ORD-2
  👨‍🍳 PREPARING: ORD-2 — kitchen started

  --- Bob tries to cancel ---
  ✗ Cannot cancel order in preparing state
  Cancel result: FAILED (too late)

  📦 READY: ORD-2 — waiting for pickup
  🚴 PICKED UP: ORD-2 by agent-2
  🏠 DELIVERED: ORD-2 to bob
```
