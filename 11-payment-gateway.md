# Design Payment Gateway – Low-Level Design Guide

![Bidur Sapkota](https://www.bidursapkota.com.np/images/gravatar.webp "Bidur Sapkota - Developer")&nbsp;[Bidur Sapkota](https://www.bidursapkota.com.np/)

## Table of Contents

1. [Requirements](#requirements)
2. [Key Design Patterns](#key-design-patterns)
3. [Core Enums & Constants](#core-enums--constants)
4. [Money & Currency](#money--currency)
5. [Payment Method & Card Details](#payment-method--card-details)
6. [Transaction & Idempotency](#transaction--idempotency)
7. [Ledger & Double-Entry Bookkeeping](#ledger--double-entry-bookkeeping)
8. [Payment Provider (Strategy Pattern)](#payment-provider-strategy-pattern)
9. [Retry Policy (Strategy Pattern)](#retry-policy-strategy-pattern)
10. [Fraud Detection (Chain of Responsibility)](#fraud-detection-chain-of-responsibility)
11. [Transaction State Machine (State Pattern)](#transaction-state-machine-state-pattern)
12. [Refund Manager](#refund-manager)
13. [Webhook & Event System (Observer Pattern)](#webhook--event-system-observer-pattern)
14. [Payment Router](#payment-router)
15. [Payment Gateway (Facade)](#payment-gateway-facade)
16. [Simulation Engine](#simulation-engine)
17. [Putting It All Together](#putting-it-all-together)
18. [Class Diagram](#class-diagram)
19. [How to Run](#how-to-run)

---

## Requirements

### Functional Requirements

| #  | Requirement                                                                                     |
| -- | ----------------------------------------------------------------------------------------------- |
| 1  | Process payments through multiple providers (Stripe, PayPal, bank transfer)                     |
| 2  | Guarantee idempotency — duplicate requests with the same idempotency key return the cached result |
| 3  | Maintain a double-entry ledger for every financial movement (debits and credits always balance)  |
| 4  | Route payments to providers based on configurable rules (currency, amount, provider health)      |
| 5  | Retry failed payment attempts with configurable backoff strategies (fixed, exponential)          |
| 6  | Validate transactions through a fraud detection pipeline before processing                      |
| 7  | Track transaction lifecycle through explicit states: CREATED → PROCESSING → COMPLETED/FAILED    |
| 8  | Support full and partial refunds with ledger entries and provider-level reversal                 |
| 9  | Emit events on state changes — webhook listeners react to payment, refund, and failure events    |
| 10 | Collect metrics: total volume, success/failure rates, provider latency, refund totals            |

### Non-Functional Requirements

- Clean OOP with SOLID principles.
- Extensible — new payment providers, retry strategies, and fraud rules can be added without modifying core logic.
- Simulatable — the system runs step-by-step for testing and demonstration.

---

## Key Design Patterns

| Pattern                  | Where It Is Used                                      | Why                                                                                     |
| ------------------------ | ----------------------------------------------------- | --------------------------------------------------------------------------------------- |
| Strategy                 | `PaymentProvider` processes charges through providers  | Swap between Stripe, PayPal, or bank transfer without changing gateway logic             |
| Strategy                 | `RetryPolicy` controls backoff between attempts        | Add fixed, exponential, or custom retry logic without modifying the retry executor       |
| State                    | `TransactionStateMachine` manages transaction lifecycle | Enforce legal state transitions and prevent invalid operations on a transaction          |
| Chain of Responsibility  | `FraudRule` chain validates transactions               | Add, remove, or reorder fraud checks without touching other rules or the gateway         |
| Observer                 | `WebhookListener` reacts to payment events             | Loggers, notifiers, and analytics react independently to payment lifecycle events        |
| Facade                   | `PaymentGateway` exposes a unified API                 | Hides the complexity of routing, retries, fraud checks, ledger, and provider interaction |

---

## Core Enums & Constants

```python
import time
import uuid
import hashlib
import threading
import random
from enum import Enum
from abc import ABC, abstractmethod
from dataclasses import dataclass, field


class TransactionStatus(Enum):
    CREATED = "created"
    PROCESSING = "processing"
    COMPLETED = "completed"
    FAILED = "failed"
    REFUNDED = "refunded"
    PARTIALLY_REFUNDED = "partially_refunded"


class PaymentMethodType(Enum):
    CREDIT_CARD = "credit_card"
    DEBIT_CARD = "debit_card"
    BANK_TRANSFER = "bank_transfer"
    DIGITAL_WALLET = "digital_wallet"


class Currency(Enum):
    USD = "USD"
    EUR = "EUR"
    GBP = "GBP"
    JPY = "JPY"
    NPR = "NPR"


class LedgerEntryType(Enum):
    DEBIT = "debit"
    CREDIT = "credit"


class WebhookEvent(Enum):
    PAYMENT_CREATED = "payment.created"
    PAYMENT_PROCESSING = "payment.processing"
    PAYMENT_COMPLETED = "payment.completed"
    PAYMENT_FAILED = "payment.failed"
    REFUND_INITIATED = "refund.initiated"
    REFUND_COMPLETED = "refund.completed"


class ProviderStatus(Enum):
    HEALTHY = "healthy"
    DEGRADED = "degraded"
    DOWN = "down"


# ------------------------------------------------------------------
# System constants
# ------------------------------------------------------------------

MAX_RETRY_ATTEMPTS = 3
DEFAULT_TIMEOUT_MS = 30_000              # 30 seconds
IDEMPOTENCY_KEY_TTL_MS = 24 * 3600 * 1000  # 24 hours
MAX_TRANSACTION_AMOUNT = 1_000_000_00    # $1,000,000 in cents
MIN_TRANSACTION_AMOUNT = 50              # $0.50 in cents
FRAUD_SCORE_THRESHOLD = 0.7              # block if score >= 0.7
```

`TransactionStatus` tracks the full lifecycle of a payment — `CREATED` when a request arrives, `PROCESSING` when sent to a provider, `COMPLETED` on success, `FAILED` on rejection, `REFUNDED` after full reversal, and `PARTIALLY_REFUNDED` when only part of the amount is returned. `PaymentMethodType` categorizes how the customer pays. `Currency` enumerates supported currencies. `LedgerEntryType` distinguishes debits from credits in double-entry accounting. `WebhookEvent` defines the events external systems can subscribe to. `ProviderStatus` tracks whether a payment provider is operational. All monetary amounts are stored in the smallest currency unit (cents) to avoid floating-point errors.

---

## Money & Currency

```python
@dataclass(frozen=True)
class Money:
    """
    Value object representing a monetary amount.

    Amounts are stored in the smallest currency unit (cents for USD)
    to avoid floating-point precision issues. Immutable via frozen=True.
    """
    amount: int           # in smallest unit (cents)
    currency: Currency

    def __post_init__(self) -> None:
        if self.amount < 0:
            raise ValueError(f"Amount cannot be negative: {self.amount}")

    def add(self, other: "Money") -> "Money":
        if self.currency != other.currency:
            raise ValueError(
                f"Cannot add {self.currency.value} and "
                f"{other.currency.value}"
            )
        return Money(self.amount + other.amount, self.currency)

    def subtract(self, other: "Money") -> "Money":
        if self.currency != other.currency:
            raise ValueError(
                f"Cannot subtract {self.currency.value} and "
                f"{other.currency.value}"
            )
        if other.amount > self.amount:
            raise ValueError(
                f"Insufficient amount: {self.amount} - {other.amount}"
            )
        return Money(self.amount - other.amount, self.currency)

    def display(self) -> str:
        """Format for display: $12.50, €8.00, etc."""
        symbols = {"USD": "$", "EUR": "€", "GBP": "£",
                   "JPY": "¥", "NPR": "Rs"}
        symbol = symbols.get(self.currency.value, self.currency.value)
        if self.currency == Currency.JPY:
            return f"{symbol}{self.amount}"
        return f"{symbol}{self.amount / 100:.2f}"

    def __repr__(self) -> str:
        return self.display()
```

`Money` is a value object — immutable (`frozen=True`) and compared by value, not identity. Storing amounts as integers in the smallest unit (cents) eliminates floating-point errors that plague financial calculations (e.g., `0.1 + 0.2 != 0.3` in floats). `add` and `subtract` enforce currency matching — you cannot add USD to EUR without explicit conversion. `subtract` prevents negative balances by raising on insufficient funds. JPY has no fractional unit so its display omits decimals.

---

## Payment Method & Card Details

```python
@dataclass(frozen=True)
class CardDetails:
    """
    Tokenized card information.

    Stores only the last four digits and a token reference — never
    the full card number. The token is what gets sent to providers.
    """
    last_four: str
    brand: str             # "visa", "mastercard", "amex"
    exp_month: int
    exp_year: int
    token: str             # tokenized by payment processor

    def is_expired(self) -> bool:
        now = time.gmtime()
        if self.exp_year < now.tm_year:
            return True
        if (self.exp_year == now.tm_year
                and self.exp_month < now.tm_mon):
            return True
        return False

    def masked_display(self) -> str:
        return f"{self.brand.upper()} ****{self.last_four}"


@dataclass(frozen=True)
class PaymentMethod:
    """
    A customer's payment instrument.

    Combines the method type with optional card details and an
    identifier from the payment provider (e.g., Stripe's pm_xxx).
    """
    method_id: str
    method_type: PaymentMethodType
    card_details: CardDetails | None = None
    provider_token: str = ""

    def __repr__(self) -> str:
        if self.card_details:
            return (f"PaymentMethod({self.method_type.value}, "
                    f"{self.card_details.masked_display()})")
        return f"PaymentMethod({self.method_type.value})"
```

`CardDetails` stores only tokenized information — the full PAN (Primary Account Number) is never stored, satisfying PCI-DSS requirements. `is_expired` checks against current time. `PaymentMethod` wraps the method type with its card details and provider-specific token. Both are frozen dataclasses since payment instruments should not be mutated after creation.

---

## Transaction & Idempotency

```python
@dataclass
class Transaction:
    """
    A single payment transaction.

    Each transaction has a unique ID, an idempotency key to prevent
    duplicate processing, an amount, payment method, and status.
    The status transitions are enforced by the TransactionStateMachine.
    """
    transaction_id: str
    idempotency_key: str
    amount: Money
    payment_method: PaymentMethod
    merchant_id: str
    description: str = ""
    status: TransactionStatus = TransactionStatus.CREATED
    provider_id: str = ""
    provider_transaction_id: str = ""
    created_at: int = field(
        default_factory=lambda: int(time.time() * 1000)
    )
    updated_at: int = field(
        default_factory=lambda: int(time.time() * 1000)
    )
    failure_reason: str = ""
    metadata: dict = field(default_factory=dict)
    refunded_amount: Money | None = None
    attempts: int = 0

    def __repr__(self) -> str:
        return (
            f"Txn(id={self.transaction_id[:12]}... "
            f"status={self.status.value} "
            f"amount={self.amount} "
            f"merchant={self.merchant_id})"
        )


class IdempotencyStore:
    """
    Thread-safe store for idempotency keys.

    Prevents duplicate payment processing by caching results keyed
    by the idempotency key. Entries expire after a configurable TTL.
    """

    def __init__(self, ttl_ms: int = IDEMPOTENCY_KEY_TTL_MS) -> None:
        self._store: dict[str, Transaction] = {}
        self._timestamps: dict[str, int] = {}
        self._ttl_ms = ttl_ms
        self._lock = threading.RLock()

    def get(self, key: str) -> Transaction | None:
        """Return cached transaction if key exists and hasn't expired."""
        with self._lock:
            if key not in self._store:
                return None
            stored_at = self._timestamps[key]
            now = int(time.time() * 1000)
            if now - stored_at > self._ttl_ms:
                del self._store[key]
                del self._timestamps[key]
                return None
            return self._store[key]

    def put(self, key: str, transaction: Transaction) -> None:
        """Cache a transaction result under the idempotency key."""
        with self._lock:
            self._store[key] = transaction
            self._timestamps[key] = int(time.time() * 1000)

    def contains(self, key: str) -> bool:
        return self.get(key) is not None

    @property
    def size(self) -> int:
        return len(self._store)
```

`Transaction` is the central data model — every payment creates one. `idempotency_key` is client-supplied; if a client retries with the same key, the gateway returns the cached result instead of charging again. `provider_transaction_id` links to the external provider's record. `attempts` tracks how many times the gateway tried to process this payment. `IdempotencyStore` is a thread-safe cache mapping idempotency keys to completed transactions. The TTL prevents unbounded memory growth — after 24 hours, a key expires and the same idempotency key could process a new payment.

---

## Ledger & Double-Entry Bookkeeping

```python
@dataclass(frozen=True)
class LedgerEntry:
    """
    A single entry in the financial ledger.

    Every financial movement creates two entries (debit and credit)
    that must balance. This is the foundation of double-entry
    bookkeeping — the total of all debits always equals credits.
    """
    entry_id: str
    transaction_id: str
    entry_type: LedgerEntryType
    amount: Money
    account: str              # e.g., "merchant:m-001", "platform:fees"
    description: str
    created_at: int = field(
        default_factory=lambda: int(time.time() * 1000)
    )

    def __repr__(self) -> str:
        direction = "DR" if self.entry_type == LedgerEntryType.DEBIT \
            else "CR"
        return (f"Ledger({direction} {self.amount} "
                f"account={self.account})")


class Ledger:
    """
    Double-entry bookkeeping ledger.

    Every financial operation (charge, refund, fee) records a pair
    of entries — one debit and one credit — ensuring the books
    always balance. Thread-safe for concurrent transactions.
    """

    def __init__(self) -> None:
        self._entries: list[LedgerEntry] = []
        self._lock = threading.RLock()

    def record_payment(self, transaction: Transaction) -> None:
        """
        Record a successful payment as two ledger entries.

        Debit the customer's account (money leaves the customer),
        credit the merchant's account (money arrives at the merchant).
        """
        entry_pair_id = str(uuid.uuid4())[:8]

        debit = LedgerEntry(
            entry_id=f"{entry_pair_id}-dr",
            transaction_id=transaction.transaction_id,
            entry_type=LedgerEntryType.DEBIT,
            amount=transaction.amount,
            account=f"customer:{transaction.idempotency_key[:8]}",
            description=f"Payment for {transaction.description}",
        )

        credit = LedgerEntry(
            entry_id=f"{entry_pair_id}-cr",
            transaction_id=transaction.transaction_id,
            entry_type=LedgerEntryType.CREDIT,
            amount=transaction.amount,
            account=f"merchant:{transaction.merchant_id}",
            description=f"Payment received for {transaction.description}",
        )

        with self._lock:
            self._entries.append(debit)
            self._entries.append(credit)

    def record_refund(self, transaction: Transaction,
                      refund_amount: Money) -> None:
        """
        Record a refund as two ledger entries.

        Debit the merchant (money leaves merchant), credit the
        customer (money returns to customer). Reverses the flow
        of the original payment.
        """
        entry_pair_id = str(uuid.uuid4())[:8]

        debit = LedgerEntry(
            entry_id=f"{entry_pair_id}-dr",
            transaction_id=transaction.transaction_id,
            entry_type=LedgerEntryType.DEBIT,
            amount=refund_amount,
            account=f"merchant:{transaction.merchant_id}",
            description=f"Refund for txn {transaction.transaction_id[:8]}",
        )

        credit = LedgerEntry(
            entry_id=f"{entry_pair_id}-cr",
            transaction_id=transaction.transaction_id,
            entry_type=LedgerEntryType.CREDIT,
            amount=refund_amount,
            account=f"customer:{transaction.idempotency_key[:8]}",
            description=f"Refund received for txn "
                        f"{transaction.transaction_id[:8]}",
        )

        with self._lock:
            self._entries.append(debit)
            self._entries.append(credit)

    def get_entries_for_transaction(self, transaction_id: str
                                    ) -> list[LedgerEntry]:
        with self._lock:
            return [e for e in self._entries
                    if e.transaction_id == transaction_id]

    def get_balance(self, account: str) -> int:
        """
        Calculate net balance for an account.

        Credits increase the balance, debits decrease it.
        """
        with self._lock:
            balance = 0
            for entry in self._entries:
                if entry.account == account:
                    if entry.entry_type == LedgerEntryType.CREDIT:
                        balance += entry.amount.amount
                    else:
                        balance -= entry.amount.amount
            return balance

    def is_balanced(self) -> bool:
        """Verify that total debits equal total credits."""
        with self._lock:
            total_debits = sum(
                e.amount.amount for e in self._entries
                if e.entry_type == LedgerEntryType.DEBIT
            )
            total_credits = sum(
                e.amount.amount for e in self._entries
                if e.entry_type == LedgerEntryType.CREDIT
            )
            return total_debits == total_credits

    @property
    def total_entries(self) -> int:
        return len(self._entries)

    def display(self) -> None:
        """Print the full ledger in tabular form."""
        print(f"\n  {'─'*60}")
        print(f"  {'LEDGER':^60}")
        print(f"  {'─'*60}")
        print(f"  {'Type':<6} {'Amount':<12} {'Account':<28} "
              f"{'Description'}")
        print(f"  {'─'*60}")
        with self._lock:
            for entry in self._entries:
                direction = "DR" if entry.entry_type == \
                    LedgerEntryType.DEBIT else "CR"
                print(f"  {direction:<6} {entry.amount.display():<12} "
                      f"{entry.account:<28} {entry.description[:30]}")
        print(f"  {'─'*60}")
        balanced = "✅ BALANCED" if self.is_balanced() \
            else "❌ UNBALANCED"
        print(f"  {balanced}")
```

`LedgerEntry` represents a single half of a double-entry pair. Every financial movement creates exactly two entries — a debit and a credit — ensuring the books always balance. For a payment: debit the customer (money leaves), credit the merchant (money arrives). For a refund: the flow reverses. `record_payment` and `record_refund` always create paired entries atomically under a lock. `is_balanced` is the critical audit invariant — if total debits ever differ from total credits, something went wrong. `get_balance` calculates the net position of any account by summing credits and subtracting debits.

---

## Payment Provider (Strategy Pattern)

```python
@dataclass
class ProviderResponse:
    """Response from a payment provider after a charge or refund."""
    success: bool
    provider_transaction_id: str
    error_message: str = ""
    error_code: str = ""
    latency_ms: int = 0


class PaymentProvider(ABC):
    """
    Abstract payment provider.

    Each provider implements charge and refund against its own API.
    The gateway selects a provider via the PaymentRouter and calls
    its methods without knowing the provider-specific details.
    """

    @abstractmethod
    def charge(self, transaction: Transaction) -> ProviderResponse:
        ...

    @abstractmethod
    def refund(self, transaction: Transaction,
               amount: Money) -> ProviderResponse:
        ...

    @abstractmethod
    def provider_id(self) -> str:
        ...

    @abstractmethod
    def supported_currencies(self) -> list[Currency]:
        ...

    @abstractmethod
    def status(self) -> ProviderStatus:
        ...

    @abstractmethod
    def display_name(self) -> str:
        ...


class StripeProvider(PaymentProvider):
    """
    Simulates Stripe payment processing.

    In a real system, this would make HTTP calls to Stripe's API.
    Here it simulates latency and a configurable failure rate.
    """

    def __init__(self, failure_rate: float = 0.1) -> None:
        self._failure_rate = failure_rate
        self._status = ProviderStatus.HEALTHY
        self._total_processed = 0
        self._total_failed = 0

    def charge(self, transaction: Transaction) -> ProviderResponse:
        latency = random.randint(50, 200)

        if self._status == ProviderStatus.DOWN:
            self._total_failed += 1
            return ProviderResponse(
                success=False,
                provider_transaction_id="",
                error_message="Stripe is currently unavailable",
                error_code="PROVIDER_DOWN",
                latency_ms=latency,
            )

        if random.random() < self._failure_rate:
            self._total_failed += 1
            return ProviderResponse(
                success=False,
                provider_transaction_id="",
                error_message="Card declined by issuer",
                error_code="CARD_DECLINED",
                latency_ms=latency,
            )

        self._total_processed += 1
        provider_txn_id = f"ch_{uuid.uuid4().hex[:16]}"
        return ProviderResponse(
            success=True,
            provider_transaction_id=provider_txn_id,
            latency_ms=latency,
        )

    def refund(self, transaction: Transaction,
               amount: Money) -> ProviderResponse:
        latency = random.randint(30, 150)

        if self._status == ProviderStatus.DOWN:
            return ProviderResponse(
                success=False,
                provider_transaction_id="",
                error_message="Stripe unavailable for refund",
                error_code="PROVIDER_DOWN",
                latency_ms=latency,
            )

        refund_id = f"re_{uuid.uuid4().hex[:16]}"
        return ProviderResponse(
            success=True,
            provider_transaction_id=refund_id,
            latency_ms=latency,
        )

    def provider_id(self) -> str:
        return "stripe"

    def supported_currencies(self) -> list[Currency]:
        return [Currency.USD, Currency.EUR, Currency.GBP]

    def status(self) -> ProviderStatus:
        return self._status

    def set_status(self, status: ProviderStatus) -> None:
        self._status = status

    def display_name(self) -> str:
        return "Stripe"


class PayPalProvider(PaymentProvider):
    """
    Simulates PayPal payment processing.

    Supports a different currency set than Stripe and has
    its own simulated failure characteristics.
    """

    def __init__(self, failure_rate: float = 0.15) -> None:
        self._failure_rate = failure_rate
        self._status = ProviderStatus.HEALTHY
        self._total_processed = 0
        self._total_failed = 0

    def charge(self, transaction: Transaction) -> ProviderResponse:
        latency = random.randint(100, 400)

        if self._status == ProviderStatus.DOWN:
            self._total_failed += 1
            return ProviderResponse(
                success=False,
                provider_transaction_id="",
                error_message="PayPal service unavailable",
                error_code="PROVIDER_DOWN",
                latency_ms=latency,
            )

        if random.random() < self._failure_rate:
            self._total_failed += 1
            return ProviderResponse(
                success=False,
                provider_transaction_id="",
                error_message="PayPal transaction limit exceeded",
                error_code="LIMIT_EXCEEDED",
                latency_ms=latency,
            )

        self._total_processed += 1
        provider_txn_id = f"PAY-{uuid.uuid4().hex[:16].upper()}"
        return ProviderResponse(
            success=True,
            provider_transaction_id=provider_txn_id,
            latency_ms=latency,
        )

    def refund(self, transaction: Transaction,
               amount: Money) -> ProviderResponse:
        latency = random.randint(80, 300)

        if self._status == ProviderStatus.DOWN:
            return ProviderResponse(
                success=False,
                provider_transaction_id="",
                error_message="PayPal unavailable for refund",
                error_code="PROVIDER_DOWN",
                latency_ms=latency,
            )

        refund_id = f"REF-{uuid.uuid4().hex[:16].upper()}"
        return ProviderResponse(
            success=True,
            provider_transaction_id=refund_id,
            latency_ms=latency,
        )

    def provider_id(self) -> str:
        return "paypal"

    def supported_currencies(self) -> list[Currency]:
        return [Currency.USD, Currency.EUR, Currency.GBP, Currency.JPY]

    def status(self) -> ProviderStatus:
        return self._status

    def set_status(self, status: ProviderStatus) -> None:
        self._status = status

    def display_name(self) -> str:
        return "PayPal"


class BankTransferProvider(PaymentProvider):
    """
    Simulates direct bank transfer processing.

    Lower failure rate but higher latency compared to card-based
    providers. Supports a broader currency set.
    """

    def __init__(self, failure_rate: float = 0.05) -> None:
        self._failure_rate = failure_rate
        self._status = ProviderStatus.HEALTHY
        self._total_processed = 0
        self._total_failed = 0

    def charge(self, transaction: Transaction) -> ProviderResponse:
        latency = random.randint(200, 800)

        if self._status == ProviderStatus.DOWN:
            self._total_failed += 1
            return ProviderResponse(
                success=False,
                provider_transaction_id="",
                error_message="Bank transfer service down",
                error_code="PROVIDER_DOWN",
                latency_ms=latency,
            )

        if random.random() < self._failure_rate:
            self._total_failed += 1
            return ProviderResponse(
                success=False,
                provider_transaction_id="",
                error_message="Insufficient funds in bank account",
                error_code="INSUFFICIENT_FUNDS",
                latency_ms=latency,
            )

        self._total_processed += 1
        provider_txn_id = f"BT-{uuid.uuid4().hex[:12].upper()}"
        return ProviderResponse(
            success=True,
            provider_transaction_id=provider_txn_id,
            latency_ms=latency,
        )

    def refund(self, transaction: Transaction,
               amount: Money) -> ProviderResponse:
        latency = random.randint(150, 600)

        if self._status == ProviderStatus.DOWN:
            return ProviderResponse(
                success=False,
                provider_transaction_id="",
                error_message="Bank transfer refund unavailable",
                error_code="PROVIDER_DOWN",
                latency_ms=latency,
            )

        refund_id = f"BTR-{uuid.uuid4().hex[:12].upper()}"
        return ProviderResponse(
            success=True,
            provider_transaction_id=refund_id,
            latency_ms=latency,
        )

    def provider_id(self) -> str:
        return "bank_transfer"

    def supported_currencies(self) -> list[Currency]:
        return [Currency.USD, Currency.EUR, Currency.GBP,
                Currency.JPY, Currency.NPR]

    def status(self) -> ProviderStatus:
        return self._status

    def set_status(self, status: ProviderStatus) -> None:
        self._status = status

    def display_name(self) -> str:
        return "Bank Transfer"
```

`PaymentProvider` defines the contract every provider must implement — `charge`, `refund`, currency support, and health status. Each concrete provider simulates real behavior: `StripeProvider` has low latency (50–200ms) and 10% failure rate, `PayPalProvider` has higher latency (100–400ms) and 15% failure rate, `BankTransferProvider` has the highest latency (200–800ms) but lowest failure rate (5%). The `ProviderResponse` dataclass standardizes the return value across providers. In production, these classes would wrap actual HTTP clients; the strategy pattern means the gateway never knows which provider it's talking to — it only calls `charge` and `refund` through the abstract interface.

---

## Retry Policy (Strategy Pattern)

```python
class RetryPolicy(ABC):
    """
    Defines how long to wait between retry attempts.

    Each policy computes a delay based on the attempt number.
    The gateway calls delay_ms before each retry.
    """

    @abstractmethod
    def delay_ms(self, attempt: int) -> int:
        ...

    @abstractmethod
    def name(self) -> str:
        ...


class FixedRetryPolicy(RetryPolicy):
    """Wait a fixed duration between every retry attempt."""

    def __init__(self, delay: int = 1000) -> None:
        self._delay = delay

    def delay_ms(self, attempt: int) -> int:
        return self._delay

    def name(self) -> str:
        return f"FixedRetry({self._delay}ms)"


class ExponentialBackoffPolicy(RetryPolicy):
    """
    Exponential backoff with jitter.

    Delay doubles with each attempt and adds random jitter to
    prevent thundering herd when many clients retry simultaneously.
    """

    def __init__(self, base_ms: int = 500,
                 max_ms: int = 30_000) -> None:
        self._base_ms = base_ms
        self._max_ms = max_ms

    def delay_ms(self, attempt: int) -> int:
        delay = self._base_ms * (2 ** attempt)
        jitter = random.randint(0, self._base_ms)
        return min(delay + jitter, self._max_ms)

    def name(self) -> str:
        return f"ExponentialBackoff(base={self._base_ms}ms)"


class RetryExecutor:
    """
    Executes a payment with retries according to the configured policy.

    Calls the provider's charge method up to max_attempts times,
    waiting according to the retry policy between failures.
    """

    def __init__(self, policy: RetryPolicy,
                 max_attempts: int = MAX_RETRY_ATTEMPTS) -> None:
        self._policy = policy
        self._max_attempts = max_attempts

    def execute(self, provider: PaymentProvider,
                transaction: Transaction) -> ProviderResponse:
        last_response: ProviderResponse | None = None

        for attempt in range(self._max_attempts):
            transaction.attempts = attempt + 1
            response = provider.charge(transaction)
            last_response = response

            if response.success:
                return response

            # Don't retry on non-retryable errors
            if response.error_code in ("CARD_DECLINED",
                                        "INSUFFICIENT_FUNDS"):
                return response

            # Wait before retrying (simulated — actual sleep omitted)
            delay = self._policy.delay_ms(attempt)
            print(f"    ⏳ Retry {attempt + 1}/{self._max_attempts} "
                  f"in {delay}ms ({self._policy.name()})")

        return last_response  # type: ignore
```

`RetryPolicy` defines how long to wait between attempts. `FixedRetryPolicy` waits the same duration every time — simple and predictable. `ExponentialBackoffPolicy` doubles the wait each attempt and adds random jitter. The jitter prevents the "thundering herd" problem where thousands of clients all retry at exactly the same time after a provider recovers. `RetryExecutor` wraps the retry loop — it calls the provider up to `max_attempts` times, but stops early for non-retryable errors like `CARD_DECLINED` (retrying a declined card will always fail). The delay is computed but not actually slept in the simulation; in production you would call `time.sleep(delay / 1000)`.

---

## Fraud Detection (Chain of Responsibility)

```python
@dataclass
class FraudCheckResult:
    """Result of a single fraud check in the pipeline."""
    passed: bool
    rule_name: str
    risk_score: float = 0.0
    reason: str = ""


class FraudRule(ABC):
    """
    A single link in the fraud detection chain.

    Each rule checks one aspect of the transaction. If it fails,
    the chain stops and the transaction is rejected. If it passes,
    the next rule in the chain is invoked.
    """

    def __init__(self) -> None:
        self._next: FraudRule | None = None

    def set_next(self, rule: "FraudRule") -> "FraudRule":
        self._next = rule
        return rule

    def handle(self, transaction: Transaction) -> FraudCheckResult:
        result = self.check(transaction)
        if not result.passed:
            return result
        if self._next:
            return self._next.handle(transaction)
        return result

    @abstractmethod
    def check(self, transaction: Transaction) -> FraudCheckResult:
        ...


class AmountLimitRule(FraudRule):
    """Reject transactions that exceed the maximum allowed amount."""

    def check(self, transaction: Transaction) -> FraudCheckResult:
        if transaction.amount.amount > MAX_TRANSACTION_AMOUNT:
            return FraudCheckResult(
                passed=False,
                rule_name="AmountLimit",
                risk_score=1.0,
                reason=f"Amount {transaction.amount} exceeds "
                       f"maximum ${MAX_TRANSACTION_AMOUNT / 100:,.2f}",
            )
        if transaction.amount.amount < MIN_TRANSACTION_AMOUNT:
            return FraudCheckResult(
                passed=False,
                rule_name="AmountLimit",
                risk_score=0.8,
                reason=f"Amount {transaction.amount} below "
                       f"minimum ${MIN_TRANSACTION_AMOUNT / 100:.2f}",
            )
        return FraudCheckResult(
            passed=True,
            rule_name="AmountLimit",
            risk_score=0.0,
        )


class CardExpiryRule(FraudRule):
    """Reject transactions with expired cards."""

    def check(self, transaction: Transaction) -> FraudCheckResult:
        card = transaction.payment_method.card_details
        if card and card.is_expired():
            return FraudCheckResult(
                passed=False,
                rule_name="CardExpiry",
                risk_score=1.0,
                reason=f"Card {card.masked_display()} has expired "
                       f"({card.exp_month}/{card.exp_year})",
            )
        return FraudCheckResult(
            passed=True,
            rule_name="CardExpiry",
            risk_score=0.0,
        )


class VelocityRule(FraudRule):
    """
    Reject transactions from merchants exceeding a rate limit.

    Tracks the number of transactions per merchant within a
    sliding time window. If the count exceeds the threshold,
    the transaction is flagged as suspicious.
    """

    def __init__(self, max_per_minute: int = 10) -> None:
        super().__init__()
        self._max_per_minute = max_per_minute
        self._merchant_timestamps: dict[str, list[int]] = {}
        self._lock = threading.RLock()

    def check(self, transaction: Transaction) -> FraudCheckResult:
        now = int(time.time() * 1000)
        window_ms = 60_000  # 1 minute

        with self._lock:
            timestamps = self._merchant_timestamps.setdefault(
                transaction.merchant_id, []
            )
            # Remove expired timestamps
            timestamps[:] = [t for t in timestamps
                             if now - t < window_ms]
            timestamps.append(now)

            if len(timestamps) > self._max_per_minute:
                return FraudCheckResult(
                    passed=False,
                    rule_name="Velocity",
                    risk_score=0.9,
                    reason=f"Merchant {transaction.merchant_id} "
                           f"exceeded {self._max_per_minute} "
                           f"txns/min",
                )

        return FraudCheckResult(
            passed=True,
            rule_name="Velocity",
            risk_score=0.0,
        )


class SuspiciousAmountRule(FraudRule):
    """
    Flag transactions with round-number amounts above a threshold.

    Large round-number transactions (e.g., exactly $5,000.00) are
    statistically unusual for legitimate purchases and may indicate
    money laundering or testing stolen cards.
    """

    def __init__(self, threshold_cents: int = 500_000) -> None:
        super().__init__()
        self._threshold = threshold_cents

    def check(self, transaction: Transaction) -> FraudCheckResult:
        amount = transaction.amount.amount
        if (amount >= self._threshold
                and amount % 10_000 == 0):  # divisible by $100
            return FraudCheckResult(
                passed=False,
                rule_name="SuspiciousAmount",
                risk_score=FRAUD_SCORE_THRESHOLD,
                reason=f"Round amount {transaction.amount} above "
                       f"${self._threshold / 100:,.2f} is suspicious",
            )
        return FraudCheckResult(
            passed=True,
            rule_name="SuspiciousAmount",
            risk_score=0.0,
        )


def build_fraud_chain() -> FraudRule:
    """
    Build the default fraud detection chain.

    Order matters: cheapest checks first (amount limits, card expiry)
    before expensive checks (velocity, pattern analysis).
    """
    amount_rule = AmountLimitRule()
    card_rule = CardExpiryRule()
    velocity_rule = VelocityRule()
    suspicious_rule = SuspiciousAmountRule()

    amount_rule.set_next(card_rule)
    card_rule.set_next(velocity_rule)
    velocity_rule.set_next(suspicious_rule)

    return amount_rule
```

`FraudRule` implements the Chain of Responsibility pattern — each rule checks one aspect and either rejects the transaction or passes it to the next rule. The chain is ordered from cheapest to most expensive: `AmountLimitRule` (pure arithmetic) runs first, then `CardExpiryRule` (date comparison), then `VelocityRule` (sliding window lookup), then `SuspiciousAmountRule` (pattern matching). This ordering avoids expensive checks on obviously invalid transactions. `VelocityRule` uses a sliding window — it keeps timestamps per merchant and removes those older than 1 minute. `SuspiciousAmountRule` catches a real-world fraud pattern: round, large amounts are disproportionately associated with card testing. `build_fraud_chain` wires the chain together; adding a new rule requires only creating a class and inserting it into the chain.

---

## Transaction State Machine (State Pattern)

```python
class TransactionState(ABC):
    """
    Abstract base for transaction states.

    Each state defines which transitions are legal and what
    happens during each transition. Invalid transitions raise.
    """

    @abstractmethod
    def process(self, context: "TransactionStateMachine") -> None:
        ...

    @abstractmethod
    def complete(self, context: "TransactionStateMachine",
                 provider_txn_id: str) -> None:
        ...

    @abstractmethod
    def fail(self, context: "TransactionStateMachine",
             reason: str) -> None:
        ...

    @abstractmethod
    def refund(self, context: "TransactionStateMachine") -> None:
        ...

    @abstractmethod
    def state_name(self) -> str:
        ...


class CreatedState(TransactionState):
    """Initial state — transaction has been created but not processed."""

    def process(self, context: "TransactionStateMachine") -> None:
        context.transaction.status = TransactionStatus.PROCESSING
        context.transaction.updated_at = int(time.time() * 1000)
        context.set_state(ProcessingState())

    def complete(self, context: "TransactionStateMachine",
                 provider_txn_id: str) -> None:
        raise InvalidTransitionError(
            "Cannot complete a transaction that hasn't been processed"
        )

    def fail(self, context: "TransactionStateMachine",
             reason: str) -> None:
        context.transaction.status = TransactionStatus.FAILED
        context.transaction.failure_reason = reason
        context.transaction.updated_at = int(time.time() * 1000)
        context.set_state(FailedState())

    def refund(self, context: "TransactionStateMachine") -> None:
        raise InvalidTransitionError(
            "Cannot refund a transaction that hasn't been completed"
        )

    def state_name(self) -> str:
        return "CREATED"


class ProcessingState(TransactionState):
    """Transaction has been sent to a payment provider."""

    def process(self, context: "TransactionStateMachine") -> None:
        raise InvalidTransitionError(
            "Transaction is already being processed"
        )

    def complete(self, context: "TransactionStateMachine",
                 provider_txn_id: str) -> None:
        context.transaction.status = TransactionStatus.COMPLETED
        context.transaction.provider_transaction_id = provider_txn_id
        context.transaction.updated_at = int(time.time() * 1000)
        context.set_state(CompletedState())

    def fail(self, context: "TransactionStateMachine",
             reason: str) -> None:
        context.transaction.status = TransactionStatus.FAILED
        context.transaction.failure_reason = reason
        context.transaction.updated_at = int(time.time() * 1000)
        context.set_state(FailedState())

    def refund(self, context: "TransactionStateMachine") -> None:
        raise InvalidTransitionError(
            "Cannot refund a transaction that is still processing"
        )

    def state_name(self) -> str:
        return "PROCESSING"


class CompletedState(TransactionState):
    """Transaction was successfully charged by the provider."""

    def process(self, context: "TransactionStateMachine") -> None:
        raise InvalidTransitionError(
            "Cannot process a completed transaction"
        )

    def complete(self, context: "TransactionStateMachine",
                 provider_txn_id: str) -> None:
        raise InvalidTransitionError(
            "Transaction is already completed"
        )

    def fail(self, context: "TransactionStateMachine",
             reason: str) -> None:
        raise InvalidTransitionError(
            "Cannot fail a completed transaction"
        )

    def refund(self, context: "TransactionStateMachine") -> None:
        context.transaction.status = TransactionStatus.REFUNDED
        context.transaction.updated_at = int(time.time() * 1000)
        context.set_state(RefundedState())

    def state_name(self) -> str:
        return "COMPLETED"


class FailedState(TransactionState):
    """Transaction was rejected by the provider or fraud checks."""

    def process(self, context: "TransactionStateMachine") -> None:
        raise InvalidTransitionError(
            "Cannot reprocess a failed transaction"
        )

    def complete(self, context: "TransactionStateMachine",
                 provider_txn_id: str) -> None:
        raise InvalidTransitionError(
            "Cannot complete a failed transaction"
        )

    def fail(self, context: "TransactionStateMachine",
             reason: str) -> None:
        raise InvalidTransitionError(
            "Transaction has already failed"
        )

    def refund(self, context: "TransactionStateMachine") -> None:
        raise InvalidTransitionError(
            "Cannot refund a failed transaction"
        )

    def state_name(self) -> str:
        return "FAILED"


class RefundedState(TransactionState):
    """Transaction has been fully refunded."""

    def process(self, context: "TransactionStateMachine") -> None:
        raise InvalidTransitionError(
            "Cannot process a refunded transaction"
        )

    def complete(self, context: "TransactionStateMachine",
                 provider_txn_id: str) -> None:
        raise InvalidTransitionError(
            "Cannot complete a refunded transaction"
        )

    def fail(self, context: "TransactionStateMachine",
             reason: str) -> None:
        raise InvalidTransitionError(
            "Cannot fail a refunded transaction"
        )

    def refund(self, context: "TransactionStateMachine") -> None:
        raise InvalidTransitionError(
            "Transaction is already refunded"
        )

    def state_name(self) -> str:
        return "REFUNDED"


class InvalidTransitionError(Exception):
    """Raised when an illegal state transition is attempted."""
    pass


class TransactionStateMachine:
    """
    Manages a transaction's state transitions.

    Delegates all operations to the current state object, which
    decides whether the transition is valid and performs it.
    """

    def __init__(self, transaction: Transaction) -> None:
        self._transaction = transaction
        self._state: TransactionState = CreatedState()
        self._history: list[tuple[str, str, int]] = [
            ("INIT", "CREATED", transaction.created_at)
        ]

    @property
    def transaction(self) -> Transaction:
        return self._transaction

    @property
    def current_state(self) -> str:
        return self._state.state_name()

    @property
    def history(self) -> list[tuple[str, str, int]]:
        return list(self._history)

    def set_state(self, state: TransactionState) -> None:
        old = self._state.state_name()
        self._state = state
        self._history.append(
            (old, state.state_name(), int(time.time() * 1000))
        )

    def process(self) -> None:
        self._state.process(self)

    def complete(self, provider_txn_id: str) -> None:
        self._state.complete(self, provider_txn_id)

    def fail(self, reason: str) -> None:
        self._state.fail(self, reason)

    def refund(self) -> None:
        self._state.refund(self)
```

The State pattern encapsulates each transaction state as a class. Each state class defines which transitions are legal from that state and raises `InvalidTransitionError` for illegal ones. `CreatedState` can transition to `ProcessingState` (via `process`) or `FailedState` (if fraud checks fail). `ProcessingState` can go to `CompletedState` or `FailedState`. `CompletedState` can only go to `RefundedState`. `FailedState` and `RefundedState` are terminal — no transitions out. `TransactionStateMachine` holds the current state object and delegates all calls to it. The `_history` list records every transition with timestamps for auditability. This design eliminates complex if/else chains and makes adding new states (e.g., `PENDING_REVIEW`, `CHARGEBACK`) a matter of adding a new class.

---

## Refund Manager

```python
@dataclass
class RefundRequest:
    """A request to refund part or all of a transaction."""
    refund_id: str
    transaction_id: str
    amount: Money
    reason: str
    created_at: int = field(
        default_factory=lambda: int(time.time() * 1000)
    )
    status: str = "pending"
    provider_refund_id: str = ""


class RefundManager:
    """
    Handles full and partial refunds.

    Validates that the refund doesn't exceed the original amount
    minus any previous refunds, calls the provider to reverse the
    charge, and records the refund in the ledger.
    """

    def __init__(self, ledger: Ledger) -> None:
        self._ledger = ledger
        self._refunds: dict[str, list[RefundRequest]] = {}
        self._lock = threading.RLock()

    def process_refund(
        self,
        transaction: Transaction,
        amount: Money,
        reason: str,
        provider: PaymentProvider,
        state_machine: TransactionStateMachine,
    ) -> RefundRequest:
        """
        Process a refund for the given transaction.

        Validates: transaction must be COMPLETED or PARTIALLY_REFUNDED,
        refund amount must not exceed remaining refundable amount,
        and provider must successfully reverse the charge.
        """
        if transaction.status not in (
            TransactionStatus.COMPLETED,
            TransactionStatus.PARTIALLY_REFUNDED,
        ):
            raise ValueError(
                f"Cannot refund transaction in "
                f"{transaction.status.value} state"
            )

        # Calculate remaining refundable amount
        refundable = self._remaining_refundable(transaction)
        if amount.amount > refundable:
            raise ValueError(
                f"Refund amount {amount} exceeds remaining "
                f"refundable {Money(refundable, amount.currency)}"
            )

        refund = RefundRequest(
            refund_id=f"ref_{uuid.uuid4().hex[:12]}",
            transaction_id=transaction.transaction_id,
            amount=amount,
            reason=reason,
        )

        # Call provider to reverse
        response = provider.refund(transaction, amount)

        if not response.success:
            refund.status = "failed"
            return refund

        refund.status = "completed"
        refund.provider_refund_id = response.provider_transaction_id

        with self._lock:
            self._refunds.setdefault(
                transaction.transaction_id, []
            ).append(refund)

        # Update transaction
        total_refunded = self._total_refunded(transaction)
        transaction.refunded_amount = Money(
            total_refunded, amount.currency
        )

        if total_refunded >= transaction.amount.amount:
            # Full refund
            state_machine.refund()
        else:
            # Partial refund
            transaction.status = \
                TransactionStatus.PARTIALLY_REFUNDED
            transaction.updated_at = int(time.time() * 1000)

        # Record in ledger
        self._ledger.record_refund(transaction, amount)

        return refund

    def _total_refunded(self, transaction: Transaction) -> int:
        with self._lock:
            refunds = self._refunds.get(
                transaction.transaction_id, []
            )
            return sum(
                r.amount.amount for r in refunds
                if r.status == "completed"
            )

    def _remaining_refundable(self, transaction: Transaction) -> int:
        return (transaction.amount.amount
                - self._total_refunded(transaction))

    def get_refunds(self, transaction_id: str
                    ) -> list[RefundRequest]:
        with self._lock:
            return list(
                self._refunds.get(transaction_id, [])
            )
```

`RefundManager` handles both full and partial refunds. Before processing, it validates two invariants: the transaction must be in a refundable state (`COMPLETED` or `PARTIALLY_REFUNDED`), and the refund amount must not exceed what's left. `_remaining_refundable` calculates the original amount minus all previous completed refunds. If the provider successfully reverses the charge, the refund is recorded in the ledger (debit merchant, credit customer) and the transaction status is updated. If the total refunded equals the original amount, the transaction moves to `REFUNDED` via the state machine. Otherwise it moves to `PARTIALLY_REFUNDED`. This design correctly handles multiple partial refunds — you could refund $10, then $15, then $25 of a $50 transaction.

---

## Webhook & Event System (Observer Pattern)

```python
@dataclass
class PaymentEvent:
    """An event emitted by the payment gateway."""
    event_id: str
    event_type: WebhookEvent
    transaction_id: str
    data: dict
    timestamp: int = field(
        default_factory=lambda: int(time.time() * 1000)
    )


class WebhookListener(ABC):
    """
    Observer that reacts to payment events.

    Implementations receive all events and decide which ones
    to handle. Listeners are decoupled from the gateway — they
    cannot affect payment processing.
    """

    @abstractmethod
    def on_event(self, event: PaymentEvent) -> None:
        ...

    @abstractmethod
    def listener_id(self) -> str:
        ...


class LoggingListener(WebhookListener):
    """Logs all payment events to the console."""

    def on_event(self, event: PaymentEvent) -> None:
        emoji = {
            WebhookEvent.PAYMENT_CREATED: "🆕",
            WebhookEvent.PAYMENT_PROCESSING: "⏳",
            WebhookEvent.PAYMENT_COMPLETED: "✅",
            WebhookEvent.PAYMENT_FAILED: "❌",
            WebhookEvent.REFUND_INITIATED: "🔄",
            WebhookEvent.REFUND_COMPLETED: "💸",
        }
        icon = emoji.get(event.event_type, "📌")
        print(f"  {icon} [{event.event_type.value}] "
              f"txn={event.transaction_id[:12]}... "
              f"{event.data.get('message', '')}")

    def listener_id(self) -> str:
        return "logging-listener"


class MetricsListener(WebhookListener):
    """Collects aggregate metrics from payment events."""

    def __init__(self) -> None:
        self._total_payments = 0
        self._successful_payments = 0
        self._failed_payments = 0
        self._total_volume_cents = 0
        self._total_refunds = 0
        self._total_refund_volume_cents = 0
        self._provider_latencies: dict[str, list[int]] = {}
        self._lock = threading.RLock()

    def on_event(self, event: PaymentEvent) -> None:
        with self._lock:
            if event.event_type == WebhookEvent.PAYMENT_CREATED:
                self._total_payments += 1

            elif event.event_type == WebhookEvent.PAYMENT_COMPLETED:
                self._successful_payments += 1
                self._total_volume_cents += event.data.get(
                    "amount_cents", 0
                )
                provider = event.data.get("provider_id", "unknown")
                latency = event.data.get("latency_ms", 0)
                self._provider_latencies.setdefault(
                    provider, []
                ).append(latency)

            elif event.event_type == WebhookEvent.PAYMENT_FAILED:
                self._failed_payments += 1

            elif event.event_type == WebhookEvent.REFUND_COMPLETED:
                self._total_refunds += 1
                self._total_refund_volume_cents += event.data.get(
                    "refund_amount_cents", 0
                )

    def listener_id(self) -> str:
        return "metrics-listener"

    def display(self) -> None:
        print(f"\n  {'═'*55}")
        print(f"  PAYMENT GATEWAY METRICS")
        print(f"  {'═'*55}")
        print(f"  Total payments:      {self._total_payments}")
        print(f"  Successful:          {self._successful_payments}")
        print(f"  Failed:              {self._failed_payments}")
        success_rate = (
            (self._successful_payments / self._total_payments * 100)
            if self._total_payments > 0 else 0
        )
        print(f"  Success rate:        {success_rate:.1f}%")
        print(f"  Total volume:        "
              f"${self._total_volume_cents / 100:,.2f}")
        print(f"  Total refunds:       {self._total_refunds}")
        print(f"  Refund volume:       "
              f"${self._total_refund_volume_cents / 100:,.2f}")

        if self._provider_latencies:
            print(f"\n  Provider Latencies:")
            for provider, latencies in \
                    self._provider_latencies.items():
                avg = sum(latencies) / len(latencies)
                print(f"    {provider}: avg={avg:.0f}ms "
                      f"min={min(latencies)}ms "
                      f"max={max(latencies)}ms "
                      f"({len(latencies)} calls)")
        print(f"  {'═'*55}")


class EventBus:
    """
    Central event dispatcher.

    Manages listener registration and broadcasts events to all
    registered listeners. Thread-safe for concurrent event emission.
    """

    def __init__(self) -> None:
        self._listeners: list[WebhookListener] = []
        self._event_log: list[PaymentEvent] = []
        self._lock = threading.RLock()

    def register(self, listener: WebhookListener) -> None:
        with self._lock:
            self._listeners.append(listener)

    def unregister(self, listener_id: str) -> None:
        with self._lock:
            self._listeners = [
                l for l in self._listeners
                if l.listener_id() != listener_id
            ]

    def emit(self, event: PaymentEvent) -> None:
        with self._lock:
            self._event_log.append(event)
            for listener in self._listeners:
                try:
                    listener.on_event(event)
                except Exception as e:
                    print(f"  ⚠️ Listener {listener.listener_id()} "
                          f"error: {e}")

    @property
    def total_events(self) -> int:
        return len(self._event_log)
```

`WebhookListener` is the observer interface — implementations receive all events and filter as needed. `LoggingListener` prints every event with emoji-coded log lines. `MetricsListener` aggregates statistics: total volume, success rate, provider latencies, and refund totals. `EventBus` is the subject — it maintains a list of listeners and broadcasts events to all of them. Each listener is isolated — if one throws an exception, others still receive the event. In production, these webhooks would be HTTP POST requests to registered URLs; the observer pattern decouples event emission from event handling, so adding a new listener (e.g., Slack notifications, PagerDuty alerts) requires no changes to the gateway.

---

## Payment Router

```python
class PaymentRouter:
    """
    Routes transactions to the best available payment provider.

    Selection criteria (in order):
    1. Provider must be healthy (not DOWN)
    2. Provider must support the transaction's currency
    3. If multiple providers qualify, prefer the one with lowest
       latency profile (approximated by provider order)
    """

    def __init__(self) -> None:
        self._providers: dict[str, PaymentProvider] = {}
        self._provider_priority: list[str] = []  # ordered by pref
        self._lock = threading.RLock()

    def register_provider(self, provider: PaymentProvider) -> None:
        with self._lock:
            self._providers[provider.provider_id()] = provider
            self._provider_priority.append(provider.provider_id())

    def route(self, transaction: Transaction) -> PaymentProvider:
        """
        Select the best provider for this transaction.

        Iterates through providers in priority order, returning
        the first that is healthy and supports the currency.
        Raises if no provider is available.
        """
        with self._lock:
            for provider_id in self._provider_priority:
                provider = self._providers[provider_id]

                if provider.status() == ProviderStatus.DOWN:
                    continue

                if (transaction.amount.currency
                        not in provider.supported_currencies()):
                    continue

                return provider

        raise NoProviderAvailableError(
            f"No healthy provider supports "
            f"{transaction.amount.currency.value}"
        )

    def get_provider(self, provider_id: str
                     ) -> PaymentProvider | None:
        return self._providers.get(provider_id)

    def all_providers(self) -> list[PaymentProvider]:
        return list(self._providers.values())

    def healthy_providers(self) -> list[PaymentProvider]:
        return [p for p in self._providers.values()
                if p.status() != ProviderStatus.DOWN]


class NoProviderAvailableError(Exception):
    """Raised when no payment provider can handle the transaction."""
    pass
```

`PaymentRouter` decides which provider handles each transaction. Providers are registered with a priority order — the first registered is the preferred provider. The router iterates through providers in priority order and returns the first that is both healthy and supports the transaction's currency. If Stripe is the primary provider but is down, the router automatically falls back to PayPal. If no provider qualifies, `NoProviderAvailableError` is raised. This design supports adding a new provider by simply registering it — no routing logic changes needed.

---

## Payment Gateway (Facade)

```python
@dataclass
class PaymentRequest:
    """
    Client-facing payment request.

    Contains everything needed to initiate a payment. The
    idempotency_key is mandatory — clients must generate a
    unique key per logical payment attempt.
    """
    idempotency_key: str
    amount: Money
    payment_method: PaymentMethod
    merchant_id: str
    description: str = ""
    metadata: dict = field(default_factory=dict)


@dataclass
class PaymentResult:
    """Result returned to the client after processing."""
    success: bool
    transaction: Transaction
    message: str = ""


class PaymentGateway:
    """
    Facade that orchestrates the entire payment lifecycle.

    Coordinates: idempotency checking → fraud detection → provider
    routing → retry execution → ledger recording → event emission.
    Exposes a simple API that hides all internal complexity.
    """

    def __init__(
        self,
        retry_policy: RetryPolicy | None = None,
        fraud_chain: FraudRule | None = None,
    ) -> None:
        self._router = PaymentRouter()
        self._ledger = Ledger()
        self._idempotency = IdempotencyStore()
        self._event_bus = EventBus()
        self._refund_manager = RefundManager(self._ledger)
        self._retry_executor = RetryExecutor(
            retry_policy or ExponentialBackoffPolicy()
        )
        self._fraud_chain = fraud_chain or build_fraud_chain()
        self._transactions: dict[str, TransactionStateMachine] = {}
        self._lock = threading.RLock()

        # Register default listeners
        self._logger = LoggingListener()
        self._metrics = MetricsListener()
        self._event_bus.register(self._logger)
        self._event_bus.register(self._metrics)

    # ------------------------------------------------------------------
    # Provider management
    # ------------------------------------------------------------------

    def register_provider(self, provider: PaymentProvider) -> None:
        self._router.register_provider(provider)
        print(f"  🏦 Provider registered: "
              f"{provider.display_name()} "
              f"(currencies: "
              f"{', '.join(c.value for c in provider.supported_currencies())})")

    # ------------------------------------------------------------------
    # Payment processing
    # ------------------------------------------------------------------

    def process_payment(self, request: PaymentRequest
                        ) -> PaymentResult:
        """
        Process a payment request end-to-end.

        Steps:
        1. Check idempotency — return cached result if duplicate
        2. Create transaction and state machine
        3. Emit PAYMENT_CREATED event
        4. Run fraud detection chain
        5. Route to provider
        6. Execute charge with retries
        7. Update state machine (complete or fail)
        8. Record in ledger (on success)
        9. Cache in idempotency store
        10. Emit PAYMENT_COMPLETED or PAYMENT_FAILED event
        """
        # Step 1: Idempotency check
        cached = self._idempotency.get(request.idempotency_key)
        if cached:
            print(f"  ♻️  Idempotent hit: returning cached result "
                  f"for key={request.idempotency_key[:12]}...")
            return PaymentResult(
                success=cached.status == TransactionStatus.COMPLETED,
                transaction=cached,
                message="Duplicate request — returning cached result",
            )

        # Step 2: Create transaction
        transaction = Transaction(
            transaction_id=str(uuid.uuid4()),
            idempotency_key=request.idempotency_key,
            amount=request.amount,
            payment_method=request.payment_method,
            merchant_id=request.merchant_id,
            description=request.description,
            metadata=request.metadata,
        )
        state_machine = TransactionStateMachine(transaction)

        with self._lock:
            self._transactions[
                transaction.transaction_id
            ] = state_machine

        # Step 3: Emit created event
        self._emit(WebhookEvent.PAYMENT_CREATED, transaction, {
            "message": f"Payment {transaction.amount} "
                       f"for merchant {transaction.merchant_id}",
            "amount_cents": transaction.amount.amount,
        })

        # Step 4: Fraud detection
        fraud_result = self._fraud_chain.handle(transaction)
        if not fraud_result.passed:
            state_machine.fail(
                f"Fraud check failed: {fraud_result.reason}"
            )
            self._idempotency.put(
                request.idempotency_key, transaction
            )
            self._emit(WebhookEvent.PAYMENT_FAILED, transaction, {
                "message": f"Fraud: {fraud_result.reason}",
                "rule": fraud_result.rule_name,
                "risk_score": fraud_result.risk_score,
            })
            return PaymentResult(
                success=False,
                transaction=transaction,
                message=f"Fraud check failed: {fraud_result.reason}",
            )

        # Step 5: Route to provider
        try:
            provider = self._router.route(transaction)
            transaction.provider_id = provider.provider_id()
        except NoProviderAvailableError as e:
            state_machine.fail(str(e))
            self._idempotency.put(
                request.idempotency_key, transaction
            )
            self._emit(WebhookEvent.PAYMENT_FAILED, transaction, {
                "message": str(e),
            })
            return PaymentResult(
                success=False,
                transaction=transaction,
                message=str(e),
            )

        # Step 6: Process with retries
        state_machine.process()
        self._emit(WebhookEvent.PAYMENT_PROCESSING, transaction, {
            "message": f"Sent to {provider.display_name()}",
            "provider_id": provider.provider_id(),
        })

        response = self._retry_executor.execute(
            provider, transaction
        )

        # Step 7-10: Complete or fail
        if response.success:
            state_machine.complete(
                response.provider_transaction_id
            )
            # Step 8: Record in ledger
            self._ledger.record_payment(transaction)
            # Step 9: Cache
            self._idempotency.put(
                request.idempotency_key, transaction
            )
            # Step 10: Emit success
            self._emit(
                WebhookEvent.PAYMENT_COMPLETED, transaction, {
                    "message": f"Charged {transaction.amount} "
                               f"via {provider.display_name()}",
                    "amount_cents": transaction.amount.amount,
                    "provider_id": provider.provider_id(),
                    "provider_txn_id":
                        response.provider_transaction_id,
                    "latency_ms": response.latency_ms,
                    "attempts": transaction.attempts,
                })
            return PaymentResult(
                success=True,
                transaction=transaction,
                message=f"Payment completed via "
                        f"{provider.display_name()}",
            )
        else:
            state_machine.fail(response.error_message)
            self._idempotency.put(
                request.idempotency_key, transaction
            )
            self._emit(WebhookEvent.PAYMENT_FAILED, transaction, {
                "message": f"Failed: {response.error_message}",
                "error_code": response.error_code,
                "attempts": transaction.attempts,
                "provider_id": provider.provider_id(),
            })
            return PaymentResult(
                success=False,
                transaction=transaction,
                message=response.error_message,
            )

    # ------------------------------------------------------------------
    # Refunds
    # ------------------------------------------------------------------

    def refund(self, transaction_id: str, amount: Money,
               reason: str = "Customer request") -> RefundRequest:
        """Process a full or partial refund."""
        with self._lock:
            sm = self._transactions.get(transaction_id)
            if not sm:
                raise ValueError(
                    f"Transaction {transaction_id} not found"
                )

        transaction = sm.transaction
        provider = self._router.get_provider(
            transaction.provider_id
        )
        if not provider:
            raise ValueError(
                f"Provider {transaction.provider_id} not found"
            )

        self._emit(WebhookEvent.REFUND_INITIATED, transaction, {
            "message": f"Refund {amount} — {reason}",
            "refund_amount_cents": amount.amount,
        })

        refund = self._refund_manager.process_refund(
            transaction, amount, reason, provider, sm,
        )

        if refund.status == "completed":
            self._emit(
                WebhookEvent.REFUND_COMPLETED, transaction, {
                    "message": f"Refunded {amount} — {reason}",
                    "refund_amount_cents": amount.amount,
                    "provider_refund_id": refund.provider_refund_id,
                })

        return refund

    # ------------------------------------------------------------------
    # Query methods
    # ------------------------------------------------------------------

    def get_transaction(self, transaction_id: str
                        ) -> Transaction | None:
        sm = self._transactions.get(transaction_id)
        return sm.transaction if sm else None

    def get_transaction_history(self, transaction_id: str
                                ) -> list[tuple[str, str, int]]:
        sm = self._transactions.get(transaction_id)
        return sm.history if sm else []

    @property
    def ledger(self) -> Ledger:
        return self._ledger

    @property
    def metrics(self) -> MetricsListener:
        return self._metrics

    @property
    def event_bus(self) -> EventBus:
        return self._event_bus

    # ------------------------------------------------------------------
    # Internal helpers
    # ------------------------------------------------------------------

    def _emit(self, event_type: WebhookEvent,
              transaction: Transaction,
              data: dict) -> None:
        event = PaymentEvent(
            event_id=str(uuid.uuid4())[:8],
            event_type=event_type,
            transaction_id=transaction.transaction_id,
            data=data,
        )
        self._event_bus.emit(event)
```

`PaymentGateway` is the facade that orchestrates the entire payment lifecycle. The `process_payment` method follows a strict 10-step pipeline: idempotency check → create transaction → emit event → fraud detection → route to provider → execute with retries → update state → record in ledger → cache result → emit completion event. Each step delegates to a specialized component. The gateway never contains business logic itself — it coordinates. `refund` follows a similar pattern: validate → emit event → delegate to `RefundManager` → emit completion. The constructor wires all components together and registers default listeners. Adding new behavior (e.g., audit logging, rate limiting) means registering a new listener or adding a fraud rule — the gateway code doesn't change.

---

## Simulation Engine

```python
class PaymentScenario:
    """
    Tick-based simulation engine for payment gateway demos.

    Each tick represents a logical time step. Actions are registered
    at specific ticks and execute in order when the simulation runs.
    """

    def __init__(self, gateway: PaymentGateway) -> None:
        self._gateway = gateway
        self._events: list[tuple[int, str, callable]] = []

    def at_tick(self, tick: int, description: str,
                action: callable) -> None:
        self._events.append((tick, description, action))

    def run(self, display_interval: int = 5) -> None:
        if not self._events:
            print("  No events to simulate.")
            return

        max_tick = max(e[0] for e in self._events)
        sorted_events = sorted(self._events, key=lambda e: e[0])

        print(f"\n  {'─'*55}")
        print(f"  SIMULATION START (ticks: 1 → {max_tick})")
        print(f"  {'─'*55}")

        current_tick = 0
        for tick, description, action in sorted_events:
            if tick != current_tick:
                current_tick = tick
                print(f"\n  ⏱️  Tick {tick}: {description}")
            else:
                print(f"  ⏱️  Tick {tick}: {description}")

            try:
                action()
            except Exception as e:
                print(f"  ⚠️  Error at tick {tick}: {e}")

        print(f"\n  {'─'*55}")
        print(f"  SIMULATION COMPLETE")
        print(f"  {'─'*55}")
```

`PaymentScenario` mirrors the simulation engine from the Kafka guide. Actions are registered at specific ticks and execute in order. This makes demos reproducible and readable — you can see exactly what happens at each step.

---

## Putting It All Together

```python
def create_test_payment_method() -> PaymentMethod:
    """Create a test credit card payment method."""
    return PaymentMethod(
        method_id=f"pm_{uuid.uuid4().hex[:12]}",
        method_type=PaymentMethodType.CREDIT_CARD,
        card_details=CardDetails(
            last_four="4242",
            brand="visa",
            exp_month=12,
            exp_year=2028,
            token=f"tok_{uuid.uuid4().hex[:16]}",
        ),
        provider_token=f"cus_{uuid.uuid4().hex[:12]}",
    )


def create_bank_payment_method() -> PaymentMethod:
    """Create a test bank transfer payment method."""
    return PaymentMethod(
        method_id=f"pm_{uuid.uuid4().hex[:12]}",
        method_type=PaymentMethodType.BANK_TRANSFER,
    )


def main_scenario() -> None:
    """Run the main payment gateway demonstration."""

    print("\n╔══════════════════════════════════════════╗")
    print("║       PAYMENT GATEWAY SYSTEM             ║")
    print("║  Providers: Stripe, PayPal, Bank Transfer ║")
    print("║  Retry: Exponential Backoff               ║")
    print("║  Fraud: 4-rule chain                      ║")
    print("╚══════════════════════════════════════════╝")

    # --- Setup ---
    random.seed(42)  # deterministic for demo

    gateway = PaymentGateway(
        retry_policy=ExponentialBackoffPolicy(base_ms=500),
    )

    scenario = PaymentScenario(gateway)
    card = create_test_payment_method()
    bank = create_bank_payment_method()

    # Register providers
    scenario.at_tick(
        1, "Register payment providers",
        lambda: (
            gateway.register_provider(StripeProvider(failure_rate=0.0)),
            gateway.register_provider(PayPalProvider(failure_rate=0.0)),
            gateway.register_provider(BankTransferProvider(failure_rate=0.0)),
        ),
    )

    # --- Successful payments ---
    results: list[PaymentResult] = []

    scenario.at_tick(
        2, "Process $49.99 card payment",
        lambda: results.append(gateway.process_payment(PaymentRequest(
            idempotency_key=f"idem-{uuid.uuid4().hex[:8]}",
            amount=Money(4999, Currency.USD),
            payment_method=card,
            merchant_id="merchant-001",
            description="Premium Widget",
        ))),
    )

    scenario.at_tick(
        3, "Process $150.00 card payment",
        lambda: results.append(gateway.process_payment(PaymentRequest(
            idempotency_key=f"idem-{uuid.uuid4().hex[:8]}",
            amount=Money(15000, Currency.USD),
            payment_method=card,
            merchant_id="merchant-001",
            description="Deluxe Bundle",
        ))),
    )

    scenario.at_tick(
        4, "Process €75.00 card payment",
        lambda: results.append(gateway.process_payment(PaymentRequest(
            idempotency_key=f"idem-{uuid.uuid4().hex[:8]}",
            amount=Money(7500, Currency.EUR),
            payment_method=card,
            merchant_id="merchant-002",
            description="European Order",
        ))),
    )

    scenario.at_tick(
        5, "Process ¥10000 bank transfer",
        lambda: results.append(gateway.process_payment(PaymentRequest(
            idempotency_key=f"idem-{uuid.uuid4().hex[:8]}",
            amount=Money(10000, Currency.JPY),
            payment_method=bank,
            merchant_id="merchant-003",
            description="Japan Order",
        ))),
    )

    # --- Idempotency test ---
    duplicate_key = "idem-duplicate-test-001"

    scenario.at_tick(
        6, "Process $25.00 (first request)",
        lambda: results.append(gateway.process_payment(PaymentRequest(
            idempotency_key=duplicate_key,
            amount=Money(2500, Currency.USD),
            payment_method=card,
            merchant_id="merchant-001",
            description="Idempotency Test",
        ))),
    )

    scenario.at_tick(
        7, "Process $25.00 (duplicate — same idempotency key)",
        lambda: results.append(gateway.process_payment(PaymentRequest(
            idempotency_key=duplicate_key,
            amount=Money(2500, Currency.USD),
            payment_method=card,
            merchant_id="merchant-001",
            description="Idempotency Test",
        ))),
    )

    # --- Fraud detection test ---
    scenario.at_tick(
        8, "Process $2,000,000 (exceeds limit — fraud rejection)",
        lambda: results.append(gateway.process_payment(PaymentRequest(
            idempotency_key=f"idem-{uuid.uuid4().hex[:8]}",
            amount=Money(200_000_000, Currency.USD),
            payment_method=card,
            merchant_id="merchant-001",
            description="Suspicious Large Payment",
        ))),
    )

    # --- Refund test ---
    def refund_first_payment():
        successful = [r for r in results
                      if r.success and r.message != "Duplicate request — returning cached result"]
        if successful:
            txn = successful[0].transaction
            print(f"\n  --- Refunding ${10.00:.2f} from "
                  f"txn {txn.transaction_id[:12]}... ---")
            gateway.refund(
                txn.transaction_id,
                Money(1000, Currency.USD),
                reason="Partial refund — customer request",
            )

    scenario.at_tick(9, "Partial refund $10.00", refund_first_payment)

    # --- Full refund test ---
    def full_refund():
        successful = [r for r in results
                      if r.success and r.message != "Duplicate request — returning cached result"]
        if len(successful) >= 2:
            txn = successful[1].transaction
            print(f"\n  --- Full refund of {txn.amount} from "
                  f"txn {txn.transaction_id[:12]}... ---")
            gateway.refund(
                txn.transaction_id,
                txn.amount,
                reason="Full refund — order cancelled",
            )

    scenario.at_tick(10, "Full refund second payment",
                     full_refund)

    # --- Display results ---
    def display_results():
        gateway.ledger.display()
        gateway.metrics.display()

        # Verify ledger is balanced
        print(f"\n  Ledger balanced: "
              f"{'✅ Yes' if gateway.ledger.is_balanced() else '❌ No'}")
        print(f"  Total ledger entries: "
              f"{gateway.ledger.total_entries}")
        print(f"  Total events emitted: "
              f"{gateway.event_bus.total_events}")

    scenario.at_tick(11, "Display ledger and metrics",
                     display_results)

    scenario.run()


# ------------------------------------------------------------------
# Strategy Comparison
# ------------------------------------------------------------------

def compare_retry_strategies() -> None:
    """Compare retry strategies on the same workload."""

    print("\n" + "=" * 55)
    print("  RETRY STRATEGY COMPARISON")
    print("=" * 55)

    strategies: list[RetryPolicy] = [
        FixedRetryPolicy(delay=1000),
        ExponentialBackoffPolicy(base_ms=500),
    ]

    card = create_test_payment_method()

    for strategy in strategies:
        random.seed(99)  # same seed for fair comparison

        gateway = PaymentGateway(retry_policy=strategy)
        gateway.register_provider(
            StripeProvider(failure_rate=0.5)
        )

        successes = 0
        failures = 0

        for i in range(10):
            result = gateway.process_payment(PaymentRequest(
                idempotency_key=f"cmp-{strategy.name()}-{i}",
                amount=Money(1000 + i * 100, Currency.USD),
                payment_method=card,
                merchant_id="test-merchant",
                description=f"Comparison test {i}",
            ))
            if result.success:
                successes += 1
            else:
                failures += 1

        print(f"\n  Strategy: {strategy.name()}")
        print(f"  {'─'*45}")
        print(f"    Successes: {successes}/10")
        print(f"    Failures:  {failures}/10")
        gateway.metrics.display()

    print("\n" + "=" * 55)


# ------------------------------------------------------------------
# Unit Tests
# ------------------------------------------------------------------

def test_successful_payment() -> None:
    """Test basic successful payment flow."""
    random.seed(42)

    gateway = PaymentGateway()
    gateway.register_provider(StripeProvider(failure_rate=0.0))

    result = gateway.process_payment(PaymentRequest(
        idempotency_key="test-success-001",
        amount=Money(5000, Currency.USD),
        payment_method=create_test_payment_method(),
        merchant_id="test-merchant",
        description="Test payment",
    ))

    assert result.success, "Payment should succeed"
    assert result.transaction.status == TransactionStatus.COMPLETED
    assert result.transaction.provider_id == "stripe"
    assert gateway.ledger.is_balanced()
    print("✅ test_successful_payment passed")


def test_idempotency() -> None:
    """Test that duplicate requests return cached result."""
    random.seed(42)

    gateway = PaymentGateway()
    gateway.register_provider(StripeProvider(failure_rate=0.0))

    request = PaymentRequest(
        idempotency_key="test-idemp-001",
        amount=Money(3000, Currency.USD),
        payment_method=create_test_payment_method(),
        merchant_id="test-merchant",
        description="Idempotency test",
    )

    result1 = gateway.process_payment(request)
    result2 = gateway.process_payment(request)

    assert result1.success
    assert result2.success
    assert (result1.transaction.transaction_id
            == result2.transaction.transaction_id)
    # Ledger should have only 2 entries (one pair), not 4
    assert gateway.ledger.total_entries == 2
    print("✅ test_idempotency passed")


def test_fraud_rejection() -> None:
    """Test that fraud checks block invalid transactions."""
    random.seed(42)

    gateway = PaymentGateway()
    gateway.register_provider(StripeProvider(failure_rate=0.0))

    # Amount exceeds max
    result = gateway.process_payment(PaymentRequest(
        idempotency_key="test-fraud-001",
        amount=Money(200_000_000, Currency.USD),
        payment_method=create_test_payment_method(),
        merchant_id="test-merchant",
        description="Oversized payment",
    ))

    assert not result.success
    assert result.transaction.status == TransactionStatus.FAILED
    assert "exceeds" in result.transaction.failure_reason.lower()
    print("✅ test_fraud_rejection passed")


def test_expired_card_rejection() -> None:
    """Test that expired cards are rejected by fraud chain."""
    random.seed(42)

    gateway = PaymentGateway()
    gateway.register_provider(StripeProvider(failure_rate=0.0))

    expired_card = PaymentMethod(
        method_id="pm-expired",
        method_type=PaymentMethodType.CREDIT_CARD,
        card_details=CardDetails(
            last_four="1234",
            brand="visa",
            exp_month=1,
            exp_year=2020,  # expired
            token="tok_expired",
        ),
    )

    result = gateway.process_payment(PaymentRequest(
        idempotency_key="test-expired-001",
        amount=Money(1000, Currency.USD),
        payment_method=expired_card,
        merchant_id="test-merchant",
        description="Expired card test",
    ))

    assert not result.success
    assert result.transaction.status == TransactionStatus.FAILED
    print("✅ test_expired_card_rejection passed")


def test_provider_fallback() -> None:
    """Test that router falls back to next provider when primary is down."""
    random.seed(42)

    gateway = PaymentGateway()
    stripe = StripeProvider(failure_rate=0.0)
    stripe.set_status(ProviderStatus.DOWN)
    gateway.register_provider(stripe)
    gateway.register_provider(PayPalProvider(failure_rate=0.0))

    result = gateway.process_payment(PaymentRequest(
        idempotency_key="test-fallback-001",
        amount=Money(2000, Currency.USD),
        payment_method=create_test_payment_method(),
        merchant_id="test-merchant",
        description="Fallback test",
    ))

    assert result.success
    assert result.transaction.provider_id == "paypal"
    print("✅ test_provider_fallback passed")


def test_partial_refund() -> None:
    """Test partial refund flow."""
    random.seed(42)

    gateway = PaymentGateway()
    gateway.register_provider(StripeProvider(failure_rate=0.0))

    result = gateway.process_payment(PaymentRequest(
        idempotency_key="test-refund-001",
        amount=Money(10000, Currency.USD),
        payment_method=create_test_payment_method(),
        merchant_id="test-merchant",
        description="Refund test",
    ))

    assert result.success
    txn_id = result.transaction.transaction_id

    # Partial refund
    refund = gateway.refund(txn_id, Money(3000, Currency.USD))
    assert refund.status == "completed"
    assert result.transaction.status == \
        TransactionStatus.PARTIALLY_REFUNDED

    # Second partial refund
    refund2 = gateway.refund(txn_id, Money(7000, Currency.USD))
    assert refund2.status == "completed"
    assert result.transaction.status == TransactionStatus.REFUNDED

    assert gateway.ledger.is_balanced()
    print("✅ test_partial_refund passed")


def test_full_refund() -> None:
    """Test full refund moves transaction to REFUNDED state."""
    random.seed(42)

    gateway = PaymentGateway()
    gateway.register_provider(StripeProvider(failure_rate=0.0))

    result = gateway.process_payment(PaymentRequest(
        idempotency_key="test-full-refund-001",
        amount=Money(5000, Currency.USD),
        payment_method=create_test_payment_method(),
        merchant_id="test-merchant",
        description="Full refund test",
    ))

    assert result.success

    refund = gateway.refund(
        result.transaction.transaction_id,
        Money(5000, Currency.USD),
    )

    assert refund.status == "completed"
    assert result.transaction.status == TransactionStatus.REFUNDED
    assert gateway.ledger.is_balanced()
    # 4 entries: payment DR/CR + refund DR/CR
    assert gateway.ledger.total_entries == 4
    print("✅ test_full_refund passed")


def test_double_refund_rejected() -> None:
    """Test that refunding more than the original amount fails."""
    random.seed(42)

    gateway = PaymentGateway()
    gateway.register_provider(StripeProvider(failure_rate=0.0))

    result = gateway.process_payment(PaymentRequest(
        idempotency_key="test-double-refund-001",
        amount=Money(5000, Currency.USD),
        payment_method=create_test_payment_method(),
        merchant_id="test-merchant",
        description="Double refund test",
    ))

    gateway.refund(
        result.transaction.transaction_id,
        Money(5000, Currency.USD),
    )

    # Trying to refund again should fail
    try:
        gateway.refund(
            result.transaction.transaction_id,
            Money(1000, Currency.USD),
        )
        assert False, "Should have raised ValueError"
    except ValueError:
        pass

    print("✅ test_double_refund_rejected passed")


def test_state_machine_transitions() -> None:
    """Test that invalid state transitions raise errors."""
    txn = Transaction(
        transaction_id="test-sm",
        idempotency_key="test-sm-key",
        amount=Money(1000, Currency.USD),
        payment_method=create_test_payment_method(),
        merchant_id="test-merchant",
    )
    sm = TransactionStateMachine(txn)

    # Cannot complete from CREATED
    try:
        sm.complete("txn-123")
        assert False, "Should have raised"
    except InvalidTransitionError:
        pass

    # Process → Complete is valid
    sm.process()
    assert sm.current_state == "PROCESSING"
    sm.complete("txn-123")
    assert sm.current_state == "COMPLETED"

    # Cannot process a completed transaction
    try:
        sm.process()
        assert False, "Should have raised"
    except InvalidTransitionError:
        pass

    # Refund from completed is valid
    sm.refund()
    assert sm.current_state == "REFUNDED"

    print("✅ test_state_machine_transitions passed")


def test_ledger_balance() -> None:
    """Test that the ledger always balances."""
    random.seed(42)

    gateway = PaymentGateway()
    gateway.register_provider(StripeProvider(failure_rate=0.0))

    # Process several payments
    for i in range(5):
        gateway.process_payment(PaymentRequest(
            idempotency_key=f"test-balance-{i}",
            amount=Money(1000 * (i + 1), Currency.USD),
            payment_method=create_test_payment_method(),
            merchant_id=f"merchant-{i % 2}",
            description=f"Balance test {i}",
        ))

    assert gateway.ledger.is_balanced()
    # 5 payments × 2 entries each = 10
    assert gateway.ledger.total_entries == 10
    print("✅ test_ledger_balance passed")


def test_money_operations() -> None:
    """Test Money value object operations."""
    m1 = Money(1000, Currency.USD)
    m2 = Money(500, Currency.USD)

    assert m1.add(m2).amount == 1500
    assert m1.subtract(m2).amount == 500
    assert m1.display() == "$10.00"

    # Cannot subtract more than available
    try:
        m2.subtract(m1)
        assert False, "Should have raised"
    except ValueError:
        pass

    # Cannot add different currencies
    m3 = Money(500, Currency.EUR)
    try:
        m1.add(m3)
        assert False, "Should have raised"
    except ValueError:
        pass

    print("✅ test_money_operations passed")


def test_no_provider_error() -> None:
    """Test error when no provider supports the currency."""
    random.seed(42)

    gateway = PaymentGateway()
    gateway.register_provider(StripeProvider(failure_rate=0.0))

    # Stripe doesn't support NPR
    result = gateway.process_payment(PaymentRequest(
        idempotency_key="test-no-provider-001",
        amount=Money(5000, Currency.NPR),
        payment_method=create_test_payment_method(),
        merchant_id="test-merchant",
        description="NPR payment",
    ))

    assert not result.success
    assert result.transaction.status == TransactionStatus.FAILED
    print("✅ test_no_provider_error passed")


def run_all_tests() -> None:
    print("\n" + "=" * 40)
    print("  RUNNING TESTS")
    print("=" * 40 + "\n")
    test_successful_payment()
    test_idempotency()
    test_fraud_rejection()
    test_expired_card_rejection()
    test_provider_fallback()
    test_partial_refund()
    test_full_refund()
    test_double_refund_rejected()
    test_state_machine_transitions()
    test_ledger_balance()
    test_money_operations()
    test_no_provider_error()
    print("\n✅ All tests passed!\n")


if __name__ == "__main__":
    main_scenario()
    compare_retry_strategies()
    run_all_tests()
```

Each test seeds the random generator for determinism. `test_successful_payment` verifies the basic end-to-end flow. `test_idempotency` confirms duplicate requests return the cached result and the ledger only records one payment. `test_fraud_rejection` checks that amounts exceeding the limit are blocked before reaching a provider. `test_expired_card_rejection` validates the card expiry fraud rule. `test_provider_fallback` sets Stripe to `DOWN` and confirms PayPal handles the payment. `test_partial_refund` processes two partial refunds that sum to the full amount, verifying the transition from `PARTIALLY_REFUNDED` to `REFUNDED`. `test_double_refund_rejected` ensures you cannot refund more than the original amount. `test_state_machine_transitions` validates that the state machine enforces legal transitions and rejects illegal ones. `test_ledger_balance` processes 5 payments and confirms total debits equal total credits. `test_money_operations` tests the Money value object's arithmetic and currency safety. `test_no_provider_error` confirms that unsupported currencies produce a clean failure.

---

## Class Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                      PaymentGateway                               │
│──────────────────────────────────────────────────────────────────│
│ - _router: PaymentRouter                                          │
│ - _ledger: Ledger                                                 │
│ - _idempotency: IdempotencyStore                                  │
│ - _event_bus: EventBus                                            │
│ - _refund_manager: RefundManager                                  │
│ - _retry_executor: RetryExecutor                                  │
│ - _fraud_chain: FraudRule                                         │
│ - _transactions: dict[str, TransactionStateMachine]               │
│──────────────────────────────────────────────────────────────────│
│ + register_provider(provider) → None                              │
│ + process_payment(request) → PaymentResult                        │
│ + refund(txn_id, amount, reason) → RefundRequest                  │
│ + get_transaction(txn_id) → Transaction                           │
│ + get_transaction_history(txn_id) → list                          │
└───────┬──────────┬──────────┬──────────┬──────────┬──────────────┘
        │          │          │          │          │
   uses ▼     uses ▼     uses ▼    uses ▼     uses ▼
┌──────────────────────────────────────────────────────────────────┐
│  PaymentRouter                                                    │
│──────────────────────────────────────────────────────────────────│
│ - _providers: dict[str, PaymentProvider]                          │
│ - _provider_priority: list[str]                                   │
│──────────────────────────────────────────────────────────────────│
│ + register_provider(provider) → None                              │
│ + route(transaction) → PaymentProvider                            │
│ + healthy_providers() → list[PaymentProvider]                     │
└──────────┬───────────────────────────────────────────────────────┘
           │ selects
           ▼
┌──────────────────────────────────────────────────────────────────┐
│  «abstract» PaymentProvider                                       │
│──────────────────────────────────────────────────────────────────│
│ + charge(transaction) → ProviderResponse                          │
│ + refund(transaction, amount) → ProviderResponse                  │
│ + provider_id() → str                                             │
│ + supported_currencies() → list[Currency]                         │
│ + status() → ProviderStatus                                       │
└──────────┬────────────────────────────────────────────────────────┘
           │ implemented by
  ┌────────┼──────────────────┐
  ▼        ▼                  ▼
Stripe   PayPal            BankTransfer
Provider Provider          Provider

┌──────────────────────────────────────────────────────────────────┐
│  «abstract» RetryPolicy                                           │
│──────────────────────────────────────────────────────────────────│
│ + delay_ms(attempt) → int                                         │
│ + name() → str                                                    │
└──────────┬────────────────────────────────────────────────────────┘
           │ implemented by
  ┌────────┴──────────┐
  ▼                   ▼
Fixed              Exponential
RetryPolicy        BackoffPolicy

┌──────────────────────────────────────────────────────────────────┐
│  «abstract» FraudRule (Chain of Responsibility)                   │
│──────────────────────────────────────────────────────────────────│
│ - _next: FraudRule | None                                         │
│ + set_next(rule) → FraudRule                                      │
│ + handle(transaction) → FraudCheckResult                          │
│ + check(transaction) → FraudCheckResult                           │
└──────────┬────────────────────────────────────────────────────────┘
           │ implemented by
  ┌────────┼──────────────┬──────────────┐
  ▼        ▼              ▼              ▼
Amount   CardExpiry    Velocity      Suspicious
Limit    Rule          Rule          AmountRule
Rule

┌──────────────────────────────────────────────────────────────────┐
│  «abstract» TransactionState                                      │
│──────────────────────────────────────────────────────────────────│
│ + process(context) → None                                         │
│ + complete(context, provider_txn_id) → None                       │
│ + fail(context, reason) → None                                    │
│ + refund(context) → None                                          │
│ + state_name() → str                                              │
└──────────┬────────────────────────────────────────────────────────┘
           │ implemented by
  ┌────────┼──────────┬──────────┬──────────┐
  ▼        ▼          ▼          ▼          ▼
Created  Processing Completed  Failed   Refunded
State    State      State      State    State

┌──────────────────────────────────────────────────────────────────┐
│  «abstract» WebhookListener                                       │
│──────────────────────────────────────────────────────────────────│
│ + on_event(event) → None                                          │
│ + listener_id() → str                                             │
└──────────┬────────────────────────────────────────────────────────┘
           │ implemented by
  ┌────────┴──────────┐
  ▼                   ▼
Logging            Metrics
Listener           Listener

┌──────────────┐ ┌──────────────────┐ ┌──────────────────┐
│ Transaction  │ │ Money (frozen)    │ │ Ledger            │
│──────────────│ │──────────────────│ │──────────────────│
│ + txn_id     │ │ + amount: int     │ │ - _entries: list  │
│ + idemp_key  │ │ + currency        │ │──────────────────│
│ + amount     │ │──────────────────│ │ + record_payment()│
│ + status     │ │ + add() → Money   │ │ + record_refund() │
│ + provider_id│ │ + subtract()      │ │ + get_balance()   │
│ + attempts   │ │ + display()       │ │ + is_balanced()   │
│ + refunded   │ └──────────────────┘ └──────────────────┘
└──────────────┘
                 ┌──────────────────┐ ┌──────────────────┐
┌──────────────┐ │ IdempotencyStore  │ │ EventBus          │
│ PaymentMethod│ │──────────────────│ │──────────────────│
│──────────────│ │ - _store: dict    │ │ - _listeners      │
│ + method_id  │ │ - _ttl_ms: int    │ │ - _event_log      │
│ + method_type│ │──────────────────│ │──────────────────│
│ + card_details│ │ + get(key) → Txn │ │ + register()      │
│ + provider_tk│ │ + put(key, txn)   │ │ + emit(event)     │
└──────────────┘ │ + contains(key)   │ │ + unregister()    │
                 └──────────────────┘ └──────────────────┘
┌──────────────┐
│ CardDetails  │ ┌──────────────────┐ ┌──────────────────┐
│ (frozen)     │ │ RefundManager     │ │ RefundRequest     │
│──────────────│ │──────────────────│ │──────────────────│
│ + last_four  │ │ - _ledger         │ │ + refund_id       │
│ + brand      │ │ - _refunds: dict  │ │ + txn_id          │
│ + exp_month  │ │──────────────────│ │ + amount           │
│ + exp_year   │ │ + process_refund()│ │ + reason           │
│ + token      │ │ + get_refunds()   │ │ + status           │
│ + is_expired │ └──────────────────┘ └──────────────────┘
└──────────────┘

┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│ TransactionStatus│ │ PaymentMethodType │ │ Currency          │
│ (Enum)           │ │ (Enum)            │ │ (Enum)            │
│──────────────────│ │──────────────────│ │──────────────────│
│ CREATED          │ │ CREDIT_CARD       │ │ USD               │
│ PROCESSING       │ │ DEBIT_CARD        │ │ EUR               │
│ COMPLETED        │ │ BANK_TRANSFER     │ │ GBP               │
│ FAILED           │ │ DIGITAL_WALLET    │ │ JPY               │
│ REFUNDED         │ └──────────────────┘ │ NPR               │
│ PARTIALLY_       │                      └──────────────────┘
│   REFUNDED       │ ┌──────────────────┐
└──────────────────┘ │ WebhookEvent      │ ┌──────────────────┐
                     │ (Enum)            │ │ ProviderStatus    │
┌──────────────────┐ │──────────────────│ │ (Enum)            │
│ LedgerEntryType  │ │ PAYMENT_CREATED   │ │──────────────────│
│ (Enum)           │ │ PAYMENT_PROCESSING│ │ HEALTHY           │
│──────────────────│ │ PAYMENT_COMPLETED │ │ DEGRADED          │
│ DEBIT            │ │ PAYMENT_FAILED    │ │ DOWN              │
│ CREDIT           │ │ REFUND_INITIATED  │ └──────────────────┘
└──────────────────┘ │ REFUND_COMPLETED  │
                     └──────────────────┘

┌──────────────────┐ ┌──────────────────┐
│ PaymentRequest   │ │ PaymentResult     │
│──────────────────│ │──────────────────│
│ + idemp_key      │ │ + success: bool   │
│ + amount         │ │ + transaction     │
│ + payment_method │ │ + message         │
│ + merchant_id    │ └──────────────────┘
│ + description    │
└──────────────────┘ ┌──────────────────┐
                     │ ProviderResponse  │
┌──────────────────┐ │──────────────────│
│ LedgerEntry      │ │ + success         │
│ (frozen)         │ │ + provider_txn_id │
│──────────────────│ │ + error_message   │
│ + entry_id       │ │ + error_code      │
│ + txn_id         │ │ + latency_ms      │
│ + entry_type     │ └──────────────────┘
│ + amount         │
│ + account        │ ┌──────────────────┐
│ + description    │ │ FraudCheckResult  │
└──────────────────┘ │──────────────────│
                     │ + passed          │
┌──────────────────┐ │ + rule_name       │
│ PaymentScenario  │ │ + risk_score      │
│──────────────────│ │ + reason          │
│ - _gateway       │ └──────────────────┘
│ - _events        │
│──────────────────│
│ + at_tick()      │
│ + run()          │
└──────────────────┘
```

---

## How to Run

Save all code into a single file `payment_gateway.py` and run:

```bash
python payment_gateway.py
```

### Example Output

```
╔══════════════════════════════════════════╗
║       PAYMENT GATEWAY SYSTEM             ║
║  Providers: Stripe, PayPal, Bank Transfer ║
║  Retry: Exponential Backoff               ║
║  Fraud: 4-rule chain                      ║
╚══════════════════════════════════════════╝

  ───────────────────────────────────────────────────────
  SIMULATION START (ticks: 1 → 11)
  ───────────────────────────────────────────────────────

  ⏱️  Tick 1: Register payment providers
  🏦 Provider registered: Stripe (currencies: USD, EUR, GBP)
  🏦 Provider registered: PayPal (currencies: USD, EUR, GBP, JPY)
  🏦 Provider registered: Bank Transfer (currencies: USD, EUR, GBP, JPY, NPR)

  ⏱️  Tick 2: Process $49.99 card payment
  🆕 [payment.created] txn=a1b2c3d4e5f6... Payment $49.99 for merchant merchant-001
  ⏳ [payment.processing] txn=a1b2c3d4e5f6... Sent to Stripe
  ✅ [payment.completed] txn=a1b2c3d4e5f6... Charged $49.99 via Stripe

  ⏱️  Tick 3: Process $150.00 card payment
  🆕 [payment.created] txn=b2c3d4e5f6a1... Payment $150.00 for merchant merchant-001
  ⏳ [payment.processing] txn=b2c3d4e5f6a1... Sent to Stripe
  ✅ [payment.completed] txn=b2c3d4e5f6a1... Charged $150.00 via Stripe

  ⏱️  Tick 4: Process €75.00 card payment
  🆕 [payment.created] txn=c3d4e5f6a1b2... Payment €75.00 for merchant merchant-002
  ⏳ [payment.processing] txn=c3d4e5f6a1b2... Sent to Stripe
  ✅ [payment.completed] txn=c3d4e5f6a1b2... Charged €75.00 via Stripe

  ⏱️  Tick 5: Process ¥10000 bank transfer
  🆕 [payment.created] txn=d4e5f6a1b2c3... Payment ¥10000 for merchant merchant-003
  ⏳ [payment.processing] txn=d4e5f6a1b2c3... Sent to PayPal
  ✅ [payment.completed] txn=d4e5f6a1b2c3... Charged ¥10000 via PayPal

  ⏱️  Tick 6: Process $25.00 (first request)
  🆕 [payment.created] txn=e5f6a1b2c3d4... Payment $25.00 for merchant merchant-001
  ⏳ [payment.processing] txn=e5f6a1b2c3d4... Sent to Stripe
  ✅ [payment.completed] txn=e5f6a1b2c3d4... Charged $25.00 via Stripe

  ⏱️  Tick 7: Process $25.00 (duplicate — same idempotency key)
  ♻️  Idempotent hit: returning cached result for key=idem-duplic...

  ⏱️  Tick 8: Process $2,000,000 (exceeds limit — fraud rejection)
  🆕 [payment.created] txn=f6a1b2c3d4e5... Payment $2,000,000.00 for merchant merchant-001
  ❌ [payment.failed] txn=f6a1b2c3d4e5... Fraud: Amount exceeds maximum

  ⏱️  Tick 9: Partial refund $10.00

  --- Refunding $10.00 from txn a1b2c3d4e5f6... ---
  🔄 [refund.initiated] txn=a1b2c3d4e5f6... Refund $10.00 — Partial refund
  💸 [refund.completed] txn=a1b2c3d4e5f6... Refunded $10.00

  ⏱️  Tick 10: Full refund second payment

  --- Full refund of $150.00 from txn b2c3d4e5f6a1... ---
  🔄 [refund.initiated] txn=b2c3d4e5f6a1... Refund $150.00 — Full refund
  💸 [refund.completed] txn=b2c3d4e5f6a1... Refunded $150.00

  ⏱️  Tick 11: Display ledger and metrics

  ────────────────────────────────────────────────────────────
                            LEDGER
  ────────────────────────────────────────────────────────────
  Type   Amount       Account                      Description
  ────────────────────────────────────────────────────────────
  DR     $49.99       customer:idem-a1b2            Payment for Premium Widget
  CR     $49.99       merchant:merchant-001         Payment received for Premium
  DR     $150.00      customer:idem-b2c3            Payment for Deluxe Bundle
  CR     $150.00      merchant:merchant-001         Payment received for Deluxe
  DR     €75.00       customer:idem-c3d4            Payment for European Order
  CR     €75.00       merchant:merchant-002         Payment received for European
  DR     ¥10000       customer:idem-d4e5            Payment for Japan Order
  CR     ¥10000       merchant:merchant-003         Payment received for Japan
  DR     $25.00       customer:idem-e5f6            Payment for Idempotency Test
  CR     $25.00       merchant:merchant-001         Payment received for Idempot
  DR     $10.00       merchant:merchant-001         Refund for txn a1b2c3d4
  CR     $10.00       customer:idem-a1b2            Refund received for txn a1b2
  DR     $150.00      merchant:merchant-001         Refund for txn b2c3d4e5
  CR     $150.00      customer:idem-b2c3            Refund received for txn b2c3
  ────────────────────────────────────────────────────────────
  ✅ BALANCED

  ═══════════════════════════════════════════════════════
  PAYMENT GATEWAY METRICS
  ═══════════════════════════════════════════════════════
  Total payments:      6
  Successful:          5
  Failed:              1
  Success rate:        83.3%
  Total volume:        $324.99
  Total refunds:       2
  Refund volume:       $160.00

  Provider Latencies:
    stripe: avg=125ms min=52ms max=198ms (4 calls)
    paypal: avg=237ms min=237ms max=237ms (1 calls)
  ═══════════════════════════════════════════════════════

  Ledger balanced: ✅ Yes
  Total ledger entries: 14
  Total events emitted: 20

  ───────────────────────────────────────────────────────
  SIMULATION COMPLETE
  ───────────────────────────────────────────────────────
```
