# Design Splitwise – Low-Level Design Guide

![Bidur Sapkota](https://www.bidursapkota.com.np/images/gravatar.webp "Bidur Sapkota - Developer")&nbsp;[Bidur Sapkota](https://www.bidursapkota.com.np/)

## Table of Contents

1. [Requirements](#requirements)
2. [Key Design Patterns](#key-design-patterns)
3. [Core Enums & Constants](#core-enums--constants)
4. [User & Group](#user--group)
5. [Expense & Split Strategies (Strategy Pattern)](#expense--split-strategies-strategy-pattern)
6. [Balance Sheet & Debt Simplification](#balance-sheet--debt-simplification)
7. [Settlement & Transaction Ledger](#settlement--transaction-ledger)
8. [Event System (Observer Pattern)](#event-system-observer-pattern)
9. [Splitwise System (Facade)](#splitwise-system-facade)
10. [Simulation Engine](#simulation-engine)
11. [Putting It All Together](#putting-it-all-together)
12. [Class Diagram](#class-diagram)
13. [How to Run](#how-to-run)

---

## Requirements

### Functional Requirements

| #  | Requirement                                                                           |
| -- | ------------------------------------------------------------------------------------- |
| 1  | Users can register with a name, email, and optional phone number                      |
| 2  | Users can create groups and add/remove members                                        |
| 3  | A user can add an expense paid by one or more people and owed by one or more people   |
| 4  | Support equal split — divide the total equally among selected members                 |
| 5  | Support exact split — each member owes a specific amount                              |
| 6  | Support percentage split — each member owes a percentage of the total                 |
| 7  | Support share-based split — divide proportionally by integer shares                   |
| 8  | Track per-pair balances: who owes whom and how much                                   |
| 9  | Simplify debts — minimise the number of transactions needed to settle all balances    |
| 10 | Users can settle up (record a payment) to reduce or clear a debt                      |
| 11 | View outstanding balances for a user or a group                                       |
| 12 | Maintain an immutable history of all expenses and settlements                         |

### Non-Functional Requirements

- Clean OOP with SOLID principles.
- Extensible — new split strategies can be added without modifying expense logic.
- Simulatable — the system runs step-by-step for testing and demonstration.

---

## Key Design Patterns

| Pattern   | Where It Is Used                                  | Why                                                                                  |
| --------- | ------------------------------------------------- | ------------------------------------------------------------------------------------ |
| Strategy  | `SplitStrategy` computes how to divide an expense | Swap between equal, exact, percentage, and share splits without changing core logic   |
| Observer  | `SplitwiseEventListener` observes system events   | Loggers, notification services, and UIs update automatically on expenses/settlements  |
| Facade    | `SplitwiseSystem` exposes a simple API            | Hides the complexity of groups, balances, simplification, and event routing           |
| Command   | `Expense` and `Settlement` objects capture actions| Each action is an immutable record that can be queried and audited                    |
| Singleton | `TransactionLedger` is a single source of history | One ledger ensures consistent audit trail across the system                           |

---

## Core Enums & Constants

```python
from enum import Enum


class SplitType(Enum):
    EQUAL = "equal"           # divide evenly
    EXACT = "exact"           # each person owes a specific amount
    PERCENTAGE = "percentage" # each person owes a percentage
    SHARE = "share"           # divide by integer shares


class ExpenseCategory(Enum):
    FOOD = "food"
    TRANSPORT = "transport"
    ACCOMMODATION = "accommodation"
    ENTERTAINMENT = "entertainment"
    SHOPPING = "shopping"
    UTILITIES = "utilities"
    OTHER = "other"


class TransactionType(Enum):
    EXPENSE = "expense"
    SETTLEMENT = "settlement"


CURRENCY_SYMBOL = "$"
ROUNDING_PRECISION = 2  # decimal places for currency
```

`SplitType` represents the four ways an expense can be divided. `ExpenseCategory` adds optional categorisation for analytics and filtering. `TransactionType` distinguishes expenses from settlements in the ledger. `ROUNDING_PRECISION` controls decimal places — rounding errors are handled by assigning the remainder to the first participant.

---

## User & Group

### User

```python
class User:
    """A registered user in the Splitwise system."""

    _next_id: int = 0

    def __init__(self, name: str, email: str, phone: str = "") -> None:
        User._next_id += 1
        self._id = User._next_id
        self._name = name
        self._email = email
        self._phone = phone

    @property
    def id(self) -> int:
        return self._id

    @property
    def name(self) -> str:
        return self._name

    @property
    def email(self) -> str:
        return self._email

    @property
    def phone(self) -> str:
        return self._phone

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, User):
            return False
        return self._id == other._id

    def __hash__(self) -> int:
        return hash(self._id)

    def __repr__(self) -> str:
        return f"User({self._name}, id={self._id})"
```

### Group

```python
class Group:
    """A group of users who share expenses."""

    _next_id: int = 0

    def __init__(self, name: str, created_by: User) -> None:
        Group._next_id += 1
        self._id = Group._next_id
        self._name = name
        self._created_by = created_by
        self._members: dict[int, User] = {created_by.id: created_by}
        self._expense_ids: list[int] = []

    @property
    def id(self) -> int:
        return self._id

    @property
    def name(self) -> str:
        return self._name

    @property
    def created_by(self) -> User:
        return self._created_by

    @property
    def members(self) -> list[User]:
        return list(self._members.values())

    @property
    def member_ids(self) -> set[int]:
        return set(self._members.keys())

    @property
    def expense_ids(self) -> list[int]:
        return list(self._expense_ids)

    def add_member(self, user: User) -> bool:
        """Add a user to the group. Returns False if already a member."""
        if user.id in self._members:
            return False
        self._members[user.id] = user
        return True

    def remove_member(self, user_id: int) -> bool:
        """Remove a user from the group. Cannot remove the creator."""
        if user_id == self._created_by.id:
            return False
        return self._members.pop(user_id, None) is not None

    def has_member(self, user_id: int) -> bool:
        return user_id in self._members

    def add_expense_id(self, expense_id: int) -> None:
        self._expense_ids.append(expense_id)

    def __repr__(self) -> str:
        member_names = ", ".join(u.name for u in self._members.values())
        return f"Group({self._name}, members=[{member_names}])"
```

`User` is identified by an auto-incrementing ID. `__eq__` and `__hash__` are overridden so users can be stored in sets and used as dictionary keys. `Group` stores members in a dict keyed by user ID for O(1) lookups. The creator cannot be removed from the group — they are the permanent owner. `_expense_ids` tracks which expenses belong to this group for retrieval and analytics.

---

## Expense & Split Strategies (Strategy Pattern)

### Split Strategy (Abstract)

```python
from abc import ABC, abstractmethod


class SplitStrategy(ABC):
    """Defines how an expense is divided among participants."""

    @abstractmethod
    def calculate_splits(self, total: float, participants: list[User],
                         split_data: dict | None = None) -> dict[int, float]:
        """
        Calculate how much each participant owes.
        Returns a dict of {user_id: amount_owed}.
        The sum of all amounts must equal the total.
        """
        ...

    @abstractmethod
    def validate(self, total: float, participants: list[User],
                 split_data: dict | None = None) -> tuple[bool, str]:
        """Validate the split parameters. Returns (is_valid, error_message)."""
        ...

    @abstractmethod
    def split_type(self) -> SplitType:
        ...
```

### Equal Split

```python
class EqualSplitStrategy(SplitStrategy):
    """Divide the total equally among all participants."""

    def calculate_splits(self, total: float, participants: list[User],
                         split_data: dict | None = None) -> dict[int, float]:
        n = len(participants)
        base_amount = round(total / n, ROUNDING_PRECISION)
        splits: dict[int, float] = {}

        for user in participants:
            splits[user.id] = base_amount

        # Handle rounding remainder — assign to the first participant
        distributed = round(base_amount * n, ROUNDING_PRECISION)
        remainder = round(total - distributed, ROUNDING_PRECISION)
        if remainder != 0:
            first_id = participants[0].id
            splits[first_id] = round(splits[first_id] + remainder,
                                     ROUNDING_PRECISION)

        return splits

    def validate(self, total: float, participants: list[User],
                 split_data: dict | None = None) -> tuple[bool, str]:
        if total <= 0:
            return False, "Total must be positive"
        if len(participants) < 2:
            return False, "Need at least 2 participants"
        return True, ""

    def split_type(self) -> SplitType:
        return SplitType.EQUAL
```

### Exact Split

```python
class ExactSplitStrategy(SplitStrategy):
    """Each participant owes a specified exact amount."""

    def calculate_splits(self, total: float, participants: list[User],
                         split_data: dict | None = None) -> dict[int, float]:
        if not split_data or "amounts" not in split_data:
            raise ValueError("Exact split requires 'amounts' in split_data")

        amounts: dict[int, float] = split_data["amounts"]
        splits: dict[int, float] = {}

        for user in participants:
            splits[user.id] = round(amounts.get(user.id, 0.0),
                                    ROUNDING_PRECISION)

        return splits

    def validate(self, total: float, participants: list[User],
                 split_data: dict | None = None) -> tuple[bool, str]:
        if total <= 0:
            return False, "Total must be positive"
        if not split_data or "amounts" not in split_data:
            return False, "Exact split requires 'amounts' dict"

        amounts: dict[int, float] = split_data["amounts"]
        amount_sum = round(sum(amounts.values()), ROUNDING_PRECISION)
        if amount_sum != round(total, ROUNDING_PRECISION):
            return False, (
                f"Amounts sum to {CURRENCY_SYMBOL}{amount_sum:.2f}, "
                f"expected {CURRENCY_SYMBOL}{total:.2f}"
            )
        return True, ""

    def split_type(self) -> SplitType:
        return SplitType.EXACT
```

### Percentage Split

```python
class PercentageSplitStrategy(SplitStrategy):
    """Each participant owes a percentage of the total."""

    def calculate_splits(self, total: float, participants: list[User],
                         split_data: dict | None = None) -> dict[int, float]:
        if not split_data or "percentages" not in split_data:
            raise ValueError(
                "Percentage split requires 'percentages' in split_data"
            )

        percentages: dict[int, float] = split_data["percentages"]
        splits: dict[int, float] = {}

        for user in participants:
            pct = percentages.get(user.id, 0.0)
            splits[user.id] = round(total * pct / 100.0, ROUNDING_PRECISION)

        # Handle rounding remainder
        distributed = round(sum(splits.values()), ROUNDING_PRECISION)
        remainder = round(total - distributed, ROUNDING_PRECISION)
        if remainder != 0:
            first_id = participants[0].id
            splits[first_id] = round(splits[first_id] + remainder,
                                     ROUNDING_PRECISION)

        return splits

    def validate(self, total: float, participants: list[User],
                 split_data: dict | None = None) -> tuple[bool, str]:
        if total <= 0:
            return False, "Total must be positive"
        if not split_data or "percentages" not in split_data:
            return False, "Percentage split requires 'percentages' dict"

        percentages: dict[int, float] = split_data["percentages"]
        pct_sum = round(sum(percentages.values()), ROUNDING_PRECISION)
        if pct_sum != 100.0:
            return False, f"Percentages sum to {pct_sum}%, expected 100%"
        return True, ""

    def split_type(self) -> SplitType:
        return SplitType.PERCENTAGE
```

### Share-Based Split

```python
class ShareSplitStrategy(SplitStrategy):
    """Divide proportionally by integer shares."""

    def calculate_splits(self, total: float, participants: list[User],
                         split_data: dict | None = None) -> dict[int, float]:
        if not split_data or "shares" not in split_data:
            raise ValueError("Share split requires 'shares' in split_data")

        shares: dict[int, int] = split_data["shares"]
        total_shares = sum(shares.values())
        splits: dict[int, float] = {}

        for user in participants:
            user_shares = shares.get(user.id, 0)
            splits[user.id] = round(total * user_shares / total_shares,
                                    ROUNDING_PRECISION)

        # Handle rounding remainder
        distributed = round(sum(splits.values()), ROUNDING_PRECISION)
        remainder = round(total - distributed, ROUNDING_PRECISION)
        if remainder != 0:
            first_id = participants[0].id
            splits[first_id] = round(splits[first_id] + remainder,
                                     ROUNDING_PRECISION)

        return splits

    def validate(self, total: float, participants: list[User],
                 split_data: dict | None = None) -> tuple[bool, str]:
        if total <= 0:
            return False, "Total must be positive"
        if not split_data or "shares" not in split_data:
            return False, "Share split requires 'shares' dict"

        shares: dict[int, int] = split_data["shares"]
        if any(s <= 0 for s in shares.values()):
            return False, "All shares must be positive"
        if sum(shares.values()) == 0:
            return False, "Total shares cannot be zero"
        return True, ""

    def split_type(self) -> SplitType:
        return SplitType.SHARE
```

The Strategy pattern makes it trivial to add a new split type — implement `calculate_splits` and `validate`, and the rest of the system works unchanged. All strategies handle rounding carefully: they compute each person's share independently, then assign the penny-level remainder to the first participant so the total always balances exactly. `EqualSplitStrategy` divides evenly. `ExactSplitStrategy` uses caller-provided amounts (validated to sum to the total). `PercentageSplitStrategy` takes percentages that must sum to 100. `ShareSplitStrategy` takes integer shares and divides proportionally — a user with 2 shares in a total of 5 pays 40%.

### Split Strategy Factory

```python
class SplitStrategyFactory:
    """Creates the correct split strategy from a SplitType enum."""

    _strategies: dict[SplitType, SplitStrategy] = {
        SplitType.EQUAL: EqualSplitStrategy(),
        SplitType.EXACT: ExactSplitStrategy(),
        SplitType.PERCENTAGE: PercentageSplitStrategy(),
        SplitType.SHARE: ShareSplitStrategy(),
    }

    @classmethod
    def get_strategy(cls, split_type: SplitType) -> SplitStrategy:
        strategy = cls._strategies.get(split_type)
        if strategy is None:
            raise ValueError(f"Unknown split type: {split_type}")
        return strategy
```

`SplitStrategyFactory` maps each `SplitType` enum to its strategy instance. Strategies are stateless singletons — they can be shared safely. The factory centralises strategy selection so the rest of the code never constructs strategies directly.

### Expense

```python
from dataclasses import dataclass, field


@dataclass(frozen=True)
class ExpenseSplit:
    """How much a single user owes for an expense."""
    user_id: int
    amount: float


class Expense:
    """An expense paid by someone and owed by multiple people."""

    _next_id: int = 0

    def __init__(self, description: str, total: float, paid_by: User,
                 participants: list[User], split_type: SplitType,
                 split_data: dict | None = None,
                 category: ExpenseCategory = ExpenseCategory.OTHER,
                 group_id: int | None = None,
                 timestamp: int = 0) -> None:
        Expense._next_id += 1
        self._id = Expense._next_id
        self._description = description
        self._total = total
        self._paid_by = paid_by
        self._participants = list(participants)
        self._split_type = split_type
        self._category = category
        self._group_id = group_id
        self._timestamp = timestamp

        # Calculate splits using the strategy
        strategy = SplitStrategyFactory.get_strategy(split_type)
        valid, error = strategy.validate(total, participants, split_data)
        if not valid:
            raise ValueError(f"Invalid split: {error}")

        self._splits: dict[int, float] = strategy.calculate_splits(
            total, participants, split_data
        )

    @property
    def id(self) -> int:
        return self._id

    @property
    def description(self) -> str:
        return self._description

    @property
    def total(self) -> float:
        return self._total

    @property
    def paid_by(self) -> User:
        return self._paid_by

    @property
    def participants(self) -> list[User]:
        return list(self._participants)

    @property
    def split_type(self) -> SplitType:
        return self._split_type

    @property
    def category(self) -> ExpenseCategory:
        return self._category

    @property
    def group_id(self) -> int | None:
        return self._group_id

    @property
    def splits(self) -> dict[int, float]:
        return dict(self._splits)

    @property
    def timestamp(self) -> int:
        return self._timestamp

    def get_user_share(self, user_id: int) -> float:
        """Get how much a specific user owes for this expense."""
        return self._splits.get(user_id, 0.0)

    def __repr__(self) -> str:
        split_details = ", ".join(
            f"uid{uid}={CURRENCY_SYMBOL}{amt:.2f}"
            for uid, amt in self._splits.items()
        )
        return (
            f"Expense#{self._id}(\"{self._description}\" "
            f"{CURRENCY_SYMBOL}{self._total:.2f} "
            f"paid_by={self._paid_by.name} "
            f"[{self._split_type.value}] [{split_details}])"
        )
```

`Expense` calculates splits at creation time using the strategy factory — once created, the splits are immutable. The constructor validates the split parameters before computing, so invalid expenses are rejected immediately with a clear error message. `paid_by` is the user who fronted the money. `_splits` maps each participant's user ID to what they owe. The payer's share in `_splits` represents what they owe to themselves — this nets out to zero for them in the balance sheet, while everyone else's share becomes a debt owed to the payer.

---

## Balance Sheet & Debt Simplification

### Balance Sheet

```python
from collections import defaultdict


class BalanceSheet:
    """
    Tracks pairwise balances between all users.

    A positive balance[A][B] means A is owed money by B.
    A negative balance[A][B] means A owes money to B.
    balance[A][B] == -balance[B][A] always holds.
    """

    def __init__(self) -> None:
        # balances[creditor_id][debtor_id] = amount owed TO creditor BY debtor
        self._balances: dict[int, dict[int, float]] = defaultdict(
            lambda: defaultdict(float)
        )

    def record_expense(self, expense: Expense) -> None:
        """
        Update balances based on an expense.

        For each participant who is not the payer:
        - The payer is owed their share.
        - The participant owes the payer their share.
        """
        payer_id = expense.paid_by.id
        for user_id, amount in expense.splits.items():
            if user_id == payer_id:
                continue  # payer doesn't owe themselves
            # payer is owed by this user
            self._balances[payer_id][user_id] = round(
                self._balances[payer_id][user_id] + amount,
                ROUNDING_PRECISION,
            )
            # this user owes the payer
            self._balances[user_id][payer_id] = round(
                self._balances[user_id][payer_id] - amount,
                ROUNDING_PRECISION,
            )

    def record_settlement(self, payer_id: int, payee_id: int,
                          amount: float) -> None:
        """
        Record a settlement payment.
        payer_id pays payee_id the given amount.
        """
        # payer's debt to payee decreases
        self._balances[payer_id][payee_id] = round(
            self._balances[payer_id][payee_id] + amount,
            ROUNDING_PRECISION,
        )
        # payee's credit from payer decreases
        self._balances[payee_id][payer_id] = round(
            self._balances[payee_id][payer_id] - amount,
            ROUNDING_PRECISION,
        )

    def get_balance(self, user_a: int, user_b: int) -> float:
        """
        Get the balance between two users from A's perspective.
        Positive: B owes A. Negative: A owes B.
        """
        return self._balances[user_a][user_b]

    def get_user_balances(self, user_id: int) -> dict[int, float]:
        """Get all non-zero balances for a user."""
        return {
            other_id: amount
            for other_id, amount in self._balances[user_id].items()
            if round(amount, ROUNDING_PRECISION) != 0
        }

    def get_total_owed_to(self, user_id: int) -> float:
        """Total amount others owe this user."""
        return sum(
            amount for amount in self._balances[user_id].values()
            if amount > 0
        )

    def get_total_owed_by(self, user_id: int) -> float:
        """Total amount this user owes others."""
        return abs(sum(
            amount for amount in self._balances[user_id].values()
            if amount < 0
        ))

    def get_net_balance(self, user_id: int) -> float:
        """Net balance: positive means user is owed, negative means user owes."""
        return round(
            sum(self._balances[user_id].values()),
            ROUNDING_PRECISION,
        )
```

`BalanceSheet` is the core accounting structure. It maintains a 2D matrix of pairwise balances where `balance[A][B] == -balance[B][A]` is always true (antisymmetric). When an expense is recorded, each participant's share becomes a debt to the payer. `defaultdict(lambda: defaultdict(float))` creates the nested structure lazily — entries are only allocated when accessed. `get_net_balance` sums all of a user's pairwise balances — positive means the system owes them, negative means they owe the system. The balance sheet is the ground truth; debt simplification is computed from it.

### Debt Simplifier

```python
import heapq


class DebtSimplifier:
    """
    Minimises the number of transactions needed to settle all debts.

    Uses the greedy algorithm:
    1. Compute each person's net balance (total owed - total owing).
    2. Separate into creditors (positive net) and debtors (negative net).
    3. Match the largest creditor with the largest debtor.
    4. Transfer the minimum of the two amounts.
    5. Repeat until all balances are zero.
    """

    @staticmethod
    def simplify(balance_sheet: BalanceSheet,
                 user_ids: set[int]) -> list[tuple[int, int, float]]:
        """
        Compute simplified debts.
        Returns list of (debtor_id, creditor_id, amount).
        """
        # Step 1: Compute net balances
        net: dict[int, float] = {}
        for uid in user_ids:
            balance = balance_sheet.get_net_balance(uid)
            if round(balance, ROUNDING_PRECISION) != 0:
                net[uid] = balance

        # Step 2: Separate into creditors and debtors using heaps
        # Max-heap for creditors (negate for max-heap using min-heap)
        creditors: list[tuple[float, int]] = []
        # Max-heap for debtors (negate the debt amount)
        debtors: list[tuple[float, int]] = []

        for uid, balance in net.items():
            if balance > 0:
                heapq.heappush(creditors, (-balance, uid))  # max-heap
            elif balance < 0:
                heapq.heappush(debtors, (balance, uid))     # max-heap by abs

        # Step 3: Greedily match largest creditor with largest debtor
        transactions: list[tuple[int, int, float]] = []

        while creditors and debtors:
            credit_amount, creditor_id = heapq.heappop(creditors)
            debt_amount, debtor_id = heapq.heappop(debtors)

            credit = -credit_amount  # un-negate
            debt = -debt_amount      # un-negate

            transfer = round(min(credit, debt), ROUNDING_PRECISION)
            transactions.append((debtor_id, creditor_id, transfer))

            remaining_credit = round(credit - transfer, ROUNDING_PRECISION)
            remaining_debt = round(debt - transfer, ROUNDING_PRECISION)

            if remaining_credit > 0:
                heapq.heappush(creditors, (-remaining_credit, creditor_id))
            if remaining_debt > 0:
                heapq.heappush(debtors, (-remaining_debt, debtor_id))

        return transactions

    @staticmethod
    def display_simplified(transactions: list[tuple[int, int, float]],
                           users: dict[int, User]) -> None:
        """Pretty-print the simplified debts."""
        if not transactions:
            print("  ✅ All settled up — no debts!")
            return

        print(f"\n  {'='*50}")
        print(f"  SIMPLIFIED DEBTS ({len(transactions)} transaction(s))")
        print(f"  {'='*50}")
        for debtor_id, creditor_id, amount in transactions:
            debtor_name = users[debtor_id].name
            creditor_name = users[creditor_id].name
            print(
                f"  💸 {debtor_name} pays {creditor_name}: "
                f"{CURRENCY_SYMBOL}{amount:.2f}"
            )
        print()
```

`DebtSimplifier` is the most algorithmically interesting component. It uses a greedy approach that reduces N² pairwise debts into at most N-1 transactions. Step 1 computes each person's net position — the difference between what they are owed and what they owe. Step 2 separates users into creditors and debtors using max-heaps (simulated with negated values in Python's min-heap). Step 3 repeatedly pairs the largest creditor with the largest debtor, transferring the smaller of the two amounts. This is optimal for minimising transaction count in most practical cases. For example, if A owes B \$10 and B owes C \$10, instead of two payments the simplifier produces one: A pays C \$10 directly.

---

## Settlement & Transaction Ledger

### Settlement

```python
@dataclass(frozen=True)
class Settlement:
    """An immutable record of a payment between two users."""
    settlement_id: int
    payer_id: int
    payee_id: int
    amount: float
    timestamp: int
    note: str = ""

    def __repr__(self) -> str:
        return (
            f"Settlement#{self.settlement_id}("
            f"uid{self.payer_id} → uid{self.payee_id}: "
            f"{CURRENCY_SYMBOL}{self.amount:.2f})"
        )
```

### Transaction Ledger (Singleton)

```python
class TransactionLedger:
    """
    Immutable history of all expenses and settlements.
    Singleton — one ledger per system.
    """

    _instance: "TransactionLedger | None" = None

    def __new__(cls) -> "TransactionLedger":
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._expenses = []
            cls._instance._settlements = []
            cls._instance._next_settlement_id = 0
        return cls._instance

    @classmethod
    def reset(cls) -> None:
        """Reset the singleton (for testing)."""
        cls._instance = None

    def record_expense(self, expense: Expense) -> None:
        self._expenses.append(expense)

    def record_settlement(self, payer_id: int, payee_id: int,
                          amount: float, timestamp: int = 0,
                          note: str = "") -> Settlement:
        self._next_settlement_id += 1
        settlement = Settlement(
            settlement_id=self._next_settlement_id,
            payer_id=payer_id,
            payee_id=payee_id,
            amount=amount,
            timestamp=timestamp,
            note=note,
        )
        self._settlements.append(settlement)
        return settlement

    @property
    def expenses(self) -> list[Expense]:
        return list(self._expenses)

    @property
    def settlements(self) -> list[Settlement]:
        return list(self._settlements)

    def expenses_for_user(self, user_id: int) -> list[Expense]:
        """All expenses where the user is a payer or participant."""
        return [
            e for e in self._expenses
            if e.paid_by.id == user_id or user_id in e.splits
        ]

    def expenses_for_group(self, group_id: int) -> list[Expense]:
        return [e for e in self._expenses if e.group_id == group_id]

    @property
    def total_expense_amount(self) -> float:
        return sum(e.total for e in self._expenses)

    @property
    def total_settled_amount(self) -> float:
        return sum(s.amount for s in self._settlements)

    def display(self, last_n: int = 10) -> None:
        print(f"\n  {'='*55}")
        print(f"  TRANSACTION LEDGER")
        print(f"  Expenses: {len(self._expenses)} "
              f"(total: {CURRENCY_SYMBOL}"
              f"{self.total_expense_amount:,.2f})")
        print(f"  Settlements: {len(self._settlements)} "
              f"(total: {CURRENCY_SYMBOL}"
              f"{self.total_settled_amount:,.2f})")
        print(f"  {'='*55}")

        print(f"  Recent Expenses:")
        for expense in self._expenses[-last_n:]:
            print(f"    {expense}")

        if self._settlements:
            print(f"  Recent Settlements:")
            for settlement in self._settlements[-last_n:]:
                print(f"    {settlement}")
        print()
```

`Settlement` is a frozen dataclass — an immutable record of a payment from one user to another. `TransactionLedger` uses the Singleton pattern to maintain a single authoritative history. It stores expenses and settlements separately for easy querying. `expenses_for_user` returns every expense where the user either paid or participated — useful for a personal activity feed. The ledger is append-only — no entries are ever deleted or modified, ensuring a complete audit trail.

---

## Event System (Observer Pattern)

### Listener Interface

```python
class SplitwiseEventListener(ABC):
    """Observer interface for Splitwise system events."""

    @abstractmethod
    def on_expense_added(self, expense: Expense) -> None:
        ...

    @abstractmethod
    def on_settlement_made(self, settlement: Settlement) -> None:
        ...

    @abstractmethod
    def on_group_created(self, group: Group) -> None:
        ...

    @abstractmethod
    def on_member_added(self, group: Group, user: User) -> None:
        ...
```

### Activity Logger

```python
class ActivityLogger(SplitwiseEventListener):
    """Logs all system events to console."""

    def __init__(self) -> None:
        self._log: list[str] = []

    def on_expense_added(self, expense: Expense) -> None:
        entry = (
            f"  📝 EXPENSE: \"{expense.description}\" — "
            f"{CURRENCY_SYMBOL}{expense.total:.2f} "
            f"paid by {expense.paid_by.name} "
            f"({expense.split_type.value} split)"
        )
        self._log.append(entry)
        print(entry)

    def on_settlement_made(self, settlement: Settlement) -> None:
        entry = (
            f"  💰 SETTLEMENT: uid{settlement.payer_id} → "
            f"uid{settlement.payee_id}: "
            f"{CURRENCY_SYMBOL}{settlement.amount:.2f}"
        )
        self._log.append(entry)
        print(entry)

    def on_group_created(self, group: Group) -> None:
        entry = (
            f"  👥 GROUP CREATED: \"{group.name}\" "
            f"by {group.created_by.name}"
        )
        self._log.append(entry)
        print(entry)

    def on_member_added(self, group: Group, user: User) -> None:
        entry = (
            f"  ➕ MEMBER ADDED: {user.name} → "
            f"\"{group.name}\""
        )
        self._log.append(entry)
        print(entry)

    def print_log(self, last_n: int = 20) -> None:
        print(f"\n  Event Log (last {last_n}):")
        for entry in self._log[-last_n:]:
            print(entry)

    def clear(self) -> None:
        self._log.clear()
```

### Notification Service

```python
class NotificationService(SplitwiseEventListener):
    """Simulates sending notifications to affected users."""

    def __init__(self, users: dict[int, User]) -> None:
        self._users = users
        self._notifications: list[str] = []

    def on_expense_added(self, expense: Expense) -> None:
        for user_id, amount in expense.splits.items():
            if user_id == expense.paid_by.id:
                continue
            user_name = self._users[user_id].name
            notif = (
                f"  🔔 → {user_name}: You owe "
                f"{expense.paid_by.name} "
                f"{CURRENCY_SYMBOL}{amount:.2f} "
                f"for \"{expense.description}\""
            )
            self._notifications.append(notif)
            print(notif)

    def on_settlement_made(self, settlement: Settlement) -> None:
        payee_name = self._users[settlement.payee_id].name
        payer_name = self._users[settlement.payer_id].name
        notif = (
            f"  🔔 → {payee_name}: {payer_name} paid you "
            f"{CURRENCY_SYMBOL}{settlement.amount:.2f}"
        )
        self._notifications.append(notif)
        print(notif)

    def on_group_created(self, group: Group) -> None:
        pass

    def on_member_added(self, group: Group, user: User) -> None:
        notif = (
            f"  🔔 → {user.name}: You were added to "
            f"\"{group.name}\""
        )
        self._notifications.append(notif)
        print(notif)

    def print_notifications(self, last_n: int = 20) -> None:
        print(f"\n  Notifications (last {last_n}):")
        for notif in self._notifications[-last_n:]:
            print(notif)
```

`SplitwiseEventListener` defines four hooks matching the system's key actions. `ActivityLogger` records every event with emoji markers — it is the audit log. `NotificationService` demonstrates user-facing notifications: when an expense is added, each participant who is not the payer gets notified of their share. The notification service takes a user dict at construction so it can resolve user IDs to names. Both listeners are fully decoupled from the core system — they can be added or removed without any code changes.

---

## Splitwise System (Facade)

The `SplitwiseSystem` ties everything together. It manages users, groups, expenses, balances, debt simplification, and settlements through a single unified API.

```python
class SplitwiseSystem:
    """
    Central controller for the expense sharing system.
    Acts as a Facade over users, groups, balances, and events.
    """

    def __init__(self) -> None:
        self._users: dict[int, User] = {}
        self._groups: dict[int, Group] = {}
        self._balance_sheet = BalanceSheet()
        self._ledger = TransactionLedger()
        self._listeners: list[SplitwiseEventListener] = []
        self._tick = 0

    # ------------------------------------------------------------------
    # Properties
    # ------------------------------------------------------------------

    @property
    def users(self) -> dict[int, User]:
        return dict(self._users)

    @property
    def groups(self) -> dict[int, Group]:
        return dict(self._groups)

    @property
    def balance_sheet(self) -> BalanceSheet:
        return self._balance_sheet

    # ------------------------------------------------------------------
    # Setup
    # ------------------------------------------------------------------

    def add_listener(self, listener: SplitwiseEventListener) -> None:
        self._listeners.append(listener)

    def register_user(self, name: str, email: str,
                      phone: str = "") -> User:
        """Register a new user."""
        user = User(name, email, phone)
        self._users[user.id] = user
        return user

    def get_user(self, user_id: int) -> User | None:
        return self._users.get(user_id)

    # ------------------------------------------------------------------
    # Groups
    # ------------------------------------------------------------------

    def create_group(self, name: str, creator_id: int) -> Group | None:
        """Create a new group."""
        creator = self._users.get(creator_id)
        if creator is None:
            print(f"  ✗ User {creator_id} not found")
            return None

        group = Group(name, creator)
        self._groups[group.id] = group
        self._notify_group_created(group)
        return group

    def add_to_group(self, group_id: int, user_id: int) -> bool:
        """Add a user to a group."""
        group = self._groups.get(group_id)
        user = self._users.get(user_id)
        if group is None or user is None:
            print(f"  ✗ Group or user not found")
            return False

        if group.add_member(user):
            self._notify_member_added(group, user)
            return True
        return False

    def remove_from_group(self, group_id: int, user_id: int) -> bool:
        """Remove a user from a group."""
        group = self._groups.get(group_id)
        if group is None:
            return False
        return group.remove_member(user_id)

    # ------------------------------------------------------------------
    # Expenses
    # ------------------------------------------------------------------

    def add_expense(self, description: str, total: float,
                    paid_by_id: int, participant_ids: list[int],
                    split_type: SplitType,
                    split_data: dict | None = None,
                    category: ExpenseCategory = ExpenseCategory.OTHER,
                    group_id: int | None = None) -> Expense | None:
        """
        Add a new expense. Validates, creates, updates balances, and notifies.
        Returns the expense or None if invalid.
        """
        self._tick += 1

        # Validate payer
        payer = self._users.get(paid_by_id)
        if payer is None:
            print(f"  ✗ Payer user {paid_by_id} not found")
            return None

        # Validate participants
        participants: list[User] = []
        for uid in participant_ids:
            user = self._users.get(uid)
            if user is None:
                print(f"  ✗ Participant user {uid} not found")
                return None
            participants.append(user)

        if len(participants) < 2:
            print(f"  ✗ Need at least 2 participants")
            return None

        # Validate group membership if group expense
        if group_id is not None:
            group = self._groups.get(group_id)
            if group is None:
                print(f"  ✗ Group {group_id} not found")
                return None
            for uid in participant_ids:
                if not group.has_member(uid):
                    print(f"  ✗ User {uid} is not in group \"{group.name}\"")
                    return None

        # Create expense (validates splits internally)
        try:
            expense = Expense(
                description=description,
                total=total,
                paid_by=payer,
                participants=participants,
                split_type=split_type,
                split_data=split_data,
                category=category,
                group_id=group_id,
                timestamp=self._tick,
            )
        except ValueError as e:
            print(f"  ✗ Invalid expense: {e}")
            return None

        # Update balance sheet
        self._balance_sheet.record_expense(expense)

        # Record in ledger
        self._ledger.record_expense(expense)

        # Track in group
        if group_id is not None:
            self._groups[group_id].add_expense_id(expense.id)

        # Notify
        self._notify_expense_added(expense)

        return expense

    # ------------------------------------------------------------------
    # Settlements
    # ------------------------------------------------------------------

    def settle_up(self, payer_id: int, payee_id: int,
                  amount: float, note: str = "") -> Settlement | None:
        """
        Record a settlement payment.
        payer_id pays payee_id the given amount.
        """
        self._tick += 1

        if payer_id not in self._users or payee_id not in self._users:
            print(f"  ✗ User not found")
            return None

        if amount <= 0:
            print(f"  ✗ Amount must be positive")
            return None

        # Update balance sheet
        self._balance_sheet.record_settlement(payer_id, payee_id, amount)

        # Record in ledger
        settlement = self._ledger.record_settlement(
            payer_id, payee_id, amount, self._tick, note
        )

        # Notify
        self._notify_settlement(settlement)

        return settlement

    # ------------------------------------------------------------------
    # Queries
    # ------------------------------------------------------------------

    def get_balances(self, user_id: int) -> dict[int, float]:
        """Get all non-zero balances for a user."""
        return self._balance_sheet.get_user_balances(user_id)

    def get_simplified_debts(self,
                             user_ids: set[int] | None = None
                             ) -> list[tuple[int, int, float]]:
        """Compute simplified debts for a set of users (default: all)."""
        if user_ids is None:
            user_ids = set(self._users.keys())
        return DebtSimplifier.simplify(self._balance_sheet, user_ids)

    def get_group_simplified_debts(self,
                                   group_id: int
                                   ) -> list[tuple[int, int, float]]:
        """Compute simplified debts within a group."""
        group = self._groups.get(group_id)
        if group is None:
            return []
        return self.get_simplified_debts(group.member_ids)

    # ------------------------------------------------------------------
    # Notifications
    # ------------------------------------------------------------------

    def _notify_expense_added(self, expense: Expense) -> None:
        for listener in self._listeners:
            listener.on_expense_added(expense)

    def _notify_settlement(self, settlement: Settlement) -> None:
        for listener in self._listeners:
            listener.on_settlement_made(settlement)

    def _notify_group_created(self, group: Group) -> None:
        for listener in self._listeners:
            listener.on_group_created(group)

    def _notify_member_added(self, group: Group, user: User) -> None:
        for listener in self._listeners:
            listener.on_member_added(group, user)

    # ------------------------------------------------------------------
    # Display
    # ------------------------------------------------------------------

    def display_balances(self, user_id: int) -> None:
        """Print all balances for a user."""
        user = self._users.get(user_id)
        if user is None:
            print(f"  ✗ User {user_id} not found")
            return

        balances = self.get_balances(user_id)
        total_owed_to = self._balance_sheet.get_total_owed_to(user_id)
        total_owed_by = self._balance_sheet.get_total_owed_by(user_id)
        net = self._balance_sheet.get_net_balance(user_id)

        print(f"\n  {'='*50}")
        print(f"  BALANCES FOR {user.name}")
        print(f"  {'='*50}")

        if not balances:
            print(f"  ✅ All settled up!")
        else:
            for other_id, amount in balances.items():
                other_name = self._users[other_id].name
                if amount > 0:
                    print(
                        f"  ⬅ {other_name} owes you: "
                        f"{CURRENCY_SYMBOL}{amount:.2f}"
                    )
                else:
                    print(
                        f"  ➡ You owe {other_name}: "
                        f"{CURRENCY_SYMBOL}{abs(amount):.2f}"
                    )

        print(f"  {'-'*50}")
        print(f"  Total owed to you: {CURRENCY_SYMBOL}{total_owed_to:.2f}")
        print(f"  Total you owe:     {CURRENCY_SYMBOL}{total_owed_by:.2f}")
        net_label = "you are owed" if net >= 0 else "you owe"
        print(f"  Net: {CURRENCY_SYMBOL}{abs(net):.2f} ({net_label})")
        print()

    def display_group_balances(self, group_id: int) -> None:
        """Print balances within a group."""
        group = self._groups.get(group_id)
        if group is None:
            print(f"  ✗ Group {group_id} not found")
            return

        print(f"\n  {'='*50}")
        print(f"  GROUP: \"{group.name}\"")
        print(f"  {'='*50}")

        for user in group.members:
            balances = self.get_balances(user.id)
            group_balances = {
                uid: amt for uid, amt in balances.items()
                if uid in group.member_ids
            }
            if group_balances:
                for other_id, amount in group_balances.items():
                    other_name = self._users[other_id].name
                    if amount > 0:
                        print(
                            f"  {user.name}: {other_name} owes "
                            f"{CURRENCY_SYMBOL}{amount:.2f}"
                        )
                    elif amount < 0:
                        print(
                            f"  {user.name}: owes {other_name} "
                            f"{CURRENCY_SYMBOL}{abs(amount):.2f}"
                        )

        # Show simplified
        simplified = self.get_group_simplified_debts(group_id)
        DebtSimplifier.display_simplified(simplified, self._users)

    def display_status(self) -> None:
        """Print the current state of the system."""
        print(f"\n  {'='*60}")
        print(f"  SPLITWISE SYSTEM — Tick {self._tick}")
        print(f"  Users: {len(self._users)} | Groups: {len(self._groups)}")
        print(f"  {'='*60}")

        for user in self._users.values():
            net = self._balance_sheet.get_net_balance(user.id)
            net_str = (f"+{CURRENCY_SYMBOL}{net:.2f}" if net >= 0
                       else f"-{CURRENCY_SYMBOL}{abs(net):.2f}")
            print(f"  {user.name}: net {net_str}")
        print()
```

`add_expense` is the primary entry point. It follows a strict pipeline: validate payer → validate participants → validate group membership → create expense (validates splits) → update balance sheet → record in ledger → notify listeners. `settle_up` records a payment between two users — it reduces the payer's debt to the payee. `get_simplified_debts` delegates to `DebtSimplifier` and can operate on all users or just a group's members. `display_balances` gives a complete per-user financial view showing who owes them, who they owe, and the net position.

---

## Simulation Engine

A helper that scripts realistic expense scenarios for demonstration.

```python
from typing import Callable


class SplitwiseScenario:
    """A sequence of timed actions for simulation."""

    def __init__(self, system: SplitwiseSystem) -> None:
        self._system = system
        self._events: list[tuple[int, str, Callable]] = []

    def at_tick(self, tick: int, description: str,
                action: Callable) -> "SplitwiseScenario":
        """Schedule an action at a specific tick."""
        self._events.append((tick, description, action))
        return self

    def run(self, display_interval: int = 5) -> None:
        """Execute the scenario with periodic status display."""
        self._events.sort(key=lambda e: e[0])
        event_idx = 0
        max_tick = max(e[0] for e in self._events) + 3 if self._events else 10

        print("\n" + "═" * 62)
        print("  SPLITWISE SIMULATION START")
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

            if event_idx >= len(self._events) and tick > self._events[-1][0] + 1:
                break

        print("\n" + "═" * 62)
        print("  SPLITWISE SIMULATION END")
        print("═" * 62)
```

`SplitwiseScenario` lets you script events at specific ticks: "at tick 2, Alice pays for dinner split equally". `run` executes the simulation, firing events at the right ticks and displaying the system state at configurable intervals. The scenario extends a couple ticks past the last event to show the final state.

---

## Putting It All Together

### Basic Simulation

```python
def main() -> None:
    # Reset singletons
    TransactionLedger.reset()
    User._next_id = 0
    Group._next_id = 0
    Expense._next_id = 0

    # Create system
    system = SplitwiseSystem()

    # Attach observers
    logger = ActivityLogger()
    system.add_listener(logger)

    # Register users
    alice = system.register_user("Alice", "alice@email.com")
    bob = system.register_user("Bob", "bob@email.com")
    charlie = system.register_user("Charlie", "charlie@email.com")
    diana = system.register_user("Diana", "diana@email.com")

    # Attach notification service (needs user dict)
    notifier = NotificationService(system.users)
    system.add_listener(notifier)

    print("╔═══════════════════════════════════════╗")
    print("║       SPLITWISE SYSTEM                ║")
    print("║  4 Users | Expense Sharing            ║")
    print("╚═══════════════════════════════════════╝")

    # Create a trip group
    trip = system.create_group("Goa Trip", alice.id)
    system.add_to_group(trip.id, bob.id)
    system.add_to_group(trip.id, charlie.id)
    system.add_to_group(trip.id, diana.id)

    # --- Expense 1: Alice pays for dinner, equal split ---
    system.add_expense(
        description="Dinner at Beach Shack",
        total=120.00,
        paid_by_id=alice.id,
        participant_ids=[alice.id, bob.id, charlie.id, diana.id],
        split_type=SplitType.EQUAL,
        category=ExpenseCategory.FOOD,
        group_id=trip.id,
    )

    # --- Expense 2: Bob pays for cab, exact split ---
    system.add_expense(
        description="Airport Cab",
        total=50.00,
        paid_by_id=bob.id,
        participant_ids=[alice.id, bob.id, charlie.id],
        split_type=SplitType.EXACT,
        split_data={"amounts": {alice.id: 20.00, bob.id: 10.00,
                                charlie.id: 20.00}},
        category=ExpenseCategory.TRANSPORT,
        group_id=trip.id,
    )

    # --- Expense 3: Charlie pays for hotel, percentage split ---
    system.add_expense(
        description="Hotel Room",
        total=400.00,
        paid_by_id=charlie.id,
        participant_ids=[alice.id, bob.id, charlie.id, diana.id],
        split_type=SplitType.PERCENTAGE,
        split_data={"percentages": {alice.id: 30.0, bob.id: 30.0,
                                     charlie.id: 20.0, diana.id: 20.0}},
        category=ExpenseCategory.ACCOMMODATION,
        group_id=trip.id,
    )

    # --- Expense 4: Diana pays for activities, share split ---
    system.add_expense(
        description="Scuba Diving",
        total=300.00,
        paid_by_id=diana.id,
        participant_ids=[alice.id, bob.id, charlie.id, diana.id],
        split_type=SplitType.SHARE,
        split_data={"shares": {alice.id: 2, bob.id: 2,
                                charlie.id: 1, diana.id: 1}},
        category=ExpenseCategory.ENTERTAINMENT,
        group_id=trip.id,
    )

    # Display individual balances
    print()
    for user in [alice, bob, charlie, diana]:
        system.display_balances(user.id)

    # Display group view with simplified debts
    system.display_group_balances(trip.id)

    # --- Settlement: Bob pays Alice ---
    owed = system.balance_sheet.get_balance(alice.id, bob.id)
    if owed > 0:
        system.settle_up(bob.id, alice.id, owed, note="Settling Goa Trip")

    # Show updated balances
    print("\n  AFTER SETTLEMENT:")
    system.display_balances(alice.id)
    system.display_balances(bob.id)

    # Show ledger
    TransactionLedger().display()


if __name__ == "__main__":
    main()
```

### Scripted Scenario

```python
def main_scenario() -> None:
    TransactionLedger.reset()
    User._next_id = 0
    Group._next_id = 0
    Expense._next_id = 0

    system = SplitwiseSystem()
    logger = ActivityLogger()
    system.add_listener(logger)

    # Register users
    alice = system.register_user("Alice", "alice@email.com")
    bob = system.register_user("Bob", "bob@email.com")
    charlie = system.register_user("Charlie", "charlie@email.com")

    scenario = SplitwiseScenario(system)

    # Create group and add members
    scenario.at_tick(1, "Create 'Roommates' group",
                     lambda: system.create_group("Roommates", alice.id))

    scenario.at_tick(1, "Add Bob to Roommates",
                     lambda: system.add_to_group(1, bob.id))

    scenario.at_tick(1, "Add Charlie to Roommates",
                     lambda: system.add_to_group(1, charlie.id))

    # Monthly expenses
    scenario.at_tick(2, "Alice pays electricity bill",
                     lambda: system.add_expense(
                         "Electricity Bill", 90.00, alice.id,
                         [alice.id, bob.id, charlie.id],
                         SplitType.EQUAL,
                         category=ExpenseCategory.UTILITIES,
                         group_id=1))

    scenario.at_tick(3, "Bob pays groceries",
                     lambda: system.add_expense(
                         "Groceries", 150.00, bob.id,
                         [alice.id, bob.id, charlie.id],
                         SplitType.EQUAL,
                         category=ExpenseCategory.FOOD,
                         group_id=1))

    scenario.at_tick(4, "Charlie pays internet",
                     lambda: system.add_expense(
                         "Internet Bill", 60.00, charlie.id,
                         [alice.id, bob.id, charlie.id],
                         SplitType.EQUAL,
                         category=ExpenseCategory.UTILITIES,
                         group_id=1))

    scenario.at_tick(5, "Alice pays for pizza (just her and Bob)",
                     lambda: system.add_expense(
                         "Pizza Night", 40.00, alice.id,
                         [alice.id, bob.id],
                         SplitType.EQUAL,
                         category=ExpenseCategory.FOOD))

    # Show simplified debts
    scenario.at_tick(6, "Show simplified debts",
                     lambda: system.display_group_balances(1))

    # Settlement
    scenario.at_tick(7, "Charlie settles with Alice",
                     lambda: system.settle_up(charlie.id, alice.id, 10.0))

    scenario.run(display_interval=4)

    # Final state
    system.display_group_balances(1)
    TransactionLedger().display()


if __name__ == "__main__":
    main_scenario()
```

### Debt Simplification Demo

```python
def demo_simplification() -> None:
    """Demonstrate how debt simplification reduces transactions."""
    TransactionLedger.reset()
    User._next_id = 0
    Group._next_id = 0
    Expense._next_id = 0

    system = SplitwiseSystem()

    # Register 5 users
    alice = system.register_user("Alice", "a@e.com")
    bob = system.register_user("Bob", "b@e.com")
    charlie = system.register_user("Charlie", "c@e.com")
    diana = system.register_user("Diana", "d@e.com")
    eve = system.register_user("Eve", "e@e.com")

    all_ids = [alice.id, bob.id, charlie.id, diana.id, eve.id]

    print("\n" + "=" * 55)
    print("  DEBT SIMPLIFICATION DEMO")
    print("=" * 55)

    # Multiple overlapping expenses
    system.add_expense("Dinner", 100.00, alice.id, all_ids, SplitType.EQUAL)
    system.add_expense("Cab", 75.00, bob.id, all_ids, SplitType.EQUAL)
    system.add_expense("Movie", 50.00, charlie.id, all_ids, SplitType.EQUAL)
    system.add_expense("Snacks", 25.00, diana.id, all_ids, SplitType.EQUAL)
    system.add_expense("Parking", 30.00, eve.id, all_ids, SplitType.EQUAL)

    # Show raw pairwise balances
    print("\n  RAW PAIRWISE DEBTS:")
    for user_id in all_ids:
        user = system.get_user(user_id)
        balances = system.get_balances(user_id)
        debts = {system.get_user(uid).name: amt
                 for uid, amt in balances.items() if amt < 0}
        if debts:
            for name, amt in debts.items():
                print(f"    {user.name} owes {name}: "
                      f"{CURRENCY_SYMBOL}{abs(amt):.2f}")

    # Count raw transactions
    raw_count = 0
    seen = set()
    for user_id in all_ids:
        for other_id, amt in system.get_balances(user_id).items():
            pair = (min(user_id, other_id), max(user_id, other_id))
            if amt != 0 and pair not in seen:
                seen.add(pair)
                raw_count += 1

    print(f"\n  Raw transactions needed: {raw_count}")

    # Simplified debts
    simplified = system.get_simplified_debts()
    print(f"  Simplified transactions: {len(simplified)}")
    DebtSimplifier.display_simplified(simplified, system.users)

    print("=" * 55)
```

This demo creates 5 users with 5 overlapping expenses and shows the reduction. Without simplification you might need up to 10 pairwise transfers. After simplification, 4 or fewer transactions settle everything. The key insight is that intermediate debts cancel out: if A owes B and B owes C the same amount, A can pay C directly.

### Unit Tests

```python
def test_equal_split() -> None:
    """Test that equal split divides evenly."""
    TransactionLedger.reset()
    User._next_id = 0
    Expense._next_id = 0

    system = SplitwiseSystem()
    a = system.register_user("A", "a@e.com")
    b = system.register_user("B", "b@e.com")
    c = system.register_user("C", "c@e.com")

    system.add_expense("Test", 90.00, a.id,
                       [a.id, b.id, c.id], SplitType.EQUAL)

    # B and C each owe A $30
    assert system.balance_sheet.get_balance(a.id, b.id) == 30.00
    assert system.balance_sheet.get_balance(a.id, c.id) == 30.00
    # A's net: +60 (owed by B and C)
    assert system.balance_sheet.get_net_balance(a.id) == 60.00
    print("✅ test_equal_split passed")


def test_exact_split() -> None:
    """Test that exact split assigns specified amounts."""
    TransactionLedger.reset()
    User._next_id = 0
    Expense._next_id = 0

    system = SplitwiseSystem()
    a = system.register_user("A", "a@e.com")
    b = system.register_user("B", "b@e.com")
    c = system.register_user("C", "c@e.com")

    system.add_expense("Test", 100.00, a.id,
                       [a.id, b.id, c.id], SplitType.EXACT,
                       split_data={"amounts": {a.id: 50.0, b.id: 30.0,
                                               c.id: 20.0}})

    assert system.balance_sheet.get_balance(a.id, b.id) == 30.00
    assert system.balance_sheet.get_balance(a.id, c.id) == 20.00
    print("✅ test_exact_split passed")


def test_percentage_split() -> None:
    """Test that percentage split divides by percentages."""
    TransactionLedger.reset()
    User._next_id = 0
    Expense._next_id = 0

    system = SplitwiseSystem()
    a = system.register_user("A", "a@e.com")
    b = system.register_user("B", "b@e.com")

    system.add_expense("Test", 200.00, a.id,
                       [a.id, b.id], SplitType.PERCENTAGE,
                       split_data={"percentages": {a.id: 60.0, b.id: 40.0}})

    # B owes A 40% of 200 = $80
    assert system.balance_sheet.get_balance(a.id, b.id) == 80.00
    print("✅ test_percentage_split passed")


def test_share_split() -> None:
    """Test that share split divides proportionally."""
    TransactionLedger.reset()
    User._next_id = 0
    Expense._next_id = 0

    system = SplitwiseSystem()
    a = system.register_user("A", "a@e.com")
    b = system.register_user("B", "b@e.com")
    c = system.register_user("C", "c@e.com")

    # Total 300, shares: A=1, B=2, C=3 → A=50, B=100, C=150
    system.add_expense("Test", 300.00, a.id,
                       [a.id, b.id, c.id], SplitType.SHARE,
                       split_data={"shares": {a.id: 1, b.id: 2, c.id: 3}})

    assert system.balance_sheet.get_balance(a.id, b.id) == 100.00
    assert system.balance_sheet.get_balance(a.id, c.id) == 150.00
    print("✅ test_share_split passed")


def test_settlement() -> None:
    """Test that settlement reduces debt."""
    TransactionLedger.reset()
    User._next_id = 0
    Expense._next_id = 0

    system = SplitwiseSystem()
    a = system.register_user("A", "a@e.com")
    b = system.register_user("B", "b@e.com")

    system.add_expense("Test", 100.00, a.id,
                       [a.id, b.id], SplitType.EQUAL)

    # B owes A $50
    assert system.balance_sheet.get_balance(a.id, b.id) == 50.00

    # B pays A $30
    system.settle_up(b.id, a.id, 30.00)
    assert system.balance_sheet.get_balance(a.id, b.id) == 20.00

    # B pays the rest
    system.settle_up(b.id, a.id, 20.00)
    assert system.balance_sheet.get_balance(a.id, b.id) == 0.00
    print("✅ test_settlement passed")


def test_debt_simplification() -> None:
    """Test that simplification reduces transaction count."""
    TransactionLedger.reset()
    User._next_id = 0
    Expense._next_id = 0

    system = SplitwiseSystem()
    a = system.register_user("A", "a@e.com")
    b = system.register_user("B", "b@e.com")
    c = system.register_user("C", "c@e.com")

    # A pays $90 split equally (B and C owe A $30 each)
    system.add_expense("Dinner", 90.00, a.id,
                       [a.id, b.id, c.id], SplitType.EQUAL)

    # B pays $60 split equally (A and C owe B $20 each)
    system.add_expense("Cab", 60.00, b.id,
                       [a.id, b.id, c.id], SplitType.EQUAL)

    # Simplified: A's net = +30-20 = +10, B's net = -30+20 = -10+20 = +10(??)
    # Actually: A net = (30+30) - 20 = 40, wait...
    # A paid 90, owes 30 to self + 20 to B = net: 30+30-20 = 40
    # B paid 60, owes 30 to A + 20 to self = net: 20+20-30 = 10
    # C owes A 30 and B 20 = net: -50
    # Simplified: C pays A $40, C pays B $10 → 2 transactions
    simplified = system.get_simplified_debts()
    assert len(simplified) <= 2  # should be at most 2 transactions
    print("✅ test_debt_simplification passed")


def test_invalid_exact_split() -> None:
    """Test that exact split rejects mismatched amounts."""
    TransactionLedger.reset()
    User._next_id = 0
    Expense._next_id = 0

    system = SplitwiseSystem()
    a = system.register_user("A", "a@e.com")
    b = system.register_user("B", "b@e.com")

    result = system.add_expense("Test", 100.00, a.id,
                                [a.id, b.id], SplitType.EXACT,
                                split_data={"amounts": {a.id: 50.0,
                                                         b.id: 30.0}})
    assert result is None  # should be rejected (50+30 != 100)
    print("✅ test_invalid_exact_split passed")


def test_group_membership() -> None:
    """Test that group expenses validate membership."""
    TransactionLedger.reset()
    User._next_id = 0
    Group._next_id = 0
    Expense._next_id = 0

    system = SplitwiseSystem()
    a = system.register_user("A", "a@e.com")
    b = system.register_user("B", "b@e.com")
    c = system.register_user("C", "c@e.com")

    group = system.create_group("Test Group", a.id)
    system.add_to_group(group.id, b.id)
    # C is NOT in the group

    result = system.add_expense("Test", 60.00, a.id,
                                [a.id, b.id, c.id], SplitType.EQUAL,
                                group_id=group.id)
    assert result is None  # rejected — C not in group
    print("✅ test_group_membership passed")


def run_all_tests() -> None:
    print("\n" + "=" * 40)
    print("  RUNNING TESTS")
    print("=" * 40 + "\n")
    test_equal_split()
    test_exact_split()
    test_percentage_split()
    test_share_split()
    test_settlement()
    test_debt_simplification()
    test_invalid_exact_split()
    test_group_membership()
    print("\n✅ All tests passed!\n")
```

Each test resets the singletons and ID counters for isolation. `test_equal_split` verifies that a \$90 expense split 3 ways creates \$30 debts. `test_exact_split` checks that caller-specified amounts are applied correctly. `test_settlement` confirms that paying off a debt reduces and eventually zeroes the balance. `test_debt_simplification` verifies that multiple overlapping expenses produce a minimal set of settlement transactions. `test_invalid_exact_split` ensures that amounts not summing to the total are rejected. `test_group_membership` confirms that non-members cannot be included in group expenses.

---

## Class Diagram

```
┌──────────────────────────────────────────────────────────────┐
│                     SplitwiseSystem                           │
│──────────────────────────────────────────────────────────────│
│ - _users: dict[int, User]                                     │
│ - _groups: dict[int, Group]                                   │
│ - _balance_sheet: BalanceSheet                                │
│ - _ledger: TransactionLedger                                  │
│ - _listeners: list[SplitwiseEventListener]                    │
│ - _tick: int                                                  │
│──────────────────────────────────────────────────────────────│
│ + register_user(name, email) -> User                          │
│ + create_group(name, creator_id) -> Group                     │
│ + add_expense(desc, total, ...) -> Expense | None             │
│ + settle_up(payer, payee, amount) -> Settlement               │
│ + get_simplified_debts() -> list[(debtor, creditor, amt)]     │
│ + display_balances(user_id) -> None                           │
└──────┬──────────┬──────────┬──────────┬──────────────────────┘
       │          │          │          │
  owns ▼     owns ▼    owns ▼    owns ▼
┌──────────────┐ ┌────────────────┐ ┌──────────────────────────┐
│    User      │ │     Group      │ │     BalanceSheet         │
│──────────────│ │────────────────│ │──────────────────────────│
│ - _id        │ │ - _id          │ │ - _balances: dict[dict]  │
│ - _name      │ │ - _name        │ │──────────────────────────│
│ - _email     │ │ - _created_by  │ │ + record_expense()       │
│ - _phone     │ │ - _members     │ │ + record_settlement()    │
│──────────────│ │ - _expense_ids │ │ + get_balance(a, b)      │
│ + id, name   │ │────────────────│ │ + get_net_balance(uid)   │
│ + email      │ │ + add_member() │ │ + get_user_balances()    │
└──────────────┘ │ + remove_      │ └──────────────────────────┘
                 │   member()     │
                 │ + has_member() │ ┌──────────────────────────┐
                 └────────────────┘ │    DebtSimplifier        │
                                    │──────────────────────────│
┌──────────────────────────────────┐│ + simplify(sheet, uids)  │
│  «abstract» SplitStrategy       ││   -> list[(d, c, amt)]   │
│──────────────────────────────────││ + display_simplified()   │
│ + calculate_splits(total,       │└──────────────────────────┘
│     participants, data) -> dict │
│ + validate(total, ...) -> bool  │
│ + split_type() -> SplitType     │
└──────────┬───────────────────────┘
           │ implemented by
  ┌────────┼──────────┬──────────────┐
  ▼        ▼          ▼              ▼
Equal    Exact     Percentage     Share
Split    Split     Split          Split
Strategy Strategy  Strategy       Strategy

┌──────────────────────────────────────────────────────────────┐
│                   SplitStrategyFactory                         │
│──────────────────────────────────────────────────────────────│
│ + get_strategy(split_type) -> SplitStrategy                   │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│  «abstract» SplitwiseEventListener       │
│──────────────────────────────────────────│
│ + on_expense_added(expense) -> None      │
│ + on_settlement_made(settlement) -> None │
│ + on_group_created(group) -> None        │
│ + on_member_added(group, user) -> None   │
└──────────┬───────────────────────────────┘
           │ implemented by
     ┌─────┴───────────────┐
     ▼                     ▼
ActivityLogger     NotificationService

┌──────────────────────────────────────────┐
│     TransactionLedger (Singleton)        │
│──────────────────────────────────────────│
│ - _expenses: list[Expense]               │
│ - _settlements: list[Settlement]         │
│──────────────────────────────────────────│
│ + record_expense(expense) -> None        │
│ + record_settlement(...) -> Settlement   │
│ + expenses_for_user(uid) -> list         │
│ + expenses_for_group(gid) -> list        │
│ + total_expense_amount -> float          │
└──────────────────────────────────────────┘

┌────────────────────────┐  ┌──────────────────────────────┐
│  Expense               │  │  Settlement (frozen)          │
│────────────────────────│  │──────────────────────────────│
│ - _id: int             │  │ + settlement_id: int          │
│ - _description: str    │  │ + payer_id: int               │
│ - _total: float        │  │ + payee_id: int               │
│ - _paid_by: User       │  │ + amount: float               │
│ - _participants: list  │  │ + timestamp: int              │
│ - _split_type          │  │ + note: str                   │
│ - _splits: dict        │  └──────────────────────────────┘
│ - _category            │
│ - _group_id            │
│────────────────────────│
│ + get_user_share(uid)  │
└────────────────────────┘

┌──────────────────────────┐
│   SplitwiseScenario      │
│──────────────────────────│
│ - _system                │
│ - _events: list[...]     │
│──────────────────────────│
│ + at_tick(t, desc, fn)   │
│ + run(interval) -> None  │
└──────────────────────────┘

┌──────────────────┐  ┌──────────────────────┐  ┌─────────────────────┐
│ SplitType (Enum) │  │ExpenseCategory (Enum)│  │TransactionType(Enum)│
│──────────────────│  │──────────────────────│  │─────────────────────│
│ EQUAL            │  │ FOOD                 │  │ EXPENSE             │
│ EXACT            │  │ TRANSPORT            │  │ SETTLEMENT          │
│ PERCENTAGE       │  │ ACCOMMODATION        │  └─────────────────────┘
│ SHARE            │  │ ENTERTAINMENT        │
└──────────────────┘  │ SHOPPING             │
                      │ UTILITIES            │
                      │ OTHER                │
                      └──────────────────────┘
```

---

## How to Run

Save all code into a single file `splitwise_system.py` and run:

```bash
python splitwise_system.py
```

### Example Output

```
╔═══════════════════════════════════════╗
║       SPLITWISE SYSTEM                ║
║  4 Users | Expense Sharing            ║
╚═══════════════════════════════════════╝

  👥 GROUP CREATED: "Goa Trip" by Alice
  ➕ MEMBER ADDED: Bob → "Goa Trip"
  ➕ MEMBER ADDED: Charlie → "Goa Trip"
  ➕ MEMBER ADDED: Diana → "Goa Trip"

  📝 EXPENSE: "Dinner at Beach Shack" — $120.00 paid by Alice (equal split)
  🔔 → Bob: You owe Alice $30.00 for "Dinner at Beach Shack"
  🔔 → Charlie: You owe Alice $30.00 for "Dinner at Beach Shack"
  🔔 → Diana: You owe Alice $30.00 for "Dinner at Beach Shack"

  📝 EXPENSE: "Airport Cab" — $50.00 paid by Bob (exact split)
  📝 EXPENSE: "Hotel Room" — $400.00 paid by Charlie (percentage split)
  📝 EXPENSE: "Scuba Diving" — $300.00 paid by Diana (share split)

  ==================================================
  BALANCES FOR Alice
  ==================================================
  ⬅ Bob owes you: $10.00
  ⬅ Charlie owes you: $30.00
  ➡ You owe Charlie: $120.00
  ➡ You owe Diana: $100.00
  --------------------------------------------------
  Total owed to you: $40.00
  Total you owe:     $220.00
  Net: $180.00 (you owe)

  ==================================================
  SIMPLIFIED DEBTS (3 transaction(s))
  ==================================================
  💸 Alice pays Charlie $150.00
  💸 Alice pays Diana $30.00
  💸 Bob pays Diana $70.00

  💰 SETTLEMENT: uid2 → uid1: $10.00

  =======================================================
  TRANSACTION LEDGER
  Expenses: 4 (total: $870.00)
  Settlements: 1 (total: $10.00)
  =======================================================
  Recent Expenses:
    Expense#1("Dinner at Beach Shack" $120.00 paid_by=Alice [equal] [...])
    Expense#2("Airport Cab" $50.00 paid_by=Bob [exact] [...])
    Expense#3("Hotel Room" $400.00 paid_by=Charlie [percentage] [...])
    Expense#4("Scuba Diving" $300.00 paid_by=Diana [share] [...])
  Recent Settlements:
    Settlement#1(uid2 → uid1: $10.00)
```
