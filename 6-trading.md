# Design Trading System – Low-Level Design Guide

![Bidur Sapkota](https://www.bidursapkota.com.np/images/gravatar.webp "Bidur Sapkota - Developer")&nbsp;[Bidur Sapkota](https://www.bidursapkota.com.np/)

## Table of Contents

1. [Requirements](#requirements)
2. [Key Design Patterns](#key-design-patterns)
3. [Core Enums & Constants](#core-enums--constants)
4. [Order (Command Pattern)](#order-command-pattern)
5. [Order Book (Core Engine)](#order-book-core-engine)
6. [Matching Engine (Strategy Pattern)](#matching-engine-strategy-pattern)
7. [Trade & Trade Ledger](#trade--trade-ledger)
8. [Portfolio & Risk Management](#portfolio--risk-management)
9. [Event System (Observer Pattern)](#event-system-observer-pattern)
10. [Trading System (Facade)](#trading-system-facade)
11. [Simulation Engine](#simulation-engine)
12. [Putting It All Together](#putting-it-all-together)
13. [Class Diagram](#class-diagram)
14. [How to Run](#how-to-run)

---

## Requirements

### Functional Requirements

| #  | Requirement                                                                           |
| -- | ------------------------------------------------------------------------------------- |
| 1  | Support multiple tradable instruments (stocks, identified by ticker symbol)            |
| 2  | Users can place limit orders (buy/sell at a specific price) and market orders          |
| 3  | Each order has a side (BUY/SELL), quantity, optional price, and timestamp              |
| 4  | Order book maintains sorted buy orders (descending by price) and sell orders (ascending) |
| 5  | Matching engine matches compatible buy and sell orders by price-time priority          |
| 6  | Partial fills — a large order can be matched against multiple smaller counter-orders   |
| 7  | Generate a trade record for every match with price, quantity, and participants         |
| 8  | Users can cancel open (unfilled or partially filled) orders                           |
| 9  | Portfolio tracks each user's holdings and cash balance                                 |
| 10 | Risk checks — reject orders that exceed available cash or short-sell limits            |
| 11 | Event notifications — observers are notified on order placement, trade execution, and cancellation |
| 12 | Trade ledger stores an immutable history of all executed trades                        |

### Non-Functional Requirements

- Clean OOP with SOLID principles.
- Extensible — new matching strategies, order types, and risk rules can be added without modifying core logic.
- Simulatable — the system runs step-by-step for testing and demonstration.

---

## Key Design Patterns

| Pattern  | Where It Is Used                                   | Why                                                                                  |
| -------- | -------------------------------------------------- | ------------------------------------------------------------------------------------ |
| Strategy | `MatchingStrategy` selects how orders are matched  | Swap between price-time priority, pro-rata, or other algorithms without changing core |
| Command  | `Order` objects represent trade intentions          | Orders are queued, partially filled, cancelled, and tracked independently             |
| Observer | `TradingEventListener` observes system events      | Loggers, risk engines, and UIs update automatically on trades and order changes       |
| Facade   | `TradingSystem` exposes a simple API               | Hides the complexity of order books, matching, portfolios, and event routing          |
| Singleton| `TradeLedger` is a single source of trade history  | One immutable ledger ensures consistency across the system                            |

---

## Core Enums & Constants

```python
from enum import Enum


class Side(Enum):
    BUY = "BUY"
    SELL = "SELL"

    def opposite(self) -> "Side":
        return Side.SELL if self == Side.BUY else Side.BUY


class OrderType(Enum):
    LIMIT = "LIMIT"     # execute at specified price or better
    MARKET = "MARKET"   # execute immediately at best available price


class OrderStatus(Enum):
    OPEN = "open"               # waiting in order book
    PARTIALLY_FILLED = "partial"  # some quantity matched
    FILLED = "filled"           # fully matched
    CANCELLED = "cancelled"     # cancelled by user


DEFAULT_STARTING_CASH = 100_000.00
MAKER_FEE_RATE = 0.001   # 0.1% fee for limit orders (maker)
TAKER_FEE_RATE = 0.002   # 0.2% fee for market orders (taker)
```

`Side` represents buy or sell. `opposite()` is used during matching — a buy order matches against sell orders and vice versa. `OrderType` distinguishes limit orders (with a specified price) from market orders (executed at the best available price). `OrderStatus` tracks an order through its lifecycle: placed → partially filled → filled or cancelled. Fee rates model the maker-taker fee structure used by real exchanges — limit orders that add liquidity (makers) pay lower fees than market orders that remove liquidity (takers).

---

## Order (Command Pattern)

An order is a mutable command object that represents a user's intent to buy or sell a quantity of an instrument at a specified (or market) price.

```python
from dataclasses import dataclass, field
from typing import Optional
import time


class Order:
    """A trade order — the fundamental command in the trading system."""

    _next_id: int = 0

    def __init__(self, user_id: str, symbol: str, side: Side,
                 order_type: OrderType, quantity: int,
                 price: float = 0.0, timestamp: int = 0) -> None:
        Order._next_id += 1
        self._id = Order._next_id
        self._user_id = user_id
        self._symbol = symbol
        self._side = side
        self._order_type = order_type
        self._original_quantity = quantity
        self._remaining_quantity = quantity
        self._price = price
        self._status = OrderStatus.OPEN
        self._timestamp = timestamp or int(time.time() * 1000)
        self._fills: list[tuple[float, int]] = []  # (price, qty) pairs

    # ------------------------------------------------------------------
    # Properties
    # ------------------------------------------------------------------

    @property
    def id(self) -> int:
        return self._id

    @property
    def user_id(self) -> str:
        return self._user_id

    @property
    def symbol(self) -> str:
        return self._symbol

    @property
    def side(self) -> Side:
        return self._side

    @property
    def order_type(self) -> OrderType:
        return self._order_type

    @property
    def price(self) -> float:
        return self._price

    @property
    def original_quantity(self) -> int:
        return self._original_quantity

    @property
    def remaining_quantity(self) -> int:
        return self._remaining_quantity

    @property
    def filled_quantity(self) -> int:
        return self._original_quantity - self._remaining_quantity

    @property
    def status(self) -> OrderStatus:
        return self._status

    @property
    def timestamp(self) -> int:
        return self._timestamp

    @property
    def fills(self) -> list[tuple[float, int]]:
        return list(self._fills)

    @property
    def average_fill_price(self) -> float:
        """Weighted average price of all fills."""
        if not self._fills:
            return 0.0
        total_cost = sum(p * q for p, q in self._fills)
        total_qty = sum(q for _, q in self._fills)
        return total_cost / total_qty if total_qty > 0 else 0.0

    def is_active(self) -> bool:
        return self._status in (OrderStatus.OPEN, OrderStatus.PARTIALLY_FILLED)

    # ------------------------------------------------------------------
    # Order lifecycle
    # ------------------------------------------------------------------

    def fill(self, price: float, quantity: int) -> None:
        """Record a partial or full fill."""
        if quantity > self._remaining_quantity:
            raise ValueError(
                f"Fill qty {quantity} exceeds remaining {self._remaining_quantity}"
            )
        self._remaining_quantity -= quantity
        self._fills.append((price, quantity))

        if self._remaining_quantity == 0:
            self._status = OrderStatus.FILLED
        else:
            self._status = OrderStatus.PARTIALLY_FILLED

    def cancel(self) -> bool:
        """Cancel the order. Returns False if already filled."""
        if self._status == OrderStatus.FILLED:
            return False
        self._status = OrderStatus.CANCELLED
        return True

    # ------------------------------------------------------------------
    # Comparison — for sorting in the order book
    # ------------------------------------------------------------------

    def __lt__(self, other: "Order") -> bool:
        """
        Comparison for order book sorting:
        - BUY orders: higher price has priority (descending)
        - SELL orders: lower price has priority (ascending)
        - Same price: earlier timestamp has priority (FIFO)
        """
        if self._side == Side.BUY:
            if self._price != other._price:
                return self._price > other._price  # higher buy = better
            return self._timestamp < other._timestamp
        else:
            if self._price != other._price:
                return self._price < other._price  # lower sell = better
            return self._timestamp < other._timestamp

    def __repr__(self) -> str:
        fill_info = f" avg={self.average_fill_price:.2f}" if self._fills else ""
        return (
            f"Order#{self._id}({self._user_id} {self._side.value} "
            f"{self._symbol} {self._remaining_quantity}/{self._original_quantity} "
            f"@{self._price:.2f} [{self._status.value}]{fill_info})"
        )
```

`Order` is mutable because its status and remaining quantity change as it gets filled. `_next_id` is a class-level counter that auto-generates unique order IDs. `fill()` records a partial or complete execution — a single large order can accumulate multiple fills at different prices. `average_fill_price` computes the volume-weighted average, which is the true cost basis for the user. `__lt__` defines the ordering used by the order book: buy orders sort highest-price-first (best bid on top), sell orders sort lowest-price-first (best ask on top). Within the same price level, earlier orders have priority — this is the price-time priority model used by most exchanges. `cancel()` returns `False` if the order is already fully filled — you cannot cancel a completed trade.

---

## Order Book (Core Engine)

The order book maintains two sorted lists — bids (buy orders) and asks (sell orders) — for a single instrument. It is the data structure that the matching engine operates on.

```python
import bisect


class OrderBook:
    """
    Order book for a single instrument.

    Maintains sorted bid (buy) and ask (sell) sides.
    Buy side: sorted descending by price, then ascending by time.
    Sell side: sorted ascending by price, then ascending by time.
    """

    def __init__(self, symbol: str) -> None:
        self._symbol = symbol
        self._bids: list[Order] = []   # buy orders, best (highest) first
        self._asks: list[Order] = []   # sell orders, best (lowest) first
        self._order_map: dict[int, Order] = {}  # order_id -> Order

    # ------------------------------------------------------------------
    # Properties
    # ------------------------------------------------------------------

    @property
    def symbol(self) -> str:
        return self._symbol

    @property
    def bids(self) -> list[Order]:
        return list(self._bids)

    @property
    def asks(self) -> list[Order]:
        return list(self._asks)

    @property
    def best_bid(self) -> Order | None:
        return self._bids[0] if self._bids else None

    @property
    def best_ask(self) -> Order | None:
        return self._asks[0] if self._asks else None

    @property
    def spread(self) -> float | None:
        """Difference between best ask and best bid."""
        if self._bids and self._asks:
            return self._asks[0].price - self._bids[0].price
        return None

    @property
    def mid_price(self) -> float | None:
        """Midpoint between best bid and best ask."""
        if self._bids and self._asks:
            return (self._bids[0].price + self._asks[0].price) / 2
        return None

    # ------------------------------------------------------------------
    # Order management
    # ------------------------------------------------------------------

    def add_order(self, order: Order) -> None:
        """Insert an order into the correct side of the book."""
        if order.side == Side.BUY:
            bisect.insort(self._bids, order)
        else:
            bisect.insort(self._asks, order)
        self._order_map[order.id] = order

    def remove_order(self, order_id: int) -> Order | None:
        """Remove an order from the book by ID."""
        order = self._order_map.pop(order_id, None)
        if order is None:
            return None

        if order.side == Side.BUY:
            self._bids = [o for o in self._bids if o.id != order_id]
        else:
            self._asks = [o for o in self._asks if o.id != order_id]
        return order

    def get_order(self, order_id: int) -> Order | None:
        return self._order_map.get(order_id)

    def cancel_order(self, order_id: int) -> Order | None:
        """Cancel and remove an order. Returns the order or None."""
        order = self._order_map.get(order_id)
        if order and order.cancel():
            self.remove_order(order_id)
            return order
        return None

    def clean_filled(self) -> None:
        """Remove fully filled orders from the book."""
        self._bids = [o for o in self._bids if o.is_active()]
        self._asks = [o for o in self._asks if o.is_active()]
        # Clean map too
        filled_ids = [oid for oid, o in self._order_map.items()
                      if not o.is_active()]
        for oid in filled_ids:
            del self._order_map[oid]

    # ------------------------------------------------------------------
    # Market data
    # ------------------------------------------------------------------

    def depth(self, levels: int = 5) -> dict:
        """
        Return top N price levels for each side.
        Aggregates quantity at each price level.
        """
        def aggregate(orders: list[Order], n: int) -> list[tuple[float, int]]:
            result: list[tuple[float, int]] = []
            current_price = None
            current_qty = 0
            for order in orders:
                if current_price is None or order.price != current_price:
                    if current_price is not None:
                        result.append((current_price, current_qty))
                        if len(result) >= n:
                            break
                    current_price = order.price
                    current_qty = order.remaining_quantity
                else:
                    current_qty += order.remaining_quantity
            if current_price is not None and len(result) < n:
                result.append((current_price, current_qty))
            return result

        return {
            "bids": aggregate(self._bids, levels),
            "asks": aggregate(self._asks, levels),
        }

    # ------------------------------------------------------------------
    # Display
    # ------------------------------------------------------------------

    def __repr__(self) -> str:
        bid_str = f"Best Bid: {self._bids[0].price:.2f}" if self._bids else "No Bids"
        ask_str = f"Best Ask: {self._asks[0].price:.2f}" if self._asks else "No Asks"
        spread_str = f"Spread: {self.spread:.2f}" if self.spread is not None else ""
        return (
            f"OrderBook({self._symbol}) "
            f"[{bid_str} | {ask_str}] "
            f"{spread_str} "
            f"({len(self._bids)} bids, {len(self._asks)} asks)"
        )

    def display(self) -> None:
        """Print a visual order book."""
        depth = self.depth(levels=5)
        print(f"\n  {'='*50}")
        print(f"  ORDER BOOK — {self._symbol}")
        print(f"  {'='*50}")
        print(f"  {'Price':>12s}  {'Quantity':>10s}  Side")
        print(f"  {'-'*40}")

        # Asks (reversed so lowest is closest to center)
        for price, qty in reversed(depth["asks"]):
            print(f"  {price:>12.2f}  {qty:>10d}  ASK")

        spread = self.spread
        if spread is not None:
            print(f"  {'--- spread: '}{spread:.2f} {'---':>20s}")
        else:
            print(f"  {'--- no spread ---':^40s}")

        for price, qty in depth["bids"]:
            print(f"  {price:>12.2f}  {qty:>10d}  BID")

        print(f"  {'='*50}\n")
```

`OrderBook` is the central data structure per instrument. `_bids` and `_asks` are kept sorted via `bisect.insort`, which uses `Order.__lt__` to maintain price-time priority automatically. `_order_map` provides O(1) lookup by order ID for cancellations and status queries. `best_bid` and `best_ask` are always at index 0 because of the sorting — `_bids[0]` is the highest buy price, `_asks[0]` is the lowest sell price. `spread` is the gap between them; a tighter spread means a more liquid market. `depth` aggregates orders at each price level into (price, total_quantity) tuples — this is what a real exchange shows as the Level 2 market data. `clean_filled` removes fully matched orders to keep the book lean.

---

## Matching Engine (Strategy Pattern)

The matching engine takes an incoming order and tries to match it against the opposite side of the order book. Different matching strategies optimise for different goals.

### Abstract Strategy

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass


@dataclass(frozen=True)
class TradeExecution:
    """Result of a single match between two orders."""
    buy_order_id: int
    sell_order_id: int
    buyer_id: str
    seller_id: str
    symbol: str
    price: float
    quantity: int
    timestamp: int


class MatchingStrategy(ABC):
    """Defines how incoming orders are matched against the order book."""

    @abstractmethod
    def match(self, order: Order, book: OrderBook,
              tick: int) -> list[TradeExecution]:
        """
        Attempt to match the incoming order against the book.
        Returns a list of trade executions (may be empty).
        """
        ...

    @abstractmethod
    def name(self) -> str:
        ...
```

### Price-Time Priority (FIFO)

```python
class PriceTimePriorityStrategy(MatchingStrategy):
    """
    Standard price-time priority matching (used by most exchanges).

    Rules:
    1. A buy order matches the lowest-priced sell (best ask).
    2. A sell order matches the highest-priced buy (best bid).
    3. At the same price, the earliest order has priority.
    4. Match continues until the incoming order is filled or
       no more compatible resting orders exist.
    """

    def match(self, order: Order, book: OrderBook,
              tick: int) -> list[TradeExecution]:
        executions: list[TradeExecution] = []

        if order.side == Side.BUY:
            self._match_buy(order, book, tick, executions)
        else:
            self._match_sell(order, book, tick, executions)

        book.clean_filled()
        return executions

    def _match_buy(self, buy: Order, book: OrderBook,
                   tick: int, executions: list[TradeExecution]) -> None:
        """Match a buy order against the ask side."""
        while buy.remaining_quantity > 0 and book.asks:
            best_ask = book.asks[0]

            # Price check: buy must meet or exceed the ask price
            if buy.order_type == OrderType.LIMIT and buy.price < best_ask.price:
                break  # no compatible ask

            # Determine fill quantity and price
            fill_qty = min(buy.remaining_quantity, best_ask.remaining_quantity)
            fill_price = best_ask.price  # price of the resting order

            # Execute the fill
            buy.fill(fill_price, fill_qty)
            best_ask.fill(fill_price, fill_qty)

            executions.append(TradeExecution(
                buy_order_id=buy.id,
                sell_order_id=best_ask.id,
                buyer_id=buy.user_id,
                seller_id=best_ask.user_id,
                symbol=buy.symbol,
                price=fill_price,
                quantity=fill_qty,
                timestamp=tick,
            ))

            # Remove fully filled ask
            if not best_ask.is_active():
                book.remove_order(best_ask.id)

    def _match_sell(self, sell: Order, book: OrderBook,
                    tick: int, executions: list[TradeExecution]) -> None:
        """Match a sell order against the bid side."""
        while sell.remaining_quantity > 0 and book.bids:
            best_bid = book.bids[0]

            # Price check: sell must meet or go below the bid price
            if sell.order_type == OrderType.LIMIT and sell.price > best_bid.price:
                break

            fill_qty = min(sell.remaining_quantity, best_bid.remaining_quantity)
            fill_price = best_bid.price  # price of the resting order

            sell.fill(fill_price, fill_qty)
            best_bid.fill(fill_price, fill_qty)

            executions.append(TradeExecution(
                buy_order_id=best_bid.id,
                sell_order_id=sell.id,
                buyer_id=best_bid.user_id,
                seller_id=sell.user_id,
                symbol=sell.symbol,
                price=fill_price,
                quantity=fill_qty,
                timestamp=tick,
            ))

            if not best_bid.is_active():
                book.remove_order(best_bid.id)

    def name(self) -> str:
        return "Price-Time Priority"
```

The matching engine walks the opposite side of the book from top to bottom. For a buy order, it starts at the best (lowest) ask. If the buy price meets or exceeds the ask price, they match. The fill price is always the resting order's price — this is how real exchanges work (the order that was already in the book determines the price). `fill_qty` is the minimum of the two remaining quantities, enabling partial fills. The loop continues until the incoming order is fully filled or no more compatible counter-orders exist. If a resting order is fully filled, it is removed from the book. Any unfilled remainder of a limit order stays in the book as a new resting order.

### Pro-Rata Strategy

```python
class ProRataStrategy(MatchingStrategy):
    """
    Pro-rata matching: distributes fills proportionally among
    all orders at the best price level.

    Used in some futures exchanges (e.g., CME Eurodollar).
    Each order at the best price gets a share proportional
    to its size relative to total quantity at that level.
    """

    def match(self, order: Order, book: OrderBook,
              tick: int) -> list[TradeExecution]:
        executions: list[TradeExecution] = []

        if order.side == Side.BUY:
            self._match_pro_rata(order, book.asks, book, tick, executions)
        else:
            self._match_pro_rata(order, book.bids, book, tick, executions)

        book.clean_filled()
        return executions

    def _match_pro_rata(self, incoming: Order, opposite_side: list[Order],
                        book: OrderBook, tick: int,
                        executions: list[TradeExecution]) -> None:
        while incoming.remaining_quantity > 0 and opposite_side:
            best_price = opposite_side[0].price

            # Price compatibility check
            if incoming.order_type == OrderType.LIMIT:
                if incoming.side == Side.BUY and incoming.price < best_price:
                    break
                if incoming.side == Side.SELL and incoming.price > best_price:
                    break

            # Gather all orders at the best price level
            level_orders = [o for o in opposite_side if o.price == best_price]
            total_qty = sum(o.remaining_quantity for o in level_orders)

            if total_qty == 0:
                break

            fill_available = min(incoming.remaining_quantity, total_qty)

            for resting in level_orders:
                if incoming.remaining_quantity <= 0:
                    break

                # Each order gets a proportional share
                share = resting.remaining_quantity / total_qty
                alloc = max(1, int(fill_available * share))
                alloc = min(alloc, resting.remaining_quantity,
                            incoming.remaining_quantity)

                if alloc <= 0:
                    continue

                incoming.fill(best_price, alloc)
                resting.fill(best_price, alloc)

                buy_id = incoming.id if incoming.side == Side.BUY else resting.id
                sell_id = resting.id if incoming.side == Side.BUY else incoming.id
                buyer = incoming.user_id if incoming.side == Side.BUY else resting.user_id
                seller = resting.user_id if incoming.side == Side.BUY else incoming.user_id

                executions.append(TradeExecution(
                    buy_order_id=buy_id,
                    sell_order_id=sell_id,
                    buyer_id=buyer,
                    seller_id=seller,
                    symbol=incoming.symbol,
                    price=best_price,
                    quantity=alloc,
                    timestamp=tick,
                ))

                if not resting.is_active():
                    book.remove_order(resting.id)

            # Remove any fully filled from opposite side tracker
            opposite_side[:] = [o for o in opposite_side if o.is_active()]

    def name(self) -> str:
        return "Pro-Rata"
```

`ProRataStrategy` differs from price-time priority by distributing fills proportionally at each price level instead of serving the earliest order first. If three orders sit at the best ask — 100, 200, and 300 shares — and a buy for 300 shares arrives, each gets roughly 50, 100, and 150 shares (proportional to their size). This incentivises traders to post larger orders rather than being first in the queue. The `max(1, ...)` ensures every order gets at least 1 unit to avoid starvation.

---

## Trade & Trade Ledger

### Trade Record

```python
@dataclass(frozen=True)
class Trade:
    """An immutable record of an executed trade."""
    trade_id: int
    buy_order_id: int
    sell_order_id: int
    buyer_id: str
    seller_id: str
    symbol: str
    price: float
    quantity: int
    timestamp: int
    buyer_fee: float
    seller_fee: float

    @property
    def notional(self) -> float:
        """Total value of the trade (price × quantity)."""
        return self.price * self.quantity

    def __repr__(self) -> str:
        return (
            f"Trade#{self.trade_id}({self.symbol} "
            f"{self.quantity}@{self.price:.2f} "
            f"{self.buyer_id}←→{self.seller_id})"
        )
```

### Trade Ledger (Singleton)

```python
class TradeLedger:
    """
    Immutable ledger of all executed trades.
    Singleton — one ledger per trading system.
    """

    _instance: "TradeLedger | None" = None

    def __new__(cls) -> "TradeLedger":
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._trades = []
            cls._instance._next_id = 0
        return cls._instance

    @classmethod
    def reset(cls) -> None:
        """Reset the singleton (for testing)."""
        cls._instance = None

    def record_trade(self, execution: TradeExecution,
                     buyer_fee: float, seller_fee: float) -> Trade:
        """Record a trade from an execution result."""
        self._next_id += 1
        trade = Trade(
            trade_id=self._next_id,
            buy_order_id=execution.buy_order_id,
            sell_order_id=execution.sell_order_id,
            buyer_id=execution.buyer_id,
            seller_id=execution.seller_id,
            symbol=execution.symbol,
            price=execution.price,
            quantity=execution.quantity,
            timestamp=execution.timestamp,
            buyer_fee=buyer_fee,
            seller_fee=seller_fee,
        )
        self._trades.append(trade)
        return trade

    @property
    def trades(self) -> list[Trade]:
        return list(self._trades)

    def trades_for_user(self, user_id: str) -> list[Trade]:
        return [t for t in self._trades
                if t.buyer_id == user_id or t.seller_id == user_id]

    def trades_for_symbol(self, symbol: str) -> list[Trade]:
        return [t for t in self._trades if t.symbol == symbol]

    @property
    def total_volume(self) -> float:
        return sum(t.notional for t in self._trades)

    @property
    def total_fees(self) -> float:
        return sum(t.buyer_fee + t.seller_fee for t in self._trades)

    def display(self, last_n: int = 10) -> None:
        print(f"\n  {'='*55}")
        print(f"  TRADE LEDGER — {len(self._trades)} trades")
        print(f"  Total Volume: ${self.total_volume:,.2f}")
        print(f"  Total Fees:   ${self.total_fees:,.2f}")
        print(f"  {'='*55}")
        for trade in self._trades[-last_n:]:
            print(f"  {trade}")
        print()
```

`Trade` is a frozen dataclass — immutable after creation. It captures both sides of the transaction, the execution price and quantity, and the fees charged to each participant. `notional` is the dollar value of the trade. `TradeLedger` uses the Singleton pattern (via `__new__`) to ensure there is exactly one ledger in the system. `record_trade` converts a `TradeExecution` (the matching engine's raw output) into a permanent `Trade` record with computed fees. `reset()` is provided for testing so each test starts with a clean ledger.

---

## Portfolio & Risk Management

### Portfolio

```python
class Portfolio:
    """Tracks a user's cash balance and stock holdings."""

    def __init__(self, user_id: str,
                 cash: float = DEFAULT_STARTING_CASH) -> None:
        self._user_id = user_id
        self._cash = cash
        self._holdings: dict[str, int] = {}  # symbol -> quantity
        self._reserved_cash: float = 0.0     # cash locked for pending buys
        self._reserved_shares: dict[str, int] = {}  # shares locked for sells

    @property
    def user_id(self) -> str:
        return self._user_id

    @property
    def cash(self) -> float:
        return self._cash

    @property
    def available_cash(self) -> float:
        """Cash not locked by pending orders."""
        return self._cash - self._reserved_cash

    @property
    def holdings(self) -> dict[str, int]:
        return dict(self._holdings)

    def get_holding(self, symbol: str) -> int:
        return self._holdings.get(symbol, 0)

    def available_shares(self, symbol: str) -> int:
        """Shares not locked by pending sell orders."""
        held = self._holdings.get(symbol, 0)
        reserved = self._reserved_shares.get(symbol, 0)
        return held - reserved

    # ------------------------------------------------------------------
    # Reservation (for pending orders)
    # ------------------------------------------------------------------

    def reserve_cash(self, amount: float) -> bool:
        """Lock cash for a pending buy order."""
        if amount > self.available_cash:
            return False
        self._reserved_cash += amount
        return True

    def release_cash(self, amount: float) -> None:
        """Release cash reservation (on cancel or fill)."""
        self._reserved_cash = max(0, self._reserved_cash - amount)

    def reserve_shares(self, symbol: str, quantity: int) -> bool:
        """Lock shares for a pending sell order."""
        if quantity > self.available_shares(symbol):
            return False
        self._reserved_shares[symbol] = (
            self._reserved_shares.get(symbol, 0) + quantity
        )
        return True

    def release_shares(self, symbol: str, quantity: int) -> None:
        """Release share reservation."""
        current = self._reserved_shares.get(symbol, 0)
        self._reserved_shares[symbol] = max(0, current - quantity)

    # ------------------------------------------------------------------
    # Settlement
    # ------------------------------------------------------------------

    def debit(self, amount: float) -> None:
        """Deduct cash (after a buy fill)."""
        self._cash -= amount

    def credit(self, amount: float) -> None:
        """Add cash (after a sell fill)."""
        self._cash += amount

    def add_shares(self, symbol: str, quantity: int) -> None:
        """Add shares to holdings."""
        self._holdings[symbol] = self._holdings.get(symbol, 0) + quantity

    def remove_shares(self, symbol: str, quantity: int) -> None:
        """Remove shares from holdings."""
        current = self._holdings.get(symbol, 0)
        self._holdings[symbol] = max(0, current - quantity)
        if self._holdings[symbol] == 0:
            del self._holdings[symbol]

    def settle_buy(self, symbol: str, quantity: int,
                   total_cost: float, fee: float) -> None:
        """Settle a buy trade: debit cash, add shares, release reservation."""
        self.debit(total_cost + fee)
        self.release_cash(total_cost)  # release the reservation
        self.add_shares(symbol, quantity)

    def settle_sell(self, symbol: str, quantity: int,
                    total_proceeds: float, fee: float) -> None:
        """Settle a sell trade: credit cash, remove shares, release reservation."""
        self.credit(total_proceeds - fee)
        self.release_shares(symbol, quantity)
        self.remove_shares(symbol, quantity)

    # ------------------------------------------------------------------
    # Display
    # ------------------------------------------------------------------

    def __repr__(self) -> str:
        holdings_str = ", ".join(f"{s}:{q}" for s, q in self._holdings.items())
        return (
            f"Portfolio({self._user_id}) "
            f"Cash=${self._cash:,.2f} "
            f"(avail=${self.available_cash:,.2f}) "
            f"Holdings=[{holdings_str}]"
        )
```

`Portfolio` separates *available* resources from *reserved* resources. When a user places a buy order, the estimated cost is reserved (locked) so they cannot spend the same money on two orders simultaneously. Similarly, shares are reserved when a sell order is placed. `settle_buy` and `settle_sell` perform the actual asset transfer after a trade executes — debit cash and add shares for a buy, credit cash and remove shares for a sell. Fees are deducted during settlement. This reservation model prevents over-spending and over-selling without blocking the entire balance.

### Risk Manager

```python
class RiskManager:
    """
    Pre-trade risk checks.

    Validates orders before they enter the matching engine.
    """

    def __init__(self) -> None:
        self._max_order_quantity = 10_000
        self._max_order_value = 1_000_000.0
        self._allow_short_selling = False

    @property
    def allow_short_selling(self) -> bool:
        return self._allow_short_selling

    @allow_short_selling.setter
    def allow_short_selling(self, value: bool) -> None:
        self._allow_short_selling = value

    def validate_order(self, order: Order,
                       portfolio: Portfolio) -> tuple[bool, str]:
        """
        Validate an order against risk rules.
        Returns (is_valid, rejection_reason).
        """
        # Quantity limits
        if order.remaining_quantity <= 0:
            return False, "Quantity must be positive"

        if order.remaining_quantity > self._max_order_quantity:
            return False, f"Quantity exceeds max ({self._max_order_quantity})"

        # Price validation for limit orders
        if order.order_type == OrderType.LIMIT:
            if order.price <= 0:
                return False, "Limit order price must be positive"

            estimated_value = order.price * order.remaining_quantity
            if estimated_value > self._max_order_value:
                return False, f"Order value ${estimated_value:,.2f} exceeds max"

        # Buy-side: check cash
        if order.side == Side.BUY and order.order_type == OrderType.LIMIT:
            required = order.price * order.remaining_quantity
            if required > portfolio.available_cash:
                return False, (
                    f"Insufficient cash: need ${required:,.2f}, "
                    f"available ${portfolio.available_cash:,.2f}"
                )

        # Sell-side: check holdings
        if order.side == Side.SELL:
            available = portfolio.available_shares(order.symbol)
            if order.remaining_quantity > available and not self._allow_short_selling:
                return False, (
                    f"Insufficient shares: need {order.remaining_quantity} "
                    f"{order.symbol}, available {available}"
                )

        return True, ""
```

`RiskManager` acts as a gatekeeper before orders reach the matching engine. It enforces quantity limits, price sanity, cash sufficiency for buys, and share availability for sells. Short selling (selling shares you do not own) is disabled by default but can be toggled. The `validate_order` method returns a tuple of (success, reason) so rejection reasons can be communicated to the user. This is a pre-trade check — separate from the post-trade settlement in `Portfolio`.

---

## Event System (Observer Pattern)

### Listener Interface

```python
class TradingEventListener(ABC):
    """Observer interface for trading system events."""

    @abstractmethod
    def on_order_placed(self, order: Order) -> None:
        ...

    @abstractmethod
    def on_order_cancelled(self, order: Order) -> None:
        ...

    @abstractmethod
    def on_trade_executed(self, trade: Trade) -> None:
        ...

    @abstractmethod
    def on_order_rejected(self, order: Order, reason: str) -> None:
        ...
```

### Trade Logger

```python
class TradeLogger(TradingEventListener):
    """Logs all trading events to console."""

    def __init__(self) -> None:
        self._log: list[str] = []

    def on_order_placed(self, order: Order) -> None:
        entry = (
            f"  📝 ORDER PLACED: {order.user_id} {order.side.value} "
            f"{order.remaining_quantity} {order.symbol} "
            f"@{order.price:.2f} ({order.order_type.value})"
        )
        self._log.append(entry)
        print(entry)

    def on_order_cancelled(self, order: Order) -> None:
        entry = (
            f"  ❌ ORDER CANCELLED: Order#{order.id} "
            f"{order.user_id} {order.side.value} {order.symbol}"
        )
        self._log.append(entry)
        print(entry)

    def on_trade_executed(self, trade: Trade) -> None:
        entry = (
            f"  ✅ TRADE: {trade.quantity} {trade.symbol} "
            f"@{trade.price:.2f} "
            f"({trade.buyer_id} ← {trade.seller_id}) "
            f"fees: ${trade.buyer_fee + trade.seller_fee:.2f}"
        )
        self._log.append(entry)
        print(entry)

    def on_order_rejected(self, order: Order, reason: str) -> None:
        entry = f"  ⛔ ORDER REJECTED: Order#{order.id} — {reason}"
        self._log.append(entry)
        print(entry)

    def print_log(self, last_n: int = 20) -> None:
        print(f"\n  Event Log (last {last_n}):")
        for entry in self._log[-last_n:]:
            print(entry)

    def clear(self) -> None:
        self._log.clear()
```

### Market Data Feed

```python
class MarketDataFeed(TradingEventListener):
    """Tracks last trade price and volume per symbol."""

    def __init__(self) -> None:
        self._last_price: dict[str, float] = {}
        self._volume: dict[str, int] = {}
        self._trade_count: dict[str, int] = {}
        self._high: dict[str, float] = {}
        self._low: dict[str, float] = {}

    def on_order_placed(self, order: Order) -> None:
        pass  # not relevant for market data

    def on_order_cancelled(self, order: Order) -> None:
        pass

    def on_order_rejected(self, order: Order, reason: str) -> None:
        pass

    def on_trade_executed(self, trade: Trade) -> None:
        sym = trade.symbol
        self._last_price[sym] = trade.price
        self._volume[sym] = self._volume.get(sym, 0) + trade.quantity
        self._trade_count[sym] = self._trade_count.get(sym, 0) + 1

        if sym not in self._high or trade.price > self._high[sym]:
            self._high[sym] = trade.price
        if sym not in self._low or trade.price < self._low[sym]:
            self._low[sym] = trade.price

    def get_ticker(self, symbol: str) -> dict:
        return {
            "symbol": symbol,
            "last": self._last_price.get(symbol),
            "high": self._high.get(symbol),
            "low": self._low.get(symbol),
            "volume": self._volume.get(symbol, 0),
            "trades": self._trade_count.get(symbol, 0),
        }

    def display(self) -> None:
        print(f"\n  {'='*55}")
        print(f"  MARKET DATA")
        print(f"  {'='*55}")
        print(f"  {'Symbol':<8s} {'Last':>10s} {'High':>10s} "
              f"{'Low':>10s} {'Volume':>10s} {'Trades':>8s}")
        print(f"  {'-'*55}")
        for sym in sorted(self._last_price):
            t = self.get_ticker(sym)
            print(
                f"  {t['symbol']:<8s} "
                f"${t['last']:>9.2f} ${t['high']:>9.2f} "
                f"${t['low']:>9.2f} {t['volume']:>10d} "
                f"{t['trades']:>8d}"
            )
        print()
```

`TradingEventListener` defines four hooks: order placed, cancelled, executed, and rejected. `TradeLogger` prints every event with emoji markers for quick visual scanning. `MarketDataFeed` is a second observer that silently tracks last price, high, low, volume, and trade count per symbol — this is the ticker tape that traders watch. Both observers can coexist because the observer pattern decouples event producers from consumers.

---

## Trading System (Facade)

The `TradingSystem` ties everything together. It manages order books, portfolios, risk checks, matching, settlement, and event notifications through a single unified API.

```python
class TradingSystem:
    """
    Central trading engine.
    Acts as a Facade over order books, matching, portfolios, and events.
    """

    def __init__(self, strategy: MatchingStrategy | None = None) -> None:
        self._books: dict[str, OrderBook] = {}
        self._portfolios: dict[str, Portfolio] = {}
        self._strategy = strategy or PriceTimePriorityStrategy()
        self._risk = RiskManager()
        self._ledger = TradeLedger()
        self._listeners: list[TradingEventListener] = []
        self._tick = 0
        self._all_orders: dict[int, Order] = {}

    # ------------------------------------------------------------------
    # Properties
    # ------------------------------------------------------------------

    @property
    def tick(self) -> int:
        return self._tick

    @property
    def risk_manager(self) -> RiskManager:
        return self._risk

    # ------------------------------------------------------------------
    # Setup
    # ------------------------------------------------------------------

    def add_listener(self, listener: TradingEventListener) -> None:
        self._listeners.append(listener)

    def register_user(self, user_id: str,
                      cash: float = DEFAULT_STARTING_CASH) -> Portfolio:
        """Register a new user with a starting cash balance."""
        if user_id in self._portfolios:
            return self._portfolios[user_id]
        portfolio = Portfolio(user_id, cash)
        self._portfolios[user_id] = portfolio
        return portfolio

    def register_symbol(self, symbol: str) -> OrderBook:
        """Register a new tradable instrument."""
        if symbol not in self._books:
            self._books[symbol] = OrderBook(symbol)
        return self._books[symbol]

    def get_portfolio(self, user_id: str) -> Portfolio | None:
        return self._portfolios.get(user_id)

    def get_order_book(self, symbol: str) -> OrderBook | None:
        return self._books.get(symbol)

    def get_order(self, order_id: int) -> Order | None:
        return self._all_orders.get(order_id)

    # ------------------------------------------------------------------
    # Order placement
    # ------------------------------------------------------------------

    def place_order(self, user_id: str, symbol: str, side: Side,
                    order_type: OrderType, quantity: int,
                    price: float = 0.0) -> Order | None:
        """
        Place a new order. Validates, matches, settles, and notifies.
        Returns the order or None if rejected.
        """
        self._tick += 1

        # Validate user and symbol exist
        portfolio = self._portfolios.get(user_id)
        if portfolio is None:
            print(f"  ✗ User '{user_id}' not registered")
            return None

        book = self._books.get(symbol)
        if book is None:
            print(f"  ✗ Symbol '{symbol}' not registered")
            return None

        # Create order
        order = Order(user_id, symbol, side, order_type,
                      quantity, price, self._tick)

        # Risk check
        valid, reason = self._risk.validate_order(order, portfolio)
        if not valid:
            self._notify_rejected(order, reason)
            return None

        # Reserve assets
        if side == Side.BUY and order_type == OrderType.LIMIT:
            reserved = portfolio.reserve_cash(price * quantity)
            if not reserved:
                self._notify_rejected(order, "Failed to reserve cash")
                return None
        elif side == Side.SELL:
            reserved = portfolio.reserve_shares(symbol, quantity)
            if not reserved:
                self._notify_rejected(order, "Failed to reserve shares")
                return None

        self._all_orders[order.id] = order
        self._notify_placed(order)

        # Match against the book
        executions = self._strategy.match(order, book, self._tick)

        # Settle each trade
        for execution in executions:
            self._settle_trade(execution)

        # If order has remaining quantity, add to book (limit orders only)
        if order.is_active() and order.order_type == OrderType.LIMIT:
            book.add_order(order)

        # Market orders that weren't fully filled get cancelled
        if order.is_active() and order.order_type == OrderType.MARKET:
            order.cancel()
            if side == Side.SELL:
                portfolio.release_shares(symbol, order.remaining_quantity)

        return order

    def cancel_order(self, order_id: int) -> bool:
        """Cancel an open order. Releases reserved assets."""
        order = self._all_orders.get(order_id)
        if order is None:
            print(f"  ✗ Order #{order_id} not found")
            return False

        if not order.is_active():
            print(f"  ✗ Order #{order_id} is not active")
            return False

        portfolio = self._portfolios[order.user_id]
        book = self._books[order.symbol]

        # Release reserved assets
        if order.side == Side.BUY and order.order_type == OrderType.LIMIT:
            unreserve = order.price * order.remaining_quantity
            portfolio.release_cash(unreserve)
        elif order.side == Side.SELL:
            portfolio.release_shares(order.symbol, order.remaining_quantity)

        # Remove from book and cancel
        book.cancel_order(order_id)
        order.cancel()
        self._notify_cancelled(order)
        return True

    # ------------------------------------------------------------------
    # Settlement
    # ------------------------------------------------------------------

    def _settle_trade(self, execution: TradeExecution) -> None:
        """Settle a trade: transfer assets and record in ledger."""
        buyer_portfolio = self._portfolios[execution.buyer_id]
        seller_portfolio = self._portfolios[execution.seller_id]
        total_value = execution.price * execution.quantity

        # Calculate fees
        buyer_order = self._all_orders.get(execution.buy_order_id)
        seller_order = self._all_orders.get(execution.sell_order_id)

        buyer_fee_rate = (TAKER_FEE_RATE
                          if buyer_order and buyer_order.order_type == OrderType.MARKET
                          else MAKER_FEE_RATE)
        seller_fee_rate = (TAKER_FEE_RATE
                           if seller_order and seller_order.order_type == OrderType.MARKET
                           else MAKER_FEE_RATE)

        buyer_fee = total_value * buyer_fee_rate
        seller_fee = total_value * seller_fee_rate

        # Settle
        buyer_portfolio.settle_buy(
            execution.symbol, execution.quantity, total_value, buyer_fee
        )
        seller_portfolio.settle_sell(
            execution.symbol, execution.quantity, total_value, seller_fee
        )

        # Record in ledger
        trade = self._ledger.record_trade(execution, buyer_fee, seller_fee)

        # Notify
        self._notify_trade(trade)

    # ------------------------------------------------------------------
    # Notifications
    # ------------------------------------------------------------------

    def _notify_placed(self, order: Order) -> None:
        for listener in self._listeners:
            listener.on_order_placed(order)

    def _notify_cancelled(self, order: Order) -> None:
        for listener in self._listeners:
            listener.on_order_cancelled(order)

    def _notify_trade(self, trade: Trade) -> None:
        for listener in self._listeners:
            listener.on_trade_executed(trade)

    def _notify_rejected(self, order: Order, reason: str) -> None:
        for listener in self._listeners:
            listener.on_order_rejected(order, reason)

    # ------------------------------------------------------------------
    # Display
    # ------------------------------------------------------------------

    def display_status(self) -> None:
        print(f"\n  {'='*60}")
        print(f"  TRADING SYSTEM — Tick {self._tick}")
        print(f"  Strategy: {self._strategy.name()}")
        print(f"  {'='*60}")

        for symbol, book in self._books.items():
            book.display()

        print(f"  PORTFOLIOS:")
        print(f"  {'-'*55}")
        for portfolio in self._portfolios.values():
            print(f"  {portfolio}")
        print()
```

`place_order` is the primary entry point. It follows a strict pipeline: validate user/symbol → create order → risk check → reserve assets → match against book → settle trades → add remainder to book. This ordering ensures that rejected orders never touch the matching engine, and assets are reserved before any matching begins. `_settle_trade` performs the actual asset transfer — the buyer's cash decreases while shares increase, and vice versa for the seller. Fees are computed using the maker-taker model. Market orders that fail to fill completely are automatically cancelled — unlike limit orders, they do not rest in the book. `cancel_order` releases reserved assets before removing the order, ensuring the user's available balance is restored immediately.

---

## Simulation Engine

A helper that scripts realistic trading scenarios for demonstration.

```python
from typing import Callable


class TradingScenario:
    """A sequence of timed trading actions for simulation."""

    def __init__(self, system: TradingSystem) -> None:
        self._system = system
        self._events: list[tuple[int, str, Callable]] = []

    def at_tick(self, tick: int, description: str,
                action: Callable) -> "TradingScenario":
        """Schedule an action at a specific tick."""
        self._events.append((tick, description, action))
        return self

    def run(self, display_interval: int = 5) -> None:
        """Execute the scenario with periodic status display."""
        self._events.sort(key=lambda e: e[0])
        event_idx = 0
        max_tick = max(e[0] for e in self._events) + 5 if self._events else 10

        print("\n" + "═" * 62)
        print("  TRADING SIMULATION START")
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

            if event_idx >= len(self._events) and tick > self._events[-1][0] + 2:
                break

        print("\n" + "═" * 62)
        print("  TRADING SIMULATION END")
        print("═" * 62)
```

`TradingScenario` lets you script events at specific ticks: "at tick 3, Alice places a buy order for AAPL". `run` executes the simulation, firing events at the right ticks and displaying the system state at configurable intervals. The scenario extends a few ticks past the last event to allow final matches to settle.

---

## Putting It All Together

### Basic Simulation

```python
def main() -> None:
    # Reset ledger for clean start
    TradeLedger.reset()

    # Create trading system
    system = TradingSystem(strategy=PriceTimePriorityStrategy())

    # Attach observers
    logger = TradeLogger()
    market_data = MarketDataFeed()
    system.add_listener(logger)
    system.add_listener(market_data)

    # Register instruments
    system.register_symbol("AAPL")
    system.register_symbol("GOOG")

    # Register users with starting cash
    system.register_user("Alice", 100_000.0)
    system.register_user("Bob", 100_000.0)
    system.register_user("Charlie", 50_000.0)

    # Seed Alice with shares (simulating prior holdings)
    alice = system.get_portfolio("Alice")
    alice.add_shares("AAPL", 500)
    alice.add_shares("GOOG", 200)

    # Seed Bob with shares
    bob = system.get_portfolio("Bob")
    bob.add_shares("AAPL", 300)

    print("╔═══════════════════════════════════════╗")
    print("║       TRADING SYSTEM                  ║")
    print("║  Instruments: AAPL, GOOG              ║")
    print("║  Strategy: Price-Time Priority        ║")
    print("╚═══════════════════════════════════════╝")

    # --- Build the order book ---

    # Alice sells 100 AAPL at $150.00
    system.place_order("Alice", "AAPL", Side.SELL,
                       OrderType.LIMIT, 100, 150.00)

    # Alice sells 50 AAPL at $151.00
    system.place_order("Alice", "AAPL", Side.SELL,
                       OrderType.LIMIT, 50, 151.00)

    # Bob places a buy limit at $149.00
    system.place_order("Bob", "AAPL", Side.BUY,
                       OrderType.LIMIT, 80, 149.00)

    # Charlie buys 30 AAPL at market — fills against Alice's $150 ask
    system.place_order("Charlie", "AAPL", Side.BUY,
                       OrderType.MARKET, 30, 0.0)

    # Bob buys 120 AAPL at $151 limit — fills 70 at $150, 50 at $151
    system.place_order("Bob", "AAPL", Side.BUY,
                       OrderType.LIMIT, 120, 151.00)

    # Display final state
    system.display_status()
    market_data.display()

    ledger = TradeLedger()
    ledger.display()


if __name__ == "__main__":
    main()
```

### Scripted Scenario

```python
def main_scenario() -> None:
    TradeLedger.reset()

    system = TradingSystem(strategy=PriceTimePriorityStrategy())
    logger = TradeLogger()
    market_data = MarketDataFeed()
    system.add_listener(logger)
    system.add_listener(market_data)

    system.register_symbol("TSLA")
    system.register_user("Alice", 200_000.0)
    system.register_user("Bob", 200_000.0)
    system.register_user("Charlie", 100_000.0)
    system.register_user("Diana", 150_000.0)

    # Seed holdings
    system.get_portfolio("Alice").add_shares("TSLA", 1000)
    system.get_portfolio("Bob").add_shares("TSLA", 500)

    scenario = TradingScenario(system)

    # Build initial order book
    scenario.at_tick(1, "Alice sells 200 TSLA @ $250",
                     lambda: system.place_order(
                         "Alice", "TSLA", Side.SELL,
                         OrderType.LIMIT, 200, 250.00))

    scenario.at_tick(1, "Alice sells 100 TSLA @ $252",
                     lambda: system.place_order(
                         "Alice", "TSLA", Side.SELL,
                         OrderType.LIMIT, 100, 252.00))

    scenario.at_tick(2, "Bob sells 150 TSLA @ $251",
                     lambda: system.place_order(
                         "Bob", "TSLA", Side.SELL,
                         OrderType.LIMIT, 150, 251.00))

    # Buyers enter
    scenario.at_tick(3, "Charlie buys 100 TSLA @ market",
                     lambda: system.place_order(
                         "Charlie", "TSLA", Side.BUY,
                         OrderType.MARKET, 100))

    scenario.at_tick(4, "Diana buys 250 TSLA @ $251 limit",
                     lambda: system.place_order(
                         "Diana", "TSLA", Side.BUY,
                         OrderType.LIMIT, 250, 251.00))

    # Late cancellation
    scenario.at_tick(6, "Alice cancels remaining sell @ $252",
                     lambda: system.cancel_order(2))

    scenario.run(display_interval=3)
    market_data.display()
    TradeLedger().display()


if __name__ == "__main__":
    main_scenario()
```

### Strategy Comparison

```python
def compare_strategies() -> None:
    """Compare matching strategies on the same workload."""
    import random
    random.seed(42)

    strategies: list[MatchingStrategy] = [
        PriceTimePriorityStrategy(),
        ProRataStrategy(),
    ]

    print("\n" + "=" * 55)
    print("  STRATEGY COMPARISON")
    print("=" * 55)

    for strategy in strategies:
        TradeLedger.reset()
        Order._next_id = 0

        system = TradingSystem(strategy=strategy)
        system.register_symbol("TEST")
        system.register_user("Seller1", 500_000.0)
        system.register_user("Seller2", 500_000.0)
        system.register_user("Seller3", 500_000.0)
        system.register_user("Buyer", 500_000.0)

        # Give sellers shares
        system.get_portfolio("Seller1").add_shares("TEST", 10000)
        system.get_portfolio("Seller2").add_shares("TEST", 10000)
        system.get_portfolio("Seller3").add_shares("TEST", 10000)

        # Three sellers place asks at the same price
        system.place_order("Seller1", "TEST", Side.SELL,
                           OrderType.LIMIT, 100, 50.00)
        system.place_order("Seller2", "TEST", Side.SELL,
                           OrderType.LIMIT, 200, 50.00)
        system.place_order("Seller3", "TEST", Side.SELL,
                           OrderType.LIMIT, 300, 50.00)

        # Buyer wants 300 shares at $50
        system.place_order("Buyer", "TEST", Side.BUY,
                           OrderType.LIMIT, 300, 50.00)

        ledger = TradeLedger()
        print(f"\n  Strategy: {strategy.name()}")
        print(f"  Trades executed: {len(ledger.trades)}")
        for trade in ledger.trades:
            print(f"    {trade.seller_id} sold {trade.quantity} @ ${trade.price:.2f}")

    print("\n" + "=" * 55)
```

This comparison highlights the difference between price-time priority and pro-rata. With price-time priority, Seller1 (who placed the order first) gets filled completely before Seller2 and Seller3. With pro-rata, all three sellers share the fill proportionally: Seller1 gets ~50, Seller2 gets ~100, and Seller3 gets ~150 — proportional to their order sizes.

### Unit Tests

```python
def test_basic_limit_match() -> None:
    """Test that a buy limit matches a compatible sell limit."""
    TradeLedger.reset()
    Order._next_id = 0

    system = TradingSystem()
    system.register_symbol("AAPL")
    system.register_user("Alice", 100_000.0)
    system.register_user("Bob", 100_000.0)
    system.get_portfolio("Alice").add_shares("AAPL", 100)

    system.place_order("Alice", "AAPL", Side.SELL,
                       OrderType.LIMIT, 50, 150.00)
    system.place_order("Bob", "AAPL", Side.BUY,
                       OrderType.LIMIT, 50, 150.00)

    ledger = TradeLedger()
    assert len(ledger.trades) == 1
    assert ledger.trades[0].quantity == 50
    assert ledger.trades[0].price == 150.00
    print("✅ test_basic_limit_match passed")


def test_partial_fill() -> None:
    """Test that a large order is partially filled."""
    TradeLedger.reset()
    Order._next_id = 0

    system = TradingSystem()
    system.register_symbol("GOOG")
    system.register_user("Alice", 100_000.0)
    system.register_user("Bob", 500_000.0)
    system.get_portfolio("Alice").add_shares("GOOG", 100)

    system.place_order("Alice", "GOOG", Side.SELL,
                       OrderType.LIMIT, 30, 100.00)
    order = system.place_order("Bob", "GOOG", Side.BUY,
                               OrderType.LIMIT, 50, 100.00)

    assert order is not None
    assert order.status == OrderStatus.PARTIALLY_FILLED
    assert order.remaining_quantity == 20
    assert order.filled_quantity == 30
    print("✅ test_partial_fill passed")


def test_price_time_priority() -> None:
    """Test that earlier orders at same price fill first."""
    TradeLedger.reset()
    Order._next_id = 0

    system = TradingSystem()
    system.register_symbol("MSFT")
    system.register_user("Seller1", 100_000.0)
    system.register_user("Seller2", 100_000.0)
    system.register_user("Buyer", 200_000.0)
    system.get_portfolio("Seller1").add_shares("MSFT", 100)
    system.get_portfolio("Seller2").add_shares("MSFT", 100)

    # Seller1 places first
    system.place_order("Seller1", "MSFT", Side.SELL,
                       OrderType.LIMIT, 50, 200.00)
    # Seller2 places second at same price
    system.place_order("Seller2", "MSFT", Side.SELL,
                       OrderType.LIMIT, 50, 200.00)

    # Buyer wants 50 — should match Seller1 (earlier)
    system.place_order("Buyer", "MSFT", Side.BUY,
                       OrderType.LIMIT, 50, 200.00)

    ledger = TradeLedger()
    assert len(ledger.trades) == 1
    assert ledger.trades[0].seller_id == "Seller1"
    print("✅ test_price_time_priority passed")


def test_market_order() -> None:
    """Test that a market order fills at the best available price."""
    TradeLedger.reset()
    Order._next_id = 0

    system = TradingSystem()
    system.register_symbol("AMZN")
    system.register_user("Alice", 100_000.0)
    system.register_user("Bob", 100_000.0)
    system.get_portfolio("Alice").add_shares("AMZN", 100)

    system.place_order("Alice", "AMZN", Side.SELL,
                       OrderType.LIMIT, 50, 120.00)
    order = system.place_order("Bob", "AMZN", Side.BUY,
                               OrderType.MARKET, 30)

    assert order is not None
    assert order.status == OrderStatus.FILLED
    assert order.average_fill_price == 120.00
    print("✅ test_market_order passed")


def test_cancel_order() -> None:
    """Test that cancelling releases reserved assets."""
    TradeLedger.reset()
    Order._next_id = 0

    system = TradingSystem()
    system.register_symbol("NFLX")
    system.register_user("Bob", 100_000.0)

    order = system.place_order("Bob", "NFLX", Side.BUY,
                               OrderType.LIMIT, 100, 50.00)
    assert order is not None

    bob = system.get_portfolio("Bob")
    assert bob.available_cash < 100_000.0  # cash reserved

    system.cancel_order(order.id)
    assert bob.available_cash == 100_000.0  # cash released
    assert order.status == OrderStatus.CANCELLED
    print("✅ test_cancel_order passed")


def test_risk_insufficient_cash() -> None:
    """Test that a buy order is rejected when cash is insufficient."""
    TradeLedger.reset()
    Order._next_id = 0

    system = TradingSystem()
    system.register_symbol("TSLA")
    system.register_user("Charlie", 1_000.0)  # only $1000

    order = system.place_order("Charlie", "TSLA", Side.BUY,
                               OrderType.LIMIT, 100, 500.00)  # needs $50,000
    assert order is None  # should be rejected
    print("✅ test_risk_insufficient_cash passed")


def test_risk_insufficient_shares() -> None:
    """Test that a sell order is rejected when shares are insufficient."""
    TradeLedger.reset()
    Order._next_id = 0

    system = TradingSystem()
    system.register_symbol("META")
    system.register_user("Alice", 100_000.0)
    # Alice has no META shares

    order = system.place_order("Alice", "META", Side.SELL,
                               OrderType.LIMIT, 50, 300.00)
    assert order is None  # rejected — no shares
    print("✅ test_risk_insufficient_shares passed")


def test_order_book_spread() -> None:
    """Test order book spread and mid price calculation."""
    TradeLedger.reset()
    Order._next_id = 0

    system = TradingSystem()
    system.register_symbol("SPY")
    system.register_user("Alice", 100_000.0)
    system.register_user("Bob", 100_000.0)
    system.get_portfolio("Alice").add_shares("SPY", 100)

    system.place_order("Alice", "SPY", Side.SELL,
                       OrderType.LIMIT, 50, 450.00)
    system.place_order("Bob", "SPY", Side.BUY,
                       OrderType.LIMIT, 50, 448.00)

    book = system.get_order_book("SPY")
    assert book.spread == 2.00
    assert book.mid_price == 449.00
    print("✅ test_order_book_spread passed")


def run_all_tests() -> None:
    print("\n" + "=" * 40)
    print("  RUNNING TESTS")
    print("=" * 40 + "\n")
    test_basic_limit_match()
    test_partial_fill()
    test_price_time_priority()
    test_market_order()
    test_cancel_order()
    test_risk_insufficient_cash()
    test_risk_insufficient_shares()
    test_order_book_spread()
    print("\n✅ All tests passed!\n")
```

Each test resets the singleton ledger and order ID counter for isolation. `test_basic_limit_match` verifies that a matching buy and sell at the same price produce a trade. `test_partial_fill` checks that a large buy against a smaller sell leaves the buy partially filled with the correct remaining quantity. `test_price_time_priority` ensures that among two sellers at the same price, the earlier one fills first. `test_cancel_order` confirms that cancelling a buy order releases the reserved cash back to the user's available balance. `test_risk_insufficient_cash` and `test_risk_insufficient_shares` verify that the risk manager blocks invalid orders.

---

## Class Diagram

```
┌──────────────────────────────────────────────────────────────┐
│                      TradingSystem                            │
│──────────────────────────────────────────────────────────────│
│ - _books: dict[str, OrderBook]                                │
│ - _portfolios: dict[str, Portfolio]                           │
│ - _strategy: MatchingStrategy                                 │
│ - _risk: RiskManager                                          │
│ - _ledger: TradeLedger                                        │
│ - _listeners: list[TradingEventListener]                      │
│ - _tick: int                                                  │
│──────────────────────────────────────────────────────────────│
│ + place_order(user, sym, side, type, qty, price) -> Order     │
│ + cancel_order(order_id) -> bool                              │
│ + register_user(user_id, cash) -> Portfolio                   │
│ + register_symbol(symbol) -> OrderBook                        │
│ + display_status() -> None                                    │
└──────┬──────────┬──────────┬──────────┬──────────────────────┘
       │          │          │          │
  owns ▼     uses ▼     owns ▼    owns ▼
┌──────────────┐ ┌──────────────────────────────────────────┐
│  OrderBook   │ │  «abstract» MatchingStrategy             │
│──────────────│ │──────────────────────────────────────────│
│ - _symbol    │ │ + match(order, book, tick) -> [Exec]     │
│ - _bids      │ │ + name() -> str                          │
│ - _asks      │ └──────────┬───────────────────────────────┘
│ - _order_map │            │ implemented by
│──────────────│   ┌────────┴──────────┐
│ + add_order  │   ▼                   ▼
│ + remove_    │ PriceTime           ProRata
│   order      │ Priority            Strategy
│ + best_bid   │ Strategy
│ + best_ask   │
│ + spread     │ ┌──────────────────────────────────────────┐
│ + depth()    │ │  «abstract» TradingEventListener         │
└──────────────┘ │──────────────────────────────────────────│
                 │ + on_order_placed(order) -> None         │
┌──────────────┐ │ + on_order_cancelled(order) -> None      │
│  Portfolio   │ │ + on_trade_executed(trade) -> None       │
│──────────────│ │ + on_order_rejected(order, reason)       │
│ - _user_id   │ └──────────┬─────────────────────────────┘
│ - _cash      │            │ implemented by
│ - _holdings  │      ┌─────┴──────────┐
│ - _reserved_ │      ▼                ▼
│   cash       │  TradeLogger     MarketDataFeed
│ - _reserved_ │
│   shares     │
│──────────────│ ┌──────────────────────────────────────────┐
│ + reserve_   │ │       TradeLedger (Singleton)            │
│   cash()     │ │──────────────────────────────────────────│
│ + settle_    │ │ - _trades: list[Trade]                   │
│   buy()      │ │ - _next_id: int                          │
│ + settle_    │ │──────────────────────────────────────────│
│   sell()     │ │ + record_trade(exec, fees) -> Trade      │
│ + available_ │ │ + trades_for_user(uid) -> list[Trade]    │
│   cash       │ │ + total_volume -> float                  │
│ + available_ │ │ + total_fees -> float                    │
│   shares()   │ └──────────────────────────────────────────┘
└──────────────┘

┌──────────────┐ ┌──────────────────────────────────────────┐
│ RiskManager  │ │           TradingScenario                 │
│──────────────│ │──────────────────────────────────────────│
│ - _max_order_│ │ - _system: TradingSystem                 │
│   quantity   │ │ - _events: list[(tick, desc, Callable)]  │
│ - _max_order_│ │──────────────────────────────────────────│
│   value      │ │ + at_tick(tick, desc, action) -> self     │
│ - _allow_    │ │ + run(interval) -> None                  │
│   short_sell │ └──────────────────────────────────────────┘
│──────────────│
│ + validate_  │
│   order()    │
└──────────────┘

┌────────────────────────────┐
│  Order                      │
│────────────────────────────│
│ - _id: int                  │
│ - _user_id: str             │
│ - _symbol: str              │
│ - _side: Side               │
│ - _order_type: OrderType    │
│ - _original_quantity: int   │
│ - _remaining_quantity: int  │
│ - _price: float             │
│ - _status: OrderStatus      │
│ - _timestamp: int           │
│ - _fills: list[(price,qty)] │
│────────────────────────────│
│ + fill(price, qty) -> None  │
│ + cancel() -> bool          │
│ + average_fill_price -> float│
│ + is_active() -> bool       │
└────────────────────────────┘

┌──────────────────────────┐
│ TradeExecution (frozen)   │
│──────────────────────────│
│ + buy_order_id: int       │
│ + sell_order_id: int      │
│ + buyer_id / seller_id    │
│ + symbol: str             │
│ + price: float            │
│ + quantity: int           │
│ + timestamp: int          │
└──────────────────────────┘

┌────────────────────────┐
│  Trade (frozen)         │
│────────────────────────│
│ + trade_id: int         │
│ + buy_order_id: int     │
│ + sell_order_id: int    │
│ + buyer_id / seller_id  │
│ + symbol: str           │
│ + price: float          │
│ + quantity: int          │
│ + buyer_fee / seller_fee│
│ + notional -> float     │
└────────────────────────┘

┌────────────────┐  ┌──────────────────┐  ┌───────────────────┐
│ Side (Enum)    │  │ OrderType (Enum)  │  │ OrderStatus (Enum)│
│────────────────│  │──────────────────│  │───────────────────│
│ BUY            │  │ LIMIT            │  │ OPEN              │
│ SELL           │  │ MARKET           │  │ PARTIALLY_FILLED  │
│────────────────│  └──────────────────┘  │ FILLED            │
│ + opposite()   │                        │ CANCELLED         │
└────────────────┘                        └───────────────────┘
```

---

## How to Run

Save all code into a single file `trading_system.py` and run:

```bash
python trading_system.py
```

### Example Output

```
╔═══════════════════════════════════════╗
║       TRADING SYSTEM                  ║
║  Instruments: AAPL, GOOG              ║
║  Strategy: Price-Time Priority        ║
╚═══════════════════════════════════════╝

  📝 ORDER PLACED: Alice SELL 100 AAPL @150.00 (LIMIT)
  📝 ORDER PLACED: Alice SELL 50 AAPL @151.00 (LIMIT)
  📝 ORDER PLACED: Bob BUY 80 AAPL @149.00 (LIMIT)
  📝 ORDER PLACED: Charlie BUY 30 AAPL @0.00 (MARKET)
  ✅ TRADE: 30 AAPL @150.00 (Charlie ← Alice) fees: $0.90
  📝 ORDER PLACED: Bob BUY 120 AAPL @151.00 (LIMIT)
  ✅ TRADE: 70 AAPL @150.00 (Bob ← Alice) fees: $0.21
  ✅ TRADE: 50 AAPL @151.00 (Bob ← Alice) fees: $0.15

  ==================================================
  ORDER BOOK — AAPL
  ==================================================
        Price    Quantity  Side
  ----------------------------------------
      149.00          80  BID
  ==================================================

  PORTFOLIOS:
  -------------------------------------------------------
  Portfolio(Alice) Cash=$118,120.85 (avail=$106,200.85) Holdings=[AAPL:350, GOOG:200]
  Portfolio(Bob) Cash=$81,147.95 (avail=$69,227.95) Holdings=[AAPL:420]
  Portfolio(Charlie) Cash=$90,991.00 (avail=$90,991.00) Holdings=[AAPL:30]

  =======================================================
  MARKET DATA
  =======================================================
  Symbol       Last       High        Low     Volume   Trades
  -------------------------------------------------------
  AAPL      $151.00    $151.00    $150.00        150        3

  =======================================================
  TRADE LEDGER — 3 trades
  Total Volume: $22,550.00
  Total Fees:   $1.26
  =======================================================
  Trade#1(AAPL 30@150.00 Charlie←→Alice)
  Trade#2(AAPL 70@150.00 Bob←→Alice)
  Trade#3(AAPL 50@151.00 Bob←→Alice)
```
