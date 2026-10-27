# Design Notification System – Low-Level Design Guide

![Bidur Sapkota](https://www.bidursapkota.com.np/images/gravatar.webp "Bidur Sapkota - Developer")&nbsp;[Bidur Sapkota](https://www.bidursapkota.com.np/)

## Table of Contents

1. [Requirements](#requirements)
2. [Key Design Patterns](#key-design-patterns)
3. [Core Enums & Constants](#core-enums--constants)
4. [Notification & Notification Result](#notification--notification-result)
5. [Notification Channel (Strategy Pattern)](#notification-channel-strategy-pattern)
6. [Template Engine](#template-engine)
7. [User Preferences](#user-preferences)
8. [Priority Queue](#priority-queue)
9. [Rate Limiter (Token Bucket)](#rate-limiter-token-bucket)
10. [Retry Manager & Dead Letter Queue](#retry-manager--dead-letter-queue)
11. [Notification Router](#notification-router)
12. [Event System (Observer Pattern)](#event-system-observer-pattern)
13. [Notification Service Facade](#notification-service-facade)
14. [Simulation Engine](#simulation-engine)
15. [Putting It All Together](#putting-it-all-together)
16. [Class Diagram](#class-diagram)
17. [How to Run](#how-to-run)

---

## Requirements

### Functional Requirements

| #  | Requirement                                                                                          |
| -- | ---------------------------------------------------------------------------------------------------- |
| 1  | `send(notification)` dispatches a notification through the appropriate channel (email, SMS, push)     |
| 2  | Multi-channel delivery — a single notification can target multiple channels simultaneously            |
| 3  | User preferences — users can opt in/out of channels and set quiet hours                              |
| 4  | Notification templates — reusable templates with variable substitution for consistent messaging       |
| 5  | Priority routing — CRITICAL notifications bypass rate limits and quiet hours                          |
| 6  | Rate limiting — per-user, per-channel limits prevent notification spam (token bucket algorithm)       |
| 7  | Retry with exponential backoff — failed deliveries are retried up to a configurable max               |
| 8  | Dead letter queue — notifications that exhaust all retries are stored for inspection and replay       |
| 9  | Observer-based event system — listeners react to sent, delivered, failed, and retried events          |
| 10 | Statistics — total sent, delivered, failed, retried, rate-limited, and per-channel breakdown          |

### Non-Functional Requirements

- Clean OOP with SOLID principles.
- Extensible — new channels, templates, and event listeners can be added without modifying core logic.
- Simulatable — the system runs step-by-step for testing and demonstration.

---

## Key Design Patterns

| Pattern   | Where It Is Used                                          | Why                                                                                       |
| --------- | --------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| Strategy  | `NotificationChannel` delivers via email, SMS, or push    | Swap or add channels without changing routing or delivery logic                            |
| Observer  | `NotificationEventListener` reacts to delivery events     | Loggers, metrics, and alerting react independently to notification state changes           |
| Template  | `TemplateEngine` renders message bodies from templates    | Separate message content from delivery mechanics; reuse templates across notifications    |
| Facade    | `NotificationService` exposes a unified API               | Hides the complexity of routing, rate limiting, retries, preferences, and event dispatch   |
| Builder   | `NotificationBuilder` constructs notifications fluently   | Complex notification objects with many optional fields are built step-by-step              |

---

## Core Enums & Constants

```python
import time
import uuid
import heapq
import threading
from enum import Enum
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from typing import Any, Callable
from collections import defaultdict


class NotificationStatus(Enum):
    PENDING = "pending"              # waiting in queue
    SENDING = "sending"              # delivery in progress
    DELIVERED = "delivered"          # successfully delivered
    FAILED = "failed"                # delivery failed
    RETRYING = "retrying"            # waiting for retry
    DEAD = "dead"                    # exhausted all retries (in DLQ)
    RATE_LIMITED = "rate_limited"    # blocked by rate limiter
    FILTERED = "filtered"            # blocked by user preferences
    CANCELLED = "cancelled"          # cancelled before delivery


class NotificationPriority(Enum):
    CRITICAL = 0                     # bypasses rate limits & quiet hours
    HIGH = 1
    MEDIUM = 2
    LOW = 3


class ChannelType(Enum):
    EMAIL = "email"
    SMS = "sms"
    PUSH = "push"


class NotificationEventType(Enum):
    QUEUED = "queued"
    SENDING = "sending"
    DELIVERED = "delivered"
    FAILED = "failed"
    RETRYING = "retrying"
    DEAD_LETTERED = "dead_lettered"
    RATE_LIMITED = "rate_limited"
    FILTERED = "filtered"
    CANCELLED = "cancelled"


class NotificationType(Enum):
    TRANSACTIONAL = "transactional"  # order confirmations, OTPs
    PROMOTIONAL = "promotional"      # marketing, offers
    SYSTEM = "system"                # alerts, maintenance
    SOCIAL = "social"                # follows, likes, comments


# ------------------------------------------------------------------
# System constants
# ------------------------------------------------------------------

DEFAULT_MAX_RETRIES = 3
DEFAULT_RETRY_BACKOFF = 1.0        # base seconds for exponential backoff
DEFAULT_RATE_LIMIT = 10            # tokens per window per user per channel
DEFAULT_RATE_WINDOW = 60.0         # rate limit window in seconds
DEFAULT_QUEUE_CAPACITY = 10_000
STATS_DISPLAY_WIDTH = 55
```

`NotificationStatus` tracks every state a notification can be in. `PENDING` means queued and waiting for delivery. `RATE_LIMITED` means the user has exceeded their notification limit for that channel. `FILTERED` means the user's preferences blocked the notification (e.g., opted out of promotional emails). `NotificationPriority` uses integer values where lower is higher priority — `CRITICAL` (0) bypasses rate limits and quiet hours. `ChannelType` defines the three delivery channels. `NotificationType` classifies notifications so user preferences can filter by type — users might accept transactional notifications but block promotional ones. `DEFAULT_RATE_LIMIT` of 10 means a user can receive at most 10 notifications per channel per 60-second window.

---

## Notification & Notification Result

```python
@dataclass
class NotificationResult:
    """
    The outcome of a notification delivery attempt.

    Created when a notification is delivered (successfully or with
    an error). Contains the delivery status, channel, timing data,
    and any error information.
    """
    notification_id: str
    channel: ChannelType
    success: bool
    value: str = ""
    error: str = ""
    delivery_time_ms: float = 0.0
    attempts: int = 1
    delivered_at: float = field(default_factory=time.time)


class Notification:
    """
    A message to be delivered to a user through one or more channels.

    Encapsulates the recipient, content, channels, priority, and
    delivery metadata. Notifications are first-class objects — they
    can be queued, inspected, retried, and compared.

    Ordering:
        Notifications are compared by (priority, created_at) so
        higher-priority notifications are delivered first. Among
        equal priorities, the oldest notification goes first (FIFO).
    """

    def __init__(
        self,
        user_id: str,
        title: str,
        body: str,
        channels: list[ChannelType] | None = None,
        priority: NotificationPriority = NotificationPriority.MEDIUM,
        notification_type: NotificationType = NotificationType.TRANSACTIONAL,
        max_retries: int = DEFAULT_MAX_RETRIES,
        metadata: dict[str, Any] | None = None,
    ) -> None:
        self.notification_id: str = str(uuid.uuid4())[:8]
        self.user_id: str = user_id
        self.title: str = title
        self.body: str = body
        self.channels: list[ChannelType] = (
            channels or [ChannelType.EMAIL]
        )
        self.priority: NotificationPriority = priority
        self.notification_type: NotificationType = notification_type
        self.status: NotificationStatus = NotificationStatus.PENDING
        self.max_retries: int = max_retries
        self.retry_count: int = 0
        self.metadata: dict[str, Any] = metadata or {}
        self.created_at: float = time.time()
        self.delivered_at: float = 0.0
        self.results: dict[ChannelType, NotificationResult] = {}
        self.error_history: list[str] = []

    @property
    def is_critical(self) -> bool:
        return self.priority == NotificationPriority.CRITICAL

    @property
    def is_terminal(self) -> bool:
        """Check if notification is in a final state."""
        return self.status in (
            NotificationStatus.DELIVERED,
            NotificationStatus.DEAD,
            NotificationStatus.CANCELLED,
            NotificationStatus.FILTERED,
        )

    @property
    def can_retry(self) -> bool:
        return self.retry_count < self.max_retries

    @property
    def all_channels_delivered(self) -> bool:
        """Check if all target channels have been delivered."""
        return all(
            ch in self.results and self.results[ch].success
            for ch in self.channels
        )

    @property
    def failed_channels(self) -> list[ChannelType]:
        """Return channels that failed delivery."""
        return [
            ch for ch in self.channels
            if ch in self.results and not self.results[ch].success
        ]

    @property
    def pending_channels(self) -> list[ChannelType]:
        """Return channels not yet attempted."""
        return [
            ch for ch in self.channels
            if ch not in self.results
        ]

    def __lt__(self, other: "Notification") -> bool:
        """Priority comparison for heap ordering."""
        if self.priority.value != other.priority.value:
            return self.priority.value < other.priority.value
        return self.created_at < other.created_at

    def __repr__(self) -> str:
        channels = ",".join(ch.value for ch in self.channels)
        return (f"Notification(id={self.notification_id}, "
                f"user={self.user_id}, "
                f"priority={self.priority.name}, "
                f"channels=[{channels}], "
                f"status={self.status.value})")


class NotificationBuilder:
    """
    Fluent builder for constructing Notification objects.

    Simplifies creation when many optional fields are involved.

    Usage:
        notification = (NotificationBuilder("user-1", "Title", "Body")
                        .channels([ChannelType.EMAIL, ChannelType.SMS])
                        .priority(NotificationPriority.HIGH)
                        .notification_type(NotificationType.TRANSACTIONAL)
                        .max_retries(5)
                        .metadata({"order_id": "ORD-123"})
                        .build())
    """

    def __init__(self, user_id: str, title: str,
                 body: str) -> None:
        self._user_id = user_id
        self._title = title
        self._body = body
        self._channels: list[ChannelType] = [ChannelType.EMAIL]
        self._priority = NotificationPriority.MEDIUM
        self._type = NotificationType.TRANSACTIONAL
        self._max_retries = DEFAULT_MAX_RETRIES
        self._metadata: dict[str, Any] = {}

    def channels(self, channels: list[ChannelType]
                 ) -> "NotificationBuilder":
        self._channels = channels
        return self

    def priority(self, priority: NotificationPriority
                 ) -> "NotificationBuilder":
        self._priority = priority
        return self

    def notification_type(self, ntype: NotificationType
                          ) -> "NotificationBuilder":
        self._type = ntype
        return self

    def max_retries(self, retries: int
                    ) -> "NotificationBuilder":
        self._max_retries = retries
        return self

    def metadata(self, data: dict[str, Any]
                 ) -> "NotificationBuilder":
        self._metadata = data
        return self

    def build(self) -> Notification:
        return Notification(
            user_id=self._user_id,
            title=self._title,
            body=self._body,
            channels=self._channels,
            priority=self._priority,
            notification_type=self._type,
            max_retries=self._max_retries,
            metadata=self._metadata,
        )
```

`Notification` is the core domain object — it encapsulates recipient, content, target channels, priority, and delivery results. `__lt__` implements heap ordering: lower priority value delivers first (`CRITICAL=0` before `LOW=3`); ties are broken by creation time (FIFO among same priority). `channels` is a list because a single notification can target multiple channels simultaneously (e.g., both email and push). `results` tracks the delivery outcome per channel — a notification targeting email and SMS might succeed on email but fail on SMS. `failed_channels` and `pending_channels` let the retry system know exactly which channels need re-delivery. `NotificationBuilder` uses the builder pattern for fluent construction — when a notification has many optional fields (metadata, retries, type), chaining `.channels(...).priority(...).build()` is cleaner than a constructor with 8 parameters.

---

## Notification Channel (Strategy Pattern)

```python
class NotificationChannel(ABC):
    """
    Defines how a notification is delivered through a specific channel.

    Each channel handles the mechanics of delivery (email via SMTP,
    SMS via API, push via FCM). The router delegates to channels
    without knowing how they work internally.
    """

    @abstractmethod
    def send(self, notification: Notification
             ) -> NotificationResult:
        """Deliver the notification through this channel."""
        ...

    @abstractmethod
    def channel_type(self) -> ChannelType:
        ...

    @abstractmethod
    def name(self) -> str:
        ...


class EmailChannel(NotificationChannel):
    """
    Delivers notifications via email.

    In production, this would integrate with SMTP or an email
    API (SendGrid, SES). Here, it simulates delivery with a
    configurable failure rate for testing retries.
    """

    def __init__(self, failure_rate: float = 0.0) -> None:
        self._failure_rate = failure_rate
        self._sent_count = 0
        self._lock = threading.RLock()

    def send(self, notification: Notification
             ) -> NotificationResult:
        with self._lock:
            start = time.time()
            self._sent_count += 1

            # Simulate failure based on configured rate
            if (self._failure_rate > 0
                    and (self._sent_count % int(
                        1 / self._failure_rate)) == 0):
                elapsed = (time.time() - start) * 1000
                return NotificationResult(
                    notification_id=notification.notification_id,
                    channel=ChannelType.EMAIL,
                    success=False,
                    error="SMTP connection timeout",
                    delivery_time_ms=elapsed,
                    attempts=notification.retry_count + 1,
                )

            elapsed = (time.time() - start) * 1000
            return NotificationResult(
                notification_id=notification.notification_id,
                channel=ChannelType.EMAIL,
                success=True,
                value=(f"Email sent to {notification.user_id}: "
                       f"{notification.title}"),
                delivery_time_ms=elapsed,
                attempts=notification.retry_count + 1,
            )

    def channel_type(self) -> ChannelType:
        return ChannelType.EMAIL

    def name(self) -> str:
        return "Email"


class SMSChannel(NotificationChannel):
    """
    Delivers notifications via SMS.

    In production, this would integrate with Twilio or a similar
    SMS gateway. Truncates long messages to 160 characters.
    """

    def __init__(self, failure_rate: float = 0.0) -> None:
        self._failure_rate = failure_rate
        self._sent_count = 0
        self._lock = threading.RLock()

    def send(self, notification: Notification
             ) -> NotificationResult:
        with self._lock:
            start = time.time()
            self._sent_count += 1

            if (self._failure_rate > 0
                    and (self._sent_count % int(
                        1 / self._failure_rate)) == 0):
                elapsed = (time.time() - start) * 1000
                return NotificationResult(
                    notification_id=notification.notification_id,
                    channel=ChannelType.SMS,
                    success=False,
                    error="SMS gateway unreachable",
                    delivery_time_ms=elapsed,
                    attempts=notification.retry_count + 1,
                )

            # Truncate to SMS length
            body = notification.body[:160]
            elapsed = (time.time() - start) * 1000
            return NotificationResult(
                notification_id=notification.notification_id,
                channel=ChannelType.SMS,
                success=True,
                value=(f"SMS sent to {notification.user_id}: "
                       f"{body}"),
                delivery_time_ms=elapsed,
                attempts=notification.retry_count + 1,
            )

    def channel_type(self) -> ChannelType:
        return ChannelType.SMS

    def name(self) -> str:
        return "SMS"


class PushChannel(NotificationChannel):
    """
    Delivers notifications via push notification.

    In production, this would integrate with FCM (Firebase Cloud
    Messaging) or APNs (Apple Push Notification service).
    """

    def __init__(self, failure_rate: float = 0.0) -> None:
        self._failure_rate = failure_rate
        self._sent_count = 0
        self._lock = threading.RLock()

    def send(self, notification: Notification
             ) -> NotificationResult:
        with self._lock:
            start = time.time()
            self._sent_count += 1

            if (self._failure_rate > 0
                    and (self._sent_count % int(
                        1 / self._failure_rate)) == 0):
                elapsed = (time.time() - start) * 1000
                return NotificationResult(
                    notification_id=notification.notification_id,
                    channel=ChannelType.PUSH,
                    success=False,
                    error="Push token expired",
                    delivery_time_ms=elapsed,
                    attempts=notification.retry_count + 1,
                )

            elapsed = (time.time() - start) * 1000
            return NotificationResult(
                notification_id=notification.notification_id,
                channel=ChannelType.PUSH,
                success=True,
                value=(f"Push sent to {notification.user_id}: "
                       f"{notification.title}"),
                delivery_time_ms=elapsed,
                attempts=notification.retry_count + 1,
            )

    def channel_type(self) -> ChannelType:
        return ChannelType.PUSH

    def name(self) -> str:
        return "Push"
```

`NotificationChannel` defines the contract for all delivery strategies. `EmailChannel` simulates email delivery with a configurable failure rate — in production, this would be an SMTP client or an API call to SendGrid/SES. `SMSChannel` truncates messages to 160 characters (standard SMS length) before sending. `PushChannel` simulates push notification delivery. Each channel is independently testable and swappable. Adding a new channel (e.g., Slack, WhatsApp, In-App) requires only implementing `send`, `channel_type`, and `name` — the router never changes.

---

## Template Engine

```python
class NotificationTemplate:
    """
    A reusable notification template with variable placeholders.

    Templates define the structure of a message. Variables are
    specified with {{variable_name}} syntax and filled in at
    render time.

    Example:
        template = NotificationTemplate(
            template_id="order-confirm",
            title="Order {{order_id}} Confirmed",
            body="Hi {{name}}, your order {{order_id}} has shipped.",
            channels=[ChannelType.EMAIL, ChannelType.PUSH],
            notification_type=NotificationType.TRANSACTIONAL,
        )
    """

    def __init__(
        self,
        template_id: str,
        title: str,
        body: str,
        channels: list[ChannelType] | None = None,
        notification_type: NotificationType = NotificationType.TRANSACTIONAL,
        default_priority: NotificationPriority = NotificationPriority.MEDIUM,
    ) -> None:
        self.template_id = template_id
        self.title = title
        self.body = body
        self.channels = channels or [ChannelType.EMAIL]
        self.notification_type = notification_type
        self.default_priority = default_priority

    def __repr__(self) -> str:
        return (f"NotificationTemplate(id={self.template_id}, "
                f"type={self.notification_type.value})")


class TemplateEngine:
    """
    Manages notification templates and renders them with variables.

    Templates are registered once and reused across notifications.
    Variable substitution uses simple string replacement of
    {{variable_name}} placeholders.
    """

    def __init__(self) -> None:
        self._templates: dict[str, NotificationTemplate] = {}
        self._lock = threading.RLock()

    def register(self, template: NotificationTemplate
                 ) -> None:
        """Register a template for later use."""
        with self._lock:
            self._templates[template.template_id] = template

    def get(self, template_id: str
            ) -> NotificationTemplate | None:
        return self._templates.get(template_id)

    def render(
        self,
        template_id: str,
        user_id: str,
        variables: dict[str, str] | None = None,
        priority: NotificationPriority | None = None,
    ) -> Notification | None:
        """
        Render a template into a Notification object.

        Substitutes {{variable}} placeholders in the title and
        body with values from the variables dict.

        Returns None if the template is not found.
        """
        with self._lock:
            template = self._templates.get(template_id)
            if not template:
                return None

            vars_ = variables or {}
            title = template.title
            body = template.body

            for key, val in vars_.items():
                placeholder = "{{" + key + "}}"
                title = title.replace(placeholder, val)
                body = body.replace(placeholder, val)

            return Notification(
                user_id=user_id,
                title=title,
                body=body,
                channels=list(template.channels),
                priority=priority or template.default_priority,
                notification_type=template.notification_type,
                metadata={"template_id": template.template_id,
                          **vars_},
            )

    @property
    def template_count(self) -> int:
        return len(self._templates)

    def list_templates(self) -> list[str]:
        return list(self._templates.keys())

    def __repr__(self) -> str:
        return (f"TemplateEngine("
                f"templates={len(self._templates)})")
```

`NotificationTemplate` separates message content from delivery logic. The `{{variable}}` syntax is intentionally simple — it avoids the complexity of a full templating engine while supporting the common case of name, order ID, and other dynamic values. `TemplateEngine.render` performs string replacement and constructs a fully formed `Notification` with the template's channels, type, and priority. Registering templates once and rendering them many times ensures consistency — every order confirmation looks the same regardless of which service sends it. The `metadata` dict carries the template ID and variables, providing traceability from a delivered notification back to its source template.

---

## User Preferences

```python
@dataclass
class QuietHours:
    """
    Time window during which non-critical notifications are suppressed.

    Uses 24-hour format. For example, QuietHours(22, 7) means
    notifications are suppressed from 10 PM to 7 AM.
    """
    start_hour: int = 22             # 10 PM
    end_hour: int = 7                # 7 AM

    def is_quiet(self, hour: int) -> bool:
        """
        Check if the given hour falls within quiet hours.

        Handles overnight ranges (e.g., 22 → 7 wraps past midnight).
        """
        if self.start_hour > self.end_hour:
            # Overnight: 22-7 means 22,23,0,1,2,3,4,5,6
            return hour >= self.start_hour or hour < self.end_hour
        # Same-day: 9-17 means 9,10,...,16
        return self.start_hour <= hour < self.end_hour


class UserPreferences:
    """
    Per-user notification preferences.

    Controls which channels and notification types a user accepts,
    and defines quiet hours during which only CRITICAL notifications
    are delivered.
    """

    def __init__(self, user_id: str) -> None:
        self.user_id = user_id
        self._enabled_channels: set[ChannelType] = {
            ChannelType.EMAIL,
            ChannelType.SMS,
            ChannelType.PUSH,
        }
        self._enabled_types: set[NotificationType] = {
            NotificationType.TRANSACTIONAL,
            NotificationType.PROMOTIONAL,
            NotificationType.SYSTEM,
            NotificationType.SOCIAL,
        }
        self._quiet_hours: QuietHours | None = None
        self._do_not_disturb: bool = False

    def enable_channel(self, channel: ChannelType) -> None:
        self._enabled_channels.add(channel)

    def disable_channel(self, channel: ChannelType) -> None:
        self._enabled_channels.discard(channel)

    def enable_type(self, ntype: NotificationType) -> None:
        self._enabled_types.add(ntype)

    def disable_type(self, ntype: NotificationType) -> None:
        self._enabled_types.discard(ntype)

    def set_quiet_hours(self, start: int, end: int) -> None:
        self._quiet_hours = QuietHours(start, end)

    def clear_quiet_hours(self) -> None:
        self._quiet_hours = None

    def set_dnd(self, enabled: bool) -> None:
        self._do_not_disturb = enabled

    def should_deliver(
        self,
        notification: Notification,
        channel: ChannelType,
    ) -> tuple[bool, str]:
        """
        Check if a notification should be delivered to this user
        on the given channel.

        CRITICAL notifications bypass all filters.

        Returns (should_deliver, reason).
        """
        # Critical bypasses everything
        if notification.is_critical:
            return True, "critical — bypasses all filters"

        # Do not disturb
        if self._do_not_disturb:
            return False, "do not disturb enabled"

        # Channel opt-out
        if channel not in self._enabled_channels:
            return (False,
                    f"user opted out of {channel.value}")

        # Notification type opt-out
        if notification.notification_type not in self._enabled_types:
            return (False,
                    f"user opted out of "
                    f"{notification.notification_type.value}")

        # Quiet hours
        if self._quiet_hours:
            import datetime
            current_hour = datetime.datetime.now().hour
            if self._quiet_hours.is_quiet(current_hour):
                return False, "quiet hours active"

        return True, "preferences allow delivery"

    def get_allowed_channels(
        self,
        notification: Notification,
    ) -> list[ChannelType]:
        """
        Filter a notification's channels to only those the user allows.

        CRITICAL notifications return all requested channels.
        """
        if notification.is_critical:
            return list(notification.channels)
        return [
            ch for ch in notification.channels
            if ch in self._enabled_channels
        ]

    def __repr__(self) -> str:
        channels = ",".join(
            ch.value for ch in self._enabled_channels
        )
        return (f"UserPreferences(user={self.user_id}, "
                f"channels=[{channels}], "
                f"dnd={self._do_not_disturb})")


class PreferenceStore:
    """
    Central store for all user preferences.

    Lazily creates default preferences for unknown users.
    Thread-safe for concurrent access.
    """

    def __init__(self) -> None:
        self._preferences: dict[str, UserPreferences] = {}
        self._lock = threading.RLock()

    def get(self, user_id: str) -> UserPreferences:
        """
        Get preferences for a user.

        Creates default preferences if none exist.
        """
        with self._lock:
            if user_id not in self._preferences:
                self._preferences[user_id] = (
                    UserPreferences(user_id)
                )
            return self._preferences[user_id]

    def set(self, prefs: UserPreferences) -> None:
        with self._lock:
            self._preferences[prefs.user_id] = prefs

    @property
    def user_count(self) -> int:
        return len(self._preferences)

    def __repr__(self) -> str:
        return (f"PreferenceStore("
                f"users={len(self._preferences)})")
```

`UserPreferences` implements per-user control over notification delivery. `should_deliver` is the core method — it checks DND, channel opt-out, notification type opt-out, and quiet hours in order. `CRITICAL` notifications bypass all filters — a security alert must always reach the user regardless of preferences. `QuietHours.is_quiet` handles the overnight wrap case correctly: `start=22, end=7` means hours 22, 23, 0, 1, 2, 3, 4, 5, 6 are quiet. `PreferenceStore` lazily creates default preferences for unknown users, so the system works without explicit configuration — users start with all channels and types enabled.

---

## Priority Queue

```python
class NotificationQueue:
    """
    Min-heap priority queue for notifications.

    Uses Python's heapq module. Notifications are ordered by
    (priority_value, created_at) so higher-priority notifications
    are dequeued first. Within the same priority, older
    notifications are dequeued first (FIFO).

    Internal structure:
        heap: [(notif1), (notif2), ...]
        Notification.__lt__ drives the ordering.
    """

    def __init__(self, capacity: int = DEFAULT_QUEUE_CAPACITY
                 ) -> None:
        self._heap: list[Notification] = []
        self._capacity = capacity
        self._lock = threading.RLock()

    def push(self, notification: Notification) -> bool:
        """
        Add a notification to the queue.

        Returns False if the queue is at capacity.
        """
        with self._lock:
            if len(self._heap) >= self._capacity:
                return False
            heapq.heappush(self._heap, notification)
            return True

    def pop(self) -> Notification | None:
        """Remove and return the highest-priority notification."""
        with self._lock:
            if not self._heap:
                return None
            return heapq.heappop(self._heap)

    def peek(self) -> Notification | None:
        """Return the highest-priority notification without removing."""
        with self._lock:
            if not self._heap:
                return None
            return self._heap[0]

    @property
    def size(self) -> int:
        return len(self._heap)

    @property
    def is_empty(self) -> bool:
        return len(self._heap) == 0

    @property
    def is_full(self) -> bool:
        return len(self._heap) >= self._capacity

    def clear(self) -> None:
        with self._lock:
            self._heap.clear()

    def to_list(self) -> list[Notification]:
        """Return notifications in priority order (non-destructive)."""
        with self._lock:
            return sorted(self._heap)

    def __repr__(self) -> str:
        return (f"NotificationQueue(size={len(self._heap)}, "
                f"capacity={self._capacity})")
```

`NotificationQueue` wraps Python's `heapq` to provide a thread-safe min-heap. `heapq.heappush` and `heapq.heappop` maintain the heap invariant in O(log n). The ordering is driven by `Notification.__lt__` — `CRITICAL` notifications float to the top. `capacity` prevents unbounded queue growth; `push` returns `False` when full, giving the caller a chance to apply backpressure.

---

## Rate Limiter (Token Bucket)

```python
class TokenBucket:
    """
    Token bucket rate limiter for a single user-channel pair.

    Tokens are added at a fixed rate. Each notification consumes
    one token. If no tokens are available, the notification is
    rate-limited.

    Token refill is lazy — tokens are calculated at check time
    based on elapsed time since the last refill.

    Example with capacity=10, refill_rate=10/60:
        - Bucket starts full with 10 tokens
        - Each notification consumes 1 token
        - Tokens refill at 10 per 60 seconds
        - After 10 notifications, the bucket is empty
        - After 6 seconds, 1 token has refilled
    """

    def __init__(
        self,
        capacity: int = DEFAULT_RATE_LIMIT,
        refill_rate: float = DEFAULT_RATE_LIMIT / DEFAULT_RATE_WINDOW,
    ) -> None:
        self._capacity = capacity
        self._refill_rate = refill_rate       # tokens per second
        self._tokens = float(capacity)
        self._last_refill = time.time()
        self._lock = threading.RLock()

    def try_consume(self) -> bool:
        """
        Try to consume one token.

        Returns True if a token was available (notification allowed).
        Returns False if the bucket is empty (rate-limited).
        """
        with self._lock:
            self._refill()
            if self._tokens >= 1.0:
                self._tokens -= 1.0
                return True
            return False

    def _refill(self) -> None:
        """Add tokens based on elapsed time since last refill."""
        now = time.time()
        elapsed = now - self._last_refill
        new_tokens = elapsed * self._refill_rate
        self._tokens = min(
            self._capacity, self._tokens + new_tokens,
        )
        self._last_refill = now

    @property
    def available_tokens(self) -> float:
        with self._lock:
            self._refill()
            return self._tokens

    def __repr__(self) -> str:
        return (f"TokenBucket(tokens={self._tokens:.1f}, "
                f"capacity={self._capacity})")


class RateLimiter:
    """
    Per-user, per-channel rate limiter.

    Maintains a separate token bucket for each (user_id, channel)
    pair. CRITICAL notifications bypass rate limits entirely.

    Example:
        - User "alice" on EMAIL channel: 10 notifications / 60s
        - User "alice" on SMS channel: 10 notifications / 60s
        - These are independent limits
    """

    def __init__(
        self,
        capacity: int = DEFAULT_RATE_LIMIT,
        window_seconds: float = DEFAULT_RATE_WINDOW,
    ) -> None:
        self._capacity = capacity
        self._refill_rate = capacity / window_seconds
        self._buckets: dict[str, TokenBucket] = {}
        self._lock = threading.RLock()

    def _get_bucket(self, user_id: str,
                    channel: ChannelType) -> TokenBucket:
        """Get or create a bucket for a user-channel pair."""
        key = f"{user_id}:{channel.value}"
        if key not in self._buckets:
            self._buckets[key] = TokenBucket(
                capacity=self._capacity,
                refill_rate=self._refill_rate,
            )
        return self._buckets[key]

    def allow(self, notification: Notification,
              channel: ChannelType) -> bool:
        """
        Check if a notification is allowed by rate limits.

        CRITICAL notifications always pass.
        """
        if notification.is_critical:
            return True

        with self._lock:
            bucket = self._get_bucket(
                notification.user_id, channel,
            )
            return bucket.try_consume()

    def get_tokens(self, user_id: str,
                   channel: ChannelType) -> float:
        """Return available tokens for a user-channel pair."""
        with self._lock:
            bucket = self._get_bucket(user_id, channel)
            return bucket.available_tokens

    def __repr__(self) -> str:
        return (f"RateLimiter(buckets={len(self._buckets)}, "
                f"capacity={self._capacity})")
```

`TokenBucket` implements the token bucket algorithm with lazy refill — instead of a background thread adding tokens, `_refill` computes how many tokens have accumulated since the last check based on elapsed time. This is efficient (no timers) and accurate. `try_consume` atomically refills and attempts to take a token. `RateLimiter` maintains separate buckets per `(user_id, channel)` pair — a user's email limit is independent of their SMS limit. `CRITICAL` notifications bypass rate limits entirely because a security alert should never be dropped due to a marketing email spree.

---

## Retry Manager & Dead Letter Queue

```python
class DeadLetterQueue:
    """
    Stores notifications that have exhausted all retries.

    Notifications in the DLQ can be inspected, replayed
    (resubmitted with fresh retry counts), or purged.
    The DLQ prevents silent delivery failures.
    """

    def __init__(self) -> None:
        self._notifications: list[Notification] = []
        self._lock = threading.RLock()

    def add(self, notification: Notification) -> None:
        """Move a dead notification to the DLQ."""
        with self._lock:
            notification.status = NotificationStatus.DEAD
            self._notifications.append(notification)

    def replay(
        self,
        notification_id: str,
        submit_fn: Callable[[Notification], None],
    ) -> bool:
        """
        Resubmit a DLQ notification with fresh retry state.

        Resets the retry count and status, then submits
        the notification back to the service.
        """
        with self._lock:
            for i, notif in enumerate(self._notifications):
                if notif.notification_id == notification_id:
                    notif.retry_count = 0
                    notif.status = NotificationStatus.PENDING
                    notif.error_history.clear()
                    notif.results.clear()
                    self._notifications.pop(i)
                    submit_fn(notif)
                    return True
            return False

    def peek(self) -> list[Notification]:
        """Return all notifications in the DLQ."""
        with self._lock:
            return list(self._notifications)

    @property
    def size(self) -> int:
        return len(self._notifications)

    def clear(self) -> None:
        with self._lock:
            self._notifications.clear()

    def display(self) -> None:
        print(f"\n  {'─'*STATS_DISPLAY_WIDTH}")
        print(f"  DEAD LETTER QUEUE "
              f"({len(self._notifications)} notifications)")
        print(f"  {'─'*STATS_DISPLAY_WIDTH}")
        for notif in self._notifications:
            errors = "; ".join(notif.error_history[-3:])
            channels = ",".join(
                ch.value for ch in notif.channels
            )
            print(f"  💀 {notif.title} "
                  f"→ [{channels}] "
                  f"(retries={notif.retry_count}, "
                  f"errors=[{errors}])")
        print(f"  {'─'*STATS_DISPLAY_WIDTH}")

    def __repr__(self) -> str:
        return (f"DeadLetterQueue("
                f"size={len(self._notifications)})")


class RetryManager:
    """
    Handles retry logic for failed notification deliveries.

    On failure, checks if the notification can retry. If yes,
    applies exponential backoff and resubmits. If no, moves
    the notification to the Dead Letter Queue.

    Backoff formula: base × 2^(retry_count - 1)
    With base=1.0: 1s → 2s → 4s → 8s → ...
    """

    def __init__(
        self,
        dlq: DeadLetterQueue,
        backoff_base: float = DEFAULT_RETRY_BACKOFF,
    ) -> None:
        self._dlq = dlq
        self._backoff_base = backoff_base

    def handle_failure(
        self,
        notification: Notification,
        resubmit_fn: Callable[[Notification], None],
    ) -> bool:
        """
        Handle a failed notification delivery.

        Returns True if the notification will be retried,
        False if it was sent to the DLQ.
        """
        if notification.can_retry:
            notification.retry_count += 1
            notification.status = NotificationStatus.RETRYING
            # In production, schedule retry after backoff
            # Here we resubmit immediately for simulation
            notification.status = NotificationStatus.PENDING
            resubmit_fn(notification)
            return True
        else:
            self._dlq.add(notification)
            return False

    def compute_backoff(self, notification: Notification
                        ) -> float:
        """
        Compute backoff delay for current retry attempt.

        Formula: base × 2^(retry_count - 1)
        """
        return (self._backoff_base
                * (2 ** (notification.retry_count - 1)))

    @property
    def dlq(self) -> DeadLetterQueue:
        return self._dlq
```

`DeadLetterQueue` is the final destination for notifications that exhaust all retries. Without a DLQ, failed deliveries would silently disappear — the DLQ preserves them for debugging, alerting, and manual replay. `replay` resets a DLQ notification's state (retry count, status, error history, results) and resubmits it to the service. `RetryManager` orchestrates the retry-or-DLQ decision. On failure, it checks `can_retry`, increments the counter, and resubmits. The exponential backoff formula (`base × 2^(retry-1)`) prevents retry storms — if an email provider is down, hammering it with retries every millisecond makes things worse.

---

## Notification Router

```python
class NotificationRouter:
    """
    Routes notifications to the correct channel(s) for delivery.

    Applies user preferences and rate limiting before delivering.
    Handles multi-channel fanout — a single notification can be
    delivered through email, SMS, and push simultaneously.

    Delivery pipeline per channel:
        1. Check user preferences (opt-out, quiet hours)
        2. Check rate limit
        3. Deliver via channel
        4. Record result
    """

    def __init__(
        self,
        preference_store: PreferenceStore,
        rate_limiter: RateLimiter,
    ) -> None:
        self._channels: dict[ChannelType, NotificationChannel] = {}
        self._preference_store = preference_store
        self._rate_limiter = rate_limiter
        self._lock = threading.RLock()

    def register_channel(self, channel: NotificationChannel
                         ) -> None:
        """Register a delivery channel."""
        with self._lock:
            self._channels[channel.channel_type()] = channel

    def route(self, notification: Notification
              ) -> list[NotificationResult]:
        """
        Route a notification through all its target channels.

        For each channel:
        1. Check user preferences
        2. Check rate limit
        3. Deliver
        4. Record result

        Returns a list of results, one per channel attempted.
        """
        with self._lock:
            results: list[NotificationResult] = []
            prefs = self._preference_store.get(
                notification.user_id
            )

            for ch_type in notification.channels:
                # Skip already-delivered channels (for retries)
                if (ch_type in notification.results
                        and notification.results[ch_type].success):
                    continue

                # 1. Check user preferences
                allowed, reason = prefs.should_deliver(
                    notification, ch_type,
                )
                if not allowed:
                    result = NotificationResult(
                        notification_id=(
                            notification.notification_id),
                        channel=ch_type,
                        success=False,
                        error=f"filtered: {reason}",
                    )
                    notification.results[ch_type] = result
                    results.append(result)
                    continue

                # 2. Check rate limit
                if not self._rate_limiter.allow(
                    notification, ch_type,
                ):
                    result = NotificationResult(
                        notification_id=(
                            notification.notification_id),
                        channel=ch_type,
                        success=False,
                        error="rate limited",
                    )
                    notification.results[ch_type] = result
                    results.append(result)
                    continue

                # 3. Deliver
                channel = self._channels.get(ch_type)
                if not channel:
                    result = NotificationResult(
                        notification_id=(
                            notification.notification_id),
                        channel=ch_type,
                        success=False,
                        error=(f"no handler for "
                               f"{ch_type.value}"),
                    )
                    notification.results[ch_type] = result
                    results.append(result)
                    continue

                notification.status = NotificationStatus.SENDING
                result = channel.send(notification)
                notification.results[ch_type] = result
                results.append(result)

            return results

    def get_channel(self, ch_type: ChannelType
                    ) -> NotificationChannel | None:
        return self._channels.get(ch_type)

    @property
    def registered_channels(self) -> list[ChannelType]:
        return list(self._channels.keys())

    def __repr__(self) -> str:
        channels = ",".join(
            ch.value for ch in self._channels.keys()
        )
        return f"NotificationRouter(channels=[{channels}])"
```

`NotificationRouter` is the delivery pipeline. For each target channel, it runs a three-step filter: user preferences → rate limit → channel delivery. This pipeline is sequential per channel, meaning a notification can succeed on email but get rate-limited on SMS. The router skips already-delivered channels on retry — if email succeeded but SMS failed, retrying only re-attempts SMS. The `register_channel` method allows dynamic channel registration, so the system can add or remove channels at runtime (e.g., disable SMS during a carrier outage).

---

## Event System (Observer Pattern)

```python
@dataclass
class NotificationEvent:
    """An event emitted during the notification lifecycle."""
    event_id: str
    event_type: NotificationEventType
    notification_id: str
    user_id: str
    title: str
    channel: str = ""
    detail: str = ""
    timestamp: float = field(default_factory=time.time)


class NotificationEventListener(ABC):
    """
    Observer that reacts to notification lifecycle events.

    Implementations receive all events and decide which to
    handle. Listeners are decoupled from the delivery pipeline.
    """

    @abstractmethod
    def on_event(self, event: NotificationEvent) -> None:
        ...

    @abstractmethod
    def listener_id(self) -> str:
        ...


class NotificationLogger(NotificationEventListener):
    """Logs all notification events to the console."""

    def on_event(self, event: NotificationEvent) -> None:
        emoji = {
            NotificationEventType.QUEUED: "📥",
            NotificationEventType.SENDING: "📤",
            NotificationEventType.DELIVERED: "✅",
            NotificationEventType.FAILED: "❌",
            NotificationEventType.RETRYING: "🔄",
            NotificationEventType.DEAD_LETTERED: "💀",
            NotificationEventType.RATE_LIMITED: "🚦",
            NotificationEventType.FILTERED: "🚫",
            NotificationEventType.CANCELLED: "⛔",
        }
        icon = emoji.get(event.event_type, "📌")
        channel = (f" [{event.channel}]"
                   if event.channel else "")
        print(f"  {icon} [{event.event_type.value:>15}] "
              f"{event.title}{channel} {event.detail}")

    def listener_id(self) -> str:
        return "notification-logger"


class NotificationMetricsCollector(NotificationEventListener):
    """Collects delivery metrics per channel."""

    def __init__(self) -> None:
        self._channel_success: dict[str, int] = defaultdict(int)
        self._channel_failure: dict[str, int] = defaultdict(int)
        self._delivery_times: list[float] = []

    def on_event(self, event: NotificationEvent) -> None:
        if event.event_type == NotificationEventType.DELIVERED:
            self._channel_success[event.channel] += 1
            if "ms" in event.detail:
                try:
                    ms_str = (event.detail
                              .split("in ")[1].split("ms")[0])
                    self._delivery_times.append(float(ms_str))
                except (IndexError, ValueError):
                    pass
        elif event.event_type == NotificationEventType.FAILED:
            self._channel_failure[event.channel] += 1

    @property
    def avg_delivery_time_ms(self) -> float:
        if not self._delivery_times:
            return 0.0
        return (sum(self._delivery_times)
                / len(self._delivery_times))

    def display(self) -> None:
        print(f"\n  {'─'*STATS_DISPLAY_WIDTH}")
        print(f"  CHANNEL METRICS")
        print(f"  {'─'*STATS_DISPLAY_WIDTH}")
        all_channels = set(
            list(self._channel_success.keys())
            + list(self._channel_failure.keys())
        )
        for ch in sorted(all_channels):
            s = self._channel_success.get(ch, 0)
            f = self._channel_failure.get(ch, 0)
            total = s + f
            rate = (s / total * 100) if total > 0 else 0
            print(f"  {ch:>6}: {s} delivered, "
                  f"{f} failed ({rate:.0f}% success)")
        if self._delivery_times:
            print(f"  Avg delivery: "
                  f"{self.avg_delivery_time_ms:.1f}ms")
        print(f"  {'─'*STATS_DISPLAY_WIDTH}")

    def listener_id(self) -> str:
        return "notification-metrics"


class NotificationEventBus:
    """
    Central event dispatcher for notification events.

    Manages listener registration and broadcasts events to
    all registered listeners. Thread-safe.
    """

    def __init__(self) -> None:
        self._listeners: list[NotificationEventListener] = []
        self._event_log: list[NotificationEvent] = []
        self._lock = threading.RLock()

    def register(self, listener: NotificationEventListener
                 ) -> None:
        with self._lock:
            self._listeners.append(listener)

    def unregister(self, listener_id: str) -> None:
        with self._lock:
            self._listeners = [
                l for l in self._listeners
                if l.listener_id() != listener_id
            ]

    def emit(self, event: NotificationEvent) -> None:
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

`NotificationEventListener` is the observer interface. `NotificationLogger` prints every event with emoji-coded log lines — deliveries, failures, and rate limits are visually distinct. `NotificationMetricsCollector` tracks per-channel success/failure counts and delivery times. The `NotificationEventBus` dispatches events to all registered listeners, isolating each listener from failures in others. Adding a new listener (e.g., Slack alerting on DLQ events, Prometheus counters) requires only implementing `on_event` and `listener_id`.

---

## Notification Service Facade

```python
class NotificationStats:
    """Aggregate notification service statistics."""

    def __init__(self) -> None:
        self.total_sent: int = 0
        self.delivered: int = 0
        self.failed: int = 0
        self.retried: int = 0
        self.dead_lettered: int = 0
        self.rate_limited: int = 0
        self.filtered: int = 0
        self.total_delivery_ms: float = 0.0

    @property
    def total_processed(self) -> int:
        return self.delivered + self.failed

    @property
    def delivery_rate(self) -> float:
        if self.total_processed == 0:
            return 0.0
        return self.delivered / self.total_processed

    @property
    def avg_delivery_ms(self) -> float:
        if self.delivered == 0:
            return 0.0
        return self.total_delivery_ms / self.delivered

    def display(self) -> None:
        print(f"\n  {'═'*STATS_DISPLAY_WIDTH}")
        print(f"  NOTIFICATION SERVICE STATISTICS")
        print(f"  {'═'*STATS_DISPLAY_WIDTH}")
        print(f"  Total sent:        {self.total_sent}")
        print(f"  Delivered:         {self.delivered}")
        print(f"  Failed:            {self.failed}")
        print(f"  Retried:           {self.retried}")
        print(f"  Dead-lettered:     {self.dead_lettered}")
        print(f"  Rate-limited:      {self.rate_limited}")
        print(f"  Filtered:          {self.filtered}")
        print(f"  Delivery rate:     {self.delivery_rate:.1%}")
        print(f"  Avg delivery:      {self.avg_delivery_ms:.1f}ms")
        print(f"  {'═'*STATS_DISPLAY_WIDTH}")

    def reset(self) -> None:
        self.total_sent = 0
        self.delivered = 0
        self.failed = 0
        self.retried = 0
        self.dead_lettered = 0
        self.rate_limited = 0
        self.filtered = 0
        self.total_delivery_ms = 0.0


class NotificationService:
    """
    Facade that orchestrates the entire notification system.

    Provides a simple API that hides the complexity of
    channels, routing, rate limiting, retries, preferences,
    templates, events, and the dead letter queue.
    """

    def __init__(
        self,
        rate_limit_capacity: int = DEFAULT_RATE_LIMIT,
        rate_limit_window: float = DEFAULT_RATE_WINDOW,
        max_retries: int = DEFAULT_MAX_RETRIES,
        retry_backoff: float = DEFAULT_RETRY_BACKOFF,
    ) -> None:
        # Event system
        self._event_bus = NotificationEventBus()
        self._logger = NotificationLogger()
        self._metrics = NotificationMetricsCollector()
        self._event_bus.register(self._logger)
        self._event_bus.register(self._metrics)

        # Core components
        self._preference_store = PreferenceStore()
        self._rate_limiter = RateLimiter(
            capacity=rate_limit_capacity,
            window_seconds=rate_limit_window,
        )
        self._router = NotificationRouter(
            preference_store=self._preference_store,
            rate_limiter=self._rate_limiter,
        )
        self._template_engine = TemplateEngine()
        self._queue = NotificationQueue()
        self._dlq = DeadLetterQueue()
        self._retry_manager = RetryManager(
            self._dlq, backoff_base=retry_backoff,
        )
        self._stats = NotificationStats()
        self._all_notifications: dict[str, Notification] = {}
        self._max_retries = max_retries
        self._lock = threading.RLock()

    # ------------------------------------------------------------------
    # Channel registration
    # ------------------------------------------------------------------

    def register_channel(self, channel: NotificationChannel
                         ) -> None:
        """Register a delivery channel."""
        self._router.register_channel(channel)

    # ------------------------------------------------------------------
    # Template management
    # ------------------------------------------------------------------

    def register_template(self, template: NotificationTemplate
                          ) -> None:
        """Register a notification template."""
        self._template_engine.register(template)

    def send_template(
        self,
        template_id: str,
        user_id: str,
        variables: dict[str, str] | None = None,
        priority: NotificationPriority | None = None,
    ) -> str | None:
        """
        Render a template and send the resulting notification.

        Returns the notification ID, or None if template not found.
        """
        notification = self._template_engine.render(
            template_id, user_id, variables, priority,
        )
        if not notification:
            return None
        return self.send(notification)

    # ------------------------------------------------------------------
    # User preferences
    # ------------------------------------------------------------------

    def get_preferences(self, user_id: str
                        ) -> UserPreferences:
        return self._preference_store.get(user_id)

    def set_preferences(self, prefs: UserPreferences
                        ) -> None:
        self._preference_store.set(prefs)

    # ------------------------------------------------------------------
    # Core operations
    # ------------------------------------------------------------------

    def send(self, notification: Notification) -> str:
        """
        Send a notification through its target channels.

        Routes through the delivery pipeline:
        1. Queue the notification
        2. Route through channels (preferences → rate limit → deliver)
        3. Handle results (retry failures, DLQ exhausted)
        4. Emit events
        """
        with self._lock:
            self._all_notifications[
                notification.notification_id
            ] = notification
            self._stats.total_sent += 1

            self._emit(NotificationEventType.QUEUED,
                       notification, "",
                       detail="queued for delivery")

            # Route and deliver
            results = self._router.route(notification)

            # Process results per channel
            has_delivery = False
            has_failure = False

            for result in results:
                if result.success:
                    has_delivery = True
                    self._stats.delivered += 1
                    self._stats.total_delivery_ms += (
                        result.delivery_time_ms
                    )
                    self._emit(
                        NotificationEventType.DELIVERED,
                        notification,
                        result.channel.value,
                        detail=(f"in "
                                f"{result.delivery_time_ms:.1f}"
                                f"ms"),
                    )
                elif "rate limited" in result.error:
                    self._stats.rate_limited += 1
                    self._emit(
                        NotificationEventType.RATE_LIMITED,
                        notification,
                        result.channel.value,
                        detail=result.error,
                    )
                elif "filtered" in result.error:
                    self._stats.filtered += 1
                    self._emit(
                        NotificationEventType.FILTERED,
                        notification,
                        result.channel.value,
                        detail=result.error,
                    )
                else:
                    has_failure = True
                    self._stats.failed += 1
                    self._emit(
                        NotificationEventType.FAILED,
                        notification,
                        result.channel.value,
                        detail=f"error={result.error}",
                    )

            # Determine overall status
            if notification.all_channels_delivered:
                notification.status = (
                    NotificationStatus.DELIVERED
                )
                notification.delivered_at = time.time()
            elif has_failure:
                # Handle retry for failed channels
                notification.error_history.extend(
                    r.error for r in results
                    if not r.success
                    and "filtered" not in r.error
                    and "rate limited" not in r.error
                )
                will_retry = self._retry_manager.handle_failure(
                    notification,
                    lambda n: self._resubmit(n),
                )
                if will_retry:
                    self._stats.retried += 1
                    self._emit(
                        NotificationEventType.RETRYING,
                        notification, "",
                        detail=(
                            f"attempt "
                            f"{notification.retry_count}/"
                            f"{notification.max_retries}"),
                    )
                else:
                    self._stats.dead_lettered += 1
                    self._emit(
                        NotificationEventType.DEAD_LETTERED,
                        notification, "",
                        detail=(
                            f"moved to DLQ after "
                            f"{notification.retry_count} "
                            f"retries"),
                    )
            elif not has_delivery:
                # All channels were filtered or rate-limited
                notification.status = (
                    NotificationStatus.FILTERED
                )

            return notification.notification_id

    def _resubmit(self, notification: Notification) -> None:
        """Resubmit a notification for retry (failed channels only)."""
        results = self._router.route(notification)

        for result in results:
            if result.success:
                self._stats.delivered += 1
                self._stats.total_delivery_ms += (
                    result.delivery_time_ms
                )
                self._emit(
                    NotificationEventType.DELIVERED,
                    notification,
                    result.channel.value,
                    detail=(f"in "
                            f"{result.delivery_time_ms:.1f}ms "
                            f"(retry)"),
                )
            else:
                self._stats.failed += 1
                self._emit(
                    NotificationEventType.FAILED,
                    notification,
                    result.channel.value,
                    detail=f"error={result.error} (retry)",
                )

        if notification.all_channels_delivered:
            notification.status = (
                NotificationStatus.DELIVERED
            )
        elif not notification.can_retry:
            self._dlq.add(notification)
            self._stats.dead_lettered += 1
            self._emit(
                NotificationEventType.DEAD_LETTERED,
                notification, "",
                detail=(f"moved to DLQ after "
                        f"{notification.retry_count} retries"),
            )

    def cancel(self, notification_id: str) -> bool:
        """Cancel a pending notification."""
        with self._lock:
            notif = self._all_notifications.get(
                notification_id
            )
            if (notif and notif.status
                    == NotificationStatus.PENDING):
                notif.status = NotificationStatus.CANCELLED
                self._emit(
                    NotificationEventType.CANCELLED,
                    notif, "", detail="",
                )
                return True
            return False

    # ------------------------------------------------------------------
    # Inspection
    # ------------------------------------------------------------------

    @property
    def stats(self) -> NotificationStats:
        return self._stats

    @property
    def dlq(self) -> DeadLetterQueue:
        return self._dlq

    @property
    def event_bus(self) -> NotificationEventBus:
        return self._event_bus

    @property
    def metrics(self) -> NotificationMetricsCollector:
        return self._metrics

    @property
    def queue_size(self) -> int:
        return self._queue.size

    def get_notification(self, notification_id: str
                         ) -> Notification | None:
        return self._all_notifications.get(notification_id)

    def display_status(self) -> None:
        self._stats.display()
        self._metrics.display()
        if self._dlq.size > 0:
            self._dlq.display()

    # ------------------------------------------------------------------
    # Internal helpers
    # ------------------------------------------------------------------

    def _emit(
        self,
        event_type: NotificationEventType,
        notification: Notification,
        channel: str,
        detail: str = "",
    ) -> None:
        event = NotificationEvent(
            event_id=str(uuid.uuid4())[:8],
            event_type=event_type,
            notification_id=notification.notification_id,
            user_id=notification.user_id,
            title=notification.title,
            channel=channel,
            detail=detail,
        )
        self._event_bus.emit(event)
```

`NotificationService` is the facade — it wires together the router, channels, rate limiter, preference store, template engine, retry manager, dead letter queue, event bus, logger, and metrics collector. `send` is the core method: it queues the notification, routes it through the delivery pipeline, processes per-channel results, handles retries or DLQ for failures, and emits events throughout. The `send_template` method combines template rendering and delivery in one call. Users interact with `send`, `send_template`, `cancel`, `get_preferences`, and `display_status` without knowing about any internal complexity.

---

## Simulation Engine

```python
class NotificationScenario:
    """
    Tick-based simulation engine for notification demos.

    Each tick represents a logical time step. Actions are registered
    at specific ticks and execute in order when the simulation runs.
    """

    def __init__(self, service: NotificationService) -> None:
        self._service = service
        self._events: list[tuple[int, str, callable]] = []

    def at_tick(self, tick: int, description: str,
                action: callable) -> None:
        self._events.append((tick, description, action))

    def run(self) -> None:
        if not self._events:
            print("  No events to simulate.")
            return

        max_tick = max(e[0] for e in self._events)
        sorted_events = sorted(
            self._events, key=lambda e: e[0],
        )

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
                action()
            except Exception as e:
                print(f"  ⚠️  Error at tick {tick}: {e}")

        print(f"\n  {'─'*STATS_DISPLAY_WIDTH}")
        print(f"  SIMULATION COMPLETE")
        print(f"  {'─'*STATS_DISPLAY_WIDTH}")
```

`NotificationScenario` mirrors the simulation engine from other guides. Actions are registered at specific ticks and execute in order, making demos reproducible and readable.

---

## Putting It All Together

```python
def main_scenario() -> None:
    """Run the main notification system demonstration."""

    print("\n╔══════════════════════════════════════════════╗")
    print("║      NOTIFICATION SYSTEM                     ║")
    print("║  Channels: Email, SMS, Push                   ║")
    print("║  Rate Limit: 10 / 60s per user per channel    ║")
    print("║  Max Retries: 3                               ║")
    print("╚══════════════════════════════════════════════╝")

    service = NotificationService(
        rate_limit_capacity=10,
        rate_limit_window=60.0,
        max_retries=3,
    )

    # Register channels
    service.register_channel(EmailChannel())
    service.register_channel(SMSChannel())
    service.register_channel(PushChannel())

    scenario = NotificationScenario(service)

    # --- Send a simple email notification ---
    scenario.at_tick(1, "Send a simple email notification",
                     lambda: service.send(Notification(
                         user_id="alice",
                         title="Welcome!",
                         body="Welcome to the platform, Alice!",
                         channels=[ChannelType.EMAIL],
                         priority=NotificationPriority.MEDIUM,
                     )))

    # --- Send multi-channel notification ---
    scenario.at_tick(2, "Send multi-channel notification (email + SMS + push)",
                     lambda: service.send(Notification(
                         user_id="bob",
                         title="Order Shipped",
                         body="Your order #1234 has shipped!",
                         channels=[ChannelType.EMAIL,
                                   ChannelType.SMS,
                                   ChannelType.PUSH],
                         priority=NotificationPriority.HIGH,
                     )))

    # --- Send using a template ---
    service.register_template(NotificationTemplate(
        template_id="order-confirm",
        title="Order {{order_id}} Confirmed",
        body="Hi {{name}}, your order {{order_id}} is confirmed. Total: {{total}}.",
        channels=[ChannelType.EMAIL, ChannelType.PUSH],
        notification_type=NotificationType.TRANSACTIONAL,
    ))

    scenario.at_tick(3, "Send notification from template",
                     lambda: service.send_template(
                         "order-confirm",
                         user_id="charlie",
                         variables={
                             "name": "Charlie",
                             "order_id": "ORD-5678",
                             "total": "$99.99",
                         },
                     ))

    # --- Send critical notification (bypasses all filters) ---
    scenario.at_tick(4, "Send CRITICAL security alert",
                     lambda: service.send(Notification(
                         user_id="alice",
                         title="Security Alert",
                         body="Suspicious login from new device.",
                         channels=[ChannelType.EMAIL,
                                   ChannelType.SMS,
                                   ChannelType.PUSH],
                         priority=NotificationPriority.CRITICAL,
                         notification_type=NotificationType.SYSTEM,
                     )))

    # --- Configure user preferences and test filtering ---
    def setup_prefs_and_send():
        prefs = service.get_preferences("dave")
        prefs.disable_channel(ChannelType.SMS)
        prefs.disable_type(NotificationType.PROMOTIONAL)
        service.set_preferences(prefs)

        service.send(Notification(
            user_id="dave",
            title="50% Off Sale!",
            body="Don't miss our biggest sale.",
            channels=[ChannelType.EMAIL, ChannelType.SMS],
            priority=NotificationPriority.LOW,
            notification_type=NotificationType.PROMOTIONAL,
        ))

    scenario.at_tick(5, "Send promotional to Dave (opted out of promo + SMS)",
                     setup_prefs_and_send)

    # --- Test with a failing channel ---
    def test_failing_channel():
        # Replace email channel with a failing one
        service.register_channel(
            EmailChannel(failure_rate=1.0)  # fails every time
        )

        service.send(Notification(
            user_id="eve",
            title="Invoice Ready",
            body="Your invoice #INV-001 is ready.",
            channels=[ChannelType.EMAIL],
            priority=NotificationPriority.HIGH,
            max_retries=2,
        ))

        # Restore healthy channel
        service.register_channel(EmailChannel())

    scenario.at_tick(6, "Send via failing email channel → retry → DLQ",
                     test_failing_channel)

    # --- Use NotificationBuilder ---
    scenario.at_tick(7, "Send notification via builder pattern",
                     lambda: service.send(
                         NotificationBuilder("frank", "Payment Received",
                                             "We received your payment of $250.")
                         .channels([ChannelType.EMAIL, ChannelType.PUSH])
                         .priority(NotificationPriority.HIGH)
                         .notification_type(NotificationType.TRANSACTIONAL)
                         .metadata({"payment_id": "PAY-789"})
                         .build()
                     ))

    # --- Display statistics ---
    scenario.at_tick(8, "Display service statistics",
                     lambda: service.display_status())

    scenario.run()


# ------------------------------------------------------------------
# Rate Limiting Demo
# ------------------------------------------------------------------

def demo_rate_limiting() -> None:
    """Demonstrate per-user, per-channel rate limiting."""

    print("\n" + "=" * STATS_DISPLAY_WIDTH)
    print("  RATE LIMITING DEMO")
    print("=" * STATS_DISPLAY_WIDTH)

    service = NotificationService(
        rate_limit_capacity=3,         # only 3 per window
        rate_limit_window=60.0,
    )
    service.register_channel(EmailChannel())

    print("\n  Sending 5 emails to alice "
          "(limit: 3 per 60s)...\n")

    for i in range(5):
        service.send(Notification(
            user_id="alice",
            title=f"Email #{i+1}",
            body=f"This is email number {i+1}.",
            channels=[ChannelType.EMAIL],
            priority=NotificationPriority.MEDIUM,
        ))

    print(f"\n  Rate-limited: {service.stats.rate_limited}")
    print(f"  Delivered: {service.stats.delivered}")
    service.stats.display()
    print("=" * STATS_DISPLAY_WIDTH)


# ------------------------------------------------------------------
# User Preferences Demo
# ------------------------------------------------------------------

def demo_user_preferences() -> None:
    """Demonstrate user preference filtering."""

    print("\n" + "=" * STATS_DISPLAY_WIDTH)
    print("  USER PREFERENCES DEMO")
    print("=" * STATS_DISPLAY_WIDTH)

    service = NotificationService()
    service.register_channel(EmailChannel())
    service.register_channel(SMSChannel())
    service.register_channel(PushChannel())

    # Setup: alice opts out of SMS and promotional
    prefs = service.get_preferences("alice")
    prefs.disable_channel(ChannelType.SMS)
    prefs.disable_type(NotificationType.PROMOTIONAL)
    service.set_preferences(prefs)

    print("\n  Alice's preferences: "
          "no SMS, no promotional\n")

    # Transactional email — should deliver
    print("  --- Transactional email ---")
    service.send(Notification(
        user_id="alice",
        title="Order Confirmed",
        body="Your order is confirmed.",
        channels=[ChannelType.EMAIL],
        notification_type=NotificationType.TRANSACTIONAL,
    ))

    # Promotional email + SMS — email filtered, SMS filtered
    print("\n  --- Promotional via email + SMS ---")
    service.send(Notification(
        user_id="alice",
        title="Big Sale!",
        body="50% off everything.",
        channels=[ChannelType.EMAIL, ChannelType.SMS],
        notification_type=NotificationType.PROMOTIONAL,
    ))

    # Critical via all channels — bypasses all
    print("\n  --- Critical via all channels ---")
    service.send(Notification(
        user_id="alice",
        title="Account Compromised",
        body="Change your password immediately.",
        channels=[ChannelType.EMAIL,
                  ChannelType.SMS,
                  ChannelType.PUSH],
        priority=NotificationPriority.CRITICAL,
        notification_type=NotificationType.SYSTEM,
    ))

    service.stats.display()
    print("=" * STATS_DISPLAY_WIDTH)


# ------------------------------------------------------------------
# Template Demo
# ------------------------------------------------------------------

def demo_templates() -> None:
    """Demonstrate template-based notifications."""

    print("\n" + "=" * STATS_DISPLAY_WIDTH)
    print("  TEMPLATE ENGINE DEMO")
    print("=" * STATS_DISPLAY_WIDTH)

    service = NotificationService()
    service.register_channel(EmailChannel())
    service.register_channel(PushChannel())

    # Register templates
    service.register_template(NotificationTemplate(
        template_id="welcome",
        title="Welcome, {{name}}!",
        body="Hi {{name}}, thanks for joining {{platform}}.",
        channels=[ChannelType.EMAIL],
        notification_type=NotificationType.TRANSACTIONAL,
    ))

    service.register_template(NotificationTemplate(
        template_id="otp",
        title="Your OTP Code",
        body="Your verification code is {{code}}. "
             "Expires in {{expiry}} minutes.",
        channels=[ChannelType.EMAIL, ChannelType.PUSH],
        notification_type=NotificationType.TRANSACTIONAL,
        default_priority=NotificationPriority.HIGH,
    ))

    service.register_template(NotificationTemplate(
        template_id="promo",
        title="{{discount}} Off — {{event_name}}!",
        body="Use code {{code}} for {{discount}} off "
             "during {{event_name}}.",
        channels=[ChannelType.EMAIL, ChannelType.PUSH],
        notification_type=NotificationType.PROMOTIONAL,
        default_priority=NotificationPriority.LOW,
    ))

    # Send from templates
    print("\n  --- Welcome email ---")
    service.send_template("welcome", "alice", {
        "name": "Alice",
        "platform": "ShopEasy",
    })

    print("\n  --- OTP code ---")
    service.send_template("otp", "bob", {
        "code": "847291",
        "expiry": "5",
    })

    print("\n  --- Promotional ---")
    service.send_template("promo", "charlie", {
        "discount": "30%",
        "event_name": "Summer Sale",
        "code": "SUMMER30",
    })

    service.stats.display()
    print("=" * STATS_DISPLAY_WIDTH)


# ------------------------------------------------------------------
# Dead Letter Queue Demo
# ------------------------------------------------------------------

def demo_dead_letter_queue() -> None:
    """Demonstrate retry exhaustion and DLQ."""

    print("\n" + "=" * STATS_DISPLAY_WIDTH)
    print("  DEAD LETTER QUEUE DEMO")
    print("=" * STATS_DISPLAY_WIDTH)

    service = NotificationService(max_retries=2)
    service.register_channel(
        EmailChannel(failure_rate=1.0),  # always fails
    )

    print("\n  Sending email via always-failing channel...\n")
    service.send(Notification(
        user_id="alice",
        title="Important Update",
        body="This will fail.",
        channels=[ChannelType.EMAIL],
        priority=NotificationPriority.HIGH,
        max_retries=2,
    ))

    print(f"\n  DLQ size: {service.dlq.size}")
    service.dlq.display()

    # Replay from DLQ with a healthy channel
    print("\n  Replacing with healthy channel "
          "and replaying from DLQ...\n")
    service.register_channel(EmailChannel())
    dlq_items = service.dlq.peek()
    if dlq_items:
        service.dlq.replay(
            dlq_items[0].notification_id,
            service.send,
        )

    service.stats.display()
    print("=" * STATS_DISPLAY_WIDTH)


# ------------------------------------------------------------------
# Unit Tests
# ------------------------------------------------------------------

def test_notification_creation() -> None:
    """Test notification creation with defaults."""
    notif = Notification(
        user_id="alice",
        title="Test",
        body="Test body",
    )
    assert notif.user_id == "alice"
    assert notif.priority == NotificationPriority.MEDIUM
    assert notif.status == NotificationStatus.PENDING
    assert notif.channels == [ChannelType.EMAIL]
    assert notif.retry_count == 0
    assert not notif.is_terminal
    print("✅ test_notification_creation passed")


def test_notification_builder() -> None:
    """Test fluent builder constructs correctly."""
    notif = (NotificationBuilder("bob", "Title", "Body")
             .channels([ChannelType.SMS, ChannelType.PUSH])
             .priority(NotificationPriority.HIGH)
             .notification_type(NotificationType.PROMOTIONAL)
             .max_retries(5)
             .metadata({"key": "val"})
             .build())

    assert notif.user_id == "bob"
    assert notif.title == "Title"
    assert notif.channels == [ChannelType.SMS, ChannelType.PUSH]
    assert notif.priority == NotificationPriority.HIGH
    assert notif.notification_type == NotificationType.PROMOTIONAL
    assert notif.max_retries == 5
    assert notif.metadata == {"key": "val"}
    print("✅ test_notification_builder passed")


def test_priority_ordering() -> None:
    """Test notifications are ordered by priority."""
    queue = NotificationQueue()

    low = Notification("a", "Low", "body",
                       priority=NotificationPriority.LOW)
    critical = Notification("a", "Critical", "body",
                            priority=NotificationPriority.CRITICAL)
    high = Notification("a", "High", "body",
                        priority=NotificationPriority.HIGH)

    queue.push(low)
    queue.push(critical)
    queue.push(high)

    n1 = queue.pop()
    n2 = queue.pop()
    n3 = queue.pop()

    assert n1.title == "Critical"
    assert n2.title == "High"
    assert n3.title == "Low"
    print("✅ test_priority_ordering passed")


def test_fifo_within_priority() -> None:
    """Test FIFO ordering among same-priority notifications."""
    queue = NotificationQueue()

    n1 = Notification("a", "First", "body",
                      priority=NotificationPriority.HIGH)
    time.sleep(0.01)
    n2 = Notification("a", "Second", "body",
                      priority=NotificationPriority.HIGH)
    time.sleep(0.01)
    n3 = Notification("a", "Third", "body",
                      priority=NotificationPriority.HIGH)

    queue.push(n1)
    queue.push(n2)
    queue.push(n3)

    assert queue.pop().title == "First"
    assert queue.pop().title == "Second"
    assert queue.pop().title == "Third"
    print("✅ test_fifo_within_priority passed")


def test_queue_capacity() -> None:
    """Test queue rejects when full."""
    queue = NotificationQueue(capacity=2)

    assert queue.push(Notification("a", "1", "body"))
    assert queue.push(Notification("a", "2", "body"))
    assert not queue.push(Notification("a", "3", "body"))
    assert queue.size == 2
    print("✅ test_queue_capacity passed")


def test_email_channel() -> None:
    """Test email channel delivery."""
    channel = EmailChannel()
    notif = Notification("alice", "Test", "Body")
    result = channel.send(notif)

    assert result.success
    assert result.channel == ChannelType.EMAIL
    assert "alice" in result.value
    print("✅ test_email_channel passed")


def test_sms_channel() -> None:
    """Test SMS channel delivery and truncation."""
    channel = SMSChannel()
    notif = Notification("bob", "Test", "x" * 200)
    result = channel.send(notif)

    assert result.success
    assert result.channel == ChannelType.SMS
    print("✅ test_sms_channel passed")


def test_push_channel() -> None:
    """Test push channel delivery."""
    channel = PushChannel()
    notif = Notification("charlie", "Test", "Body")
    result = channel.send(notif)

    assert result.success
    assert result.channel == ChannelType.PUSH
    print("✅ test_push_channel passed")


def test_template_rendering() -> None:
    """Test template variable substitution."""
    engine = TemplateEngine()
    engine.register(NotificationTemplate(
        template_id="test",
        title="Hello {{name}}",
        body="Your code is {{code}}.",
    ))

    notif = engine.render("test", "alice", {
        "name": "Alice",
        "code": "1234",
    })

    assert notif is not None
    assert notif.title == "Hello Alice"
    assert notif.body == "Your code is 1234."
    assert notif.metadata["template_id"] == "test"
    print("✅ test_template_rendering passed")


def test_template_not_found() -> None:
    """Test rendering a non-existent template."""
    engine = TemplateEngine()
    result = engine.render("nonexistent", "alice")
    assert result is None
    print("✅ test_template_not_found passed")


def test_user_preferences_channel_opt_out() -> None:
    """Test channel opt-out filtering."""
    prefs = UserPreferences("alice")
    prefs.disable_channel(ChannelType.SMS)

    notif = Notification("alice", "Test", "body",
                         channels=[ChannelType.SMS])

    allowed, reason = prefs.should_deliver(
        notif, ChannelType.SMS,
    )
    assert not allowed
    assert "opted out" in reason
    print("✅ test_user_preferences_channel_opt_out passed")


def test_user_preferences_type_opt_out() -> None:
    """Test notification type opt-out filtering."""
    prefs = UserPreferences("alice")
    prefs.disable_type(NotificationType.PROMOTIONAL)

    notif = Notification(
        "alice", "Sale!", "body",
        notification_type=NotificationType.PROMOTIONAL,
    )

    allowed, reason = prefs.should_deliver(
        notif, ChannelType.EMAIL,
    )
    assert not allowed
    assert "opted out" in reason
    print("✅ test_user_preferences_type_opt_out passed")


def test_critical_bypasses_preferences() -> None:
    """Test CRITICAL bypasses all preference filters."""
    prefs = UserPreferences("alice")
    prefs.set_dnd(True)
    prefs.disable_channel(ChannelType.EMAIL)

    notif = Notification(
        "alice", "Alert", "body",
        priority=NotificationPriority.CRITICAL,
    )

    allowed, reason = prefs.should_deliver(
        notif, ChannelType.EMAIL,
    )
    assert allowed
    assert "critical" in reason
    print("✅ test_critical_bypasses_preferences passed")


def test_quiet_hours() -> None:
    """Test quiet hours detection."""
    qh = QuietHours(22, 7)

    assert qh.is_quiet(23)      # 11 PM — quiet
    assert qh.is_quiet(0)       # midnight — quiet
    assert qh.is_quiet(6)       # 6 AM — quiet
    assert not qh.is_quiet(7)   # 7 AM — not quiet
    assert not qh.is_quiet(14)  # 2 PM — not quiet
    assert qh.is_quiet(22)      # 10 PM — quiet
    print("✅ test_quiet_hours passed")


def test_token_bucket() -> None:
    """Test token bucket allows and rejects."""
    bucket = TokenBucket(capacity=3, refill_rate=0.0)

    assert bucket.try_consume()      # 2 left
    assert bucket.try_consume()      # 1 left
    assert bucket.try_consume()      # 0 left
    assert not bucket.try_consume()  # rejected
    print("✅ test_token_bucket passed")


def test_rate_limiter_per_user_channel() -> None:
    """Test independent limits per user-channel pair."""
    limiter = RateLimiter(capacity=2, window_seconds=60.0)

    n1 = Notification("alice", "T1", "b")
    n2 = Notification("alice", "T2", "b")
    n3 = Notification("alice", "T3", "b")

    assert limiter.allow(n1, ChannelType.EMAIL)
    assert limiter.allow(n2, ChannelType.EMAIL)
    assert not limiter.allow(n3, ChannelType.EMAIL)

    # SMS is a separate bucket — should still allow
    assert limiter.allow(n1, ChannelType.SMS)
    print("✅ test_rate_limiter_per_user_channel passed")


def test_critical_bypasses_rate_limit() -> None:
    """Test CRITICAL bypasses rate limits."""
    limiter = RateLimiter(capacity=1, window_seconds=60.0)

    normal = Notification("alice", "Normal", "b")
    limiter.allow(normal, ChannelType.EMAIL)  # exhaust

    critical = Notification(
        "alice", "Critical", "b",
        priority=NotificationPriority.CRITICAL,
    )
    assert limiter.allow(critical, ChannelType.EMAIL)
    print("✅ test_critical_bypasses_rate_limit passed")


def test_dead_letter_queue() -> None:
    """Test DLQ stores and replays notifications."""
    dlq = DeadLetterQueue()

    notif = Notification("alice", "Dead", "body",
                         max_retries=0)
    notif.status = NotificationStatus.DEAD
    notif.error_history.append("permanent error")
    dlq.add(notif)

    assert dlq.size == 1

    replayed: list[Notification] = []
    dlq.replay(notif.notification_id,
               lambda n: replayed.append(n))

    assert dlq.size == 0
    assert len(replayed) == 1
    assert replayed[0].retry_count == 0
    print("✅ test_dead_letter_queue passed")


def test_retry_manager() -> None:
    """Test retry manager retries or sends to DLQ."""
    dlq = DeadLetterQueue()
    manager = RetryManager(dlq)

    # Notification with retries left
    notif = Notification("alice", "Retry", "body",
                         max_retries=2)
    resubmitted: list[Notification] = []
    will_retry = manager.handle_failure(
        notif, lambda n: resubmitted.append(n),
    )
    assert will_retry
    assert notif.retry_count == 1
    assert len(resubmitted) == 1

    # Notification with no retries left
    notif2 = Notification("bob", "NoRetry", "body",
                          max_retries=0)
    will_retry = manager.handle_failure(
        notif2, lambda n: resubmitted.append(n),
    )
    assert not will_retry
    assert dlq.size == 1
    print("✅ test_retry_manager passed")


def test_preference_store_lazy_creation() -> None:
    """Test preference store creates defaults for unknown users."""
    store = PreferenceStore()

    prefs = store.get("new-user")
    assert prefs.user_id == "new-user"
    assert store.user_count == 1
    print("✅ test_preference_store_lazy_creation passed")


def test_notification_result() -> None:
    """Test notification result properties."""
    result = NotificationResult(
        notification_id="abc",
        channel=ChannelType.EMAIL,
        success=True,
        value="sent",
        delivery_time_ms=5.5,
    )
    assert result.success
    assert result.delivery_time_ms == 5.5
    assert result.channel == ChannelType.EMAIL
    print("✅ test_notification_result passed")


def test_full_lifecycle() -> None:
    """Test full send → deliver lifecycle."""
    service = NotificationService()
    service.register_channel(EmailChannel())

    nid = service.send(Notification(
        user_id="alice",
        title="Lifecycle Test",
        body="Full lifecycle.",
        channels=[ChannelType.EMAIL],
    ))

    notif = service.get_notification(nid)
    assert notif is not None
    assert notif.status == NotificationStatus.DELIVERED
    assert service.stats.delivered == 1
    print("✅ test_full_lifecycle passed")


def test_multi_channel_delivery() -> None:
    """Test delivery across multiple channels."""
    service = NotificationService()
    service.register_channel(EmailChannel())
    service.register_channel(SMSChannel())
    service.register_channel(PushChannel())

    nid = service.send(Notification(
        user_id="bob",
        title="Multi-Channel",
        body="Delivered everywhere.",
        channels=[ChannelType.EMAIL,
                  ChannelType.SMS,
                  ChannelType.PUSH],
    ))

    notif = service.get_notification(nid)
    assert notif is not None
    assert notif.all_channels_delivered
    assert service.stats.delivered == 3
    print("✅ test_multi_channel_delivery passed")


def test_notification_stats() -> None:
    """Test statistics tracking."""
    service = NotificationService()
    service.register_channel(EmailChannel())

    service.send(Notification("a", "T1", "b"))
    service.send(Notification("a", "T2", "b"))

    assert service.stats.total_sent == 2
    assert service.stats.delivered == 2
    assert service.stats.delivery_rate == 1.0
    print("✅ test_notification_stats passed")


def run_all_tests() -> None:
    print("\n" + "=" * 40)
    print("  RUNNING TESTS")
    print("=" * 40 + "\n")
    test_notification_creation()
    test_notification_builder()
    test_priority_ordering()
    test_fifo_within_priority()
    test_queue_capacity()
    test_email_channel()
    test_sms_channel()
    test_push_channel()
    test_template_rendering()
    test_template_not_found()
    test_user_preferences_channel_opt_out()
    test_user_preferences_type_opt_out()
    test_critical_bypasses_preferences()
    test_quiet_hours()
    test_token_bucket()
    test_rate_limiter_per_user_channel()
    test_critical_bypasses_rate_limit()
    test_dead_letter_queue()
    test_retry_manager()
    test_preference_store_lazy_creation()
    test_notification_result()
    test_full_lifecycle()
    test_multi_channel_delivery()
    test_notification_stats()
    print("\n✅ All tests passed!\n")


if __name__ == "__main__":
    main_scenario()
    demo_rate_limiting()
    demo_user_preferences()
    demo_templates()
    demo_dead_letter_queue()
    run_all_tests()
```

Each test isolates a single behavior. `test_notification_creation` verifies default values are applied correctly. `test_notification_builder` confirms the fluent builder constructs the notification with all specified fields. `test_priority_ordering` is the core test — it verifies that `CRITICAL` notifications dequeue before `HIGH` before `LOW`. `test_fifo_within_priority` uses `time.sleep(0.01)` to ensure distinct creation timestamps, verifying FIFO among same-priority notifications. `test_template_rendering` confirms `{{variable}}` substitution works in both title and body. `test_user_preferences_channel_opt_out` and `test_user_preferences_type_opt_out` verify that disabled channels and types are filtered. `test_critical_bypasses_preferences` confirms that `CRITICAL` notifications bypass DND and channel opt-outs. `test_quiet_hours` tests the overnight wrap case (22:00 → 07:00). `test_token_bucket` verifies tokens are consumed and exhausted. `test_rate_limiter_per_user_channel` confirms independent limits per user-channel pair. `test_full_lifecycle` runs a complete send → deliver lifecycle through the facade. `test_multi_channel_delivery` verifies simultaneous delivery across email, SMS, and push.

---

## Class Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                    NotificationService                              │
│──────────────────────────────────────────────────────────────────│
│ - _router: NotificationRouter                                       │
│ - _queue: NotificationQueue                                         │
│ - _rate_limiter: RateLimiter                                        │
│ - _preference_store: PreferenceStore                                │
│ - _template_engine: TemplateEngine                                  │
│ - _retry_manager: RetryManager                                      │
│ - _dlq: DeadLetterQueue                                            │
│ - _event_bus: NotificationEventBus                                  │
│ - _stats: NotificationStats                                         │
│──────────────────────────────────────────────────────────────────│
│ + send(notification) → notification_id                              │
│ + send_template(template_id, user_id, vars) → str | None           │
│ + cancel(notification_id) → bool                                    │
│ + register_channel(channel) → None                                  │
│ + register_template(template) → None                                │
│ + get_preferences(user_id) → UserPreferences                        │
│ + display_status() → None                                           │
└───────┬──────────┬──────────┬────────────────────────────────────┘
        │          │          │
   uses ▼     uses ▼     uses ▼
┌──────────────────────────────────────────────────────────────────┐
│  NotificationRouter                                                 │
│──────────────────────────────────────────────────────────────────│
│ - _channels: dict[ChannelType, NotificationChannel]                 │
│ - _preference_store: PreferenceStore                                │
│ - _rate_limiter: RateLimiter                                        │
│──────────────────────────────────────────────────────────────────│
│ + register_channel(channel) → None                                  │
│ + route(notification) → list[NotificationResult]                    │
│   Pipeline: preferences → rate limit → channel.send()               │
└──────────┬────────────────────────────────────────────────────────┘
           │ delegates to
           ▼
┌──────────────────────────────────────────────────────────────────┐
│  «abstract» NotificationChannel                                     │
│──────────────────────────────────────────────────────────────────│
│ + send(notification) → NotificationResult                           │
│ + channel_type() → ChannelType                                      │
│ + name() → str                                                      │
└──────────┬────────────────────────────────────────────────────────┘
           │ implemented by
  ┌────────┼──────────────────┐
  ▼        ▼                  ▼
Email    SMS               Push
Channel  Channel            Channel

┌──────────────────────────────────────────────────────────────────┐
│  Notification                                                       │
│──────────────────────────────────────────────────────────────────│
│ + notification_id: str                                              │
│ + user_id: str                                                      │
│ + title: str                                                        │
│ + body: str                                                         │
│ + channels: list[ChannelType]                                       │
│ + priority: NotificationPriority                                    │
│ + notification_type: NotificationType                               │
│ + status: NotificationStatus                                        │
│ + max_retries: int                                                  │
│ + retry_count: int                                                  │
│ + results: dict[ChannelType, NotificationResult]                    │
│──────────────────────────────────────────────────────────────────│
│ + is_critical → bool                                                │
│ + is_terminal → bool                                                │
│ + can_retry → bool                                                  │
│ + all_channels_delivered → bool                                     │
│ + failed_channels → list[ChannelType]                               │
│ + __lt__(other) → bool   (heap ordering)                            │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  NotificationQueue (Heap-Based)                                     │
│──────────────────────────────────────────────────────────────────│
│ - _heap: list[Notification]                                         │
│ - _capacity: int                                                    │
│──────────────────────────────────────────────────────────────────│
│ + push(notification) → bool             O(log n)                    │
│ + pop() → Notification | None           O(log n)                    │
│ + peek() → Notification | None          O(1)                        │
│ + size → int                             O(1)                        │
│ + is_empty → bool                                                   │
│ + is_full → bool                                                    │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  RateLimiter                                                        │
│──────────────────────────────────────────────────────────────────│
│ - _buckets: dict[str, TokenBucket]                                  │
│ - _capacity: int                                                    │
│──────────────────────────────────────────────────────────────────│
│ + allow(notification, channel) → bool                               │
│   CRITICAL always passes                                            │
│ + get_tokens(user_id, channel) → float                              │
└──────────┬───────────────────────────────────────────────────────┘
           │ contains
           ▼
┌──────────────────────────────────────────────────────────────────┐
│  TokenBucket                                                        │
│──────────────────────────────────────────────────────────────────│
│ - _capacity: int                                                    │
│ - _tokens: float                                                    │
│ - _refill_rate: float        tokens/second                          │
│──────────────────────────────────────────────────────────────────│
│ + try_consume() → bool                                              │
│ - _refill() → None           lazy refill on access                  │
│ + available_tokens → float                                          │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  RetryManager                                                       │
│──────────────────────────────────────────────────────────────────│
│ - _dlq: DeadLetterQueue                                             │
│ - _backoff_base: float                                              │
│──────────────────────────────────────────────────────────────────│
│ + handle_failure(notif, resubmit_fn) → bool                         │
│   → True: notification will be retried                              │
│   → False: notification sent to DLQ                                 │
│ + compute_backoff(notif) → float                                    │
│   base × 2^(retry_count - 1)                                       │
└──────────┬───────────────────────────────────────────────────────┘
           │ uses
           ▼
┌──────────────────────────────────────────────────────────────────┐
│  DeadLetterQueue                                                    │
│──────────────────────────────────────────────────────────────────│
│ - _notifications: list[Notification]                                │
│──────────────────────────────────────────────────────────────────│
│ + add(notification) → None                                          │
│ + replay(notification_id, submit_fn) → bool                         │
│ + peek() → list[Notification]                                       │
│ + display() → None                                                  │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  TemplateEngine                                                     │
│──────────────────────────────────────────────────────────────────│
│ - _templates: dict[str, NotificationTemplate]                       │
│──────────────────────────────────────────────────────────────────│
│ + register(template) → None                                         │
│ + render(template_id, user_id, vars) → Notification | None          │
│ + get(template_id) → NotificationTemplate | None                    │
└──────────┬───────────────────────────────────────────────────────┘
           │ contains
           ▼
┌──────────────────────────────────────────────────────────────────┐
│  NotificationTemplate                                               │
│──────────────────────────────────────────────────────────────────│
│ + template_id: str                                                  │
│ + title: str              (with {{var}} placeholders)               │
│ + body: str               (with {{var}} placeholders)               │
│ + channels: list[ChannelType]                                       │
│ + notification_type: NotificationType                               │
│ + default_priority: NotificationPriority                            │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  PreferenceStore                                                    │
│──────────────────────────────────────────────────────────────────│
│ - _preferences: dict[str, UserPreferences]                          │
│──────────────────────────────────────────────────────────────────│
│ + get(user_id) → UserPreferences     (lazy creation)                │
│ + set(prefs) → None                                                 │
└──────────┬───────────────────────────────────────────────────────┘
           │ contains
           ▼
┌──────────────────────────────────────────────────────────────────┐
│  UserPreferences                                                    │
│──────────────────────────────────────────────────────────────────│
│ - _enabled_channels: set[ChannelType]                               │
│ - _enabled_types: set[NotificationType]                             │
│ - _quiet_hours: QuietHours | None                                   │
│ - _do_not_disturb: bool                                             │
│──────────────────────────────────────────────────────────────────│
│ + should_deliver(notif, channel) → (bool, str)                      │
│ + enable_channel(channel) / disable_channel(channel)                │
│ + enable_type(type) / disable_type(type)                            │
│ + set_quiet_hours(start, end) / set_dnd(bool)                       │
│   CRITICAL bypasses all filters                                     │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  «abstract» NotificationEventListener                               │
│──────────────────────────────────────────────────────────────────│
│ + on_event(event) → None                                            │
│ + listener_id() → str                                               │
└──────────┬────────────────────────────────────────────────────────┘
           │ implemented by
  ┌────────┴──────────────────┐
  ▼                           ▼
Notification              Notification
Logger                    MetricsCollector

┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│ NotificationResult│ │ NotificationStats│ │ NotificationEvent│
│──────────────────│ │──────────────────│ │──────────────────│
│ + notif_id       │ │ + total_sent     │ │ + event_id       │
│ + channel        │ │ + delivered      │ │ + event_type     │
│ + success        │ │ + failed         │ │ + notif_id       │
│ + value          │ │ + retried        │ │ + user_id        │
│ + error          │ │ + dead_lettered  │ │ + title          │
│ + delivery_ms    │ │ + rate_limited   │ │ + channel        │
│ + attempts       │ │ + filtered       │ │ + detail         │
└──────────────────┘ │ + delivery_rate  │ │ + timestamp      │
                     │ + avg_delivery   │ └──────────────────┘
                     └──────────────────┘
                                          ┌──────────────────┐
┌──────────────────┐ ┌──────────────────┐ │ NotificationEvent│
│ NotificationStatus│ │ NotificationPri. │ │ Bus              │
│ (Enum)           │ │ (Enum)           │ │──────────────────│
│──────────────────│ │──────────────────│ │ - _listeners     │
│ PENDING          │ │ CRITICAL = 0     │ │ - _event_log     │
│ SENDING          │ │ HIGH = 1         │ │──────────────────│
│ DELIVERED        │ │ MEDIUM = 2       │ │ + register()     │
│ FAILED           │ │ LOW = 3          │ │ + emit()         │
│ RETRYING         │ └──────────────────┘ │ + unregister()   │
│ DEAD             │                      └──────────────────┘
│ RATE_LIMITED     │ ┌──────────────────┐
│ FILTERED         │ │ ChannelType      │ ┌──────────────────┐
│ CANCELLED        │ │ (Enum)           │ │ NotificationType │
└──────────────────┘ │──────────────────│ │ (Enum)           │
                     │ EMAIL            │ │──────────────────│
┌──────────────────┐ │ SMS              │ │ TRANSACTIONAL    │
│ QuietHours       │ │ PUSH             │ │ PROMOTIONAL      │
│──────────────────│ └──────────────────┘ │ SYSTEM           │
│ + start_hour     │                      │ SOCIAL           │
│ + end_hour       │ ┌──────────────────┐ └──────────────────┘
│──────────────────│ │ NotificationScen.│
│ + is_quiet(hour) │ │──────────────────│ ┌──────────────────┐
│   handles wrap   │ │ - _service       │ │ NotificationBld. │
└──────────────────┘ │ - _events        │ │──────────────────│
                     │──────────────────│ │ + channels()     │
                     │ + at_tick()      │ │ + priority()     │
                     │ + run()          │ │ + metadata()     │
                     └──────────────────┘ │ + build()        │
                                          └──────────────────┘
```

---

## How to Run

Save all code into a single file `notification_system.py` and run:

```bash
python notification_system.py
```

### Example Output

```
╔══════════════════════════════════════════════╗
║      NOTIFICATION SYSTEM                     ║
║  Channels: Email, SMS, Push                   ║
║  Rate Limit: 10 / 60s per user per channel    ║
║  Max Retries: 3                               ║
╚══════════════════════════════════════════════╝

  ───────────────────────────────────────────────────────
  SIMULATION START (ticks: 1 → 8)
  ───────────────────────────────────────────────────────

  ⏱️  Tick 1: Send a simple email notification
  📥 [         queued] Welcome! queued for delivery
  ✅ [      delivered] Welcome! [email] in 0.0ms

  ⏱️  Tick 2: Send multi-channel notification (email + SMS + push)
  📥 [         queued] Order Shipped queued for delivery
  ✅ [      delivered] Order Shipped [email] in 0.0ms
  ✅ [      delivered] Order Shipped [sms] in 0.0ms
  ✅ [      delivered] Order Shipped [push] in 0.0ms

  ⏱️  Tick 3: Send notification from template
  📥 [         queued] Order ORD-5678 Confirmed queued for delivery
  ✅ [      delivered] Order ORD-5678 Confirmed [email] in 0.0ms
  ✅ [      delivered] Order ORD-5678 Confirmed [push] in 0.0ms

  ⏱️  Tick 4: Send CRITICAL security alert
  📥 [         queued] Security Alert queued for delivery
  ✅ [      delivered] Security Alert [email] in 0.0ms
  ✅ [      delivered] Security Alert [sms] in 0.0ms
  ✅ [      delivered] Security Alert [push] in 0.0ms

  ⏱️  Tick 5: Send promotional to Dave (opted out of promo + SMS)
  📥 [         queued] 50% Off Sale! queued for delivery
  🚫 [       filtered] 50% Off Sale! [email] filtered: user opted out of promotional
  🚫 [       filtered] 50% Off Sale! [sms] filtered: user opted out of promotional

  ⏱️  Tick 6: Send via failing email channel → retry → DLQ
  📥 [         queued] Invoice Ready queued for delivery
  ❌ [         failed] Invoice Ready [email] error=SMTP connection timeout
  🔄 [       retrying] Invoice Ready attempt 1/2
  ❌ [         failed] Invoice Ready [email] error=SMTP connection timeout
  💀 [  dead_lettered] Invoice Ready moved to DLQ after 2 retries

  ⏱️  Tick 7: Send notification via builder pattern
  📥 [         queued] Payment Received queued for delivery
  ✅ [      delivered] Payment Received [email] in 0.0ms
  ✅ [      delivered] Payment Received [push] in 0.0ms

  ⏱️  Tick 8: Display service statistics

  ═══════════════════════════════════════════════════════
  NOTIFICATION SERVICE STATISTICS
  ═══════════════════════════════════════════════════════
  Total sent:        7
  Delivered:         11
  Failed:            2
  Retried:           1
  Dead-lettered:     1
  Rate-limited:      0
  Filtered:          2
  Delivery rate:     84.6%
  Avg delivery:      0.0ms
  ═══════════════════════════════════════════════════════

  ───────────────────────────────────────────────────────
  CHANNEL METRICS
  ───────────────────────────────────────────────────────
   email: 5 delivered, 2 failed (71% success)
    push: 4 delivered, 0 failed (100% success)
     sms: 2 delivered, 0 failed (100% success)
  ───────────────────────────────────────────────────────

  ───────────────────────────────────────────────────────
  DEAD LETTER QUEUE (1 notifications)
  ───────────────────────────────────────────────────────
  💀 Invoice Ready → [email] (retries=2, errors=[SMTP connection timeout])
  ───────────────────────────────────────────────────────

  ───────────────────────────────────────────────────────
  SIMULATION COMPLETE
  ───────────────────────────────────────────────────────

=======================================================
  RATE LIMITING DEMO
=======================================================

  Sending 5 emails to alice (limit: 3 per 60s)...

  📥 [         queued] Email #1 queued for delivery
  ✅ [      delivered] Email #1 [email] in 0.0ms
  📥 [         queued] Email #2 queued for delivery
  ✅ [      delivered] Email #2 [email] in 0.0ms
  📥 [         queued] Email #3 queued for delivery
  ✅ [      delivered] Email #3 [email] in 0.0ms
  📥 [         queued] Email #4 queued for delivery
  🚦 [   rate_limited] Email #4 [email] rate limited
  📥 [         queued] Email #5 queued for delivery
  🚦 [   rate_limited] Email #5 [email] rate limited

  Rate-limited: 2
  Delivered: 3

========================================
  RUNNING TESTS
========================================

✅ test_notification_creation passed
✅ test_notification_builder passed
✅ test_priority_ordering passed
✅ test_fifo_within_priority passed
✅ test_queue_capacity passed
✅ test_email_channel passed
✅ test_sms_channel passed
✅ test_push_channel passed
✅ test_template_rendering passed
✅ test_template_not_found passed
✅ test_user_preferences_channel_opt_out passed
✅ test_user_preferences_type_opt_out passed
✅ test_critical_bypasses_preferences passed
✅ test_quiet_hours passed
✅ test_token_bucket passed
✅ test_rate_limiter_per_user_channel passed
✅ test_critical_bypasses_rate_limit passed
✅ test_dead_letter_queue passed
✅ test_retry_manager passed
✅ test_preference_store_lazy_creation passed
✅ test_notification_result passed
✅ test_full_lifecycle passed
✅ test_multi_channel_delivery passed
✅ test_notification_stats passed

✅ All tests passed!
```
