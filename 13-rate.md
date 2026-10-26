# Design Rate Limiter – Low-Level Design Guide

![Bidur Sapkota](https://www.bidursapkota.com.np/images/gravatar.webp "Bidur Sapkota - Developer")&nbsp;[Bidur Sapkota](https://www.bidursapkota.com.np/)

## Table of Contents

1. [Requirements](#requirements)
2. [Key Design Patterns](#key-design-patterns)
3. [Core Enums & Constants](#core-enums--constants)
4. [Rate Limit Result](#rate-limit-result)
5. [Rate Limiting Algorithm (Strategy Pattern)](#rate-limiting-algorithm-strategy-pattern)
6. [Token Bucket Algorithm](#token-bucket-algorithm)
7. [Sliding Window Log Algorithm](#sliding-window-log-algorithm)
8. [Sliding Window Counter Algorithm](#sliding-window-counter-algorithm)
9. [Fixed Window Counter Algorithm](#fixed-window-counter-algorithm)
10. [Rate Limiter Core](#rate-limiter-core)
11. [Rate Limit Event System (Observer Pattern)](#rate-limit-event-system-observer-pattern)
12. [Per-Client Rate Limiter (Decorator Pattern)](#per-client-rate-limiter-decorator-pattern)
13. [Rate Limit Rule Engine](#rate-limit-rule-engine)
14. [Distributed Rate Limiter](#distributed-rate-limiter)
15. [Rate Limiter Middleware](#rate-limiter-middleware)
16. [Rate Limiter Facade](#rate-limiter-facade)
17. [Simulation Engine](#simulation-engine)
18. [Putting It All Together](#putting-it-all-together)
19. [Class Diagram](#class-diagram)
20. [How to Run](#how-to-run)

---

## Requirements

### Functional Requirements

| #  | Requirement                                                                                       |
| -- | ------------------------------------------------------------------------------------------------- |
| 1  | `allow(client_id)` returns whether a request is allowed in O(1) amortized time                    |
| 2  | Pluggable algorithms — Token Bucket, Sliding Window Log, Sliding Window Counter, Fixed Window     |
| 3  | Per-client rate limiting — each client has independent counters and limits                         |
| 4  | Configurable rules — different endpoints or tiers can have different rate limits                   |
| 5  | Observer-based event system — listeners react to allowed, throttled, and rule-change events        |
| 6  | Rate limit headers — remaining tokens, reset time, and retry-after are returned with each request |
| 7  | Distributed support — a shared backing store enables rate limiting across multiple instances       |
| 8  | Middleware integration — rate limiter wraps request handlers transparently                         |
| 9  | Statistics — total allowed, throttled, per-client counts, and throttle rate                        |
| 10 | Thread-safe — all operations are safe for concurrent access via locking                           |

### Non-Functional Requirements

- Clean OOP with SOLID principles.
- Extensible — new algorithms, rules, and event listeners can be added without modifying core logic.
- Simulatable — the system runs step-by-step for testing and demonstration.

---

## Key Design Patterns

| Pattern   | Where It Is Used                                   | Why                                                                                         |
| --------- | -------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| Strategy  | `RateLimitAlgorithm` decides how to count requests | Swap between Token Bucket, Sliding Window, or Fixed Window without changing limiter logic    |
| Observer  | `RateLimitEventListener` reacts to limiter events  | Loggers, metrics exporters, and alerting react independently to allowed and throttled events |
| Decorator | `PerClientRateLimiter` wraps per-client instances   | Add per-client isolation transparently without modifying the underlying algorithm            |
| Facade    | `RateLimiterSystem` exposes a unified API           | Hides the complexity of algorithms, rules, events, and per-client management                |

---

## Core Enums & Constants

```python
import time
import uuid
import threading
from enum import Enum
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from typing import Any, Callable
from collections import defaultdict


class RateLimitDecision(Enum):
    ALLOWED = "allowed"          # request is permitted
    THROTTLED = "throttled"      # request is rejected (rate limit exceeded)


class RateLimitEventType(Enum):
    ALLOWED = "allowed"
    THROTTLED = "throttled"
    CLIENT_REGISTERED = "client_registered"
    RULE_CHANGED = "rule_changed"
    LIMIT_RESET = "limit_reset"


class AlgorithmType(Enum):
    TOKEN_BUCKET = "token_bucket"
    SLIDING_WINDOW_LOG = "sliding_window_log"
    SLIDING_WINDOW_COUNTER = "sliding_window_counter"
    FIXED_WINDOW = "fixed_window"


# ------------------------------------------------------------------
# System constants
# ------------------------------------------------------------------

DEFAULT_MAX_REQUESTS = 10          # requests per window
DEFAULT_WINDOW_SECONDS = 60        # 1 minute window
DEFAULT_BUCKET_CAPACITY = 10       # max tokens in bucket
DEFAULT_REFILL_RATE = 1.0          # tokens added per second
STATS_DISPLAY_WIDTH = 50
```

`RateLimitDecision` is the binary outcome of every rate limit check — either the request goes through or it doesn't. `RateLimitEventType` categorizes observable events for the observer pattern. `AlgorithmType` labels the four supported algorithms so rules can specify which algorithm to use for a given client or endpoint. `DEFAULT_REFILL_RATE` of 1.0 means one token is added per second in Token Bucket. `DEFAULT_WINDOW_SECONDS` of 60 means the sliding and fixed window algorithms count requests within a one-minute window.

---

## Rate Limit Result

```python
@dataclass
class RateLimitResult:
    """
    The outcome of a rate limit check.

    Returned by every call to allow(). Contains the decision
    (allowed or throttled), the number of remaining requests,
    when the limit resets, and how long to wait before retrying
    if throttled.
    """
    decision: RateLimitDecision
    remaining: int                   # requests remaining in window
    limit: int                       # max requests allowed
    reset_at: float                  # Unix timestamp when limit resets
    retry_after: float = 0.0         # seconds to wait before retrying
    client_id: str = ""

    @property
    def is_allowed(self) -> bool:
        return self.decision == RateLimitDecision.ALLOWED

    @property
    def headers(self) -> dict[str, str]:
        """
        Generate rate limit response headers.

        Standard headers used by APIs to communicate rate
        limit state back to clients.
        """
        h = {
            "X-RateLimit-Limit": str(self.limit),
            "X-RateLimit-Remaining": str(max(0, self.remaining)),
            "X-RateLimit-Reset": str(int(self.reset_at)),
        }
        if not self.is_allowed:
            h["Retry-After"] = str(int(self.retry_after) + 1)
        return h

    def __repr__(self) -> str:
        status = "✅ ALLOWED" if self.is_allowed else "🚫 THROTTLED"
        return (f"RateLimitResult({status}, "
                f"remaining={self.remaining}/{self.limit}, "
                f"retry_after={self.retry_after:.1f}s)")
```

`RateLimitResult` is the response object for every rate limit check. `remaining` tells the client how many more requests they can make before hitting the limit. `reset_at` is the Unix timestamp when the window resets and the counter goes back to zero. `retry_after` is only meaningful when throttled — it tells the client exactly how long to wait. The `headers` property generates standard HTTP rate limit headers (`X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, `Retry-After`) that APIs return to clients. This is a data transfer object — algorithms create it, the limiter returns it, and callers inspect it.

---

## Rate Limiting Algorithm (Strategy Pattern)

```python
class RateLimitAlgorithm(ABC):
    """
    Defines how requests are counted and whether they are allowed.

    Each algorithm maintains its own internal state (tokens,
    timestamps, counters). The rate limiter delegates the allow/deny
    decision to the algorithm without knowing how the decision
    is made.
    """

    @abstractmethod
    def allow(self) -> RateLimitResult:
        """Check if a request is allowed under the current rate limit."""
        ...

    @abstractmethod
    def reset(self) -> None:
        """Reset all internal state."""
        ...

    @abstractmethod
    def name(self) -> str:
        """Return the algorithm name."""
        ...

    @abstractmethod
    def current_usage(self) -> int:
        """Return the current number of consumed requests/tokens."""
        ...

    @abstractmethod
    def max_capacity(self) -> int:
        """Return the maximum allowed requests/tokens."""
        ...
```

`RateLimitAlgorithm` defines the contract for all rate limiting strategies. Every algorithm must implement `allow` (the core decision), `reset` (clear state), and inspection methods for usage monitoring. The rate limiter calls `allow()` on every incoming request and acts on the returned `RateLimitResult`. Adding a new algorithm (e.g., Leaky Bucket, Adaptive) requires only implementing these five methods — the limiter never changes.

---

## Token Bucket Algorithm

```python
class TokenBucket(RateLimitAlgorithm):
    """
    Token Bucket rate limiting algorithm.

    A bucket holds up to `capacity` tokens. Each request consumes
    one token. Tokens are refilled at a fixed rate. If the bucket
    is empty, the request is rejected.

    Properties:
    - Allows bursts up to bucket capacity
    - Smooth long-term rate = refill_rate requests/second
    - Memory: O(1) — only stores token count and timestamp

    Example: capacity=10, refill_rate=2.0
    → Allows bursts of 10 requests instantly
    → Sustains 2 requests/second long-term
    → A fully drained bucket refills in 5 seconds
    """

    def __init__(self, capacity: int = DEFAULT_BUCKET_CAPACITY,
                 refill_rate: float = DEFAULT_REFILL_RATE) -> None:
        self._capacity = capacity
        self._refill_rate = refill_rate           # tokens per second
        self._tokens = float(capacity)            # start full
        self._last_refill = time.time()
        self._lock = threading.RLock()

    def allow(self) -> RateLimitResult:
        """
        Consume a token if available.

        Before checking, refill tokens based on elapsed time.
        This is lazy refill — tokens accumulate passively
        between calls instead of using a background timer.
        """
        with self._lock:
            self._refill()

            if self._tokens >= 1.0:
                self._tokens -= 1.0
                return RateLimitResult(
                    decision=RateLimitDecision.ALLOWED,
                    remaining=int(self._tokens),
                    limit=self._capacity,
                    reset_at=time.time() + (
                        self._capacity / self._refill_rate
                    ),
                )
            else:
                # How long until 1 token is available?
                wait = (1.0 - self._tokens) / self._refill_rate
                return RateLimitResult(
                    decision=RateLimitDecision.THROTTLED,
                    remaining=0,
                    limit=self._capacity,
                    reset_at=time.time() + wait,
                    retry_after=wait,
                )

    def _refill(self) -> None:
        """
        Add tokens based on elapsed time since last refill.

        tokens_to_add = elapsed_seconds × refill_rate
        Tokens are capped at capacity — you cannot exceed
        the burst limit by waiting longer.
        """
        now = time.time()
        elapsed = now - self._last_refill
        tokens_to_add = elapsed * self._refill_rate
        self._tokens = min(
            self._capacity,
            self._tokens + tokens_to_add,
        )
        self._last_refill = now

    def reset(self) -> None:
        with self._lock:
            self._tokens = float(self._capacity)
            self._last_refill = time.time()

    def name(self) -> str:
        return "TokenBucket"

    def current_usage(self) -> int:
        with self._lock:
            self._refill()
            return self._capacity - int(self._tokens)

    def max_capacity(self) -> int:
        return self._capacity

    @property
    def tokens(self) -> float:
        with self._lock:
            self._refill()
            return self._tokens

    def __repr__(self) -> str:
        return (f"TokenBucket(capacity={self._capacity}, "
                f"rate={self._refill_rate}/s, "
                f"tokens={self._tokens:.1f})")
```

`TokenBucket` is the most widely used rate limiting algorithm. The bucket starts full at `capacity` tokens. Each `allow()` call consumes one token. `_refill` adds tokens lazily based on elapsed time — no background thread needed. `tokens_to_add = elapsed × refill_rate` computes how many tokens have accumulated since the last check. Tokens are capped at `capacity` — you cannot stockpile beyond the burst limit. When the bucket is empty, `retry_after` tells the client exactly how long until one token regenerates: `(1.0 - current_tokens) / refill_rate`. The algorithm is O(1) in both time and space — it stores only two numbers (token count and last refill timestamp). Token Bucket is ideal when you want to allow short bursts while enforcing a smooth long-term rate.

---

## Sliding Window Log Algorithm

```python
class SlidingWindowLog(RateLimitAlgorithm):
    """
    Sliding Window Log rate limiting algorithm.

    Keeps a log of timestamps for every request within the
    current window. On each request, expired timestamps are
    purged, then the count is checked against the limit.

    Properties:
    - Exact counting — no approximation
    - Memory: O(n) where n = max_requests (stores every timestamp)
    - Smooth window — no boundary spikes

    Example: max_requests=5, window_seconds=60
    → Tracks every request's timestamp
    → Purges entries older than 60 seconds
    → Rejects if 5+ timestamps remain in the window
    """

    def __init__(self, max_requests: int = DEFAULT_MAX_REQUESTS,
                 window_seconds: int = DEFAULT_WINDOW_SECONDS
                 ) -> None:
        self._max_requests = max_requests
        self._window_seconds = window_seconds
        self._timestamps: list[float] = []
        self._lock = threading.RLock()

    def allow(self) -> RateLimitResult:
        """
        Log the request timestamp and check the count.

        Steps:
        1. Remove all timestamps older than the window
        2. If count < limit, append current timestamp → ALLOWED
        3. If count >= limit → THROTTLED
        """
        with self._lock:
            now = time.time()
            cutoff = now - self._window_seconds

            # Purge expired timestamps
            self._timestamps = [
                ts for ts in self._timestamps if ts > cutoff
            ]

            if len(self._timestamps) < self._max_requests:
                self._timestamps.append(now)
                remaining = (self._max_requests
                             - len(self._timestamps))
                # Reset = when the oldest entry expires
                reset_at = (
                    self._timestamps[0] + self._window_seconds
                    if self._timestamps else now + self._window_seconds
                )
                return RateLimitResult(
                    decision=RateLimitDecision.ALLOWED,
                    remaining=remaining,
                    limit=self._max_requests,
                    reset_at=reset_at,
                )
            else:
                # Retry when the oldest timestamp expires
                oldest = self._timestamps[0]
                retry_after = (oldest + self._window_seconds) - now
                return RateLimitResult(
                    decision=RateLimitDecision.THROTTLED,
                    remaining=0,
                    limit=self._max_requests,
                    reset_at=oldest + self._window_seconds,
                    retry_after=max(0, retry_after),
                )

    def reset(self) -> None:
        with self._lock:
            self._timestamps.clear()

    def name(self) -> str:
        return "SlidingWindowLog"

    def current_usage(self) -> int:
        with self._lock:
            now = time.time()
            cutoff = now - self._window_seconds
            return sum(1 for ts in self._timestamps if ts > cutoff)

    def max_capacity(self) -> int:
        return self._max_requests

    def __repr__(self) -> str:
        return (f"SlidingWindowLog(max={self._max_requests}, "
                f"window={self._window_seconds}s, "
                f"entries={len(self._timestamps)})")
```

`SlidingWindowLog` provides exact rate limiting — no approximation. It stores the timestamp of every request within the window. On each call, it purges timestamps older than `window_seconds` and counts what remains. If the count is under the limit, the request is logged and allowed. The `retry_after` value is precise: it's the time until the oldest timestamp expires, freeing one slot. Memory usage is O(max_requests) because we store at most `max_requests` timestamps (once full, new requests are rejected). This algorithm is ideal when accuracy matters more than memory — for example, API rate limits where users should never be incorrectly throttled.

---

## Sliding Window Counter Algorithm

```python
class SlidingWindowCounter(RateLimitAlgorithm):
    """
    Sliding Window Counter rate limiting algorithm.

    Combines fixed window efficiency with sliding window accuracy.
    Keeps counters for the current and previous fixed windows,
    then estimates the count using a weighted average.

    Properties:
    - Approximate counting — weighted blend of two windows
    - Memory: O(1) — only stores two counters and timestamps
    - Eliminates boundary spikes of fixed window

    Example: max_requests=10, window_seconds=60
    → At 45 seconds into the current window:
      weight = (60 - 45) / 60 = 0.25
      estimated = prev_count × 0.25 + curr_count
    → If estimated >= 10, request is rejected

    This smooths the transition between windows. A burst at
    the boundary of two fixed windows cannot exceed the limit
    because the previous window's count is partially included.
    """

    def __init__(self, max_requests: int = DEFAULT_MAX_REQUESTS,
                 window_seconds: int = DEFAULT_WINDOW_SECONDS
                 ) -> None:
        self._max_requests = max_requests
        self._window_seconds = window_seconds
        self._current_count = 0
        self._previous_count = 0
        self._window_start = time.time()
        self._lock = threading.RLock()

    def allow(self) -> RateLimitResult:
        """
        Estimate the request count using weighted window blending.

        Steps:
        1. Roll forward windows if the current one has expired
        2. Compute weight = fraction of previous window still
           overlapping with the sliding window
        3. estimated = previous × weight + current
        4. If estimated < limit → ALLOWED
        """
        with self._lock:
            now = time.time()
            self._roll_window(now)

            elapsed = now - self._window_start
            weight = max(0.0,
                         (self._window_seconds - elapsed)
                         / self._window_seconds)
            estimated = (self._previous_count * weight
                         + self._current_count)

            if estimated < self._max_requests:
                self._current_count += 1
                remaining = int(
                    self._max_requests - estimated - 1
                )
                reset_at = (self._window_start
                            + self._window_seconds)
                return RateLimitResult(
                    decision=RateLimitDecision.ALLOWED,
                    remaining=max(0, remaining),
                    limit=self._max_requests,
                    reset_at=reset_at,
                )
            else:
                reset_at = (self._window_start
                            + self._window_seconds)
                retry_after = reset_at - now
                return RateLimitResult(
                    decision=RateLimitDecision.THROTTLED,
                    remaining=0,
                    limit=self._max_requests,
                    reset_at=reset_at,
                    retry_after=max(0, retry_after),
                )

    def _roll_window(self, now: float) -> None:
        """
        Advance the window if the current one has expired.

        When the window rolls over:
        - Previous count becomes the current count
        - Current count resets to zero
        - Window start advances by window_seconds

        If multiple windows have passed (long idle period),
        both counters reset to zero.
        """
        elapsed = now - self._window_start
        if elapsed >= self._window_seconds:
            windows_passed = int(elapsed / self._window_seconds)
            if windows_passed == 1:
                self._previous_count = self._current_count
            else:
                self._previous_count = 0
            self._current_count = 0
            self._window_start += (
                windows_passed * self._window_seconds
            )

    def reset(self) -> None:
        with self._lock:
            self._current_count = 0
            self._previous_count = 0
            self._window_start = time.time()

    def name(self) -> str:
        return "SlidingWindowCounter"

    def current_usage(self) -> int:
        with self._lock:
            now = time.time()
            self._roll_window(now)
            elapsed = now - self._window_start
            weight = max(0.0,
                         (self._window_seconds - elapsed)
                         / self._window_seconds)
            return int(self._previous_count * weight
                       + self._current_count)

    def max_capacity(self) -> int:
        return self._max_requests

    def __repr__(self) -> str:
        return (f"SlidingWindowCounter("
                f"max={self._max_requests}, "
                f"window={self._window_seconds}s, "
                f"current={self._current_count}, "
                f"previous={self._previous_count})")
```

`SlidingWindowCounter` is the best trade-off between accuracy and memory. It keeps only two counters (current and previous window) and blends them with a weight based on how far we are into the current window. At the start of a new window, the previous window still has full weight; as time progresses, its weight drops to zero. This eliminates the boundary spike problem of fixed windows — a burst at second 59 of one window and second 1 of the next cannot exceed the limit because the previous window's count is partially included. `_roll_window` handles the transition: if one window has passed, the current count becomes the previous count; if multiple windows have passed (long idle), both counters reset. Memory is O(1) and computation is O(1).

---

## Fixed Window Counter Algorithm

```python
class FixedWindowCounter(RateLimitAlgorithm):
    """
    Fixed Window Counter rate limiting algorithm.

    Divides time into fixed-size windows and counts requests
    in each window. When the window expires, the counter resets.

    Properties:
    - Simplest algorithm — just a counter and timestamp
    - Memory: O(1)
    - Boundary spike problem — 2× burst possible at window edges

    Example: max_requests=10, window_seconds=60
    → 10 requests at second 59 + 10 at second 61 = 20 in 2 seconds
    → Both are in different windows, so both are allowed

    Simple and fast, but the boundary problem makes it unsuitable
    for strict rate limiting. Best for approximate throttling.
    """

    def __init__(self, max_requests: int = DEFAULT_MAX_REQUESTS,
                 window_seconds: int = DEFAULT_WINDOW_SECONDS
                 ) -> None:
        self._max_requests = max_requests
        self._window_seconds = window_seconds
        self._count = 0
        self._window_start = time.time()
        self._lock = threading.RLock()

    def allow(self) -> RateLimitResult:
        """
        Increment the counter and check against the limit.

        If the current window has expired, reset the counter
        and start a new window.
        """
        with self._lock:
            now = time.time()

            # Check if window has expired
            if now - self._window_start >= self._window_seconds:
                self._count = 0
                self._window_start = now

            if self._count < self._max_requests:
                self._count += 1
                remaining = self._max_requests - self._count
                reset_at = (self._window_start
                            + self._window_seconds)
                return RateLimitResult(
                    decision=RateLimitDecision.ALLOWED,
                    remaining=remaining,
                    limit=self._max_requests,
                    reset_at=reset_at,
                )
            else:
                reset_at = (self._window_start
                            + self._window_seconds)
                retry_after = reset_at - now
                return RateLimitResult(
                    decision=RateLimitDecision.THROTTLED,
                    remaining=0,
                    limit=self._max_requests,
                    reset_at=reset_at,
                    retry_after=max(0, retry_after),
                )

    def reset(self) -> None:
        with self._lock:
            self._count = 0
            self._window_start = time.time()

    def name(self) -> str:
        return "FixedWindow"

    def current_usage(self) -> int:
        with self._lock:
            now = time.time()
            if now - self._window_start >= self._window_seconds:
                return 0
            return self._count

    def max_capacity(self) -> int:
        return self._max_requests

    def __repr__(self) -> str:
        return (f"FixedWindowCounter("
                f"max={self._max_requests}, "
                f"window={self._window_seconds}s, "
                f"count={self._count})")
```

`FixedWindowCounter` is the simplest algorithm — a counter and a timestamp. When the window expires, the counter resets to zero. The critical weakness is the boundary spike: 10 requests at second 59 and 10 requests at second 61 yield 20 requests in 2 seconds, but both windows see only 10. This 2× burst at window boundaries makes it unsuitable for strict enforcement. However, it's the fastest and most memory-efficient option (O(1) space, O(1) time), making it a reasonable choice when approximate throttling is acceptable.

---

## Rate Limiter Core

```python
class RateLimiterStats:
    """
    Aggregate rate limiter performance statistics.

    Tracks total allowed, throttled, and per-client counts
    to compute the overall throttle rate.
    """

    def __init__(self) -> None:
        self.total_allowed: int = 0
        self.total_throttled: int = 0
        self.per_client_allowed: dict[str, int] = defaultdict(int)
        self.per_client_throttled: dict[str, int] = defaultdict(int)

    @property
    def total_requests(self) -> int:
        return self.total_allowed + self.total_throttled

    @property
    def throttle_rate(self) -> float:
        if self.total_requests == 0:
            return 0.0
        return self.total_throttled / self.total_requests

    @property
    def allow_rate(self) -> float:
        return 1.0 - self.throttle_rate

    def record(self, client_id: str,
               decision: RateLimitDecision) -> None:
        if decision == RateLimitDecision.ALLOWED:
            self.total_allowed += 1
            self.per_client_allowed[client_id] += 1
        else:
            self.total_throttled += 1
            self.per_client_throttled[client_id] += 1

    def display(self) -> None:
        print(f"\n  {'═'*STATS_DISPLAY_WIDTH}")
        print(f"  RATE LIMITER STATISTICS")
        print(f"  {'═'*STATS_DISPLAY_WIDTH}")
        print(f"  Total requests:    {self.total_requests}")
        print(f"  Allowed:           {self.total_allowed}")
        print(f"  Throttled:         {self.total_throttled}")
        print(f"  Throttle rate:     {self.throttle_rate:.1%}")
        print(f"  Allow rate:        {self.allow_rate:.1%}")

        if self.per_client_allowed or self.per_client_throttled:
            print(f"  {'─'*STATS_DISPLAY_WIDTH}")
            all_clients = sorted(
                set(list(self.per_client_allowed.keys())
                    + list(self.per_client_throttled.keys()))
            )
            for cid in all_clients:
                allowed = self.per_client_allowed.get(cid, 0)
                throttled = self.per_client_throttled.get(cid, 0)
                total = allowed + throttled
                rate = throttled / total if total > 0 else 0.0
                print(f"  {cid:>16}: "
                      f"{allowed} allowed, "
                      f"{throttled} throttled "
                      f"({rate:.0%} throttled)")

        print(f"  {'═'*STATS_DISPLAY_WIDTH}")

    def reset(self) -> None:
        self.total_allowed = 0
        self.total_throttled = 0
        self.per_client_allowed.clear()
        self.per_client_throttled.clear()


class RateLimiter:
    """
    Core rate limiter backed by a pluggable algorithm.

    Wraps any RateLimitAlgorithm and adds client tracking,
    statistics, and thread safety.

    Internal flow on allow(client_id):
        1. Acquire lock
        2. Delegate to algorithm.allow()
        3. Record the result in statistics
        4. Return RateLimitResult with client_id
    """

    def __init__(
        self,
        algorithm: RateLimitAlgorithm | None = None,
        name: str = "default",
    ) -> None:
        self._algorithm = algorithm or TokenBucket()
        self._name = name
        self._stats = RateLimiterStats()
        self._lock = threading.RLock()

    def allow(self, client_id: str = "anonymous") -> RateLimitResult:
        """
        Check if a request from the given client is allowed.

        Delegates to the algorithm and records the result.
        """
        with self._lock:
            result = self._algorithm.allow()
            result.client_id = client_id
            self._stats.record(client_id, result.decision)
            return result

    def reset(self) -> None:
        """Reset the algorithm state and statistics."""
        with self._lock:
            self._algorithm.reset()
            self._stats.reset()

    @property
    def algorithm(self) -> RateLimitAlgorithm:
        return self._algorithm

    @property
    def stats(self) -> RateLimiterStats:
        return self._stats

    @property
    def limiter_name(self) -> str:
        return self._name

    def __repr__(self) -> str:
        return (f"RateLimiter(name={self._name}, "
                f"algorithm={self._algorithm.name()}, "
                f"usage={self._algorithm.current_usage()}"
                f"/{self._algorithm.max_capacity()})")
```

`RateLimiterStats` tracks all allowed and throttled requests with per-client breakdowns. The `throttle_rate` is the primary health metric — a high throttle rate means clients are hitting limits frequently. `record` is called on every `allow()` call. `RateLimiter` is the core class. It wraps any `RateLimitAlgorithm` and adds tracking without modifying the algorithm. The algorithm is swappable at construction — `RateLimiter(TokenBucket())` and `RateLimiter(SlidingWindowLog())` behave identically from the caller's perspective. Every operation acquires the reentrant lock for thread safety.

---

## Rate Limit Event System (Observer Pattern)

```python
@dataclass
class RateLimitEvent:
    """An event emitted by the rate limiter."""
    event_id: str
    event_type: RateLimitEventType
    client_id: str
    decision: RateLimitDecision | None = None
    remaining: int = 0
    detail: str = ""
    timestamp: float = field(
        default_factory=lambda: time.time()
    )


class RateLimitEventListener(ABC):
    """
    Observer that reacts to rate limiter events.

    Implementations receive all events and decide which ones
    to handle. Listeners are decoupled from the limiter — they
    cannot affect rate limit decisions.
    """

    @abstractmethod
    def on_event(self, event: RateLimitEvent) -> None:
        ...

    @abstractmethod
    def listener_id(self) -> str:
        ...


class RateLimitLogger(RateLimitEventListener):
    """Logs all rate limit events to the console."""

    def on_event(self, event: RateLimitEvent) -> None:
        emoji = {
            RateLimitEventType.ALLOWED: "✅",
            RateLimitEventType.THROTTLED: "🚫",
            RateLimitEventType.CLIENT_REGISTERED: "📋",
            RateLimitEventType.RULE_CHANGED: "📝",
            RateLimitEventType.LIMIT_RESET: "🔄",
        }
        icon = emoji.get(event.event_type, "📌")
        detail = ""
        if event.event_type == RateLimitEventType.ALLOWED:
            detail = (f"client={event.client_id} "
                      f"remaining={event.remaining}")
        elif event.event_type == RateLimitEventType.THROTTLED:
            detail = (f"client={event.client_id} "
                      f"({event.detail})")
        elif event.event_type in (
            RateLimitEventType.CLIENT_REGISTERED,
            RateLimitEventType.RULE_CHANGED,
            RateLimitEventType.LIMIT_RESET,
        ):
            detail = (f"client={event.client_id} "
                      f"{event.detail}")

        print(f"  {icon} [{event.event_type.value:>20}] {detail}")

    def listener_id(self) -> str:
        return "rate-limit-logger"


class ThrottleRateMonitor(RateLimitEventListener):
    """Tracks throttle rate over a sliding window of requests."""

    def __init__(self, window_size: int = 100) -> None:
        self._window_size = window_size
        self._history: list[bool] = []  # True = throttled

    def on_event(self, event: RateLimitEvent) -> None:
        if event.event_type == RateLimitEventType.THROTTLED:
            self._history.append(True)
        elif event.event_type == RateLimitEventType.ALLOWED:
            self._history.append(False)
        else:
            return

        if len(self._history) > self._window_size:
            self._history = self._history[-self._window_size:]

    @property
    def windowed_throttle_rate(self) -> float:
        if not self._history:
            return 0.0
        return sum(self._history) / len(self._history)

    def listener_id(self) -> str:
        return "throttle-rate-monitor"

    def display(self) -> None:
        print(f"\n  Sliding window throttle rate "
              f"(last {len(self._history)} requests): "
              f"{self.windowed_throttle_rate:.1%}")


class RateLimitEventBus:
    """
    Central event dispatcher for rate limiter events.

    Manages listener registration and broadcasts events to all
    registered listeners. Thread-safe for concurrent emission.
    """

    def __init__(self) -> None:
        self._listeners: list[RateLimitEventListener] = []
        self._event_log: list[RateLimitEvent] = []
        self._lock = threading.RLock()

    def register(self, listener: RateLimitEventListener) -> None:
        with self._lock:
            self._listeners.append(listener)

    def unregister(self, listener_id: str) -> None:
        with self._lock:
            self._listeners = [
                l for l in self._listeners
                if l.listener_id() != listener_id
            ]

    def emit(self, event: RateLimitEvent) -> None:
        with self._lock:
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
```

`RateLimitEventListener` is the observer interface. `RateLimitLogger` prints every event with emoji-coded log lines — allowed events show remaining quota, throttled events show the reason. `ThrottleRateMonitor` tracks the throttle rate over a sliding window of the last N requests, which reveals recent pressure trends. A spike in throttle rate might indicate a traffic surge or an attack. The `RateLimitEventBus` dispatches events to all registered listeners, isolating each listener from failures in others. Adding a new listener (e.g., Slack alerting, Prometheus exporter) requires only implementing `on_event` and `listener_id`.

---

## Per-Client Rate Limiter (Decorator Pattern)

```python
class PerClientRateLimiter:
    """
    Maintains a separate rate limiter instance per client.

    Each client gets their own independent algorithm instance
    so one client's traffic does not consume another's quota.

    Internal structure:

        _clients: { client_id → RateLimiter }

    When a new client_id is seen, a fresh algorithm is
    created using the factory function. This ensures each
    client starts with a full bucket or empty window.
    """

    def __init__(
        self,
        algorithm_factory: Callable[[], RateLimitAlgorithm],
        event_bus: RateLimitEventBus | None = None,
    ) -> None:
        self._factory = algorithm_factory
        self._clients: dict[str, RateLimiter] = {}
        self._event_bus = event_bus or RateLimitEventBus()
        self._stats = RateLimiterStats()
        self._lock = threading.RLock()

    def allow(self, client_id: str) -> RateLimitResult:
        """
        Check if a request from the given client is allowed.

        If the client has not been seen before, a new rate
        limiter is created with a fresh algorithm instance.
        """
        with self._lock:
            if client_id not in self._clients:
                self._register_client(client_id)

            limiter = self._clients[client_id]
            result = limiter.allow(client_id)
            self._stats.record(client_id, result.decision)

            # Emit event
            event_type = (RateLimitEventType.ALLOWED
                          if result.is_allowed
                          else RateLimitEventType.THROTTLED)
            self._event_bus.emit(RateLimitEvent(
                event_id=str(uuid.uuid4())[:8],
                event_type=event_type,
                client_id=client_id,
                decision=result.decision,
                remaining=result.remaining,
                detail=(f"remaining={result.remaining}"
                        if result.is_allowed
                        else f"retry_after="
                             f"{result.retry_after:.1f}s"),
            ))

            return result

    def _register_client(self, client_id: str) -> None:
        """Create a new rate limiter for a first-time client."""
        algorithm = self._factory()
        self._clients[client_id] = RateLimiter(
            algorithm=algorithm, name=client_id,
        )
        self._event_bus.emit(RateLimitEvent(
            event_id=str(uuid.uuid4())[:8],
            event_type=RateLimitEventType.CLIENT_REGISTERED,
            client_id=client_id,
            detail=f"algorithm={algorithm.name()}",
        ))

    def reset_client(self, client_id: str) -> bool:
        """Reset a specific client's rate limiter."""
        with self._lock:
            if client_id in self._clients:
                self._clients[client_id].reset()
                self._event_bus.emit(RateLimitEvent(
                    event_id=str(uuid.uuid4())[:8],
                    event_type=RateLimitEventType.LIMIT_RESET,
                    client_id=client_id,
                    detail="counters reset",
                ))
                return True
            return False

    def reset_all(self) -> None:
        """Reset all client rate limiters."""
        with self._lock:
            for client_id, limiter in self._clients.items():
                limiter.reset()
            self._stats.reset()

    @property
    def client_count(self) -> int:
        return len(self._clients)

    @property
    def stats(self) -> RateLimiterStats:
        return self._stats

    @property
    def event_bus(self) -> RateLimitEventBus:
        return self._event_bus

    def get_client_usage(self, client_id: str) -> int | None:
        """Return current usage for a client, or None if unknown."""
        with self._lock:
            limiter = self._clients.get(client_id)
            if limiter is None:
                return None
            return limiter.algorithm.current_usage()

    def __repr__(self) -> str:
        return (f"PerClientRateLimiter("
                f"clients={len(self._clients)}, "
                f"total={self._stats.total_requests})")
```

`PerClientRateLimiter` is the decorator — it wraps algorithm instances per client. The `algorithm_factory` is a callable that returns a fresh algorithm instance for each new client. This ensures client A's token bucket is independent of client B's. When `allow("user-123")` is called for the first time, a new `RateLimiter` with a fresh `TokenBucket` (or whatever the factory returns) is created and cached. Subsequent calls reuse the existing instance. `reset_client` lets admins clear a specific client's quota. The factory pattern (passing `lambda: TokenBucket(10, 2.0)` instead of a pre-built instance) is critical — if we shared a single algorithm instance, all clients would share the same token bucket.

---

## Rate Limit Rule Engine

```python
@dataclass
class RateLimitRule:
    """
    A rule defining rate limit parameters for a scope.

    Rules map a scope (e.g., endpoint, tier, client group)
    to an algorithm configuration. The rule engine matches
    requests to rules and creates algorithm instances.
    """
    scope: str                       # e.g. "/api/v1/users", "premium"
    algorithm_type: AlgorithmType
    max_requests: int
    window_seconds: int = DEFAULT_WINDOW_SECONDS
    bucket_capacity: int = DEFAULT_BUCKET_CAPACITY
    refill_rate: float = DEFAULT_REFILL_RATE
    priority: int = 0               # higher = more specific

    def create_algorithm(self) -> RateLimitAlgorithm:
        """Instantiate the algorithm configured by this rule."""
        if self.algorithm_type == AlgorithmType.TOKEN_BUCKET:
            return TokenBucket(
                capacity=self.bucket_capacity,
                refill_rate=self.refill_rate,
            )
        elif self.algorithm_type == AlgorithmType.SLIDING_WINDOW_LOG:
            return SlidingWindowLog(
                max_requests=self.max_requests,
                window_seconds=self.window_seconds,
            )
        elif self.algorithm_type == AlgorithmType.SLIDING_WINDOW_COUNTER:
            return SlidingWindowCounter(
                max_requests=self.max_requests,
                window_seconds=self.window_seconds,
            )
        elif self.algorithm_type == AlgorithmType.FIXED_WINDOW:
            return FixedWindowCounter(
                max_requests=self.max_requests,
                window_seconds=self.window_seconds,
            )
        else:
            raise ValueError(
                f"Unknown algorithm: {self.algorithm_type}"
            )


class RateLimitRuleEngine:
    """
    Matches requests to rules based on scope.

    Rules are stored in priority order. When a request arrives,
    the engine finds the highest-priority matching rule and
    returns its algorithm configuration.

    Example:
        rules = [
            Rule("/api/v1/users", TokenBucket, priority=10),
            Rule("premium", SlidingWindow, priority=5),
            Rule("default", FixedWindow, priority=0),
        ]
        → Request to "/api/v1/users" matches priority 10 rule
        → Request from premium tier matches priority 5 rule
        → Everything else matches the default rule
    """

    def __init__(self) -> None:
        self._rules: list[RateLimitRule] = []
        self._lock = threading.RLock()

    def add_rule(self, rule: RateLimitRule) -> None:
        """Add a rule, maintaining priority order (highest first)."""
        with self._lock:
            self._rules.append(rule)
            self._rules.sort(key=lambda r: -r.priority)

    def remove_rule(self, scope: str) -> bool:
        """Remove all rules for a given scope."""
        with self._lock:
            before = len(self._rules)
            self._rules = [
                r for r in self._rules if r.scope != scope
            ]
            return len(self._rules) < before

    def match(self, scope: str) -> RateLimitRule | None:
        """
        Find the highest-priority rule matching the scope.

        Matching is by exact string equality or prefix match.
        """
        with self._lock:
            for rule in self._rules:
                if (rule.scope == scope
                        or scope.startswith(rule.scope)
                        or rule.scope == "default"):
                    return rule
            return None

    def create_algorithm_for(
        self, scope: str
    ) -> RateLimitAlgorithm:
        """
        Create an algorithm instance for the matching rule.

        Falls back to a default TokenBucket if no rule matches.
        """
        rule = self.match(scope)
        if rule:
            return rule.create_algorithm()
        return TokenBucket()

    @property
    def rules(self) -> list[RateLimitRule]:
        return list(self._rules)

    def display(self) -> None:
        print(f"\n  {'─'*STATS_DISPLAY_WIDTH}")
        print(f"  RATE LIMIT RULES")
        print(f"  {'─'*STATS_DISPLAY_WIDTH}")
        for r in self._rules:
            print(f"  [{r.priority:>3}] {r.scope:<20} "
                  f"{r.algorithm_type.value:<24} "
                  f"max={r.max_requests}")
        print(f"  {'─'*STATS_DISPLAY_WIDTH}")
```

`RateLimitRule` encapsulates the configuration for a single scope — the algorithm type, max requests, window size, and priority. `create_algorithm` is a factory method that instantiates the correct algorithm from the rule's configuration. `RateLimitRuleEngine` matches incoming requests to rules by scope. Rules are sorted by priority (highest first). Matching uses exact string equality, prefix matching, or a "default" fallback rule. This enables tiered rate limiting: premium users get `TokenBucket(1000, 50.0)`, free users get `FixedWindowCounter(10, 60)`, and specific endpoints get their own limits. The engine is the single source of truth for rate limit configuration.

---

## Distributed Rate Limiter

```python
class RateLimitStore(ABC):
    """
    Abstract store for distributed rate limiting.

    In a distributed system, multiple application instances
    must share rate limit state. The store provides atomic
    increment and get operations backed by Redis, Memcached,
    or another shared store.
    """

    @abstractmethod
    def increment(self, key: str, window: int) -> int:
        """
        Atomically increment the counter for the key.

        Returns the new count. The key expires after
        `window` seconds.
        """
        ...

    @abstractmethod
    def get_count(self, key: str) -> int:
        """Return the current count for the key."""
        ...

    @abstractmethod
    def reset(self, key: str) -> None:
        """Reset the counter for the key."""
        ...

    @abstractmethod
    def name(self) -> str:
        ...


class InMemoryRateLimitStore(RateLimitStore):
    """
    In-memory store simulating a distributed counter.

    In production, this would be replaced by a Redis-backed
    implementation using INCR and EXPIRE commands.
    """

    def __init__(self) -> None:
        self._counters: dict[str, int] = defaultdict(int)
        self._expiry: dict[str, float] = {}
        self._lock = threading.RLock()

    def increment(self, key: str, window: int) -> int:
        with self._lock:
            now = time.time()

            # Check expiry
            if key in self._expiry:
                if now >= self._expiry[key]:
                    self._counters[key] = 0
                    self._expiry[key] = now + window

            if key not in self._expiry:
                self._expiry[key] = now + window

            self._counters[key] += 1
            return self._counters[key]

    def get_count(self, key: str) -> int:
        with self._lock:
            now = time.time()
            if key in self._expiry and now >= self._expiry[key]:
                return 0
            return self._counters.get(key, 0)

    def reset(self, key: str) -> None:
        with self._lock:
            self._counters.pop(key, None)
            self._expiry.pop(key, None)

    def name(self) -> str:
        return "InMemoryStore"

    @property
    def data(self) -> dict[str, int]:
        return dict(self._counters)


class DistributedRateLimiter:
    """
    Rate limiter backed by a shared store.

    Uses the Fixed Window algorithm with atomic counters.
    In production, each application instance calls INCR on
    Redis. The counter key includes the client ID and the
    current window timestamp, ensuring automatic expiry.

    Key format: rate:{client_id}:{window_timestamp}
    """

    def __init__(
        self,
        store: RateLimitStore,
        max_requests: int = DEFAULT_MAX_REQUESTS,
        window_seconds: int = DEFAULT_WINDOW_SECONDS,
    ) -> None:
        self._store = store
        self._max_requests = max_requests
        self._window_seconds = window_seconds
        self._lock = threading.RLock()

    def allow(self, client_id: str) -> RateLimitResult:
        """
        Check the shared counter for the client.

        The key includes the current window's start timestamp,
        so counters automatically separate across windows.
        """
        with self._lock:
            now = time.time()
            window_start = int(
                now / self._window_seconds
            ) * self._window_seconds
            key = f"rate:{client_id}:{window_start}"

            count = self._store.increment(
                key, self._window_seconds
            )

            if count <= self._max_requests:
                remaining = self._max_requests - count
                reset_at = window_start + self._window_seconds
                return RateLimitResult(
                    decision=RateLimitDecision.ALLOWED,
                    remaining=remaining,
                    limit=self._max_requests,
                    reset_at=reset_at,
                    client_id=client_id,
                )
            else:
                reset_at = window_start + self._window_seconds
                retry_after = reset_at - now
                return RateLimitResult(
                    decision=RateLimitDecision.THROTTLED,
                    remaining=0,
                    limit=self._max_requests,
                    reset_at=reset_at,
                    retry_after=max(0, retry_after),
                    client_id=client_id,
                )

    def __repr__(self) -> str:
        return (f"DistributedRateLimiter("
                f"max={self._max_requests}, "
                f"window={self._window_seconds}s, "
                f"store={self._store.name()})")
```

`RateLimitStore` abstracts the distributed counter. `InMemoryRateLimitStore` simulates Redis with `increment`, `get_count`, and key expiry. In production, `increment` maps to Redis's `INCR` followed by `EXPIRE`. `DistributedRateLimiter` generates a key combining the client ID and the current window's start timestamp: `rate:user-123:1698300000`. This key naturally separates windows — when the timestamp changes, a new key is created. The store's `increment` is atomic, so concurrent requests from multiple instances get a consistent count. The counter expires after `window_seconds`, so Redis automatically garbage-collects old keys.

---

## Rate Limiter Middleware

```python
@dataclass
class Request:
    """Simulated HTTP request."""
    client_id: str
    endpoint: str
    method: str = "GET"
    request_id: str = field(
        default_factory=lambda: str(uuid.uuid4())[:8]
    )
    timestamp: float = field(default_factory=time.time)


@dataclass
class Response:
    """Simulated HTTP response."""
    status_code: int
    body: dict[str, Any]
    headers: dict[str, str] = field(default_factory=dict)


class RateLimitMiddleware:
    """
    Middleware that intercepts requests and applies rate limiting.

    Sits between the client and the handler. If the request is
    allowed, it passes through to the handler. If throttled,
    a 429 Too Many Requests response is returned immediately.

    Flow:
        Client → Middleware → [Rate Limiter] → Handler
                    ↓ (if throttled)
              429 Response
    """

    def __init__(
        self,
        limiter: PerClientRateLimiter,
        handler: Callable[[Request], Response] | None = None,
    ) -> None:
        self._limiter = limiter
        self._handler = handler or self._default_handler

    def handle(self, request: Request) -> Response:
        """
        Process a request through the rate limiter.

        Steps:
        1. Check rate limit for the client
        2. If allowed, pass to handler
        3. If throttled, return 429 with rate limit headers
        """
        result = self._limiter.allow(request.client_id)

        if result.is_allowed:
            response = self._handler(request)
            response.headers.update(result.headers)
            return response
        else:
            return Response(
                status_code=429,
                body={
                    "error": "Too Many Requests",
                    "message": (
                        f"Rate limit exceeded. "
                        f"Retry after {result.retry_after:.0f}s"
                    ),
                },
                headers=result.headers,
            )

    @staticmethod
    def _default_handler(request: Request) -> Response:
        """Default handler that returns 200 OK."""
        return Response(
            status_code=200,
            body={
                "message": "OK",
                "endpoint": request.endpoint,
                "request_id": request.request_id,
            },
        )

    def __repr__(self) -> str:
        return f"RateLimitMiddleware(limiter={self._limiter})"
```

`RateLimitMiddleware` implements the middleware pattern — it intercepts requests before they reach the handler. On every request, it calls `allow(client_id)` on the per-client rate limiter. If allowed, the request passes through to the handler and the response gets rate limit headers added. If throttled, the middleware short-circuits with a 429 response that includes `Retry-After` and the standard `X-RateLimit-*` headers. The handler never knows about rate limiting — separation of concerns. The `_default_handler` returns a simple 200 OK for simulation purposes; in production, this would be the actual request handler (e.g., a Django view or FastAPI endpoint).

---

## Rate Limiter Facade

```python
class RateLimiterSystem:
    """
    Facade that orchestrates rate limiter creation and configuration.

    Provides factory methods for common setups and exposes
    a simple API that hides all internal complexity.
    """

    def __init__(
        self,
        algorithm_factory: Callable[
            [], RateLimitAlgorithm
        ] | None = None,
        rule_engine: RateLimitRuleEngine | None = None,
    ) -> None:
        self._event_bus = RateLimitEventBus()
        self._logger = RateLimitLogger()
        self._monitor = ThrottleRateMonitor()
        self._event_bus.register(self._logger)
        self._event_bus.register(self._monitor)

        self._rule_engine = rule_engine
        factory = algorithm_factory or (lambda: TokenBucket())

        self._limiter = PerClientRateLimiter(
            algorithm_factory=factory,
            event_bus=self._event_bus,
        )
        self._middleware = RateLimitMiddleware(self._limiter)

    # ------------------------------------------------------------------
    # Core operations
    # ------------------------------------------------------------------

    def allow(self, client_id: str) -> RateLimitResult:
        return self._limiter.allow(client_id)

    def handle_request(self, request: Request) -> Response:
        return self._middleware.handle(request)

    def reset_client(self, client_id: str) -> bool:
        return self._limiter.reset_client(client_id)

    def reset_all(self) -> None:
        self._limiter.reset_all()

    # ------------------------------------------------------------------
    # Inspection
    # ------------------------------------------------------------------

    @property
    def stats(self) -> RateLimiterStats:
        return self._limiter.stats

    @property
    def monitor(self) -> ThrottleRateMonitor:
        return self._monitor

    @property
    def event_bus(self) -> RateLimitEventBus:
        return self._event_bus

    @property
    def limiter(self) -> PerClientRateLimiter:
        return self._limiter

    @property
    def middleware(self) -> RateLimitMiddleware:
        return self._middleware

    @property
    def rule_engine(self) -> RateLimitRuleEngine | None:
        return self._rule_engine

    def display_status(self) -> None:
        self._limiter.stats.display()
        self._monitor.display()
```

`RateLimiterSystem` is the facade — it wires together the per-client rate limiter, event bus, logger, monitor, rule engine, and middleware. Users interact with `allow`, `handle_request`, and `reset_client` without knowing about algorithms, event dispatch, or per-client management. The constructor registers a logger and throttle rate monitor by default. `display_status` prints both the cumulative statistics and the sliding window throttle rate. Adding new functionality (e.g., Prometheus export, webhook alerting) means registering a new listener — the facade code does not change.

---

## Simulation Engine

```python
class RateLimitScenario:
    """
    Tick-based simulation engine for rate limiter demos.

    Each tick represents a logical time step. Actions are registered
    at specific ticks and execute in order when the simulation runs.
    """

    def __init__(self, system: RateLimiterSystem) -> None:
        self._system = system
        self._events: list[tuple[int, str, callable]] = []

    def at_tick(self, tick: int, description: str,
                action: callable) -> None:
        self._events.append((tick, description, action))

    def run(self) -> None:
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

`RateLimitScenario` mirrors the simulation engine from other guides. Actions are registered at specific ticks and execute in order, making demos reproducible and readable.

---

## Putting It All Together

```python
def main_scenario() -> None:
    """Run the main rate limiter demonstration."""

    print("\n╔══════════════════════════════════════════╗")
    print("║       RATE LIMITER SYSTEM                ║")
    print("║  Algorithm: Token Bucket                  ║")
    print("║  Capacity: 5 tokens                       ║")
    print("║  Refill: 2.0 tokens/sec                   ║")
    print("╚══════════════════════════════════════════╝")

    system = RateLimiterSystem(
        algorithm_factory=lambda: TokenBucket(
            capacity=5, refill_rate=2.0,
        ),
    )

    scenario = RateLimitScenario(system)

    # --- Client A sends 5 requests (uses all tokens) ---
    scenario.at_tick(1, "Client A: 5 requests (drain bucket)",
                     lambda: [
                         system.allow("client-A")
                         for _ in range(5)
                     ])

    # --- Client A is now throttled ---
    scenario.at_tick(2, "Client A: 1 more request (throttled)",
                     lambda: print(
                         f"    Result: {system.allow('client-A')}"
                     ))

    # --- Client B has their own bucket ---
    scenario.at_tick(3, "Client B: 3 requests (independent bucket)",
                     lambda: [
                         system.allow("client-B")
                         for _ in range(3)
                     ])

    # --- Wait for refill ---
    scenario.at_tick(4, "Wait 1 second for token refill",
                     lambda: time.sleep(1.0))

    # --- Client A should have tokens again ---
    scenario.at_tick(5, "Client A: 1 request after refill",
                     lambda: print(
                         f"    Result: {system.allow('client-A')}"
                     ))

    # --- Reset Client A ---
    scenario.at_tick(6, "Reset Client A's rate limiter",
                     lambda: system.reset_client("client-A"))

    # --- Client A has full bucket again ---
    scenario.at_tick(7, "Client A: 3 requests after reset",
                     lambda: [
                         system.allow("client-A")
                         for _ in range(3)
                     ])

    # --- Display statistics ---
    scenario.at_tick(8, "Display rate limiter statistics",
                     lambda: system.display_status())

    scenario.run()


# ------------------------------------------------------------------
# Algorithm Comparison
# ------------------------------------------------------------------

def compare_algorithms() -> None:
    """Compare all four algorithms on the same workload."""

    print("\n" + "=" * 55)
    print("  ALGORITHM COMPARISON")
    print("=" * 55)

    algorithms: list[tuple[str, Callable[[], RateLimitAlgorithm]]] = [
        ("TokenBucket",
         lambda: TokenBucket(capacity=5, refill_rate=1.0)),
        ("SlidingWindowLog",
         lambda: SlidingWindowLog(max_requests=5, window_seconds=60)),
        ("SlidingWindowCounter",
         lambda: SlidingWindowCounter(
             max_requests=5, window_seconds=60)),
        ("FixedWindow",
         lambda: FixedWindowCounter(
             max_requests=5, window_seconds=60)),
    ]

    for name, factory in algorithms:
        alg = factory()
        allowed_count = 0
        throttled_count = 0

        # Send 10 requests rapidly
        for _ in range(10):
            result = alg.allow()
            if result.is_allowed:
                allowed_count += 1
            else:
                throttled_count += 1

        total = allowed_count + throttled_count
        rate = throttled_count / total if total > 0 else 0.0
        print(f"\n  Algorithm: {name}")
        print(f"  {'─'*45}")
        print(f"    Allowed:       {allowed_count}")
        print(f"    Throttled:     {throttled_count}")
        print(f"    Throttle rate: {rate:.1%}")
        print(f"    Usage:         "
              f"{alg.current_usage()}/{alg.max_capacity()}")

    print("\n" + "=" * 55)


def demo_middleware() -> None:
    """Demonstrate rate limiting middleware with HTTP requests."""

    print("\n" + "=" * 55)
    print("  MIDDLEWARE DEMO")
    print("=" * 55)

    system = RateLimiterSystem(
        algorithm_factory=lambda: FixedWindowCounter(
            max_requests=3, window_seconds=60,
        ),
    )

    # Simulate 5 HTTP requests from same client
    for i in range(5):
        request = Request(
            client_id="user-42",
            endpoint="/api/v1/data",
            method="GET",
        )
        response = system.handle_request(request)

        status = ("✅" if response.status_code == 200
                  else "🚫")
        print(f"\n  {status} Request {i + 1}: "
              f"status={response.status_code}")
        print(f"    Body: {response.body}")
        print(f"    Headers: {response.headers}")

    print("\n" + "=" * 55)


def demo_rule_engine() -> None:
    """Demonstrate tiered rate limiting with rules."""

    print("\n" + "=" * 55)
    print("  RULE ENGINE DEMO")
    print("=" * 55)

    engine = RateLimitRuleEngine()

    # Premium tier: generous limits
    engine.add_rule(RateLimitRule(
        scope="premium",
        algorithm_type=AlgorithmType.TOKEN_BUCKET,
        max_requests=100,
        bucket_capacity=20,
        refill_rate=5.0,
        priority=10,
    ))

    # Free tier: strict limits
    engine.add_rule(RateLimitRule(
        scope="free",
        algorithm_type=AlgorithmType.FIXED_WINDOW,
        max_requests=5,
        window_seconds=60,
        priority=5,
    ))

    # Default fallback
    engine.add_rule(RateLimitRule(
        scope="default",
        algorithm_type=AlgorithmType.SLIDING_WINDOW_COUNTER,
        max_requests=10,
        window_seconds=60,
        priority=0,
    ))

    engine.display()

    # Test matching
    for scope in ["premium", "free", "unknown"]:
        rule = engine.match(scope)
        if rule:
            alg = rule.create_algorithm()
            print(f"\n  Scope '{scope}' → "
                  f"{alg.name()} "
                  f"(max={rule.max_requests})")

            # Send 8 requests
            results = [alg.allow() for _ in range(8)]
            allowed = sum(1 for r in results if r.is_allowed)
            print(f"    8 requests → "
                  f"{allowed} allowed, "
                  f"{8 - allowed} throttled")

    print("\n" + "=" * 55)


def demo_distributed() -> None:
    """Demonstrate distributed rate limiting with shared store."""

    print("\n" + "=" * 55)
    print("  DISTRIBUTED RATE LIMITER DEMO")
    print("=" * 55)

    store = InMemoryRateLimitStore()
    limiter = DistributedRateLimiter(
        store=store,
        max_requests=5,
        window_seconds=60,
    )

    # Simulate requests from two "instances" sharing the store
    print("\n  Simulating 2 app instances sharing a store:")
    for i in range(8):
        instance = f"instance-{(i % 2) + 1}"
        result = limiter.allow("shared-user")
        status = "✅" if result.is_allowed else "🚫"
        print(f"  {status} [{instance}] Request {i + 1}: "
              f"remaining={result.remaining}")

    print(f"\n  Store contents: {store.data}")
    print("\n" + "=" * 55)


# ------------------------------------------------------------------
# Unit Tests
# ------------------------------------------------------------------

def test_token_bucket_basic() -> None:
    """Test Token Bucket allows requests up to capacity."""
    bucket = TokenBucket(capacity=3, refill_rate=1.0)

    r1 = bucket.allow()
    r2 = bucket.allow()
    r3 = bucket.allow()
    r4 = bucket.allow()

    assert r1.is_allowed
    assert r2.is_allowed
    assert r3.is_allowed
    assert not r4.is_allowed
    assert r4.retry_after > 0
    print("✅ test_token_bucket_basic passed")


def test_token_bucket_refill() -> None:
    """Test Token Bucket refills tokens over time."""
    bucket = TokenBucket(capacity=2, refill_rate=10.0)

    # Drain the bucket
    bucket.allow()
    bucket.allow()
    r = bucket.allow()
    assert not r.is_allowed

    # Wait for refill (10 tokens/sec = 1 token in 0.1s)
    time.sleep(0.15)

    r = bucket.allow()
    assert r.is_allowed
    print("✅ test_token_bucket_refill passed")


def test_sliding_window_log() -> None:
    """Test Sliding Window Log exact counting."""
    log = SlidingWindowLog(max_requests=3, window_seconds=60)

    r1 = log.allow()
    r2 = log.allow()
    r3 = log.allow()
    r4 = log.allow()

    assert r1.is_allowed
    assert r2.is_allowed
    assert r3.is_allowed
    assert not r4.is_allowed
    assert r4.remaining == 0
    print("✅ test_sliding_window_log passed")


def test_sliding_window_counter() -> None:
    """Test Sliding Window Counter approximate counting."""
    counter = SlidingWindowCounter(
        max_requests=5, window_seconds=60,
    )

    results = [counter.allow() for _ in range(7)]
    allowed = sum(1 for r in results if r.is_allowed)
    throttled = sum(1 for r in results if not r.is_allowed)

    assert allowed == 5
    assert throttled == 2
    print("✅ test_sliding_window_counter passed")


def test_fixed_window_counter() -> None:
    """Test Fixed Window Counter basic operation."""
    fw = FixedWindowCounter(max_requests=3, window_seconds=60)

    r1 = fw.allow()
    r2 = fw.allow()
    r3 = fw.allow()
    r4 = fw.allow()

    assert r1.is_allowed
    assert r2.is_allowed
    assert r3.is_allowed
    assert not r4.is_allowed
    print("✅ test_fixed_window_counter passed")


def test_rate_limit_result_headers() -> None:
    """Test that rate limit headers are generated correctly."""
    result = RateLimitResult(
        decision=RateLimitDecision.THROTTLED,
        remaining=0,
        limit=10,
        reset_at=1698300060.0,
        retry_after=30.0,
    )

    headers = result.headers
    assert headers["X-RateLimit-Limit"] == "10"
    assert headers["X-RateLimit-Remaining"] == "0"
    assert "Retry-After" in headers
    assert not result.is_allowed
    print("✅ test_rate_limit_result_headers passed")


def test_per_client_isolation() -> None:
    """Test that per-client rate limiters are independent."""
    limiter = PerClientRateLimiter(
        algorithm_factory=lambda: TokenBucket(
            capacity=2, refill_rate=0.1,
        ),
    )

    # Client A uses their tokens
    limiter.allow("A")
    limiter.allow("A")
    r = limiter.allow("A")
    assert not r.is_allowed  # A is throttled

    # Client B still has tokens
    r = limiter.allow("B")
    assert r.is_allowed  # B is independent
    print("✅ test_per_client_isolation passed")


def test_per_client_reset() -> None:
    """Test resetting a specific client."""
    limiter = PerClientRateLimiter(
        algorithm_factory=lambda: TokenBucket(
            capacity=2, refill_rate=0.1,
        ),
    )

    limiter.allow("A")
    limiter.allow("A")
    r = limiter.allow("A")
    assert not r.is_allowed

    limiter.reset_client("A")
    r = limiter.allow("A")
    assert r.is_allowed  # full bucket after reset
    print("✅ test_per_client_reset passed")


def test_middleware_allows() -> None:
    """Test that middleware passes allowed requests."""
    limiter = PerClientRateLimiter(
        algorithm_factory=lambda: TokenBucket(
            capacity=5, refill_rate=1.0,
        ),
    )
    middleware = RateLimitMiddleware(limiter)

    request = Request(
        client_id="user-1",
        endpoint="/api/test",
    )
    response = middleware.handle(request)

    assert response.status_code == 200
    assert "X-RateLimit-Limit" in response.headers
    print("✅ test_middleware_allows passed")


def test_middleware_throttles() -> None:
    """Test that middleware returns 429 when throttled."""
    limiter = PerClientRateLimiter(
        algorithm_factory=lambda: TokenBucket(
            capacity=1, refill_rate=0.01,
        ),
    )
    middleware = RateLimitMiddleware(limiter)

    # First request is allowed
    r1 = middleware.handle(Request(
        client_id="user-1", endpoint="/api/test",
    ))
    assert r1.status_code == 200

    # Second request is throttled
    r2 = middleware.handle(Request(
        client_id="user-1", endpoint="/api/test",
    ))
    assert r2.status_code == 429
    assert "Retry-After" in r2.headers
    print("✅ test_middleware_throttles passed")


def test_rule_engine_matching() -> None:
    """Test that rule engine matches scopes correctly."""
    engine = RateLimitRuleEngine()

    engine.add_rule(RateLimitRule(
        scope="/api/v1",
        algorithm_type=AlgorithmType.TOKEN_BUCKET,
        max_requests=100,
        priority=10,
    ))
    engine.add_rule(RateLimitRule(
        scope="default",
        algorithm_type=AlgorithmType.FIXED_WINDOW,
        max_requests=10,
        priority=0,
    ))

    r1 = engine.match("/api/v1/users")
    assert r1 is not None
    assert r1.scope == "/api/v1"

    r2 = engine.match("unknown-scope")
    assert r2 is not None
    assert r2.scope == "default"
    print("✅ test_rule_engine_matching passed")


def test_rule_creates_algorithm() -> None:
    """Test that rules create the correct algorithm."""
    rule = RateLimitRule(
        scope="test",
        algorithm_type=AlgorithmType.SLIDING_WINDOW_LOG,
        max_requests=20,
        window_seconds=30,
    )
    alg = rule.create_algorithm()

    assert alg.name() == "SlidingWindowLog"
    assert alg.max_capacity() == 20
    print("✅ test_rule_creates_algorithm passed")


def test_distributed_limiter() -> None:
    """Test distributed rate limiter with shared store."""
    store = InMemoryRateLimitStore()
    limiter = DistributedRateLimiter(
        store=store, max_requests=3, window_seconds=60,
    )

    r1 = limiter.allow("user-1")
    r2 = limiter.allow("user-1")
    r3 = limiter.allow("user-1")
    r4 = limiter.allow("user-1")

    assert r1.is_allowed
    assert r2.is_allowed
    assert r3.is_allowed
    assert not r4.is_allowed
    print("✅ test_distributed_limiter passed")


def test_stats_tracking() -> None:
    """Test that statistics are tracked correctly."""
    limiter = PerClientRateLimiter(
        algorithm_factory=lambda: TokenBucket(
            capacity=2, refill_rate=0.01,
        ),
    )

    limiter.allow("A")  # allowed
    limiter.allow("A")  # allowed
    limiter.allow("A")  # throttled
    limiter.allow("B")  # allowed

    stats = limiter.stats
    assert stats.total_allowed == 3
    assert stats.total_throttled == 1
    assert stats.per_client_allowed["A"] == 2
    assert stats.per_client_throttled["A"] == 1
    assert stats.per_client_allowed["B"] == 1
    print("✅ test_stats_tracking passed")


def test_algorithm_reset() -> None:
    """Test that algorithm reset restores full capacity."""
    bucket = TokenBucket(capacity=3, refill_rate=0.01)

    bucket.allow()
    bucket.allow()
    bucket.allow()
    r = bucket.allow()
    assert not r.is_allowed

    bucket.reset()
    r = bucket.allow()
    assert r.is_allowed
    assert r.remaining == 2
    print("✅ test_algorithm_reset passed")


def test_token_bucket_burst() -> None:
    """Test that Token Bucket allows bursts up to capacity."""
    bucket = TokenBucket(capacity=10, refill_rate=1.0)

    # All 10 should be allowed in a burst
    results = [bucket.allow() for _ in range(10)]
    assert all(r.is_allowed for r in results)

    # 11th should be throttled
    r = bucket.allow()
    assert not r.is_allowed
    print("✅ test_token_bucket_burst passed")


def test_throttle_rate_calculation() -> None:
    """Test throttle rate computation."""
    stats = RateLimiterStats()

    for _ in range(7):
        stats.record("A", RateLimitDecision.ALLOWED)
    for _ in range(3):
        stats.record("A", RateLimitDecision.THROTTLED)

    assert stats.total_requests == 10
    assert stats.throttle_rate == 0.3
    assert stats.allow_rate == 0.7
    print("✅ test_throttle_rate_calculation passed")


def test_sliding_window_log_window_expiry() -> None:
    """Test that expired timestamps are purged."""
    log = SlidingWindowLog(max_requests=2, window_seconds=1)

    log.allow()
    log.allow()
    r = log.allow()
    assert not r.is_allowed  # full

    time.sleep(1.1)  # wait for window to expire

    r = log.allow()
    assert r.is_allowed  # window expired, counter reset
    print("✅ test_sliding_window_log_window_expiry passed")


def test_in_memory_store_expiry() -> None:
    """Test that in-memory store expires keys."""
    store = InMemoryRateLimitStore()

    store.increment("key1", window=1)
    store.increment("key1", window=1)

    assert store.get_count("key1") == 2

    time.sleep(1.1)

    assert store.get_count("key1") == 0
    print("✅ test_in_memory_store_expiry passed")


def run_all_tests() -> None:
    print("\n" + "=" * 40)
    print("  RUNNING TESTS")
    print("=" * 40 + "\n")
    test_token_bucket_basic()
    test_token_bucket_refill()
    test_sliding_window_log()
    test_sliding_window_counter()
    test_fixed_window_counter()
    test_rate_limit_result_headers()
    test_per_client_isolation()
    test_per_client_reset()
    test_middleware_allows()
    test_middleware_throttles()
    test_rule_engine_matching()
    test_rule_creates_algorithm()
    test_distributed_limiter()
    test_stats_tracking()
    test_algorithm_reset()
    test_token_bucket_burst()
    test_throttle_rate_calculation()
    test_sliding_window_log_window_expiry()
    test_in_memory_store_expiry()
    print("\n✅ All tests passed!\n")


if __name__ == "__main__":
    main_scenario()
    compare_algorithms()
    demo_middleware()
    demo_rule_engine()
    demo_distributed()
    run_all_tests()
```

Each test isolates a single behavior. `test_token_bucket_basic` verifies requests are allowed up to capacity and then throttled. `test_token_bucket_refill` confirms tokens regenerate over time — with `refill_rate=10.0`, one token appears in 0.1 seconds. `test_sliding_window_log` checks exact counting — 3 requests allowed, 4th rejected. `test_per_client_isolation` is critical: client A's throttling does not affect client B because each has an independent algorithm instance. `test_middleware_throttles` verifies the middleware returns 429 with `Retry-After` headers. `test_rule_engine_matching` confirms prefix matching and default fallback. `test_distributed_limiter` validates shared counter semantics. `test_sliding_window_log_window_expiry` uses a 1-second window and sleeps 1.1 seconds to confirm old timestamps are purged.

---

## Class Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                    RateLimiterSystem                               │
│──────────────────────────────────────────────────────────────────│
│ - _limiter: PerClientRateLimiter                                   │
│ - _event_bus: RateLimitEventBus                                    │
│ - _logger: RateLimitLogger                                         │
│ - _monitor: ThrottleRateMonitor                                    │
│ - _middleware: RateLimitMiddleware                                  │
│ - _rule_engine: RateLimitRuleEngine                                │
│──────────────────────────────────────────────────────────────────│
│ + allow(client_id) → RateLimitResult                               │
│ + handle_request(request) → Response                               │
│ + reset_client(client_id) → bool                                   │
│ + reset_all() → None                                               │
│ + display_status() → None                                          │
└───────┬──────────┬──────────┬────────────────────────────────────┘
        │          │          │
   uses ▼     uses ▼     uses ▼
┌──────────────────────────────────────────────────────────────────┐
│  PerClientRateLimiter (Decorator)                                  │
│──────────────────────────────────────────────────────────────────│
│ - _factory: Callable[[], RateLimitAlgorithm]                       │
│ - _clients: dict[str, RateLimiter]                                 │
│ - _event_bus: RateLimitEventBus                                    │
│ - _stats: RateLimiterStats                                         │
│──────────────────────────────────────────────────────────────────│
│ + allow(client_id) → RateLimitResult                               │
│ + reset_client(client_id) → bool                                   │
│ + reset_all() → None                                               │
│ + get_client_usage(client_id) → int | None                         │
└──────────┬───────────────────────────────────────────────────────┘
           │ creates per-client
           ▼
┌──────────────────────────────────────────────────────────────────┐
│  RateLimiter                                                       │
│──────────────────────────────────────────────────────────────────│
│ - _algorithm: RateLimitAlgorithm                                   │
│ - _name: str                                                       │
│ - _stats: RateLimiterStats                                         │
│ - _lock: RLock                                                     │
│──────────────────────────────────────────────────────────────────│
│ + allow(client_id) → RateLimitResult                               │
│ + reset() → None                                                   │
└──────────┬───────────────────────────────────────────────────────┘
           │ delegates to
           ▼
┌──────────────────────────────────────────────────────────────────┐
│  «abstract» RateLimitAlgorithm                                     │
│──────────────────────────────────────────────────────────────────│
│ + allow() → RateLimitResult                                        │
│ + reset() → None                                                   │
│ + name() → str                                                     │
│ + current_usage() → int                                            │
│ + max_capacity() → int                                             │
└──────────┬────────────────────────────────────────────────────────┘
           │ implemented by
  ┌────────┼────────────┬──────────────┐
  ▼        ▼            ▼              ▼
Token    Sliding      Sliding        Fixed
Bucket   Window       Window         Window
         Log          Counter        Counter

┌──────────────────────────────────────────────────────────────────┐
│  RateLimitMiddleware                                               │
│──────────────────────────────────────────────────────────────────│
│ - _limiter: PerClientRateLimiter                                   │
│ - _handler: Callable[[Request], Response]                          │
│──────────────────────────────────────────────────────────────────│
│ + handle(request) → Response                                       │
│   → Allowed: pass to handler, add headers                          │
│   → Throttled: return 429 with Retry-After                         │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  RateLimitRuleEngine                                               │
│──────────────────────────────────────────────────────────────────│
│ - _rules: list[RateLimitRule]                                      │
│──────────────────────────────────────────────────────────────────│
│ + add_rule(rule) → None                                            │
│ + match(scope) → RateLimitRule | None                              │
│ + create_algorithm_for(scope) → RateLimitAlgorithm                 │
│ + display() → None                                                 │
└──────────┬───────────────────────────────────────────────────────┘
           │ contains
           ▼
┌──────────────────────────────────────────────────────────────────┐
│  RateLimitRule                                                     │
│──────────────────────────────────────────────────────────────────│
│ + scope: str                                                       │
│ + algorithm_type: AlgorithmType                                    │
│ + max_requests: int                                                │
│ + window_seconds: int                                              │
│ + bucket_capacity: int                                             │
│ + refill_rate: float                                               │
│ + priority: int                                                    │
│──────────────────────────────────────────────────────────────────│
│ + create_algorithm() → RateLimitAlgorithm                          │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  «abstract» RateLimitStore                                         │
│──────────────────────────────────────────────────────────────────│
│ + increment(key, window) → int                                     │
│ + get_count(key) → int                                             │
│ + reset(key) → None                                                │
│ + name() → str                                                     │
└──────────┬────────────────────────────────────────────────────────┘
           │ implemented by
           ▼
       InMemory
       RateLimit
       Store

┌──────────────────────────────────────────────────────────────────┐
│  DistributedRateLimiter                                            │
│──────────────────────────────────────────────────────────────────│
│ - _store: RateLimitStore                                           │
│ - _max_requests: int                                               │
│ - _window_seconds: int                                             │
│──────────────────────────────────────────────────────────────────│
│ + allow(client_id) → RateLimitResult                               │
│   key = "rate:{client}:{window_start}"                             │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  «abstract» RateLimitEventListener                                 │
│──────────────────────────────────────────────────────────────────│
│ + on_event(event) → None                                           │
│ + listener_id() → str                                              │
└──────────┬────────────────────────────────────────────────────────┘
           │ implemented by
  ┌────────┴──────────┐
  ▼                   ▼
RateLimit          Throttle
Logger             Rate
                   Monitor

┌──────────────┐ ┌──────────────────┐ ┌──────────────────┐
│ RateLimitResult│ │ RateLimiterStats │ │ RateLimitEvent   │
│──────────────│ │──────────────────│ │──────────────────│
│ + decision   │ │ + total_allowed  │ │ + event_id       │
│ + remaining  │ │ + total_throttled│ │ + event_type     │
│ + limit      │ │ + per_client_    │ │ + client_id      │
│ + reset_at   │ │   allowed        │ │ + decision       │
│ + retry_after│ │ + per_client_    │ │ + remaining      │
│ + client_id  │ │   throttled      │ │ + detail         │
│ + is_allowed │ │ + throttle_rate  │ │ + timestamp      │
│ + headers    │ │ + allow_rate     │ └──────────────────┘
└──────────────┘ └──────────────────┘

┌──────────────┐ ┌──────────────────┐ ┌──────────────────┐
│ Request      │ │ Response         │ │ RateLimitEventBus│
│──────────────│ │──────────────────│ │──────────────────│
│ + client_id  │ │ + status_code    │ │ - _listeners     │
│ + endpoint   │ │ + body           │ │ - _event_log     │
│ + method     │ │ + headers        │ │──────────────────│
│ + request_id │ └──────────────────┘ │ + register()     │
│ + timestamp  │                      │ + emit()         │
└──────────────┘ ┌──────────────────┐ │ + unregister()   │
                 │ RateLimitScenario│ └──────────────────┘
                 │──────────────────│
                 │ - _system        │
                 │ - _events        │
                 │──────────────────│
                 │ + at_tick()      │
                 │ + run()          │
                 └──────────────────┘

┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│ RateLimitDecision│ │ RateLimitEvent   │ │ AlgorithmType    │
│ (Enum)           │ │ Type (Enum)      │ │ (Enum)           │
│──────────────────│ │──────────────────│ │──────────────────│
│ ALLOWED          │ │ ALLOWED          │ │ TOKEN_BUCKET     │
│ THROTTLED        │ │ THROTTLED        │ │ SLIDING_WINDOW_  │
└──────────────────┘ │ CLIENT_REGISTERED│ │   LOG            │
                     │ RULE_CHANGED     │ │ SLIDING_WINDOW_  │
                     │ LIMIT_RESET      │ │   COUNTER        │
                     └──────────────────┘ │ FIXED_WINDOW     │
                                          └──────────────────┘
```

---

## How to Run

Save all code into a single file `rate_limiter.py` and run:

```bash
python rate_limiter.py
```

### Example Output

```
╔══════════════════════════════════════════╗
║       RATE LIMITER SYSTEM                ║
║  Algorithm: Token Bucket                  ║
║  Capacity: 5 tokens                       ║
║  Refill: 2.0 tokens/sec                   ║
╚══════════════════════════════════════════╝

  ───────────────────────────────────────────────────────
  SIMULATION START (ticks: 1 → 8)
  ───────────────────────────────────────────────────────

  ⏱️  Tick 1: Client A: 5 requests (drain bucket)
  📋 [   client_registered] client=client-A algorithm=TokenBucket
  ✅ [             allowed] client=client-A remaining=4
  ✅ [             allowed] client=client-A remaining=3
  ✅ [             allowed] client=client-A remaining=2
  ✅ [             allowed] client=client-A remaining=1
  ✅ [             allowed] client=client-A remaining=0

  ⏱️  Tick 2: Client A: 1 more request (throttled)
  🚫 [           throttled] client=client-A (retry_after=0.5s)
    Result: RateLimitResult(🚫 THROTTLED, remaining=0/5, retry_after=0.5s)

  ⏱️  Tick 3: Client B: 3 requests (independent bucket)
  📋 [   client_registered] client=client-B algorithm=TokenBucket
  ✅ [             allowed] client=client-B remaining=4
  ✅ [             allowed] client=client-B remaining=3
  ✅ [             allowed] client=client-B remaining=2

  ⏱️  Tick 4: Wait 1 second for token refill

  ⏱️  Tick 5: Client A: 1 request after refill
  ✅ [             allowed] client=client-A remaining=1
    Result: RateLimitResult(✅ ALLOWED, remaining=1/5, retry_after=0.0s)

  ⏱️  Tick 6: Reset Client A's rate limiter
  🔄 [         limit_reset] client=client-A counters reset

  ⏱️  Tick 7: Client A: 3 requests after reset
  ✅ [             allowed] client=client-A remaining=4
  ✅ [             allowed] client=client-A remaining=3
  ✅ [             allowed] client=client-A remaining=2

  ⏱️  Tick 8: Display rate limiter statistics

  ══════════════════════════════════════════════════
  RATE LIMITER STATISTICS
  ══════════════════════════════════════════════════
  Total requests:    12
  Allowed:           11
  Throttled:         1
  Throttle rate:     8.3%
  Allow rate:        91.7%
  ──────────────────────────────────────────────────
        client-A: 8 allowed, 1 throttled (11% throttled)
        client-B: 3 allowed, 0 throttled (0% throttled)
  ══════════════════════════════════════════════════

  Sliding window throttle rate (last 12 requests): 8.3%

  ───────────────────────────────────────────────────────
  SIMULATION COMPLETE
  ───────────────────────────────────────────────────────

=======================================================
  ALGORITHM COMPARISON
=======================================================

  Algorithm: TokenBucket
  ─────────────────────────────────────────────
    Allowed:       5
    Throttled:     5
    Throttle rate: 50.0%
    Usage:         5/5

  Algorithm: SlidingWindowLog
  ─────────────────────────────────────────────
    Allowed:       5
    Throttled:     5
    Throttle rate: 50.0%
    Usage:         5/5

  Algorithm: SlidingWindowCounter
  ─────────────────────────────────────────────
    Allowed:       5
    Throttled:     5
    Throttle rate: 50.0%
    Usage:         5/5

  Algorithm: FixedWindow
  ─────────────────────────────────────────────
    Allowed:       5
    Throttled:     5
    Throttle rate: 50.0%
    Usage:         5/5

=======================================================

========================================
  RUNNING TESTS
========================================

✅ test_token_bucket_basic passed
✅ test_token_bucket_refill passed
✅ test_sliding_window_log passed
✅ test_sliding_window_counter passed
✅ test_fixed_window_counter passed
✅ test_rate_limit_result_headers passed
✅ test_per_client_isolation passed
✅ test_per_client_reset passed
✅ test_middleware_allows passed
✅ test_middleware_throttles passed
✅ test_rule_engine_matching passed
✅ test_rule_creates_algorithm passed
✅ test_distributed_limiter passed
✅ test_stats_tracking passed
✅ test_algorithm_reset passed
✅ test_token_bucket_burst passed
✅ test_throttle_rate_calculation passed
✅ test_sliding_window_log_window_expiry passed
✅ test_in_memory_store_expiry passed

✅ All tests passed!
```
