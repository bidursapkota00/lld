# Design Vending Machine – Low-Level Design Guide

![Bidur Sapkota](https://www.bidursapkota.com.np/images/gravatar.webp "Bidur Sapkota - Developer")&nbsp;[Bidur Sapkota](https://www.bidursapkota.com.np/)

## Table of Contents

1. [Requirements](#requirements)
2. [Key Design Patterns](#key-design-patterns)
3. [Core Enums & Constants](#core-enums--constants)
4. [Product & Inventory](#product--inventory)
5. [Payment System (Strategy Pattern)](#payment-system-strategy-pattern)
6. [Vending Machine State (State Pattern)](#vending-machine-state-state-pattern)
7. [State Implementations](#state-implementations)
8. [Event System (Observer Pattern)](#event-system-observer-pattern)
9. [Transaction Ledger](#transaction-ledger)
10. [Change Calculator](#change-calculator)
11. [Vending Machine Facade](#vending-machine-facade)
12. [Simulation Engine](#simulation-engine)
13. [Putting It All Together](#putting-it-all-together)
14. [Class Diagram](#class-diagram)
15. [How to Run](#how-to-run)

---

## Requirements

### Functional Requirements

| #  | Requirement                                                                                       |
| -- | ------------------------------------------------------------------------------------------------- |
| 1  | `select_product(code)` selects a product by its slot code (e.g., A1, B3)                         |
| 2  | `insert_money(amount)` accepts payment in discrete denominations (coins and bills)                |
| 3  | `dispense()` dispenses the selected product once payment is sufficient                            |
| 4  | `cancel()` cancels the current transaction and refunds all inserted money                         |
| 5  | `refund_change()` returns the correct change after a purchase using available denominations       |
| 6  | Pluggable payment methods — coin, bill, and card payments can be swapped or combined              |
| 7  | Inventory management — track stock per slot, restock, and detect out-of-stock                     |
| 8  | State machine — enforce valid transitions (Idle → Selecting → Payment → Dispensing → Idle)        |
| 9  | Observer-based event system — listeners react to selections, payments, dispenses, and errors      |
| 10 | Transaction ledger — record every completed and cancelled transaction for audit                   |
| 11 | Statistics — total revenue, items sold, transactions completed, most popular product              |

### Non-Functional Requirements

- Clean OOP with SOLID principles.
- Extensible — new payment methods, products, and event listeners can be added without modifying core logic.
- Simulatable — the system runs step-by-step for testing and demonstration.

---

## Key Design Patterns

| Pattern  | Where It Is Used                                    | Why                                                                                       |
| -------- | --------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| State    | `VendingState` controls behavior per machine state  | Each state handles inputs differently; transitions are explicit and validated              |
| Strategy | `PaymentMethod` defines how money is accepted       | Swap between coin, bill, or card payment without changing machine logic                   |
| Observer | `VendingEventListener` reacts to machine events     | Loggers, analytics, and alerting react independently to state changes                     |
| Facade   | `VendingMachine` exposes a unified API              | Hides the complexity of states, inventory, payments, change, events, and the ledger       |

---

## Core Enums & Constants

```python
import time
import uuid
from enum import Enum
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from typing import Any
from collections import defaultdict


class MachineState(Enum):
    IDLE = "idle"                    # waiting for customer
    SELECTING = "selecting"          # customer is choosing a product
    ACCEPTING_PAYMENT = "payment"    # waiting for sufficient payment
    DISPENSING = "dispensing"         # product is being dispensed
    OUT_OF_SERVICE = "out_of_service" # machine is disabled


class EventType(Enum):
    PRODUCT_SELECTED = "product_selected"
    MONEY_INSERTED = "money_inserted"
    PRODUCT_DISPENSED = "product_dispensed"
    CHANGE_RETURNED = "change_returned"
    TRANSACTION_CANCELLED = "transaction_cancelled"
    REFUND_ISSUED = "refund_issued"
    OUT_OF_STOCK = "out_of_stock"
    INSUFFICIENT_FUNDS = "insufficient_funds"
    RESTOCKED = "restocked"
    STATE_CHANGED = "state_changed"
    ERROR = "error"


class Denomination(Enum):
    """
    Supported currency denominations.

    Values are in cents to avoid floating-point issues.
    All internal money calculations use cents.
    """
    PENNY = 1
    NICKEL = 5
    DIME = 10
    QUARTER = 25
    HALF_DOLLAR = 50
    DOLLAR = 100
    FIVE_DOLLARS = 500
    TEN_DOLLARS = 1000
    TWENTY_DOLLARS = 2000


# ------------------------------------------------------------------
# System constants
# ------------------------------------------------------------------

STATS_DISPLAY_WIDTH = 55
MAX_SLOTS = 50
DEFAULT_CAPACITY_PER_SLOT = 10
```

`MachineState` tracks the five possible machine states. The machine starts `IDLE`, moves to `SELECTING` when a product is chosen, then `ACCEPTING_PAYMENT` while collecting money, `DISPENSING` when the product is released, and back to `IDLE`. `OUT_OF_SERVICE` is a terminal state for maintenance. `Denomination` uses cent values instead of floating-point dollars — `QUARTER = 25` means 25 cents. This eliminates rounding errors entirely: `0.1 + 0.2 != 0.3` in floats, but `10 + 20 == 30` in integers. All internal arithmetic uses cents; conversion to dollars is only for display.

---

## Product & Inventory

```python
@dataclass
class Product:
    """
    A product available in the vending machine.

    price_cents stores the price in cents to avoid
    floating-point precision issues. A $1.50 soda
    is stored as 150.
    """
    name: str
    price_cents: int
    category: str = "general"

    @property
    def price_display(self) -> str:
        dollars = self.price_cents / 100
        return f"${dollars:.2f}"

    def __repr__(self) -> str:
        return (f"Product({self.name}, "
                f"{self.price_display})")


@dataclass
class Slot:
    """
    A physical slot in the vending machine.

    Each slot has a code (e.g., A1, B3), holds one type
    of product, and tracks current stock and capacity.
    """
    code: str
    product: Product
    quantity: int = 0
    capacity: int = DEFAULT_CAPACITY_PER_SLOT

    @property
    def is_empty(self) -> bool:
        return self.quantity <= 0

    @property
    def is_full(self) -> bool:
        return self.quantity >= self.capacity

    def dispense_one(self) -> Product | None:
        """Remove and return one product, or None if empty."""
        if self.is_empty:
            return None
        self.quantity -= 1
        return self.product

    def restock(self, count: int) -> int:
        """
        Add stock up to capacity.

        Returns the number of items actually added.
        """
        can_add = min(count, self.capacity - self.quantity)
        self.quantity += can_add
        return can_add

    def __repr__(self) -> str:
        return (f"Slot({self.code}: {self.product.name}, "
                f"qty={self.quantity}/{self.capacity})")


class Inventory:
    """
    Manages all product slots in the vending machine.

    Slots are keyed by their code (A1, A2, B1, etc.).
    Provides lookup, stock checking, dispensing, and
    restocking operations.
    """

    def __init__(self) -> None:
        self._slots: dict[str, Slot] = {}

    def add_slot(self, code: str, product: Product,
                 quantity: int = 0,
                 capacity: int = DEFAULT_CAPACITY_PER_SLOT,
                 ) -> None:
        """Add or replace a slot."""
        self._slots[code] = Slot(
            code=code,
            product=product,
            quantity=quantity,
            capacity=capacity,
        )

    def get_slot(self, code: str) -> Slot | None:
        return self._slots.get(code)

    def has_stock(self, code: str) -> bool:
        slot = self._slots.get(code)
        return slot is not None and not slot.is_empty

    def dispense(self, code: str) -> Product | None:
        """Dispense one item from the slot."""
        slot = self._slots.get(code)
        if slot is None:
            return None
        return slot.dispense_one()

    def restock(self, code: str, count: int) -> int:
        """Restock a slot. Returns items actually added."""
        slot = self._slots.get(code)
        if slot is None:
            return 0
        return slot.restock(count)

    def restock_all(self, count: int) -> dict[str, int]:
        """Restock every slot. Returns items added per slot."""
        result: dict[str, int] = {}
        for code, slot in self._slots.items():
            added = slot.restock(count)
            if added > 0:
                result[code] = added
        return result

    @property
    def slot_codes(self) -> list[str]:
        return sorted(self._slots.keys())

    @property
    def total_items(self) -> int:
        return sum(s.quantity for s in self._slots.values())

    @property
    def empty_slots(self) -> list[str]:
        return [code for code, slot in self._slots.items()
                if slot.is_empty]

    def display(self) -> None:
        print(f"\n  {'─'*STATS_DISPLAY_WIDTH}")
        print(f"  INVENTORY")
        print(f"  {'─'*STATS_DISPLAY_WIDTH}")
        print(f"  {'Code':<6} {'Product':<20} "
              f"{'Price':>7} {'Stock':>8}")
        print(f"  {'─'*STATS_DISPLAY_WIDTH}")
        for code in sorted(self._slots.keys()):
            slot = self._slots[code]
            status = ("EMPTY" if slot.is_empty
                      else f"{slot.quantity}/{slot.capacity}")
            print(f"  {slot.code:<6} "
                  f"{slot.product.name:<20} "
                  f"{slot.product.price_display:>7} "
                  f"{status:>8}")
        print(f"  {'─'*STATS_DISPLAY_WIDTH}")

    def __repr__(self) -> str:
        return (f"Inventory(slots={len(self._slots)}, "
                f"items={self.total_items})")
```

`Product` is a simple data class — `price_cents` stores the price as an integer to avoid floating-point issues. A `$1.75` drink is `175` cents internally. `Slot` represents a physical position in the machine — each slot holds one product type with a limited capacity. `dispense_one` decrements the quantity and returns the product, or `None` if empty. `restock` adds items up to capacity and returns how many were actually added, preventing overfilling. `Inventory` manages all slots with O(1) lookup by code. `empty_slots` returns slots that need restocking — useful for maintenance alerts.

---

## Payment System (Strategy Pattern)

```python
class PaymentMethod(ABC):
    """
    Defines how money is accepted by the machine.

    Each payment method validates denominations, tracks
    the amount inserted, and handles refunds. The vending
    machine delegates payment handling to the strategy
    without knowing the specifics.
    """

    @abstractmethod
    def accept(self, amount_cents: int) -> bool:
        """Accept a payment of the given amount."""
        ...

    @abstractmethod
    def get_balance(self) -> int:
        """Return the total amount inserted (cents)."""
        ...

    @abstractmethod
    def reset(self) -> int:
        """Reset and return the balance (for refund)."""
        ...

    @abstractmethod
    def name(self) -> str:
        ...


class CoinPayment(PaymentMethod):
    """
    Accept coins (penny through dollar).

    Only coin-sized denominations are valid. Bills
    and card payments are rejected.
    """

    VALID = {
        Denomination.PENNY.value,
        Denomination.NICKEL.value,
        Denomination.DIME.value,
        Denomination.QUARTER.value,
        Denomination.HALF_DOLLAR.value,
        Denomination.DOLLAR.value,
    }

    def __init__(self) -> None:
        self._balance: int = 0

    def accept(self, amount_cents: int) -> bool:
        if amount_cents not in self.VALID:
            return False
        self._balance += amount_cents
        return True

    def get_balance(self) -> int:
        return self._balance

    def reset(self) -> int:
        refund = self._balance
        self._balance = 0
        return refund

    def name(self) -> str:
        return "Coin"


class BillPayment(PaymentMethod):
    """
    Accept paper bills ($1 through $20).

    Only bill denominations are valid. Coins are rejected.
    """

    VALID = {
        Denomination.DOLLAR.value,
        Denomination.FIVE_DOLLARS.value,
        Denomination.TEN_DOLLARS.value,
        Denomination.TWENTY_DOLLARS.value,
    }

    def __init__(self) -> None:
        self._balance: int = 0

    def accept(self, amount_cents: int) -> bool:
        if amount_cents not in self.VALID:
            return False
        self._balance += amount_cents
        return True

    def get_balance(self) -> int:
        return self._balance

    def reset(self) -> int:
        refund = self._balance
        self._balance = 0
        return refund

    def name(self) -> str:
        return "Bill"


class CombinedPayment(PaymentMethod):
    """
    Accept both coins and bills.

    Delegates to CoinPayment and BillPayment internally.
    An amount is accepted if either sub-method accepts it.
    """

    def __init__(self) -> None:
        self._coin = CoinPayment()
        self._bill = BillPayment()

    def accept(self, amount_cents: int) -> bool:
        if self._coin.accept(amount_cents):
            return True
        return self._bill.accept(amount_cents)

    def get_balance(self) -> int:
        return (self._coin.get_balance()
                + self._bill.get_balance())

    def reset(self) -> int:
        return self._coin.reset() + self._bill.reset()

    def name(self) -> str:
        return "Coin+Bill"


class CardPayment(PaymentMethod):
    """
    Accept card payments (exact amount charged).

    Card payments do not involve physical denominations.
    The accept method charges the exact amount needed.
    No change is returned for card payments.
    """

    def __init__(self) -> None:
        self._balance: int = 0

    def accept(self, amount_cents: int) -> bool:
        if amount_cents <= 0:
            return False
        self._balance += amount_cents
        return True

    def get_balance(self) -> int:
        return self._balance

    def reset(self) -> int:
        refund = self._balance
        self._balance = 0
        return refund

    def name(self) -> str:
        return "Card"
```

`PaymentMethod` is the strategy interface. `CoinPayment` only accepts coin denominations (1¢ through $1), rejecting bills. `BillPayment` only accepts bills ($1 through $20), rejecting coins. `CombinedPayment` composes both — it tries coins first, then bills, so any physical currency is accepted. `CardPayment` accepts any positive amount, modeling electronic payments where denominations don't apply. Adding a new payment method (mobile pay, cryptocurrency) requires only implementing `accept`, `get_balance`, `reset`, and `name`. The machine never knows which strategy is active — it calls `accept(amount)` and checks the result.

---

## Vending Machine State (State Pattern)

```python
class VendingContext:
    """
    Shared context passed to every state.

    The context gives states access to the inventory,
    payment method, event emitter, and the currently
    selected product. States modify the context to
    communicate with each other across transitions.
    """

    def __init__(
        self,
        inventory: Inventory,
        payment: PaymentMethod,
        emit_fn: callable,
    ) -> None:
        self.inventory = inventory
        self.payment = payment
        self.emit_fn = emit_fn
        self.selected_code: str = ""
        self.selected_slot: Slot | None = None
        self.current_state: MachineState = MachineState.IDLE

    @property
    def amount_due(self) -> int:
        """Remaining amount needed (cents)."""
        if self.selected_slot is None:
            return 0
        return max(0, (self.selected_slot.product.price_cents
                       - self.payment.get_balance()))

    @property
    def is_paid(self) -> bool:
        return self.amount_due <= 0

    def clear_selection(self) -> None:
        self.selected_code = ""
        self.selected_slot = None

    def cents_to_display(self, cents: int) -> str:
        return f"${cents / 100:.2f}"


class VendingState(ABC):
    """
    Abstract state in the vending machine state machine.

    Each state defines how the machine responds to
    select, insert_money, dispense, and cancel actions.
    Invalid actions for the current state return False
    without side effects.
    """

    @abstractmethod
    def select_product(self, ctx: VendingContext,
                       code: str) -> bool:
        ...

    @abstractmethod
    def insert_money(self, ctx: VendingContext,
                     amount_cents: int) -> bool:
        ...

    @abstractmethod
    def dispense(self, ctx: VendingContext) -> bool:
        ...

    @abstractmethod
    def cancel(self, ctx: VendingContext) -> int:
        """Cancel and return refund amount in cents."""
        ...

    @abstractmethod
    def state_name(self) -> MachineState:
        ...
```

`VendingContext` is the shared data object that all states can read and modify — it holds the inventory, payment strategy, selected product, and event emitter. `amount_due` computes how much more the customer needs to pay. `VendingState` is the state interface — each method corresponds to a user action. States return `bool` for success/failure, except `cancel` which returns the refund amount. Invalid actions (e.g., inserting money in `IDLE` state) return `False` / `0` without side effects. This prevents illegal state transitions — the machine cannot dispense without payment, and cannot accept money without a selection.

---

## State Implementations

```python
class IdleState(VendingState):
    """
    Machine is idle, waiting for a customer.

    Valid actions: select_product
    Invalid: insert_money, dispense, cancel (nothing to cancel)

    Transition: → ACCEPTING_PAYMENT (after valid selection)
    """

    def select_product(self, ctx: VendingContext,
                       code: str) -> bool:
        slot = ctx.inventory.get_slot(code)
        if slot is None:
            ctx.emit_fn(EventType.ERROR, "",
                        f"invalid slot code: {code}")
            return False

        if slot.is_empty:
            ctx.emit_fn(EventType.OUT_OF_STOCK, code,
                        f"{slot.product.name} is out of stock")
            return False

        ctx.selected_code = code
        ctx.selected_slot = slot
        ctx.current_state = MachineState.ACCEPTING_PAYMENT
        ctx.emit_fn(EventType.PRODUCT_SELECTED, code,
                    f"{slot.product.name} "
                    f"{slot.product.price_display}")
        ctx.emit_fn(EventType.STATE_CHANGED, "",
                    f"IDLE → ACCEPTING_PAYMENT")
        return True

    def insert_money(self, ctx: VendingContext,
                     amount_cents: int) -> bool:
        ctx.emit_fn(EventType.ERROR, "",
                    "select a product first")
        return False

    def dispense(self, ctx: VendingContext) -> bool:
        ctx.emit_fn(EventType.ERROR, "",
                    "no product selected")
        return False

    def cancel(self, ctx: VendingContext) -> int:
        return 0  # nothing to cancel

    def state_name(self) -> MachineState:
        return MachineState.IDLE


class AcceptingPaymentState(VendingState):
    """
    Machine is waiting for sufficient payment.

    Valid actions: insert_money, cancel
    Invalid: select_product (already selected), dispense
             (unless payment is sufficient, auto-transitions)

    Transition: → DISPENSING (when fully paid)
                → IDLE (on cancel)
    """

    def select_product(self, ctx: VendingContext,
                       code: str) -> bool:
        ctx.emit_fn(EventType.ERROR, "",
                    "product already selected — "
                    "cancel first to change")
        return False

    def insert_money(self, ctx: VendingContext,
                     amount_cents: int) -> bool:
        if not ctx.payment.accept(amount_cents):
            ctx.emit_fn(EventType.ERROR, "",
                        f"invalid denomination: "
                        f"{ctx.cents_to_display(amount_cents)}")
            return False

        ctx.emit_fn(
            EventType.MONEY_INSERTED,
            ctx.selected_code,
            f"inserted {ctx.cents_to_display(amount_cents)}"
            f" | balance: "
            f"{ctx.cents_to_display(ctx.payment.get_balance())}"
            f" | due: "
            f"{ctx.cents_to_display(ctx.amount_due)}")

        if ctx.is_paid:
            ctx.current_state = MachineState.DISPENSING
            ctx.emit_fn(EventType.STATE_CHANGED, "",
                        "ACCEPTING_PAYMENT → DISPENSING")
        return True

    def dispense(self, ctx: VendingContext) -> bool:
        if not ctx.is_paid:
            ctx.emit_fn(
                EventType.INSUFFICIENT_FUNDS,
                ctx.selected_code,
                f"need {ctx.cents_to_display(ctx.amount_due)}"
                f" more")
            return False
        # Should not reach here — insert_money auto-transitions
        ctx.current_state = MachineState.DISPENSING
        return True

    def cancel(self, ctx: VendingContext) -> int:
        refund = ctx.payment.reset()
        ctx.clear_selection()
        ctx.current_state = MachineState.IDLE
        if refund > 0:
            ctx.emit_fn(EventType.REFUND_ISSUED, "",
                        f"refunded "
                        f"{ctx.cents_to_display(refund)}")
        ctx.emit_fn(EventType.TRANSACTION_CANCELLED, "",
                    "transaction cancelled")
        ctx.emit_fn(EventType.STATE_CHANGED, "",
                    "ACCEPTING_PAYMENT → IDLE")
        return refund

    def state_name(self) -> MachineState:
        return MachineState.ACCEPTING_PAYMENT


class DispensingState(VendingState):
    """
    Machine is dispensing the product.

    Valid actions: dispense (completes the transaction)
    Invalid: select_product, insert_money, cancel

    Transition: → IDLE (after dispensing and returning change)
    """

    def select_product(self, ctx: VendingContext,
                       code: str) -> bool:
        ctx.emit_fn(EventType.ERROR, "",
                    "dispensing in progress")
        return False

    def insert_money(self, ctx: VendingContext,
                     amount_cents: int) -> bool:
        ctx.emit_fn(EventType.ERROR, "",
                    "dispensing in progress")
        return False

    def dispense(self, ctx: VendingContext) -> bool:
        if ctx.selected_slot is None:
            ctx.emit_fn(EventType.ERROR, "",
                        "no product selected")
            ctx.current_state = MachineState.IDLE
            return False

        product = ctx.inventory.dispense(ctx.selected_code)
        if product is None:
            ctx.emit_fn(EventType.OUT_OF_STOCK,
                        ctx.selected_code,
                        "stock depleted during transaction")
            refund = ctx.payment.reset()
            if refund > 0:
                ctx.emit_fn(EventType.REFUND_ISSUED, "",
                            f"refunded "
                            f"{ctx.cents_to_display(refund)}")
            ctx.clear_selection()
            ctx.current_state = MachineState.IDLE
            return False

        # Calculate change
        overpayment = (ctx.payment.get_balance()
                       - product.price_cents)
        ctx.payment.reset()

        ctx.emit_fn(EventType.PRODUCT_DISPENSED,
                    ctx.selected_code,
                    f"dispensed {product.name}")

        if overpayment > 0:
            ctx.emit_fn(EventType.CHANGE_RETURNED, "",
                        f"change: "
                        f"{ctx.cents_to_display(overpayment)}")

        ctx.clear_selection()
        ctx.current_state = MachineState.IDLE
        ctx.emit_fn(EventType.STATE_CHANGED, "",
                    "DISPENSING → IDLE")
        return True

    def cancel(self, ctx: VendingContext) -> int:
        ctx.emit_fn(EventType.ERROR, "",
                    "cannot cancel during dispensing")
        return 0

    def state_name(self) -> MachineState:
        return MachineState.DISPENSING


class OutOfServiceState(VendingState):
    """
    Machine is out of service.

    All customer actions are rejected. Only admin
    operations (restock, set_state) can bring the
    machine back to IDLE.
    """

    def select_product(self, ctx: VendingContext,
                       code: str) -> bool:
        ctx.emit_fn(EventType.ERROR, "",
                    "machine is out of service")
        return False

    def insert_money(self, ctx: VendingContext,
                     amount_cents: int) -> bool:
        ctx.emit_fn(EventType.ERROR, "",
                    "machine is out of service")
        return False

    def dispense(self, ctx: VendingContext) -> bool:
        ctx.emit_fn(EventType.ERROR, "",
                    "machine is out of service")
        return False

    def cancel(self, ctx: VendingContext) -> int:
        ctx.emit_fn(EventType.ERROR, "",
                    "machine is out of service")
        return 0

    def state_name(self) -> MachineState:
        return MachineState.OUT_OF_SERVICE
```

Each state class handles actions differently. `IdleState` only allows product selection — inserting money or dispensing is rejected. On valid selection, it transitions to `ACCEPTING_PAYMENT`. `AcceptingPaymentState` accepts money and auto-transitions to `DISPENSING` when the balance meets or exceeds the price. Selecting a different product requires cancelling first — this prevents confusion about which product is being purchased. `DispensingState` physically dispenses the product, calculates overpayment (change), resets payment, and transitions back to `IDLE`. If stock was depleted between selection and dispensing (race condition in a concurrent system), it refunds everything. `OutOfServiceState` rejects all customer actions — only admin operations can recover the machine.

---

## Event System (Observer Pattern)

```python
@dataclass
class VendingEvent:
    """An event emitted by the vending machine."""
    event_id: str
    event_type: EventType
    slot_code: str
    detail: str = ""
    timestamp: float = field(default_factory=time.time)


class VendingEventListener(ABC):
    """
    Observer that reacts to vending machine events.

    Implementations receive all events and decide which
    ones to handle. Listeners are decoupled from the
    machine — they cannot affect its state.
    """

    @abstractmethod
    def on_event(self, event: VendingEvent) -> None:
        ...

    @abstractmethod
    def listener_id(self) -> str:
        ...


class VendingLogger(VendingEventListener):
    """Logs all vending machine events to the console."""

    def on_event(self, event: VendingEvent) -> None:
        emoji = {
            EventType.PRODUCT_SELECTED: "🔘",
            EventType.MONEY_INSERTED: "💰",
            EventType.PRODUCT_DISPENSED: "📦",
            EventType.CHANGE_RETURNED: "💵",
            EventType.TRANSACTION_CANCELLED: "🚫",
            EventType.REFUND_ISSUED: "↩️ ",
            EventType.OUT_OF_STOCK: "🚨",
            EventType.INSUFFICIENT_FUNDS: "⚠️ ",
            EventType.RESTOCKED: "📥",
            EventType.STATE_CHANGED: "🔄",
            EventType.ERROR: "❌",
        }
        icon = emoji.get(event.event_type, "📌")
        print(f"  {icon} [{event.event_type.value:>22}] "
              f"{event.detail}")

    def listener_id(self) -> str:
        return "vending-logger"


class SalesTracker(VendingEventListener):
    """Tracks sales and revenue metrics."""

    def __init__(self) -> None:
        self._items_sold: int = 0
        self._revenue_cents: int = 0
        self._product_counts: dict[str, int] = defaultdict(int)
        self._cancelled: int = 0

    def on_event(self, event: VendingEvent) -> None:
        if event.event_type == EventType.PRODUCT_DISPENSED:
            self._items_sold += 1
            if event.slot_code:
                self._product_counts[event.slot_code] += 1
        elif event.event_type == EventType.TRANSACTION_CANCELLED:
            self._cancelled += 1

    def record_sale(self, amount_cents: int) -> None:
        """Called by the machine to record revenue."""
        self._revenue_cents += amount_cents

    @property
    def items_sold(self) -> int:
        return self._items_sold

    @property
    def revenue_cents(self) -> int:
        return self._revenue_cents

    @property
    def revenue_display(self) -> str:
        return f"${self._revenue_cents / 100:.2f}"

    @property
    def most_popular(self) -> str | None:
        if not self._product_counts:
            return None
        return max(self._product_counts,
                   key=self._product_counts.get)

    def listener_id(self) -> str:
        return "sales-tracker"

    def display(self) -> None:
        print(f"\n  {'─'*STATS_DISPLAY_WIDTH}")
        print(f"  SALES METRICS")
        print(f"  {'─'*STATS_DISPLAY_WIDTH}")
        print(f"  Items sold:      {self._items_sold}")
        print(f"  Revenue:         {self.revenue_display}")
        print(f"  Cancelled:       {self._cancelled}")
        if self.most_popular:
            print(f"  Most popular:    {self.most_popular}")
        print(f"  {'─'*STATS_DISPLAY_WIDTH}")


class VendingEventBus:
    """
    Central event dispatcher for vending machine events.

    Manages listener registration and broadcasts events
    to all registered listeners.
    """

    def __init__(self) -> None:
        self._listeners: list[VendingEventListener] = []
        self._event_log: list[VendingEvent] = []

    def register(self, listener: VendingEventListener
                 ) -> None:
        self._listeners.append(listener)

    def unregister(self, listener_id: str) -> None:
        self._listeners = [
            l for l in self._listeners
            if l.listener_id() != listener_id
        ]

    def emit(self, event: VendingEvent) -> None:
        self._event_log.append(event)
        for listener in self._listeners:
            try:
                listener.on_event(event)
            except Exception as e:
                print(f"  ⚠️ Listener "
                      f"{listener.listener_id()} "
                      f"error: {e}")

    @property
    def total_events(self) -> int:
        return len(self._event_log)

    @property
    def event_log(self) -> list[VendingEvent]:
        return list(self._event_log)
```

`VendingEventListener` is the observer interface. `VendingLogger` prints every event with emoji-coded log lines — selections, payments, dispenses, and errors are visually distinct. `SalesTracker` collects business metrics: items sold, revenue, cancellation count, and most popular product. The `VendingEventBus` dispatches events to all registered listeners, isolating each listener from failures in others. Adding a new listener (e.g., a low-stock alerter that triggers restocking when inventory drops below a threshold) requires only implementing `on_event` and `listener_id`.

---

## Transaction Ledger

```python
@dataclass
class Transaction:
    """
    Record of a completed or cancelled vending transaction.

    Stores the product, amount paid, change returned,
    and whether the transaction completed successfully.
    """
    transaction_id: str
    slot_code: str
    product_name: str
    price_cents: int
    amount_paid_cents: int
    change_cents: int
    completed: bool
    timestamp: float = field(default_factory=time.time)

    @property
    def is_sale(self) -> bool:
        return self.completed

    def __repr__(self) -> str:
        status = "✅" if self.completed else "🚫"
        return (f"{status} {self.product_name} "
                f"${self.price_cents/100:.2f} "
                f"(paid ${self.amount_paid_cents/100:.2f}, "
                f"change ${self.change_cents/100:.2f})")


class TransactionLedger:
    """
    Append-only ledger of all transactions.

    Every completed purchase and every cancellation is
    recorded for auditing. The ledger is immutable —
    entries cannot be modified or deleted.
    """

    def __init__(self) -> None:
        self._transactions: list[Transaction] = []

    def record(self, transaction: Transaction) -> None:
        self._transactions.append(transaction)

    @property
    def all_transactions(self) -> list[Transaction]:
        return list(self._transactions)

    @property
    def completed_transactions(self) -> list[Transaction]:
        return [t for t in self._transactions
                if t.completed]

    @property
    def cancelled_transactions(self) -> list[Transaction]:
        return [t for t in self._transactions
                if not t.completed]

    @property
    def total_revenue_cents(self) -> int:
        return sum(t.price_cents for t in self._transactions
                   if t.completed)

    @property
    def total_change_dispensed_cents(self) -> int:
        return sum(t.change_cents for t in self._transactions
                   if t.completed)

    @property
    def count(self) -> int:
        return len(self._transactions)

    def display(self) -> None:
        print(f"\n  {'─'*STATS_DISPLAY_WIDTH}")
        print(f"  TRANSACTION LEDGER "
              f"({len(self._transactions)} entries)")
        print(f"  {'─'*STATS_DISPLAY_WIDTH}")
        for t in self._transactions:
            print(f"  {t}")
        revenue = self.total_revenue_cents / 100
        change = self.total_change_dispensed_cents / 100
        print(f"  {'─'*STATS_DISPLAY_WIDTH}")
        print(f"  Total revenue:   ${revenue:.2f}")
        print(f"  Total change:    ${change:.2f}")
        print(f"  {'─'*STATS_DISPLAY_WIDTH}")

    def __repr__(self) -> str:
        return (f"TransactionLedger("
                f"entries={len(self._transactions)})")
```

`Transaction` is an immutable record of a single vending interaction — it captures what was bought, how much was paid, and how much change was returned. `TransactionLedger` is append-only by design — transactions cannot be modified or deleted, mirroring a real financial ledger. `total_revenue_cents` sums only completed transactions; cancellations are tracked separately. This separation ensures accurate financial reporting.

---

## Change Calculator

```python
class ChangeCalculator:
    """
    Calculates change using available denominations.

    Uses a greedy algorithm to return change in the
    fewest coins/bills possible. Tracks the machine's
    internal coin/bill reserves.

    Greedy works here because US denominations form a
    canonical coin system — greedy always produces the
    optimal solution.
    """

    def __init__(self) -> None:
        self._reserves: dict[int, int] = {
            d.value: 20 for d in Denomination
        }

    def make_change(self, amount_cents: int
                    ) -> list[tuple[int, int]] | None:
        """
        Calculate change for the given amount.

        Returns a list of (denomination_cents, count) tuples,
        or None if exact change cannot be made.
        """
        if amount_cents <= 0:
            return []

        result: list[tuple[int, int]] = []
        remaining = amount_cents
        denominations = sorted(self._reserves.keys(),
                                reverse=True)

        # Greedy: use largest denomination first
        for denom in denominations:
            if remaining <= 0:
                break
            available = self._reserves.get(denom, 0)
            count = min(remaining // denom, available)
            if count > 0:
                result.append((denom, count))
                remaining -= denom * count

        if remaining > 0:
            return None  # cannot make exact change

        # Deduct from reserves
        for denom, count in result:
            self._reserves[denom] -= count

        return result

    def add_to_reserves(self, denom_cents: int,
                        count: int = 1) -> None:
        """Add coins/bills to the machine's reserves."""
        self._reserves[denom_cents] = (
            self._reserves.get(denom_cents, 0) + count
        )

    @property
    def total_reserves_cents(self) -> int:
        return sum(d * c
                   for d, c in self._reserves.items())

    def display_reserves(self) -> None:
        print(f"\n  {'─'*STATS_DISPLAY_WIDTH}")
        print(f"  CHANGE RESERVES")
        print(f"  {'─'*STATS_DISPLAY_WIDTH}")
        for denom in sorted(self._reserves.keys(),
                            reverse=True):
            count = self._reserves[denom]
            if count > 0:
                name = f"${denom/100:.2f}"
                print(f"  {name:<10} × {count}")
        total = self.total_reserves_cents / 100
        print(f"  {'─'*STATS_DISPLAY_WIDTH}")
        print(f"  Total: ${total:.2f}")
        print(f"  {'─'*STATS_DISPLAY_WIDTH}")

    @staticmethod
    def format_change(
        change: list[tuple[int, int]],
    ) -> str:
        """Format change as human-readable string."""
        if not change:
            return "no change"
        parts = []
        for denom, count in change:
            name = f"${denom/100:.2f}"
            parts.append(f"{count}×{name}")
        return ", ".join(parts)

    def __repr__(self) -> str:
        total = self.total_reserves_cents / 100
        return f"ChangeCalculator(reserves=${total:.2f})"
```

`ChangeCalculator` uses a greedy algorithm — take as many of the largest denomination as possible, then move to the next. This produces the minimum number of coins/bills because US denominations form a canonical system (greedy is always optimal). If the reserves cannot cover the exact change (e.g., 15¢ change but no nickels and no pennies), `make_change` returns `None`, signaling the machine cannot complete the transaction. `add_to_reserves` lets the machine absorb inserted coins — every coin a customer inserts becomes part of the reserve pool for future change.

---

## Vending Machine Facade

```python
class VendingMachine:
    """
    Facade that orchestrates the entire vending machine.

    Provides a simple API that hides the complexity of
    states, inventory, payments, change calculation,
    events, and the transaction ledger.
    """

    def __init__(
        self,
        payment: PaymentMethod | None = None,
    ) -> None:
        self._inventory = Inventory()
        self._payment = payment or CombinedPayment()
        self._change_calc = ChangeCalculator()
        self._ledger = TransactionLedger()
        self._event_bus = VendingEventBus()
        self._logger = VendingLogger()
        self._sales = SalesTracker()
        self._event_bus.register(self._logger)
        self._event_bus.register(self._sales)

        self._ctx = VendingContext(
            inventory=self._inventory,
            payment=self._payment,
            emit_fn=self._emit,
        )

        self._states: dict[MachineState, VendingState] = {
            MachineState.IDLE: IdleState(),
            MachineState.ACCEPTING_PAYMENT: AcceptingPaymentState(),
            MachineState.DISPENSING: DispensingState(),
            MachineState.OUT_OF_SERVICE: OutOfServiceState(),
        }

    # ------------------------------------------------------------------
    # Setup
    # ------------------------------------------------------------------

    def add_product(self, code: str, name: str,
                    price_cents: int,
                    quantity: int = DEFAULT_CAPACITY_PER_SLOT,
                    category: str = "general") -> None:
        """Add a product to a slot."""
        product = Product(
            name=name,
            price_cents=price_cents,
            category=category,
        )
        self._inventory.add_slot(
            code, product, quantity,
        )

    # ------------------------------------------------------------------
    # Customer operations
    # ------------------------------------------------------------------

    def select_product(self, code: str) -> bool:
        """Select a product by slot code."""
        state = self._current_state
        return state.select_product(self._ctx, code)

    def insert_money(self, amount_cents: int) -> bool:
        """Insert money (denomination in cents)."""
        state = self._current_state
        result = state.insert_money(self._ctx, amount_cents)
        if result:
            self._change_calc.add_to_reserves(amount_cents)
        # Auto-dispense if fully paid and in DISPENSING state
        if self._ctx.current_state == MachineState.DISPENSING:
            self._do_dispense()
        return result

    def cancel(self) -> int:
        """Cancel the current transaction. Returns refund in cents."""
        state = self._current_state
        selected_code = self._ctx.selected_code
        product_name = ""
        price = 0
        paid = self._ctx.payment.get_balance()

        if self._ctx.selected_slot:
            product_name = self._ctx.selected_slot.product.name
            price = self._ctx.selected_slot.product.price_cents

        refund = state.cancel(self._ctx)

        if paid > 0 or selected_code:
            self._ledger.record(Transaction(
                transaction_id=str(uuid.uuid4())[:8],
                slot_code=selected_code,
                product_name=product_name,
                price_cents=price,
                amount_paid_cents=paid,
                change_cents=refund,
                completed=False,
            ))

        return refund

    # ------------------------------------------------------------------
    # Admin operations
    # ------------------------------------------------------------------

    def restock(self, code: str, count: int) -> int:
        """Restock a slot. Returns items actually added."""
        added = self._inventory.restock(code, count)
        if added > 0:
            slot = self._inventory.get_slot(code)
            self._emit(EventType.RESTOCKED, code,
                       f"+{added} {slot.product.name} "
                       f"(now {slot.quantity}/"
                       f"{slot.capacity})")
        return added

    def restock_all(self, count: int) -> dict[str, int]:
        """Restock all slots."""
        result = self._inventory.restock_all(count)
        for code, added in result.items():
            slot = self._inventory.get_slot(code)
            self._emit(EventType.RESTOCKED, code,
                       f"+{added} {slot.product.name}")
        return result

    def set_out_of_service(self) -> None:
        """Put the machine out of service."""
        # Refund any current transaction
        if self._ctx.current_state != MachineState.IDLE:
            self.cancel()
        self._ctx.current_state = MachineState.OUT_OF_SERVICE
        self._emit(EventType.STATE_CHANGED, "",
                   "→ OUT_OF_SERVICE")

    def set_in_service(self) -> None:
        """Bring the machine back in service."""
        self._ctx.current_state = MachineState.IDLE
        self._emit(EventType.STATE_CHANGED, "",
                   "→ IDLE (back in service)")

    # ------------------------------------------------------------------
    # Inspection
    # ------------------------------------------------------------------

    @property
    def state(self) -> MachineState:
        return self._ctx.current_state

    @property
    def inventory(self) -> Inventory:
        return self._inventory

    @property
    def ledger(self) -> TransactionLedger:
        return self._ledger

    @property
    def sales(self) -> SalesTracker:
        return self._sales

    @property
    def event_bus(self) -> VendingEventBus:
        return self._event_bus

    @property
    def change_calculator(self) -> ChangeCalculator:
        return self._change_calc

    @property
    def balance_cents(self) -> int:
        return self._payment.get_balance()

    def display_status(self) -> None:
        print(f"\n  {'═'*STATS_DISPLAY_WIDTH}")
        print(f"  VENDING MACHINE STATUS")
        print(f"  {'═'*STATS_DISPLAY_WIDTH}")
        print(f"  State:           "
              f"{self._ctx.current_state.value}")
        print(f"  Payment method:  {self._payment.name()}")
        print(f"  Total items:     "
              f"{self._inventory.total_items}")
        print(f"  Transactions:    {self._ledger.count}")
        empty = self._inventory.empty_slots
        if empty:
            print(f"  Empty slots:     {', '.join(empty)}")
        print(f"  {'═'*STATS_DISPLAY_WIDTH}")
        self._inventory.display()
        self._sales.display()
        self._ledger.display()

    # ------------------------------------------------------------------
    # Internal helpers
    # ------------------------------------------------------------------

    @property
    def _current_state(self) -> VendingState:
        return self._states[self._ctx.current_state]

    def _do_dispense(self) -> None:
        """Execute the dispense operation and record the transaction."""
        slot = self._ctx.selected_slot
        if slot is None:
            return

        product = slot.product
        paid = self._ctx.payment.get_balance()
        overpayment = paid - product.price_cents

        state = self._current_state
        success = state.dispense(self._ctx)

        if success:
            # Calculate change breakdown
            change_breakdown = []
            if overpayment > 0:
                change_breakdown = (
                    self._change_calc.make_change(overpayment)
                )
                if change_breakdown:
                    formatted = ChangeCalculator.format_change(
                        change_breakdown,
                    )
                    self._emit(EventType.CHANGE_RETURNED, "",
                               f"breakdown: {formatted}")

            self._sales.record_sale(product.price_cents)
            self._ledger.record(Transaction(
                transaction_id=str(uuid.uuid4())[:8],
                slot_code=slot.code,
                product_name=product.name,
                price_cents=product.price_cents,
                amount_paid_cents=paid,
                change_cents=overpayment,
                completed=True,
            ))

    def _emit(self, event_type: EventType,
              slot_code: str, detail: str = "") -> None:
        event = VendingEvent(
            event_id=str(uuid.uuid4())[:8],
            event_type=event_type,
            slot_code=slot_code,
            detail=detail,
        )
        self._event_bus.emit(event)
```

`VendingMachine` is the facade — it wires together inventory, payment strategy, change calculator, transaction ledger, event bus, logger, sales tracker, vending context, and state objects. `select_product` delegates to the current state object, which validates the action and performs the transition. `insert_money` delegates to the state, and if the state auto-transitions to `DISPENSING`, immediately calls `_do_dispense`. `cancel` records a cancelled transaction in the ledger for audit. `_do_dispense` handles the complete dispensing workflow: dispense the product, calculate change breakdown, record the sale, and log the transaction. Users interact with `select_product`, `insert_money`, `cancel`, `restock`, and inspection methods without knowing about states, contexts, or event buses.

---

## Simulation Engine

```python
class VendingScenario:
    """
    Tick-based simulation engine for vending machine demos.

    Each tick represents a logical time step. Actions are
    registered at specific ticks and execute in order when
    the simulation runs.
    """

    def __init__(self, machine: VendingMachine) -> None:
        self._machine = machine
        self._events: list[tuple[int, str, callable]] = []

    def at_tick(self, tick: int, description: str,
                action: callable) -> None:
        self._events.append((tick, description, action))

    def run(self) -> None:
        if not self._events:
            print("  No events to simulate.")
            return

        max_tick = max(e[0] for e in self._events)
        sorted_events = sorted(self._events,
                                key=lambda e: e[0])

        print(f"\n  {'─'*STATS_DISPLAY_WIDTH}")
        print(f"  SIMULATION START (ticks: 1 → {max_tick})")
        print(f"  {'─'*STATS_DISPLAY_WIDTH}")

        current_tick = 0
        for tick, description, action in sorted_events:
            if tick != current_tick:
                current_tick = tick
                print(f"\n  ⏱️  Tick {tick}: {description}")
            else:
                print(f"  ⏱️  Tick {tick}: {description}")

            try:
                result = action()
                if result is not None:
                    print(f"       → {result}")
            except Exception as e:
                print(f"  ⚠️  Error at tick {tick}: {e}")

        print(f"\n  {'─'*STATS_DISPLAY_WIDTH}")
        print(f"  SIMULATION COMPLETE")
        print(f"  {'─'*STATS_DISPLAY_WIDTH}")
```

`VendingScenario` mirrors the simulation engine from other guides. Actions are registered at specific ticks and execute in order, making demos reproducible and readable.

---

## Putting It All Together

```python
def main_scenario() -> None:
    """Run the main vending machine demonstration."""

    print("\n╔══════════════════════════════════════════╗")
    print("║      VENDING MACHINE SYSTEM              ║")
    print("║  State + Strategy + Observer              ║")
    print("╚══════════════════════════════════════════╝")

    machine = VendingMachine(
        payment=CombinedPayment(),
    )

    # Stock the machine
    machine.add_product("A1", "Cola", 150, quantity=5)
    machine.add_product("A2", "Diet Cola", 150, quantity=3)
    machine.add_product("A3", "Lemon Soda", 175, quantity=4)
    machine.add_product("B1", "Water", 100, quantity=8)
    machine.add_product("B2", "Orange Juice", 200, quantity=3)
    machine.add_product("C1", "Chips", 125, quantity=6)
    machine.add_product("C2", "Candy Bar", 100, quantity=10)
    machine.add_product("C3", "Pretzels", 150, quantity=2)

    scenario = VendingScenario(machine)

    # --- Successful purchase with exact payment ---
    scenario.at_tick(1, "Select Cola (A1, $1.50)",
                     lambda: machine.select_product("A1"))

    scenario.at_tick(2, "Insert $1.00 bill",
                     lambda: machine.insert_money(
                         Denomination.DOLLAR.value))

    scenario.at_tick(2, "Insert 2 quarters ($0.50)",
                     lambda: (
                         machine.insert_money(
                             Denomination.QUARTER.value),
                         machine.insert_money(
                             Denomination.QUARTER.value),
                     ))

    # --- Purchase with change ---
    scenario.at_tick(3, "Select Water (B1, $1.00)",
                     lambda: machine.select_product("B1"))

    scenario.at_tick(4, "Insert $5.00 bill (expect $4.00 change)",
                     lambda: machine.insert_money(
                         Denomination.FIVE_DOLLARS.value))

    # --- Cancelled transaction ---
    scenario.at_tick(5, "Select Orange Juice (B2, $2.00)",
                     lambda: machine.select_product("B2"))

    scenario.at_tick(5, "Insert $1.00",
                     lambda: machine.insert_money(
                         Denomination.DOLLAR.value))

    scenario.at_tick(6, "Cancel transaction (expect $1.00 refund)",
                     lambda: machine.cancel())

    # --- Invalid actions ---
    scenario.at_tick(7, "Try dispensing without selection",
                     lambda: machine.select_product("Z9"))

    scenario.at_tick(7, "Insert money without selection",
                     lambda: machine.insert_money(
                         Denomination.QUARTER.value))

    # --- Multiple coin payment ---
    scenario.at_tick(8, "Select Candy Bar (C2, $1.00)",
                     lambda: machine.select_product("C2"))

    scenario.at_tick(9, "Pay with 4 quarters",
                     lambda: (
                         machine.insert_money(
                             Denomination.QUARTER.value),
                         machine.insert_money(
                             Denomination.QUARTER.value),
                         machine.insert_money(
                             Denomination.QUARTER.value),
                         machine.insert_money(
                             Denomination.QUARTER.value),
                     ))

    # --- Display final status ---
    scenario.at_tick(10, "Display machine status",
                      lambda: machine.display_status())

    scenario.run()


def demo_out_of_stock() -> None:
    """Demonstrate out-of-stock handling."""

    print("\n" + "=" * STATS_DISPLAY_WIDTH)
    print("  OUT OF STOCK DEMO")
    print("=" * STATS_DISPLAY_WIDTH)

    machine = VendingMachine()
    machine.add_product("A1", "Last Cola", 100, quantity=1)

    # Buy the only one
    print("\n  Purchase the last Cola:")
    machine.select_product("A1")
    machine.insert_money(Denomination.DOLLAR.value)

    # Try to buy again
    print("\n  Try to buy again (should fail):")
    result = machine.select_product("A1")
    print(f"  Selection succeeded? {result}")

    # Restock and buy
    print("\n  Restocking...")
    machine.restock("A1", 5)
    machine.select_product("A1")
    machine.insert_money(Denomination.DOLLAR.value)

    machine.ledger.display()
    print("\n" + "=" * STATS_DISPLAY_WIDTH)


def demo_payment_strategies() -> None:
    """Compare different payment strategies."""

    print("\n" + "=" * STATS_DISPLAY_WIDTH)
    print("  PAYMENT STRATEGIES DEMO")
    print("=" * STATS_DISPLAY_WIDTH)

    strategies: list[tuple[str, PaymentMethod]] = [
        ("Coin Only", CoinPayment()),
        ("Bill Only", BillPayment()),
        ("Combined", CombinedPayment()),
        ("Card", CardPayment()),
    ]

    for label, strategy in strategies:
        print(f"\n  Strategy: {label}")
        print(f"  {'─'*45}")
        # Try accepting a quarter (25 cents)
        q = strategy.accept(Denomination.QUARTER.value)
        print(f"    Accept quarter (25¢):  {q}")
        # Try accepting a dollar bill (100 cents)
        d = strategy.accept(Denomination.DOLLAR.value)
        print(f"    Accept dollar (100¢):  {d}")
        # Try accepting $5 bill (500 cents)
        f = strategy.accept(Denomination.FIVE_DOLLARS.value)
        print(f"    Accept $5 bill (500¢): {f}")
        print(f"    Balance: "
              f"${strategy.get_balance()/100:.2f}")
        strategy.reset()

    print("\n" + "=" * STATS_DISPLAY_WIDTH)


def demo_change_calculator() -> None:
    """Demonstrate change calculation."""

    print("\n" + "=" * STATS_DISPLAY_WIDTH)
    print("  CHANGE CALCULATOR DEMO")
    print("=" * STATS_DISPLAY_WIDTH)

    calc = ChangeCalculator()

    test_amounts = [75, 130, 41, 250, 999]
    for amount in test_amounts:
        change = calc.make_change(amount)
        if change is not None:
            formatted = ChangeCalculator.format_change(change)
            print(f"\n  ${amount/100:.2f} → {formatted}")
        else:
            print(f"\n  ${amount/100:.2f} → cannot make change")

    print("\n" + "=" * STATS_DISPLAY_WIDTH)


def demo_out_of_service() -> None:
    """Demonstrate out-of-service state."""

    print("\n" + "=" * STATS_DISPLAY_WIDTH)
    print("  OUT OF SERVICE DEMO")
    print("=" * STATS_DISPLAY_WIDTH)

    machine = VendingMachine()
    machine.add_product("A1", "Cola", 150, quantity=5)

    print("\n  Normal operation:")
    machine.select_product("A1")
    machine.cancel()

    print("\n  Set out of service:")
    machine.set_out_of_service()

    print("\n  Try to use (should fail):")
    result = machine.select_product("A1")
    print(f"  Selection succeeded? {result}")

    print("\n  Restore service:")
    machine.set_in_service()
    result = machine.select_product("A1")
    print(f"  Selection succeeded? {result}")
    machine.cancel()

    print("\n" + "=" * STATS_DISPLAY_WIDTH)


# ------------------------------------------------------------------
# Unit Tests
# ------------------------------------------------------------------

def test_product_creation() -> None:
    """Test product creation and display price."""
    product = Product("Cola", 150)
    assert product.name == "Cola"
    assert product.price_cents == 150
    assert product.price_display == "$1.50"
    print("✅ test_product_creation passed")


def test_slot_operations() -> None:
    """Test slot dispense and restock."""
    product = Product("Water", 100)
    slot = Slot("A1", product, quantity=2, capacity=5)

    assert not slot.is_empty
    assert not slot.is_full

    p1 = slot.dispense_one()
    assert p1 is not None
    assert slot.quantity == 1

    p2 = slot.dispense_one()
    assert p2 is not None
    assert slot.is_empty

    p3 = slot.dispense_one()
    assert p3 is None

    added = slot.restock(10)
    assert added == 5  # capped at capacity
    assert slot.is_full
    print("✅ test_slot_operations passed")


def test_inventory() -> None:
    """Test inventory add, dispense, restock."""
    inv = Inventory()
    inv.add_slot("A1", Product("Cola", 150), quantity=3)
    inv.add_slot("B1", Product("Water", 100), quantity=0)

    assert inv.has_stock("A1")
    assert not inv.has_stock("B1")
    assert inv.total_items == 3
    assert "B1" in inv.empty_slots

    product = inv.dispense("A1")
    assert product is not None
    assert product.name == "Cola"
    assert inv.total_items == 2
    print("✅ test_inventory passed")


def test_coin_payment() -> None:
    """Test coin payment accepts only coins."""
    pay = CoinPayment()
    assert pay.accept(Denomination.QUARTER.value)
    assert pay.accept(Denomination.DIME.value)
    assert not pay.accept(
        Denomination.FIVE_DOLLARS.value)
    assert pay.get_balance() == 35
    refund = pay.reset()
    assert refund == 35
    assert pay.get_balance() == 0
    print("✅ test_coin_payment passed")


def test_bill_payment() -> None:
    """Test bill payment accepts only bills."""
    pay = BillPayment()
    assert pay.accept(Denomination.DOLLAR.value)
    assert pay.accept(Denomination.FIVE_DOLLARS.value)
    assert not pay.accept(Denomination.QUARTER.value)
    assert pay.get_balance() == 600
    print("✅ test_bill_payment passed")


def test_combined_payment() -> None:
    """Test combined payment accepts both."""
    pay = CombinedPayment()
    assert pay.accept(Denomination.QUARTER.value)
    assert pay.accept(Denomination.DOLLAR.value)
    assert pay.accept(Denomination.FIVE_DOLLARS.value)
    assert pay.get_balance() == 625
    print("✅ test_combined_payment passed")


def test_card_payment() -> None:
    """Test card payment accepts any positive amount."""
    pay = CardPayment()
    assert pay.accept(150)
    assert pay.accept(275)
    assert not pay.accept(0)
    assert not pay.accept(-10)
    assert pay.get_balance() == 425
    print("✅ test_card_payment passed")


def test_state_idle_select() -> None:
    """Test selecting a product from idle state."""
    machine = VendingMachine()
    machine.add_product("A1", "Cola", 150, quantity=3)

    assert machine.state == MachineState.IDLE
    assert machine.select_product("A1")
    assert machine.state == MachineState.ACCEPTING_PAYMENT
    print("✅ test_state_idle_select passed")


def test_state_idle_invalid_actions() -> None:
    """Test that idle state rejects money and dispense."""
    machine = VendingMachine()
    machine.add_product("A1", "Cola", 150, quantity=3)

    assert not machine.insert_money(
        Denomination.QUARTER.value)
    print("✅ test_state_idle_invalid_actions passed")


def test_state_payment_insert() -> None:
    """Test inserting money in payment state."""
    machine = VendingMachine()
    machine.add_product("A1", "Cola", 150, quantity=3)

    machine.select_product("A1")
    assert machine.insert_money(
        Denomination.DOLLAR.value)
    assert machine.state == MachineState.ACCEPTING_PAYMENT

    # Insert enough to auto-dispense
    machine.insert_money(Denomination.QUARTER.value)
    machine.insert_money(Denomination.QUARTER.value)
    assert machine.state == MachineState.IDLE
    print("✅ test_state_payment_insert passed")


def test_state_payment_cancel() -> None:
    """Test cancelling during payment."""
    machine = VendingMachine()
    machine.add_product("A1", "Cola", 150, quantity=3)

    machine.select_product("A1")
    machine.insert_money(Denomination.DOLLAR.value)
    refund = machine.cancel()

    assert refund == 100
    assert machine.state == MachineState.IDLE
    print("✅ test_state_payment_cancel passed")


def test_out_of_stock() -> None:
    """Test selecting an out-of-stock product."""
    machine = VendingMachine()
    machine.add_product("A1", "Cola", 150, quantity=0)

    assert not machine.select_product("A1")
    assert machine.state == MachineState.IDLE
    print("✅ test_out_of_stock passed")


def test_invalid_slot() -> None:
    """Test selecting a non-existent slot."""
    machine = VendingMachine()
    assert not machine.select_product("Z9")
    print("✅ test_invalid_slot passed")


def test_exact_payment() -> None:
    """Test purchase with exact payment."""
    machine = VendingMachine()
    machine.add_product("A1", "Cola", 100, quantity=5)

    machine.select_product("A1")
    machine.insert_money(Denomination.DOLLAR.value)

    assert machine.state == MachineState.IDLE
    assert machine.sales.items_sold == 1
    print("✅ test_exact_payment passed")


def test_overpayment_change() -> None:
    """Test purchase with overpayment returns change."""
    machine = VendingMachine()
    machine.add_product("A1", "Water", 100, quantity=5)

    machine.select_product("A1")
    machine.insert_money(
        Denomination.FIVE_DOLLARS.value)

    assert machine.state == MachineState.IDLE
    txn = machine.ledger.completed_transactions
    assert len(txn) == 1
    assert txn[0].change_cents == 400
    print("✅ test_overpayment_change passed")


def test_restock() -> None:
    """Test restocking a slot."""
    machine = VendingMachine()
    machine.add_product("A1", "Cola", 150, quantity=0)

    assert not machine.select_product("A1")
    machine.restock("A1", 5)
    assert machine.select_product("A1")
    machine.cancel()
    print("✅ test_restock passed")


def test_transaction_ledger() -> None:
    """Test that transactions are recorded."""
    machine = VendingMachine()
    machine.add_product("A1", "Cola", 100, quantity=5)

    # Completed purchase
    machine.select_product("A1")
    machine.insert_money(Denomination.DOLLAR.value)

    # Cancelled transaction
    machine.select_product("A1")
    machine.insert_money(Denomination.QUARTER.value)
    machine.cancel()

    assert len(machine.ledger.completed_transactions) == 1
    assert len(machine.ledger.cancelled_transactions) == 1
    assert machine.ledger.count == 2
    print("✅ test_transaction_ledger passed")


def test_out_of_service() -> None:
    """Test out-of-service state rejects all actions."""
    machine = VendingMachine()
    machine.add_product("A1", "Cola", 150, quantity=5)

    machine.set_out_of_service()
    assert machine.state == MachineState.OUT_OF_SERVICE
    assert not machine.select_product("A1")

    machine.set_in_service()
    assert machine.state == MachineState.IDLE
    assert machine.select_product("A1")
    machine.cancel()
    print("✅ test_out_of_service passed")


def test_change_calculator() -> None:
    """Test change calculation with greedy algorithm."""
    calc = ChangeCalculator()

    # $0.75 → 3 quarters
    change = calc.make_change(75)
    assert change is not None
    total = sum(d * c for d, c in change)
    assert total == 75

    # $1.30 → 1 dollar + 1 quarter + 1 nickel
    change = calc.make_change(130)
    assert change is not None
    total = sum(d * c for d, c in change)
    assert total == 130
    print("✅ test_change_calculator passed")


def test_multiple_purchases() -> None:
    """Test multiple sequential purchases."""
    machine = VendingMachine()
    machine.add_product("A1", "Cola", 100, quantity=3)

    for _ in range(3):
        machine.select_product("A1")
        machine.insert_money(Denomination.DOLLAR.value)

    assert machine.sales.items_sold == 3
    assert machine.ledger.total_revenue_cents == 300
    assert not machine.inventory.has_stock("A1")
    print("✅ test_multiple_purchases passed")


def test_denomination_values() -> None:
    """Test denomination cent values."""
    assert Denomination.PENNY.value == 1
    assert Denomination.QUARTER.value == 25
    assert Denomination.DOLLAR.value == 100
    assert Denomination.TWENTY_DOLLARS.value == 2000
    print("✅ test_denomination_values passed")


def test_full_lifecycle() -> None:
    """Test full lifecycle: setup → buy → cancel → restock."""
    machine = VendingMachine()
    machine.add_product("A1", "Cola", 150, quantity=2)

    # Purchase 1: success
    machine.select_product("A1")
    machine.insert_money(Denomination.DOLLAR.value)
    machine.insert_money(Denomination.HALF_DOLLAR.value)
    assert machine.state == MachineState.IDLE
    assert machine.sales.items_sold == 1

    # Purchase 2: cancel
    machine.select_product("A1")
    machine.insert_money(Denomination.QUARTER.value)
    refund = machine.cancel()
    assert refund == 25
    assert machine.state == MachineState.IDLE

    # Purchase 3: success (last item)
    machine.select_product("A1")
    machine.insert_money(Denomination.DOLLAR.value)
    machine.insert_money(Denomination.DOLLAR.value)
    assert machine.state == MachineState.IDLE
    assert machine.sales.items_sold == 2

    # Out of stock
    assert not machine.select_product("A1")

    # Restock and buy
    machine.restock("A1", 1)
    machine.select_product("A1")
    machine.insert_money(Denomination.DOLLAR.value)
    machine.insert_money(Denomination.HALF_DOLLAR.value)
    assert machine.sales.items_sold == 3
    print("✅ test_full_lifecycle passed")


def run_all_tests() -> None:
    print("\n" + "=" * 40)
    print("  RUNNING TESTS")
    print("=" * 40 + "\n")
    test_product_creation()
    test_slot_operations()
    test_inventory()
    test_coin_payment()
    test_bill_payment()
    test_combined_payment()
    test_card_payment()
    test_state_idle_select()
    test_state_idle_invalid_actions()
    test_state_payment_insert()
    test_state_payment_cancel()
    test_out_of_stock()
    test_invalid_slot()
    test_exact_payment()
    test_overpayment_change()
    test_restock()
    test_transaction_ledger()
    test_out_of_service()
    test_change_calculator()
    test_multiple_purchases()
    test_denomination_values()
    test_full_lifecycle()
    print("\n✅ All tests passed!\n")


if __name__ == "__main__":
    main_scenario()
    demo_out_of_stock()
    demo_payment_strategies()
    demo_change_calculator()
    demo_out_of_service()
    run_all_tests()
```

Each test isolates a single behavior. `test_product_creation` verifies the price display format. `test_slot_operations` confirms dispense decrements stock, returns `None` when empty, and restock caps at capacity. `test_coin_payment` verifies coins are accepted but $5 bills are rejected. `test_combined_payment` verifies both coins and bills are accepted. `test_state_idle_select` confirms the machine transitions from `IDLE` to `ACCEPTING_PAYMENT` on valid selection. `test_state_idle_invalid_actions` verifies that money insertion is rejected before a product is selected. `test_state_payment_cancel` confirms that cancelling returns the inserted money and transitions back to `IDLE`. `test_overpayment_change` verifies that paying $5 for a $1 item records $4 change. `test_transaction_ledger` checks that both completed and cancelled transactions are recorded separately. `test_out_of_service` confirms the machine rejects all actions when out of service and resumes when restored. `test_change_calculator` verifies the greedy algorithm computes correct change. `test_full_lifecycle` runs a complete buy → cancel → buy → out-of-stock → restock → buy sequence.

---

## Class Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                       VendingMachine                                │
│──────────────────────────────────────────────────────────────────│
│ - _inventory: Inventory                                             │
│ - _payment: PaymentMethod                                           │
│ - _change_calc: ChangeCalculator                                    │
│ - _ledger: TransactionLedger                                        │
│ - _event_bus: VendingEventBus                                       │
│ - _sales: SalesTracker                                              │
│ - _ctx: VendingContext                                              │
│ - _states: dict[MachineState, VendingState]                         │
│──────────────────────────────────────────────────────────────────│
│ + add_product(code, name, price, qty) → None                        │
│ + select_product(code) → bool                                       │
│ + insert_money(amount_cents) → bool                                 │
│ + cancel() → int                                                    │
│ + restock(code, count) → int                                        │
│ + set_out_of_service() → None                                       │
│ + set_in_service() → None                                           │
│ + display_status() → None                                           │
└───────┬──────────┬──────────┬────────────────────────────────────┘
        │          │          │
   uses ▼     uses ▼     uses ▼
┌──────────────────────────────────────────────────────────────────┐
│  «abstract» VendingState (State Pattern)                            │
│──────────────────────────────────────────────────────────────────│
│ + select_product(ctx, code) → bool                                  │
│ + insert_money(ctx, amount) → bool                                  │
│ + dispense(ctx) → bool                                              │
│ + cancel(ctx) → int                                                 │
│ + state_name() → MachineState                                       │
└──────────┬───────────────────────────────────────────────────────┘
           │ implemented by
  ┌────────┼──────────────┬────────────────┐
  ▼        ▼              ▼                ▼
Idle     Accepting      Dispensing      OutOf
State    Payment        State           Service
         State                          State

┌──────────────────────────────────────────────────────────────────┐
│  VendingContext (Shared State)                                      │
│──────────────────────────────────────────────────────────────────│
│ + inventory: Inventory                                              │
│ + payment: PaymentMethod                                            │
│ + emit_fn: callable                                                 │
│ + selected_code: str                                                │
│ + selected_slot: Slot | None                                        │
│ + current_state: MachineState                                       │
│──────────────────────────────────────────────────────────────────│
│ + amount_due → int                                                  │
│ + is_paid → bool                                                    │
│ + clear_selection() → None                                          │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  «abstract» PaymentMethod (Strategy Pattern)                        │
│──────────────────────────────────────────────────────────────────│
│ + accept(amount_cents) → bool                                       │
│ + get_balance() → int                                               │
│ + reset() → int                                                     │
│ + name() → str                                                      │
└──────────┬───────────────────────────────────────────────────────┘
           │ implemented by
  ┌────────┼────────────┬────────────┐
  ▼        ▼            ▼            ▼
Coin     Bill        Combined     Card
Payment  Payment     Payment      Payment

┌──────────────────────────────────────────────────────────────────┐
│  Inventory                                                          │
│──────────────────────────────────────────────────────────────────│
│ - _slots: dict[str, Slot]                                           │
│──────────────────────────────────────────────────────────────────│
│ + add_slot(code, product, qty) → None                               │
│ + get_slot(code) → Slot | None                                      │
│ + has_stock(code) → bool                                            │
│ + dispense(code) → Product | None                                   │
│ + restock(code, count) → int                                        │
│ + restock_all(count) → dict                                         │
│ + display() → None                                                  │
└──────────┬───────────────────────────────────────────────────────┘
           │ contains
           ▼
┌──────────────────────────────────────────────────────────────────┐
│  Slot                                                               │
│──────────────────────────────────────────────────────────────────│
│ + code: str                                                         │
│ + product: Product                                                  │
│ + quantity: int                                                     │
│ + capacity: int                                                     │
│──────────────────────────────────────────────────────────────────│
│ + is_empty → bool                                                   │
│ + is_full → bool                                                    │
│ + dispense_one() → Product | None                                   │
│ + restock(count) → int                                              │
└──────────┬───────────────────────────────────────────────────────┘
           │ contains
           ▼
┌──────────────────────────────────────────────────────────────────┐
│  Product                                                            │
│──────────────────────────────────────────────────────────────────│
│ + name: str                                                         │
│ + price_cents: int                                                  │
│ + category: str                                                     │
│ + price_display → str                                               │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  ChangeCalculator                                                   │
│──────────────────────────────────────────────────────────────────│
│ - _reserves: dict[int, int]                                         │
│──────────────────────────────────────────────────────────────────│
│ + make_change(amount) → list[(denom, count)] | None                 │
│ + add_to_reserves(denom, count) → None                              │
│ + format_change(change) → str                                       │
│ + total_reserves_cents → int                                        │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  TransactionLedger                                                  │
│──────────────────────────────────────────────────────────────────│
│ - _transactions: list[Transaction]                                  │
│──────────────────────────────────────────────────────────────────│
│ + record(transaction) → None                                        │
│ + completed_transactions → list[Transaction]                        │
│ + cancelled_transactions → list[Transaction]                        │
│ + total_revenue_cents → int                                         │
│ + display() → None                                                  │
└──────────┬───────────────────────────────────────────────────────┘
           │ contains
           ▼
┌──────────────────────────────────────────────────────────────────┐
│  Transaction                                                        │
│──────────────────────────────────────────────────────────────────│
│ + transaction_id: str                                               │
│ + slot_code: str                                                    │
│ + product_name: str                                                 │
│ + price_cents: int                                                  │
│ + amount_paid_cents: int                                            │
│ + change_cents: int                                                 │
│ + completed: bool                                                   │
│ + timestamp: float                                                  │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  «abstract» VendingEventListener (Observer Pattern)                 │
│──────────────────────────────────────────────────────────────────│
│ + on_event(event) → None                                            │
│ + listener_id() → str                                               │
└──────────┬────────────────────────────────────────────────────────┘
           │ implemented by
  ┌────────┴──────────┐
  ▼                   ▼
Vending            Sales
Logger             Tracker

┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│ VendingEvent     │ │ VendingEventBus  │ │ VendingScenario  │
│──────────────────│ │──────────────────│ │──────────────────│
│ + event_id       │ │ - _listeners     │ │ - _machine       │
│ + event_type     │ │ - _event_log     │ │ - _events        │
│ + slot_code      │ │──────────────────│ │──────────────────│
│ + detail         │ │ + register()     │ │ + at_tick()      │
│ + timestamp      │ │ + emit()         │ │ + run()          │
└──────────────────┘ │ + unregister()   │ └──────────────────┘
                     └──────────────────┘

┌──────────────────┐ ┌──────────────────┐
│ MachineState     │ │ EventType (Enum) │
│ (Enum)           │ │──────────────────│
│──────────────────│ │ PRODUCT_SELECTED │
│ IDLE             │ │ MONEY_INSERTED   │
│ SELECTING        │ │ PRODUCT_DISPENSED│
│ ACCEPTING_PAYMENT│ │ CHANGE_RETURNED  │
│ DISPENSING       │ │ TRANSACTION_     │
│ OUT_OF_SERVICE   │ │   CANCELLED      │
└──────────────────┘ │ REFUND_ISSUED    │
                     │ OUT_OF_STOCK     │
┌──────────────────┐ │ INSUFFICIENT_    │
│ Denomination     │ │   FUNDS          │
│ (Enum, cents)    │ │ RESTOCKED        │
│──────────────────│ │ STATE_CHANGED    │
│ PENNY = 1        │ │ ERROR            │
│ NICKEL = 5       │ └──────────────────┘
│ DIME = 10        │
│ QUARTER = 25     │
│ HALF_DOLLAR = 50 │
│ DOLLAR = 100     │
│ FIVE_DOLLARS= 500│
│ TEN_DOLLARS=1000 │
│ TWENTY_DOLLARS   │
│         = 2000   │
└──────────────────┘
```

---

## How to Run

Save all code into a single file `vending_machine.py` and run:

```bash
python vending_machine.py
```

### Example Output

```
╔══════════════════════════════════════════╗
║      VENDING MACHINE SYSTEM              ║
║  State + Strategy + Observer              ║
╚══════════════════════════════════════════╝

  ───────────────────────────────────────────────────────
  SIMULATION START (ticks: 1 → 10)
  ───────────────────────────────────────────────────────

  ⏱️  Tick 1: Select Cola (A1, $1.50)
  🔘 [      product_selected] Cola $1.50
  🔄 [         state_changed] IDLE → ACCEPTING_PAYMENT

  ⏱️  Tick 2: Insert $1.00 bill
  💰 [        money_inserted] inserted $1.00 | balance: $1.00 | due: $0.50
  ⏱️  Tick 2: Insert 2 quarters ($0.50)
  💰 [        money_inserted] inserted $0.25 | balance: $1.25 | due: $0.25
  💰 [        money_inserted] inserted $0.25 | balance: $1.50 | due: $0.00
  🔄 [         state_changed] ACCEPTING_PAYMENT → DISPENSING
  📦 [     product_dispensed] dispensed Cola
  🔄 [         state_changed] DISPENSING → IDLE

  ⏱️  Tick 3: Select Water (B1, $1.00)
  🔘 [      product_selected] Water $1.00
  🔄 [         state_changed] IDLE → ACCEPTING_PAYMENT

  ⏱️  Tick 4: Insert $5.00 bill (expect $4.00 change)
  💰 [        money_inserted] inserted $5.00 | balance: $5.00 | due: $0.00
  🔄 [         state_changed] ACCEPTING_PAYMENT → DISPENSING
  📦 [     product_dispensed] dispensed Water
  💵 [       change_returned] change: $4.00
  🔄 [         state_changed] DISPENSING → IDLE
  💵 [       change_returned] breakdown: 2×$20.00

  ⏱️  Tick 5: Select Orange Juice (B2, $2.00)
  🔘 [      product_selected] Orange Juice $2.00
  🔄 [         state_changed] IDLE → ACCEPTING_PAYMENT
  ⏱️  Tick 5: Insert $1.00
  💰 [        money_inserted] inserted $1.00 | balance: $1.00 | due: $1.00

  ⏱️  Tick 6: Cancel transaction (expect $1.00 refund)
  ↩️  [        refund_issued] refunded $1.00
  🚫 [transaction_cancelled] transaction cancelled
  🔄 [         state_changed] ACCEPTING_PAYMENT → IDLE

  ⏱️  Tick 7: Try dispensing without selection
  ❌ [               error] invalid slot code: Z9
  ⏱️  Tick 7: Insert money without selection
  ❌ [               error] select a product first

  ⏱️  Tick 8: Select Candy Bar (C2, $1.00)
  🔘 [      product_selected] Candy Bar $1.00
  🔄 [         state_changed] IDLE → ACCEPTING_PAYMENT

  ⏱️  Tick 9: Pay with 4 quarters
  💰 [        money_inserted] inserted $0.25 | balance: $0.25 | due: $0.75
  💰 [        money_inserted] inserted $0.25 | balance: $0.50 | due: $0.50
  💰 [        money_inserted] inserted $0.25 | balance: $0.75 | due: $0.25
  💰 [        money_inserted] inserted $0.25 | balance: $1.00 | due: $0.00
  🔄 [         state_changed] ACCEPTING_PAYMENT → DISPENSING
  📦 [     product_dispensed] dispensed Candy Bar
  🔄 [         state_changed] DISPENSING → IDLE

  ───────────────────────────────────────────────────────
  SIMULATION COMPLETE
  ───────────────────────────────────────────────────────

========================================
  RUNNING TESTS
========================================

✅ test_product_creation passed
✅ test_slot_operations passed
✅ test_inventory passed
✅ test_coin_payment passed
✅ test_bill_payment passed
✅ test_combined_payment passed
✅ test_card_payment passed
✅ test_state_idle_select passed
✅ test_state_idle_invalid_actions passed
✅ test_state_payment_insert passed
✅ test_state_payment_cancel passed
✅ test_out_of_stock passed
✅ test_invalid_slot passed
✅ test_exact_payment passed
✅ test_overpayment_change passed
✅ test_restock passed
✅ test_transaction_ledger passed
✅ test_out_of_service passed
✅ test_change_calculator passed
✅ test_multiple_purchases passed
✅ test_denomination_values passed
✅ test_full_lifecycle passed

✅ All tests passed!
```
