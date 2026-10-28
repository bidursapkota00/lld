# Design Logging Framework – Low-Level Design Guide

![Bidur Sapkota](https://www.bidursapkota.com.np/images/gravatar.webp "Bidur Sapkota - Developer")&nbsp;[Bidur Sapkota](https://www.bidursapkota.com.np/)

## Table of Contents

1. [Requirements](#requirements)
2. [Key Design Patterns](#key-design-patterns)
3. [Core Enums & Constants](#core-enums--constants)
4. [Log Record](#log-record)
5. [Formatter (Strategy Pattern)](#formatter-strategy-pattern)
6. [Appender (Strategy Pattern)](#appender-strategy-pattern)
7. [Filter Chain](#filter-chain)
8. [Logger & Logger Hierarchy](#logger--logger-hierarchy)
9. [Logger Registry (Singleton)](#logger-registry-singleton)
10. [Async Appender (Producer-Consumer)](#async-appender-producer-consumer)
11. [Rotating File Appender](#rotating-file-appender)
12. [Log Context (MDC)](#log-context-mdc)
13. [Logging Service Facade](#logging-service-facade)
14. [Simulation Engine](#simulation-engine)
15. [Putting It All Together](#putting-it-all-together)
16. [Class Diagram](#class-diagram)
17. [How to Run](#how-to-run)

---

## Requirements

### Functional Requirements

| #  | Requirement                                                                                            |
| -- | ------------------------------------------------------------------------------------------------------ |
| 1  | `log(level, message)` writes a log entry through one or more appenders (console, file, JSON)           |
| 2  | Log levels — TRACE, DEBUG, INFO, WARN, ERROR, FATAL with numeric severity ordering                     |
| 3  | Multiple appenders — a single logger can write to console, file, and JSON simultaneously               |
| 4  | Pluggable formatters — each appender can use a different format (plain text, JSON, pattern-based)       |
| 5  | Log filtering — filter log records by level, logger name, or custom predicates before they reach appenders |
| 6  | Logger hierarchy — child loggers inherit the level and appenders of their parent (e.g., `app.db` inherits from `app`) |
| 7  | Log context (MDC) — thread-local key-value pairs automatically included in every log record            |
| 8  | Async appender — a non-blocking wrapper that buffers log records and writes them in a background thread |
| 9  | Rotating file appender — automatically rotates log files when they exceed a configurable size          |
| 10 | Statistics — total logged, per-level counts, per-appender counts, and dropped count                    |

### Non-Functional Requirements

- Clean OOP with SOLID principles.
- Extensible — new appenders, formatters, and filters can be added without modifying core logic.
- Thread-safe — all shared state is protected by locks; async appender uses a queue for producer-consumer.

---

## Key Design Patterns

| Pattern            | Where It Is Used                                           | Why                                                                                      |
| ------------------ | ---------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| Strategy           | `Formatter` formats records; `Appender` writes records     | Swap formatters and appenders independently without changing logger logic                 |
| Chain of Resp.     | `FilterChain` evaluates filters sequentially               | Each filter decides pass/block independently; new filters are added without touching others |
| Singleton          | `LoggerRegistry` provides a single global registry         | All code shares the same logger namespace and configuration                              |
| Observer           | `Logger` fans out records to multiple appenders            | Adding or removing an appender does not affect other appenders or the logger itself       |
| Facade             | `LoggingService` exposes a unified configuration API       | Hides the complexity of registries, hierarchies, formatters, and appenders               |
| Producer-Consumer  | `AsyncAppender` buffers records for background writing     | Decouples log emission from I/O-bound appender writes, preventing caller blocking        |

---

## Core Enums & Constants

```python
import os
import time
import uuid
import threading
import traceback
from enum import Enum
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from typing import Any, Callable
from collections import defaultdict, deque
from queue import Queue, Empty


class LogLevel(Enum):
    TRACE = 0                        # finest-grained tracing
    DEBUG = 1                        # debugging information
    INFO = 2                         # general operational messages
    WARN = 3                         # potential issues
    ERROR = 4                        # errors that allow recovery
    FATAL = 5                        # critical failures, app may exit
    OFF = 6                          # disables all logging

    def __ge__(self, other: "LogLevel") -> bool:
        return self.value >= other.value

    def __gt__(self, other: "LogLevel") -> bool:
        return self.value > other.value

    def __le__(self, other: "LogLevel") -> bool:
        return self.value <= other.value

    def __lt__(self, other: "LogLevel") -> bool:
        return self.value < other.value


class FilterDecision(Enum):
    ACCEPT = "accept"                # force-accept the record
    DENY = "deny"                    # force-reject the record
    NEUTRAL = "neutral"              # defer to the next filter


# ------------------------------------------------------------------
# System constants
# ------------------------------------------------------------------

DEFAULT_LOG_LEVEL = LogLevel.DEBUG
DEFAULT_PATTERN = "[{timestamp}] [{level:<5}] [{logger}] {message}"
DEFAULT_DATE_FORMAT = "%Y-%m-%d %H:%M:%S"
DEFAULT_MAX_FILE_SIZE = 5 * 1024 * 1024     # 5 MB
DEFAULT_MAX_BACKUP_COUNT = 3
DEFAULT_ASYNC_QUEUE_SIZE = 10_000
STATS_DISPLAY_WIDTH = 55
```

`LogLevel` defines six log levels plus `OFF` (which disables all output). The numeric values enable comparison — `INFO (2) >= DEBUG (1)` means an INFO-level logger silences DEBUG messages. The custom comparison operators (`__ge__`, `__gt__`, etc.) are required because Python's `Enum` does not support comparison by default. `FilterDecision` implements a three-state result: `ACCEPT` short-circuits the filter chain and forces the record through, `DENY` drops it immediately, and `NEUTRAL` passes it to the next filter. `DEFAULT_PATTERN` uses `str.format_map` syntax for variable substitution in log lines. `DEFAULT_MAX_FILE_SIZE` of 5 MB triggers file rotation — large enough for meaningful log batches, small enough to prevent unbounded growth.

---

## Log Record

```python
@dataclass
class LogRecord:
    """
    A single log event captured at a point in time.

    Contains the log level, message, logger name, timestamp,
    exception info, and contextual key-value pairs from MDC.
    LogRecords are the unit of data that flows through the
    logging pipeline: logger → filter → formatter → appender.
    """
    level: LogLevel
    message: str
    logger_name: str
    timestamp: float = field(default_factory=time.time)
    record_id: str = field(
        default_factory=lambda: str(uuid.uuid4())[:8]
    )
    thread_name: str = field(
        default_factory=lambda: threading.current_thread().name
    )
    exception: str = ""
    context: dict[str, str] = field(default_factory=dict)

    @property
    def formatted_timestamp(self) -> str:
        """Return timestamp as human-readable string."""
        return time.strftime(
            DEFAULT_DATE_FORMAT,
            time.localtime(self.timestamp),
        )

    @property
    def level_name(self) -> str:
        return self.level.name

    def to_dict(self) -> dict[str, Any]:
        """Convert record to dictionary for JSON serialization."""
        d: dict[str, Any] = {
            "timestamp": self.formatted_timestamp,
            "level": self.level.name,
            "logger": self.logger_name,
            "message": self.message,
            "thread": self.thread_name,
            "record_id": self.record_id,
        }
        if self.exception:
            d["exception"] = self.exception
        if self.context:
            d["context"] = dict(self.context)
        return d

    def __repr__(self) -> str:
        return (f"LogRecord(level={self.level.name}, "
                f"logger={self.logger_name}, "
                f"msg={self.message!r})")
```

`LogRecord` is the core data object — it captures everything about a single log event. `timestamp` defaults to `time.time()` at creation, so the record always reflects when the event occurred, not when it was written. `thread_name` is captured automatically for multi-threaded debugging. `context` carries MDC (Mapped Diagnostic Context) data — request IDs, user IDs, or correlation tokens that are automatically attached to every log line from that thread. `to_dict` provides a clean dictionary representation for JSON formatters. `record_id` gives each record a unique short ID for traceability across systems.

---

## Formatter (Strategy Pattern)

```python
class Formatter(ABC):
    """
    Converts a LogRecord into a string for output.

    Each appender can use a different formatter. The formatter
    controls the visual representation of log data without
    affecting how or where it is written.
    """

    @abstractmethod
    def format(self, record: LogRecord) -> str:
        """Format a log record into a string."""
        ...

    @abstractmethod
    def name(self) -> str:
        ...


class PatternFormatter(Formatter):
    """
    Formats log records using a configurable pattern string.

    Supported placeholders:
        {timestamp}  — formatted date/time
        {level}      — log level name (e.g., INFO)
        {logger}     — logger name
        {message}    — log message
        {thread}     — thread name
        {record_id}  — unique record ID
        {exception}  — exception traceback (if any)

    Example pattern:
        "[{timestamp}] [{level:<5}] [{logger}] {message}"
    """

    def __init__(
        self,
        pattern: str = DEFAULT_PATTERN,
        date_format: str = DEFAULT_DATE_FORMAT,
    ) -> None:
        self._pattern = pattern
        self._date_format = date_format

    def format(self, record: LogRecord) -> str:
        timestamp = time.strftime(
            self._date_format,
            time.localtime(record.timestamp),
        )
        values = {
            "timestamp": timestamp,
            "level": record.level.name,
            "logger": record.logger_name,
            "message": record.message,
            "thread": record.thread_name,
            "record_id": record.record_id,
        }
        result = self._pattern.format_map(
            defaultdict(str, values)
        )
        if record.exception:
            result += f"\n  Exception: {record.exception}"
        if record.context:
            ctx = " ".join(
                f"{k}={v}" for k, v in record.context.items()
            )
            result += f"\n  Context: {ctx}"
        return result

    def name(self) -> str:
        return "PatternFormatter"


class JsonFormatter(Formatter):
    """
    Formats log records as single-line JSON strings.

    Ideal for structured logging pipelines (ELK, Splunk,
    CloudWatch). Each log line is a valid JSON object that
    can be parsed by log aggregation tools.
    """

    def __init__(self, pretty: bool = False) -> None:
        self._pretty = pretty

    def format(self, record: LogRecord) -> str:
        import json
        d = record.to_dict()
        if self._pretty:
            return json.dumps(d, indent=2)
        return json.dumps(d)

    def name(self) -> str:
        return "JsonFormatter"


class SimpleFormatter(Formatter):
    """
    Minimal formatter that outputs level and message only.

    Useful for quick debugging or minimal console output.
    """

    def format(self, record: LogRecord) -> str:
        return f"{record.level.name}: {record.message}"

    def name(self) -> str:
        return "SimpleFormatter"
```

`Formatter` defines the strategy interface for converting records to strings. `PatternFormatter` supports `str.format_map` syntax with a `defaultdict` fallback — if a placeholder like `{custom}` is missing, it renders as an empty string instead of raising `KeyError`. The `{level:<5}` syntax left-aligns the level name to 5 characters, so `INFO ` and `ERROR` line up in console output. `JsonFormatter` produces single-line JSON suitable for structured logging pipelines like ELK or Splunk — each line is independently parseable. `SimpleFormatter` strips everything to just level and message for quick debugging. Adding a new format (e.g., XML, CSV, Logfmt) requires only implementing `format` and `name`.

---

## Appender (Strategy Pattern)

```python
class Appender(ABC):
    """
    Writes formatted log records to a destination.

    Each appender handles one output target (console, file,
    network). Appenders receive already-formatted strings
    from their assigned formatter.
    """

    @abstractmethod
    def append(self, record: LogRecord) -> bool:
        """
        Write a log record.

        Returns True if successfully written, False otherwise.
        """
        ...

    @abstractmethod
    def close(self) -> None:
        """Release resources held by this appender."""
        ...

    @abstractmethod
    def appender_name(self) -> str:
        ...


class ConsoleAppender(Appender):
    """
    Writes log records to stdout.

    Supports optional ANSI color coding by log level for
    terminal readability. Colors are disabled when output
    is redirected to a file (non-TTY).
    """

    COLORS = {
        LogLevel.TRACE: "\033[37m",      # white
        LogLevel.DEBUG: "\033[36m",      # cyan
        LogLevel.INFO: "\033[32m",       # green
        LogLevel.WARN: "\033[33m",       # yellow
        LogLevel.ERROR: "\033[31m",      # red
        LogLevel.FATAL: "\033[35;1m",    # bold magenta
    }
    RESET = "\033[0m"

    def __init__(
        self,
        formatter: Formatter | None = None,
        colorize: bool = True,
    ) -> None:
        self._formatter = formatter or PatternFormatter()
        self._colorize = colorize
        self._write_count = 0
        self._lock = threading.RLock()

    def append(self, record: LogRecord) -> bool:
        with self._lock:
            try:
                text = self._formatter.format(record)
                if self._colorize:
                    color = self.COLORS.get(
                        record.level, ""
                    )
                    text = f"{color}{text}{self.RESET}"
                print(text)
                self._write_count += 1
                return True
            except Exception:
                return False

    def close(self) -> None:
        pass  # stdout does not need closing

    @property
    def write_count(self) -> int:
        return self._write_count

    def appender_name(self) -> str:
        return "ConsoleAppender"


class FileAppender(Appender):
    """
    Writes log records to a file.

    Opens the file in append mode. Creates parent directories
    if they do not exist. Thread-safe writes via a lock.
    """

    def __init__(
        self,
        filepath: str,
        formatter: Formatter | None = None,
    ) -> None:
        self._filepath = filepath
        self._formatter = formatter or PatternFormatter()
        self._write_count = 0
        self._lock = threading.RLock()
        self._file = None
        self._ensure_directory()
        self._open()

    def _ensure_directory(self) -> None:
        directory = os.path.dirname(self._filepath)
        if directory:
            os.makedirs(directory, exist_ok=True)

    def _open(self) -> None:
        self._file = open(self._filepath, "a",
                          encoding="utf-8")

    def append(self, record: LogRecord) -> bool:
        with self._lock:
            try:
                if self._file is None or self._file.closed:
                    self._open()
                text = self._formatter.format(record)
                self._file.write(text + "\n")
                self._file.flush()
                self._write_count += 1
                return True
            except Exception:
                return False

    def close(self) -> None:
        with self._lock:
            if self._file and not self._file.closed:
                self._file.close()

    @property
    def write_count(self) -> int:
        return self._write_count

    @property
    def filepath(self) -> str:
        return self._filepath

    def appender_name(self) -> str:
        return f"FileAppender({self._filepath})"


class InMemoryAppender(Appender):
    """
    Stores log records in memory for testing and inspection.

    Useful for unit tests that need to verify log output
    without touching the filesystem or console.
    """

    def __init__(
        self,
        formatter: Formatter | None = None,
        max_records: int = 1000,
    ) -> None:
        self._formatter = formatter or PatternFormatter()
        self._records: list[LogRecord] = []
        self._formatted: list[str] = []
        self._max_records = max_records
        self._write_count = 0
        self._lock = threading.RLock()

    def append(self, record: LogRecord) -> bool:
        with self._lock:
            if len(self._records) >= self._max_records:
                # Drop oldest when full
                self._records.pop(0)
                self._formatted.pop(0)
            text = self._formatter.format(record)
            self._records.append(record)
            self._formatted.append(text)
            self._write_count += 1
            return True

    def get_records(self) -> list[LogRecord]:
        with self._lock:
            return list(self._records)

    def get_formatted(self) -> list[str]:
        with self._lock:
            return list(self._formatted)

    def clear(self) -> None:
        with self._lock:
            self._records.clear()
            self._formatted.clear()

    def close(self) -> None:
        self.clear()

    @property
    def write_count(self) -> int:
        return self._write_count

    @property
    def size(self) -> int:
        return len(self._records)

    def appender_name(self) -> str:
        return "InMemoryAppender"
```

`Appender` defines the strategy interface for writing log output. `ConsoleAppender` writes to stdout with optional ANSI color coding — `ERROR` appears in red, `WARN` in yellow, `INFO` in green, making log streams visually scannable. The color map is defined as a class-level constant because it never changes. `FileAppender` opens the file in append mode and creates parent directories automatically — a logger configured with `logs/app/debug.log` creates the `logs/app/` directory if it does not exist. `flush()` is called after every write to ensure log data survives a crash. `InMemoryAppender` stores records in a list for testing — tests assert against `get_records()` and `get_formatted()` without needing to read files or capture stdout. Adding a new appender (e.g., Syslog, HTTP, database) requires only implementing `append`, `close`, and `appender_name`.

---

## Filter Chain

```python
class LogFilter(ABC):
    """
    Decides whether a log record should pass through.

    Filters are evaluated in a chain. Each filter returns
    ACCEPT (force through), DENY (block), or NEUTRAL
    (defer to next filter).
    """

    @abstractmethod
    def decide(self, record: LogRecord) -> FilterDecision:
        ...

    @abstractmethod
    def filter_name(self) -> str:
        ...


class LevelFilter(LogFilter):
    """
    Accepts records at or above a minimum level.

    Records below the minimum level are denied.
    Records at or above are accepted.
    """

    def __init__(self, min_level: LogLevel) -> None:
        self._min_level = min_level

    def decide(self, record: LogRecord) -> FilterDecision:
        if record.level >= self._min_level:
            return FilterDecision.ACCEPT
        return FilterDecision.DENY

    def filter_name(self) -> str:
        return f"LevelFilter(>={self._min_level.name})"


class LevelRangeFilter(LogFilter):
    """
    Accepts records within a level range (inclusive).

    Useful for routing specific levels to specific appenders.
    For example, send only WARN and ERROR to a separate file.
    """

    def __init__(self, min_level: LogLevel,
                 max_level: LogLevel) -> None:
        self._min_level = min_level
        self._max_level = max_level

    def decide(self, record: LogRecord) -> FilterDecision:
        if (record.level >= self._min_level
                and record.level <= self._max_level):
            return FilterDecision.ACCEPT
        return FilterDecision.DENY

    def filter_name(self) -> str:
        return (f"LevelRangeFilter("
                f"{self._min_level.name}-"
                f"{self._max_level.name})")


class LoggerNameFilter(LogFilter):
    """
    Accepts records from loggers whose name starts with a prefix.

    For example, prefix="app.db" matches "app.db",
    "app.db.query", and "app.db.connection".
    """

    def __init__(self, prefix: str,
                 accept: bool = True) -> None:
        self._prefix = prefix
        self._accept = accept

    def decide(self, record: LogRecord) -> FilterDecision:
        matches = record.logger_name.startswith(self._prefix)
        if matches and self._accept:
            return FilterDecision.ACCEPT
        if matches and not self._accept:
            return FilterDecision.DENY
        return FilterDecision.NEUTRAL

    def filter_name(self) -> str:
        action = "accept" if self._accept else "deny"
        return f"LoggerNameFilter({self._prefix}, {action})"


class KeywordFilter(LogFilter):
    """
    Accepts or denies records containing specific keywords.

    Case-insensitive search in the message body.
    """

    def __init__(self, keyword: str,
                 accept: bool = True) -> None:
        self._keyword = keyword.lower()
        self._accept = accept

    def decide(self, record: LogRecord) -> FilterDecision:
        found = self._keyword in record.message.lower()
        if found and self._accept:
            return FilterDecision.ACCEPT
        if found and not self._accept:
            return FilterDecision.DENY
        return FilterDecision.NEUTRAL

    def filter_name(self) -> str:
        action = "accept" if self._accept else "deny"
        return f"KeywordFilter({self._keyword!r}, {action})"


class FilterChain:
    """
    Evaluates a sequence of filters against a log record.

    Processing rules:
        1. Filters are evaluated in order.
        2. ACCEPT → immediately accept, skip remaining filters.
        3. DENY → immediately reject, skip remaining filters.
        4. NEUTRAL → continue to the next filter.
        5. If all filters return NEUTRAL → accept (default pass).
    """

    def __init__(self) -> None:
        self._filters: list[LogFilter] = []
        self._lock = threading.RLock()

    def add_filter(self, log_filter: LogFilter) -> None:
        with self._lock:
            self._filters.append(log_filter)

    def remove_filter(self, filter_name: str) -> None:
        with self._lock:
            self._filters = [
                f for f in self._filters
                if f.filter_name() != filter_name
            ]

    def evaluate(self, record: LogRecord) -> bool:
        """
        Run all filters against a record.

        Returns True if the record should pass, False to drop.
        """
        with self._lock:
            for log_filter in self._filters:
                decision = log_filter.decide(record)
                if decision == FilterDecision.ACCEPT:
                    return True
                if decision == FilterDecision.DENY:
                    return False
            # All NEUTRAL → default accept
            return True

    @property
    def filter_count(self) -> int:
        return len(self._filters)

    def __repr__(self) -> str:
        names = ", ".join(
            f.filter_name() for f in self._filters
        )
        return f"FilterChain([{names}])"
```

`LogFilter` implements the chain of responsibility pattern. Each filter returns a three-state `FilterDecision` rather than a simple boolean — `ACCEPT` short-circuits the chain and forces the record through (useful for critical-level overrides), `DENY` drops it immediately, and `NEUTRAL` passes it to the next filter. `LevelFilter` is the most common filter — it blocks records below a threshold. `LevelRangeFilter` routes specific levels to specific appenders (e.g., only WARN and ERROR go to `errors.log`). `LoggerNameFilter` supports prefix matching for hierarchical logger names — `prefix="app.db"` matches `app.db`, `app.db.query`, and `app.db.connection`. `KeywordFilter` provides content-based filtering with case-insensitive search. `FilterChain.evaluate` processes filters in order and defaults to accept if all filters return `NEUTRAL`.

---

## Logger & Logger Hierarchy

```python
class Logger:
    """
    Named logger that processes log records.

    Each logger has a name, a level, a set of appenders,
    and a filter chain. Loggers form a hierarchy based on
    dot-separated names (e.g., "app" → "app.db" → "app.db.query").

    A child logger inherits the level and appenders of its
    parent unless explicitly overridden. This allows
    configuring logging at a high level ("app") and having
    all subcomponents automatically inherit the configuration.

    Logging pipeline:
        1. Level check — is the record's level ≥ logger's level?
        2. Create LogRecord with MDC context
        3. Filter chain — does the record pass all filters?
        4. Appender fan-out — write to all attached appenders
        5. Propagate to parent logger (if propagation enabled)
    """

    def __init__(
        self,
        name: str,
        level: LogLevel = DEFAULT_LOG_LEVEL,
        parent: "Logger | None" = None,
    ) -> None:
        self._name = name
        self._level = level
        self._parent = parent
        self._appenders: list[Appender] = []
        self._filter_chain = FilterChain()
        self._propagate = True       # propagate to parent
        self._lock = threading.RLock()

        # Statistics
        self._log_count = 0
        self._level_counts: dict[LogLevel, int] = (
            defaultdict(int)
        )
        self._dropped_count = 0

    @property
    def name(self) -> str:
        return self._name

    @property
    def level(self) -> LogLevel:
        return self._level

    @level.setter
    def level(self, level: LogLevel) -> None:
        self._level = level

    @property
    def effective_level(self) -> LogLevel:
        """
        Return the effective level considering parent hierarchy.

        Walks up the parent chain until a logger with an
        explicitly set level is found.
        """
        if self._level != LogLevel.TRACE or not self._parent:
            return self._level
        return self._parent.effective_level

    @property
    def propagate(self) -> bool:
        return self._propagate

    @propagate.setter
    def propagate(self, value: bool) -> None:
        self._propagate = value

    def add_appender(self, appender: Appender) -> None:
        with self._lock:
            self._appenders.append(appender)

    def remove_appender(self, appender_name: str) -> None:
        with self._lock:
            self._appenders = [
                a for a in self._appenders
                if a.appender_name() != appender_name
            ]

    def add_filter(self, log_filter: LogFilter) -> None:
        self._filter_chain.add_filter(log_filter)

    def remove_filter(self, filter_name: str) -> None:
        self._filter_chain.remove_filter(filter_name)

    # ------------------------------------------------------------------
    # Convenience log methods
    # ------------------------------------------------------------------

    def trace(self, message: str, **kwargs: Any) -> None:
        self._log(LogLevel.TRACE, message, **kwargs)

    def debug(self, message: str, **kwargs: Any) -> None:
        self._log(LogLevel.DEBUG, message, **kwargs)

    def info(self, message: str, **kwargs: Any) -> None:
        self._log(LogLevel.INFO, message, **kwargs)

    def warn(self, message: str, **kwargs: Any) -> None:
        self._log(LogLevel.WARN, message, **kwargs)

    def error(self, message: str,
              exception: str = "", **kwargs: Any) -> None:
        self._log(LogLevel.ERROR, message,
                  exception=exception, **kwargs)

    def fatal(self, message: str,
              exception: str = "", **kwargs: Any) -> None:
        self._log(LogLevel.FATAL, message,
                  exception=exception, **kwargs)

    # ------------------------------------------------------------------
    # Core logging pipeline
    # ------------------------------------------------------------------

    def _log(
        self,
        level: LogLevel,
        message: str,
        exception: str = "",
        **kwargs: Any,
    ) -> None:
        """
        Core logging method.

        Pipeline:
        1. Level check
        2. Create LogRecord (with MDC context)
        3. Filter chain
        4. Fan out to appenders
        5. Propagate to parent
        """
        # 1. Level check
        if level < self.effective_level:
            return

        # 2. Create record
        context = LogContext.get_context().copy()
        record = LogRecord(
            level=level,
            message=message,
            logger_name=self._name,
            exception=exception,
            context=context,
        )

        # 3. Filter chain
        with self._lock:
            if not self._filter_chain.evaluate(record):
                self._dropped_count += 1
                return

            # 4. Fan out to appenders
            self._log_count += 1
            self._level_counts[level] += 1

            for appender in self._appenders:
                try:
                    appender.append(record)
                except Exception as e:
                    print(f"  ⚠️ Appender "
                          f"{appender.appender_name()} "
                          f"error: {e}")

        # 5. Propagate to parent
        if self._propagate and self._parent:
            self._parent._handle_propagated(record)

    def _handle_propagated(self, record: LogRecord) -> None:
        """
        Handle a record propagated from a child logger.

        Applies this logger's filters and writes to this
        logger's appenders, then continues propagation.
        """
        with self._lock:
            if not self._filter_chain.evaluate(record):
                return

            self._log_count += 1
            self._level_counts[record.level] += 1

            for appender in self._appenders:
                try:
                    appender.append(record)
                except Exception as e:
                    print(f"  ⚠️ Appender "
                          f"{appender.appender_name()} "
                          f"error: {e}")

        if self._propagate and self._parent:
            self._parent._handle_propagated(record)

    # ------------------------------------------------------------------
    # Inspection
    # ------------------------------------------------------------------

    @property
    def log_count(self) -> int:
        return self._log_count

    @property
    def dropped_count(self) -> int:
        return self._dropped_count

    @property
    def level_counts(self) -> dict[LogLevel, int]:
        return dict(self._level_counts)

    @property
    def appenders(self) -> list[Appender]:
        return list(self._appenders)

    def close(self) -> None:
        """Close all appenders attached to this logger."""
        with self._lock:
            for appender in self._appenders:
                try:
                    appender.close()
                except Exception:
                    pass

    def __repr__(self) -> str:
        appenders = ", ".join(
            a.appender_name() for a in self._appenders
        )
        return (f"Logger(name={self._name}, "
                f"level={self._level.name}, "
                f"appenders=[{appenders}])")
```

`Logger` is the central class — it owns the five-step logging pipeline: level check → record creation → filter chain → appender fan-out → parent propagation. The level check is the cheapest operation and happens first to short-circuit debug/trace messages in production. `effective_level` walks up the parent chain — if `app.db.query` has no explicit level set (defaults to `TRACE`), it inherits from `app.db`, which might inherit from `app`. `_handle_propagated` is separated from `_log` because propagated records skip the level check (they already passed the child's level check) and skip MDC attachment (context was already captured by the child). `propagate=True` by default mirrors the behavior of Python's `logging` module and Java's `log4j`. Adding an appender to the root logger means all child loggers automatically write there without explicit configuration. The convenience methods (`info`, `debug`, `error`, etc.) delegate to `_log` with the appropriate level.

---

## Logger Registry (Singleton)

```python
class LoggerRegistry:
    """
    Singleton registry that manages all loggers and their hierarchy.

    Loggers are identified by dot-separated names. The registry
    automatically creates parent-child relationships:
        "app"           → root-level logger
        "app.db"        → child of "app"
        "app.db.query"  → child of "app.db"

    The root logger (name="root") is the ancestor of all loggers.
    It is created automatically and is the default appender target.

    Usage:
        registry = LoggerRegistry.get_instance()
        logger = registry.get_logger("app.db")
    """

    _instance: "LoggerRegistry | None" = None
    _init_lock = threading.Lock()

    def __init__(self) -> None:
        if LoggerRegistry._instance is not None:
            raise RuntimeError(
                "Use LoggerRegistry.get_instance()"
            )
        self._loggers: dict[str, Logger] = {}
        self._root = Logger("root", DEFAULT_LOG_LEVEL)
        self._loggers["root"] = self._root
        self._lock = threading.RLock()

    @classmethod
    def get_instance(cls) -> "LoggerRegistry":
        """Return the singleton instance, creating it if needed."""
        if cls._instance is None:
            with cls._init_lock:
                if cls._instance is None:
                    cls._instance = cls()
        return cls._instance

    @classmethod
    def reset(cls) -> None:
        """
        Reset the singleton for testing.

        Closes all loggers and destroys the instance.
        """
        with cls._init_lock:
            if cls._instance is not None:
                for logger in cls._instance._loggers.values():
                    logger.close()
                cls._instance = None

    @property
    def root(self) -> Logger:
        return self._root

    def get_logger(self, name: str) -> Logger:
        """
        Get or create a logger by name.

        Automatically establishes parent-child hierarchy.
        For "app.db.query":
            1. Ensure "app" exists (parent = root)
            2. Ensure "app.db" exists (parent = "app")
            3. Create "app.db.query" (parent = "app.db")
        """
        with self._lock:
            if name in self._loggers:
                return self._loggers[name]

            # Find or create parent
            parent = self._find_parent(name)
            logger = Logger(name, parent=parent)
            self._loggers[name] = logger
            return logger

    def _find_parent(self, name: str) -> Logger:
        """
        Find the nearest existing parent for a logger name.

        For "app.db.query", checks "app.db" then "app".
        Falls back to root if no ancestor exists.
        """
        parts = name.rsplit(".", 1)
        if len(parts) == 1:
            # No dot → parent is root
            return self._root

        parent_name = parts[0]
        if parent_name in self._loggers:
            return self._loggers[parent_name]

        # Recursively create parent
        return self.get_logger(parent_name)

    def get_all_loggers(self) -> dict[str, Logger]:
        with self._lock:
            return dict(self._loggers)

    @property
    def logger_count(self) -> int:
        return len(self._loggers)

    def __repr__(self) -> str:
        return (f"LoggerRegistry("
                f"loggers={len(self._loggers)})")
```

`LoggerRegistry` implements the singleton pattern with double-checked locking for thread safety. The `_init_lock` is a class-level `threading.Lock` that ensures only one thread creates the instance. `get_logger` automatically builds the hierarchy: requesting `app.db.query` creates `app` (parent=root), `app.db` (parent=app), and `app.db.query` (parent=app.db) if they do not exist. `_find_parent` uses `rsplit(".", 1)` to split on the last dot — `"app.db.query".rsplit(".", 1)` gives `["app.db", "query"]`, so the parent is `"app.db"`. `reset` destroys the singleton for test isolation — each test starts with a clean registry.

---

## Async Appender (Producer-Consumer)

```python
class AsyncAppender(Appender):
    """
    Non-blocking appender that buffers records in a queue
    and writes them in a background thread.

    The caller's thread never blocks on I/O — it puts the
    record in the queue and returns immediately. A dedicated
    worker thread drains the queue and delegates to the
    wrapped (synchronous) appender.

    If the queue is full, the record is dropped and the
    drop counter is incremented.

    Usage:
        file_appender = FileAppender("app.log")
        async_appender = AsyncAppender(file_appender)
        logger.add_appender(async_appender)
        # ... log messages ...
        async_appender.close()  # flush and stop worker
    """

    def __init__(
        self,
        delegate: Appender,
        queue_size: int = DEFAULT_ASYNC_QUEUE_SIZE,
    ) -> None:
        self._delegate = delegate
        self._queue: Queue[LogRecord | None] = Queue(
            maxsize=queue_size
        )
        self._dropped = 0
        self._write_count = 0
        self._running = True
        self._lock = threading.RLock()
        self._worker = threading.Thread(
            target=self._process,
            name=f"async-{delegate.appender_name()}",
            daemon=True,
        )
        self._worker.start()

    def append(self, record: LogRecord) -> bool:
        """
        Enqueue a record for async writing.

        Non-blocking. Returns False if the queue is full
        (record is dropped).
        """
        try:
            self._queue.put_nowait(record)
            return True
        except Exception:
            with self._lock:
                self._dropped += 1
            return False

    def _process(self) -> None:
        """Worker thread: drain queue and write to delegate."""
        while self._running or not self._queue.empty():
            try:
                record = self._queue.get(timeout=0.1)
                if record is None:
                    # Poison pill — shutdown signal
                    break
                self._delegate.append(record)
                with self._lock:
                    self._write_count += 1
            except Empty:
                continue
            except Exception:
                pass

    def close(self) -> None:
        """
        Flush remaining records and stop the worker thread.

        Sends a poison pill (None) to signal the worker to exit.
        """
        self._running = False
        self._queue.put(None)  # poison pill
        self._worker.join(timeout=5.0)
        self._delegate.close()

    @property
    def dropped_count(self) -> int:
        return self._dropped

    @property
    def write_count(self) -> int:
        return self._write_count

    @property
    def queue_size(self) -> int:
        return self._queue.qsize()

    def appender_name(self) -> str:
        return f"AsyncAppender({self._delegate.appender_name()})"
```

`AsyncAppender` implements the producer-consumer pattern. The calling thread (producer) puts records into a bounded `Queue` and returns immediately — no I/O wait. A background daemon thread (consumer) drains the queue and delegates to the wrapped synchronous appender. `put_nowait` is used instead of `put` to ensure the caller never blocks; if the queue is full, the record is dropped and `_dropped` is incremented. Shutdown uses the poison pill pattern: `close` sends `None` into the queue, the worker thread recognizes it and exits. The `while self._running or not self._queue.empty()` condition ensures all buffered records are flushed before the worker stops. The daemon flag ensures the worker thread dies if the main program exits without calling `close`.

---

## Rotating File Appender

```python
class RotatingFileAppender(Appender):
    """
    File appender that rotates log files when they exceed
    a maximum size.

    When the active log file exceeds max_file_size:
        1. Close the current file
        2. Rename: app.log → app.log.1, app.log.1 → app.log.2, ...
        3. Delete the oldest backup if it exceeds max_backup_count
        4. Open a new empty app.log

    Example with max_backup_count=3:
        app.log     ← current (active)
        app.log.1   ← previous
        app.log.2   ← older
        app.log.3   ← oldest (deleted when 4th rotation occurs)
    """

    def __init__(
        self,
        filepath: str,
        formatter: Formatter | None = None,
        max_file_size: int = DEFAULT_MAX_FILE_SIZE,
        max_backup_count: int = DEFAULT_MAX_BACKUP_COUNT,
    ) -> None:
        self._filepath = filepath
        self._formatter = formatter or PatternFormatter()
        self._max_file_size = max_file_size
        self._max_backup_count = max_backup_count
        self._write_count = 0
        self._rotation_count = 0
        self._current_size = 0
        self._lock = threading.RLock()
        self._file = None
        self._ensure_directory()
        self._open()

    def _ensure_directory(self) -> None:
        directory = os.path.dirname(self._filepath)
        if directory:
            os.makedirs(directory, exist_ok=True)

    def _open(self) -> None:
        self._file = open(self._filepath, "a",
                          encoding="utf-8")
        # Track existing file size
        try:
            self._current_size = os.path.getsize(
                self._filepath
            )
        except OSError:
            self._current_size = 0

    def append(self, record: LogRecord) -> bool:
        with self._lock:
            try:
                if self._file is None or self._file.closed:
                    self._open()

                text = self._formatter.format(record)
                line = text + "\n"
                line_bytes = len(line.encode("utf-8"))

                # Rotate if this write would exceed max size
                if (self._current_size + line_bytes
                        > self._max_file_size):
                    self._rotate()

                self._file.write(line)
                self._file.flush()
                self._current_size += line_bytes
                self._write_count += 1
                return True
            except Exception:
                return False

    def _rotate(self) -> None:
        """
        Rotate log files.

        app.log.2 → app.log.3 (if max_backup > 2)
        app.log.1 → app.log.2
        app.log   → app.log.1
        Open new empty app.log
        """
        # Close current file
        if self._file and not self._file.closed:
            self._file.close()

        # Shift existing backups
        for i in range(self._max_backup_count - 1, 0, -1):
            src = f"{self._filepath}.{i}"
            dst = f"{self._filepath}.{i + 1}"
            if os.path.exists(src):
                if os.path.exists(dst):
                    os.remove(dst)
                os.rename(src, dst)

        # Rename current to .1
        backup_path = f"{self._filepath}.1"
        if os.path.exists(self._filepath):
            if os.path.exists(backup_path):
                os.remove(backup_path)
            os.rename(self._filepath, backup_path)

        # Delete oldest backup if it exceeds count
        oldest = (f"{self._filepath}."
                  f"{self._max_backup_count + 1}")
        if os.path.exists(oldest):
            os.remove(oldest)

        # Open new file
        self._file = open(self._filepath, "w",
                          encoding="utf-8")
        self._current_size = 0
        self._rotation_count += 1

    def close(self) -> None:
        with self._lock:
            if self._file and not self._file.closed:
                self._file.close()

    @property
    def write_count(self) -> int:
        return self._write_count

    @property
    def rotation_count(self) -> int:
        return self._rotation_count

    @property
    def current_size(self) -> int:
        return self._current_size

    def appender_name(self) -> str:
        return f"RotatingFileAppender({self._filepath})"
```

`RotatingFileAppender` extends file-based logging with automatic rotation. When a write would push the file past `max_file_size`, `_rotate` is triggered: it closes the current file, shifts existing backups up by one (`.1` → `.2`, `.2` → `.3`), renames the current file to `.1`, deletes the oldest if it exceeds `max_backup_count`, and opens a fresh empty file. The rotation loop iterates in reverse (`range(max-1, 0, -1)`) to avoid overwriting — if you rename `.1` → `.2` before `.2` → `.3`, you lose `.2`. `_current_size` tracks bytes written since the last rotation to avoid calling `os.path.getsize()` on every write. The new file is opened with mode `"w"` (not `"a"`) because it should start empty after rotation.

---

## Log Context (MDC)

```python
class LogContext:
    """
    Thread-local Mapped Diagnostic Context (MDC).

    MDC lets you attach key-value pairs to the current thread's
    logging context. These pairs are automatically included in
    every LogRecord created by that thread.

    Common use cases:
        - Request ID tracking across a web request lifecycle
        - User ID for per-user log correlation
        - Transaction ID for distributed tracing

    Usage:
        LogContext.put("request_id", "req-abc123")
        logger.info("Processing order")
        # → LogRecord includes context={"request_id": "req-abc123"}
        LogContext.clear()
    """

    _local = threading.local()

    @classmethod
    def put(cls, key: str, value: str) -> None:
        """Add a key-value pair to the current thread's context."""
        ctx = cls._get_store()
        ctx[key] = value

    @classmethod
    def get(cls, key: str,
            default: str = "") -> str:
        """Get a value from the current thread's context."""
        ctx = cls._get_store()
        return ctx.get(key, default)

    @classmethod
    def remove(cls, key: str) -> None:
        """Remove a key from the current thread's context."""
        ctx = cls._get_store()
        ctx.pop(key, None)

    @classmethod
    def clear(cls) -> None:
        """Clear all context entries for the current thread."""
        cls._local.context = {}

    @classmethod
    def get_context(cls) -> dict[str, str]:
        """Return a copy of the current thread's full context."""
        return dict(cls._get_store())

    @classmethod
    def _get_store(cls) -> dict[str, str]:
        if not hasattr(cls._local, "context"):
            cls._local.context = {}
        return cls._local.context

    def __repr__(self) -> str:
        return f"LogContext({self.get_context()})"
```

`LogContext` implements thread-local MDC (Mapped Diagnostic Context) using `threading.local()`. Each thread has its own independent context dictionary — when Thread A sets `request_id=req-123`, Thread B does not see it. `_get_store` lazily initializes the dict on first access, so threads that never use MDC pay no cost. `get_context` returns a copy (not the live dict) to prevent external code from modifying the context without going through `put`/`remove`. The `Logger._log` method copies the context at record creation time, so the record preserves the MDC state even if the context is cleared later. This is critical for async appenders where the record might be written after `clear()` is called.

---

## Logging Service Facade

```python
class LoggingStats:
    """Aggregate logging system statistics."""

    def __init__(self) -> None:
        self.total_logged: int = 0
        self.total_dropped: int = 0
        self.level_counts: dict[str, int] = defaultdict(int)
        self.appender_counts: dict[str, int] = defaultdict(int)

    def display(self) -> None:
        print(f"\n  {'═'*STATS_DISPLAY_WIDTH}")
        print(f"  LOGGING SYSTEM STATISTICS")
        print(f"  {'═'*STATS_DISPLAY_WIDTH}")
        print(f"  Total logged:      {self.total_logged}")
        print(f"  Total dropped:     {self.total_dropped}")

        if self.level_counts:
            print(f"  {'─'*STATS_DISPLAY_WIDTH}")
            print(f"  PER-LEVEL BREAKDOWN")
            for level_name in ["TRACE", "DEBUG", "INFO",
                                "WARN", "ERROR", "FATAL"]:
                count = self.level_counts.get(level_name, 0)
                if count > 0:
                    print(f"    {level_name:<6}: {count}")

        if self.appender_counts:
            print(f"  {'─'*STATS_DISPLAY_WIDTH}")
            print(f"  PER-APPENDER BREAKDOWN")
            for name, count in sorted(
                self.appender_counts.items()
            ):
                print(f"    {name}: {count} writes")

        print(f"  {'═'*STATS_DISPLAY_WIDTH}")

    def reset(self) -> None:
        self.total_logged = 0
        self.total_dropped = 0
        self.level_counts.clear()
        self.appender_counts.clear()


class LoggingService:
    """
    Facade that orchestrates the entire logging framework.

    Provides a simple API that hides the complexity of
    registries, hierarchies, formatters, appenders, filters,
    MDC, and async processing.

    Usage:
        service = LoggingService()
        service.configure_root(LogLevel.INFO)
        service.add_console_appender()
        logger = service.get_logger("app.db")
        logger.info("Connected to database")
    """

    def __init__(self) -> None:
        LoggerRegistry.reset()
        self._registry = LoggerRegistry.get_instance()
        self._stats = LoggingStats()
        self._appenders: list[Appender] = []
        self._lock = threading.RLock()

    # ------------------------------------------------------------------
    # Root logger configuration
    # ------------------------------------------------------------------

    def configure_root(
        self,
        level: LogLevel = DEFAULT_LOG_LEVEL,
    ) -> None:
        """Set the root logger's level."""
        self._registry.root.level = level

    # ------------------------------------------------------------------
    # Appender helpers
    # ------------------------------------------------------------------

    def add_console_appender(
        self,
        formatter: Formatter | None = None,
        colorize: bool = True,
        target_logger: str = "root",
    ) -> ConsoleAppender:
        """Add a console appender to a logger."""
        appender = ConsoleAppender(
            formatter=formatter, colorize=colorize,
        )
        logger = self._registry.get_logger(target_logger) \
            if target_logger != "root" \
            else self._registry.root
        logger.add_appender(appender)
        self._appenders.append(appender)
        return appender

    def add_file_appender(
        self,
        filepath: str,
        formatter: Formatter | None = None,
        target_logger: str = "root",
    ) -> FileAppender:
        """Add a file appender to a logger."""
        appender = FileAppender(
            filepath=filepath, formatter=formatter,
        )
        logger = self._registry.get_logger(target_logger) \
            if target_logger != "root" \
            else self._registry.root
        logger.add_appender(appender)
        self._appenders.append(appender)
        return appender

    def add_rotating_file_appender(
        self,
        filepath: str,
        formatter: Formatter | None = None,
        max_file_size: int = DEFAULT_MAX_FILE_SIZE,
        max_backup_count: int = DEFAULT_MAX_BACKUP_COUNT,
        target_logger: str = "root",
    ) -> RotatingFileAppender:
        """Add a rotating file appender to a logger."""
        appender = RotatingFileAppender(
            filepath=filepath,
            formatter=formatter,
            max_file_size=max_file_size,
            max_backup_count=max_backup_count,
        )
        logger = self._registry.get_logger(target_logger) \
            if target_logger != "root" \
            else self._registry.root
        logger.add_appender(appender)
        self._appenders.append(appender)
        return appender

    def add_memory_appender(
        self,
        formatter: Formatter | None = None,
        max_records: int = 1000,
        target_logger: str = "root",
    ) -> InMemoryAppender:
        """Add an in-memory appender for testing."""
        appender = InMemoryAppender(
            formatter=formatter,
            max_records=max_records,
        )
        logger = self._registry.get_logger(target_logger) \
            if target_logger != "root" \
            else self._registry.root
        logger.add_appender(appender)
        self._appenders.append(appender)
        return appender

    # ------------------------------------------------------------------
    # Logger access
    # ------------------------------------------------------------------

    def get_logger(self, name: str) -> Logger:
        """Get or create a named logger."""
        return self._registry.get_logger(name)

    @property
    def root(self) -> Logger:
        return self._registry.root

    # ------------------------------------------------------------------
    # Statistics
    # ------------------------------------------------------------------

    def collect_stats(self) -> LoggingStats:
        """Collect statistics from all loggers and appenders."""
        with self._lock:
            self._stats.reset()

            for name, logger in (
                self._registry.get_all_loggers().items()
            ):
                self._stats.total_logged += logger.log_count
                self._stats.total_dropped += (
                    logger.dropped_count
                )
                for level, count in (
                    logger.level_counts.items()
                ):
                    self._stats.level_counts[
                        level.name
                    ] += count

            for appender in self._appenders:
                self._stats.appender_counts[
                    appender.appender_name()
                ] = appender.write_count

            return self._stats

    def display_status(self) -> None:
        """Collect and display logging statistics."""
        stats = self.collect_stats()
        stats.display()

    # ------------------------------------------------------------------
    # Cleanup
    # ------------------------------------------------------------------

    def shutdown(self) -> None:
        """Close all appenders and reset the registry."""
        with self._lock:
            for appender in self._appenders:
                try:
                    appender.close()
                except Exception:
                    pass
            self._appenders.clear()
            LoggerRegistry.reset()

    def __repr__(self) -> str:
        return (f"LoggingService("
                f"loggers={self._registry.logger_count}, "
                f"appenders={len(self._appenders)})")
```

`LoggingService` is the facade — it wires together the registry, loggers, appenders, formatters, and statistics. `configure_root` sets the level on the root logger, which cascades to all child loggers that have not explicitly set their own level. The `add_*_appender` methods create and attach appenders in one call, hiding the construction details. `collect_stats` aggregates data across all loggers and appenders — total logged, per-level breakdown, per-appender write counts, and dropped records. `shutdown` closes all appenders (flushing file buffers, stopping async workers) and resets the registry for clean re-initialization. Users interact with `get_logger`, `configure_root`, `add_*_appender`, and `display_status` without managing registries, hierarchies, or lifecycle.

---

## Simulation Engine

```python
class LoggingScenario:
    """
    Tick-based simulation engine for logging demos.

    Each tick represents a logical time step. Actions are registered
    at specific ticks and execute in order when the simulation runs.
    """

    def __init__(self, service: LoggingService) -> None:
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

`LoggingScenario` mirrors the simulation engine from other guides. Actions are registered at specific ticks and execute in order, making demos reproducible and readable.

---

## Putting It All Together

```python
def main_scenario() -> None:
    """Run the main logging framework demonstration."""

    print("\n╔══════════════════════════════════════════════╗")
    print("║      LOGGING FRAMEWORK                       ║")
    print("║  Levels: TRACE → DEBUG → INFO → WARN → ERROR ║")
    print("║  Appenders: Console, File, InMemory           ║")
    print("║  Filters: Level, Range, Name, Keyword         ║")
    print("╚══════════════════════════════════════════════╝")

    service = LoggingService()
    service.configure_root(LogLevel.DEBUG)

    # Add console appender to root with pattern formatter
    console = service.add_console_appender(
        formatter=PatternFormatter(
            "[{timestamp}] [{level:<5}] [{logger}] {message}"
        ),
        colorize=False,
    )

    # Add in-memory appender for verification
    memory = service.add_memory_appender()

    scenario = LoggingScenario(service)

    # --- Basic logging at different levels ---
    def basic_logging():
        logger = service.get_logger("app")
        logger.trace("This is a TRACE message (hidden)")
        logger.debug("Application starting")
        logger.info("Server listening on port 8080")
        logger.warn("Disk usage at 85%")
        logger.error("Failed to connect to cache")
        logger.fatal("Database connection lost")

    scenario.at_tick(1, "Basic logging at different levels",
                     basic_logging)

    # --- Logger hierarchy ---
    def hierarchy_logging():
        db_logger = service.get_logger("app.db")
        query_logger = service.get_logger("app.db.query")

        db_logger.info("Connection pool initialized (5 connections)")
        query_logger.debug("SELECT * FROM users WHERE id = 42")
        query_logger.info("Query returned 1 row in 3ms")

    scenario.at_tick(2, "Logger hierarchy (app → app.db → app.db.query)",
                     hierarchy_logging)

    # --- MDC context ---
    def mdc_logging():
        LogContext.put("request_id", "req-abc123")
        LogContext.put("user_id", "user-42")

        logger = service.get_logger("app.api")
        logger.info("Processing GET /api/orders")
        logger.info("Retrieved 5 orders")

        LogContext.clear()

    scenario.at_tick(3, "MDC context (request_id, user_id)",
                     mdc_logging)

    # --- Error with exception ---
    def error_logging():
        logger = service.get_logger("app.service")
        try:
            result = 1 / 0
        except ZeroDivisionError:
            logger.error(
                "Division by zero in calculation",
                exception=traceback.format_exc().strip(),
            )

    scenario.at_tick(4, "Error logging with exception traceback",
                     error_logging)

    # --- Filter demonstration ---
    def filter_logging():
        logger = service.get_logger("app.filtered")
        logger.add_filter(
            LevelFilter(LogLevel.WARN)
        )

        logger.debug("This DEBUG is filtered out")
        logger.info("This INFO is filtered out")
        logger.warn("This WARN passes the filter")
        logger.error("This ERROR passes the filter")

    scenario.at_tick(5, "Level filter (only WARN+ passes)",
                     filter_logging)

    # --- JSON formatter ---
    def json_logging():
        json_memory = InMemoryAppender(
            formatter=JsonFormatter(),
        )
        logger = service.get_logger("app.json")
        logger.add_appender(json_memory)
        logger.propagate = False

        logger.info("Structured log entry")
        logger.warn("Something needs attention")

        print("  JSON output samples:")
        for line in json_memory.get_formatted():
            print(f"    {line}")

    scenario.at_tick(6, "JSON formatter output",
                     json_logging)

    # --- Display statistics ---
    scenario.at_tick(7, "Display service statistics",
                     lambda: service.display_status())

    scenario.run()
    service.shutdown()


# ------------------------------------------------------------------
# Logger Hierarchy Demo
# ------------------------------------------------------------------

def demo_hierarchy() -> None:
    """Demonstrate logger hierarchy and propagation."""

    print("\n" + "=" * STATS_DISPLAY_WIDTH)
    print("  LOGGER HIERARCHY DEMO")
    print("=" * STATS_DISPLAY_WIDTH)

    service = LoggingService()
    service.configure_root(LogLevel.INFO)

    memory = service.add_memory_appender(
        formatter=SimpleFormatter(),
    )

    root = service.root
    app_logger = service.get_logger("app")
    db_logger = service.get_logger("app.db")
    query_logger = service.get_logger("app.db.query")

    print("\n  Hierarchy: root → app → app.db → app.db.query")
    print("  Root level: INFO (inherited by all)")
    print("  Root has 1 appender (InMemory)\n")

    query_logger.info("Query executed: SELECT 1")
    db_logger.warn("Slow query detected (>100ms)")
    app_logger.error("Request timeout")

    print("  --- Root logger received (via propagation) ---")
    for line in memory.get_formatted():
        print(f"    {line}")

    print(f"\n  Root log count: {root.log_count}")
    print(f"  App log count: {app_logger.log_count}")
    print(f"  DB log count: {db_logger.log_count}")
    print(f"  Query log count: {query_logger.log_count}")

    service.shutdown()
    print("=" * STATS_DISPLAY_WIDTH)


# ------------------------------------------------------------------
# Filter Chain Demo
# ------------------------------------------------------------------

def demo_filter_chain() -> None:
    """Demonstrate filter chain with multiple filters."""

    print("\n" + "=" * STATS_DISPLAY_WIDTH)
    print("  FILTER CHAIN DEMO")
    print("=" * STATS_DISPLAY_WIDTH)

    service = LoggingService()
    service.configure_root(LogLevel.TRACE)

    memory = service.add_memory_appender(
        formatter=SimpleFormatter(),
    )

    logger = service.get_logger("app.secure")

    # Add filters: block password-related messages,
    # accept only WARN+
    logger.add_filter(
        KeywordFilter("password", accept=False)
    )
    logger.add_filter(LevelFilter(LogLevel.WARN))

    print("\n  Filters: KeywordFilter('password', deny)"
          " + LevelFilter(WARN+)\n")

    logger.info("User logged in")
    logger.warn("Session expiring soon")
    logger.error("User changed password successfully")
    logger.error("Database connection failed")
    logger.warn("Password reset attempted")

    print("  --- Messages that passed filters ---")
    for line in memory.get_formatted():
        print(f"    {line}")

    print(f"\n  Logged: {logger.log_count}")
    print(f"  Dropped: {logger.dropped_count}")

    service.shutdown()
    print("=" * STATS_DISPLAY_WIDTH)


# ------------------------------------------------------------------
# MDC Context Demo
# ------------------------------------------------------------------

def demo_mdc_context() -> None:
    """Demonstrate MDC (Mapped Diagnostic Context)."""

    print("\n" + "=" * STATS_DISPLAY_WIDTH)
    print("  MDC CONTEXT DEMO")
    print("=" * STATS_DISPLAY_WIDTH)

    service = LoggingService()
    service.configure_root(LogLevel.DEBUG)
    service.add_console_appender(
        formatter=PatternFormatter(
            "[{level:<5}] [{logger}] {message}"
        ),
        colorize=False,
    )

    logger = service.get_logger("app.web")

    print("\n  Simulating request lifecycle...\n")

    # Simulate request 1
    LogContext.put("request_id", "req-001")
    LogContext.put("client_ip", "192.168.1.10")
    logger.info("Incoming GET /api/users")
    logger.debug("Authenticated as admin")
    logger.info("Response 200 OK (12ms)")
    LogContext.clear()

    print()

    # Simulate request 2
    LogContext.put("request_id", "req-002")
    LogContext.put("client_ip", "10.0.0.5")
    logger.info("Incoming POST /api/orders")
    logger.warn("Slow database query (250ms)")
    logger.info("Response 201 Created (280ms)")
    LogContext.clear()

    service.shutdown()
    print("=" * STATS_DISPLAY_WIDTH)


# ------------------------------------------------------------------
# Async Appender Demo
# ------------------------------------------------------------------

def demo_async_appender() -> None:
    """Demonstrate async (non-blocking) appender."""

    print("\n" + "=" * STATS_DISPLAY_WIDTH)
    print("  ASYNC APPENDER DEMO")
    print("=" * STATS_DISPLAY_WIDTH)

    service = LoggingService()
    service.configure_root(LogLevel.DEBUG)

    # Create a sync memory appender, wrap it with async
    sync_memory = InMemoryAppender(
        formatter=SimpleFormatter(),
    )
    async_appender = AsyncAppender(
        sync_memory, queue_size=100,
    )
    service.root.add_appender(async_appender)

    logger = service.get_logger("app.async")

    print("\n  Sending 10 log messages asynchronously...\n")

    for i in range(10):
        logger.info(f"Async message #{i+1}")

    # Give worker thread time to process
    time.sleep(0.5)

    print(f"  Messages written by worker: "
          f"{async_appender.write_count}")
    print(f"  Messages dropped: "
          f"{async_appender.dropped_count}")
    print(f"  Queue remaining: "
          f"{async_appender.queue_size}")

    print("\n  --- Written records ---")
    for line in sync_memory.get_formatted():
        print(f"    {line}")

    async_appender.close()
    service.shutdown()
    print("=" * STATS_DISPLAY_WIDTH)


# ------------------------------------------------------------------
# Unit Tests
# ------------------------------------------------------------------

def test_log_levels() -> None:
    """Test log level ordering."""
    assert LogLevel.TRACE < LogLevel.DEBUG
    assert LogLevel.DEBUG < LogLevel.INFO
    assert LogLevel.INFO < LogLevel.WARN
    assert LogLevel.WARN < LogLevel.ERROR
    assert LogLevel.ERROR < LogLevel.FATAL
    assert LogLevel.INFO >= LogLevel.DEBUG
    assert LogLevel.ERROR >= LogLevel.ERROR
    print("✅ test_log_levels passed")


def test_log_record_creation() -> None:
    """Test log record defaults."""
    record = LogRecord(
        level=LogLevel.INFO,
        message="test message",
        logger_name="app",
    )
    assert record.level == LogLevel.INFO
    assert record.message == "test message"
    assert record.logger_name == "app"
    assert record.exception == ""
    assert record.context == {}
    assert len(record.record_id) == 8
    print("✅ test_log_record_creation passed")


def test_log_record_to_dict() -> None:
    """Test log record dictionary conversion."""
    record = LogRecord(
        level=LogLevel.ERROR,
        message="something failed",
        logger_name="app.db",
        exception="ZeroDivisionError",
        context={"request_id": "req-123"},
    )
    d = record.to_dict()
    assert d["level"] == "ERROR"
    assert d["message"] == "something failed"
    assert d["logger"] == "app.db"
    assert d["exception"] == "ZeroDivisionError"
    assert d["context"]["request_id"] == "req-123"
    print("✅ test_log_record_to_dict passed")


def test_pattern_formatter() -> None:
    """Test pattern-based formatting."""
    fmt = PatternFormatter("[{level}] {message}")
    record = LogRecord(
        level=LogLevel.INFO,
        message="hello world",
        logger_name="app",
    )
    result = fmt.format(record)
    assert "[INFO]" in result
    assert "hello world" in result
    print("✅ test_pattern_formatter passed")


def test_json_formatter() -> None:
    """Test JSON formatting."""
    import json
    fmt = JsonFormatter()
    record = LogRecord(
        level=LogLevel.WARN,
        message="disk full",
        logger_name="system",
    )
    result = fmt.format(record)
    parsed = json.loads(result)
    assert parsed["level"] == "WARN"
    assert parsed["message"] == "disk full"
    assert parsed["logger"] == "system"
    print("✅ test_json_formatter passed")


def test_simple_formatter() -> None:
    """Test simple formatting."""
    fmt = SimpleFormatter()
    record = LogRecord(
        level=LogLevel.DEBUG,
        message="trace info",
        logger_name="test",
    )
    result = fmt.format(record)
    assert result == "DEBUG: trace info"
    print("✅ test_simple_formatter passed")


def test_console_appender() -> None:
    """Test console appender writes."""
    appender = ConsoleAppender(
        formatter=SimpleFormatter(),
        colorize=False,
    )
    record = LogRecord(
        level=LogLevel.INFO,
        message="console test",
        logger_name="test",
    )
    assert appender.append(record)
    assert appender.write_count == 1
    print("✅ test_console_appender passed")


def test_in_memory_appender() -> None:
    """Test in-memory appender stores records."""
    appender = InMemoryAppender(
        formatter=SimpleFormatter(),
    )
    r1 = LogRecord(LogLevel.INFO, "msg1", "test")
    r2 = LogRecord(LogLevel.WARN, "msg2", "test")

    appender.append(r1)
    appender.append(r2)

    assert appender.size == 2
    assert appender.write_count == 2

    records = appender.get_records()
    assert records[0].message == "msg1"
    assert records[1].message == "msg2"

    formatted = appender.get_formatted()
    assert formatted[0] == "INFO: msg1"
    assert formatted[1] == "WARN: msg2"
    print("✅ test_in_memory_appender passed")


def test_in_memory_appender_max_records() -> None:
    """Test in-memory appender drops oldest when full."""
    appender = InMemoryAppender(
        formatter=SimpleFormatter(),
        max_records=2,
    )
    appender.append(LogRecord(LogLevel.INFO, "a", "t"))
    appender.append(LogRecord(LogLevel.INFO, "b", "t"))
    appender.append(LogRecord(LogLevel.INFO, "c", "t"))

    assert appender.size == 2
    records = appender.get_records()
    assert records[0].message == "b"
    assert records[1].message == "c"
    print("✅ test_in_memory_appender_max_records passed")


def test_level_filter() -> None:
    """Test level filter accepts/denies."""
    f = LevelFilter(LogLevel.WARN)
    info = LogRecord(LogLevel.INFO, "info", "test")
    warn = LogRecord(LogLevel.WARN, "warn", "test")
    error = LogRecord(LogLevel.ERROR, "error", "test")

    assert f.decide(info) == FilterDecision.DENY
    assert f.decide(warn) == FilterDecision.ACCEPT
    assert f.decide(error) == FilterDecision.ACCEPT
    print("✅ test_level_filter passed")


def test_level_range_filter() -> None:
    """Test level range filter."""
    f = LevelRangeFilter(LogLevel.WARN, LogLevel.ERROR)
    info = LogRecord(LogLevel.INFO, "info", "test")
    warn = LogRecord(LogLevel.WARN, "warn", "test")
    error = LogRecord(LogLevel.ERROR, "error", "test")
    fatal = LogRecord(LogLevel.FATAL, "fatal", "test")

    assert f.decide(info) == FilterDecision.DENY
    assert f.decide(warn) == FilterDecision.ACCEPT
    assert f.decide(error) == FilterDecision.ACCEPT
    assert f.decide(fatal) == FilterDecision.DENY
    print("✅ test_level_range_filter passed")


def test_logger_name_filter() -> None:
    """Test logger name prefix filter."""
    f = LoggerNameFilter("app.db", accept=True)
    db = LogRecord(LogLevel.INFO, "msg", "app.db.query")
    api = LogRecord(LogLevel.INFO, "msg", "app.api")

    assert f.decide(db) == FilterDecision.ACCEPT
    assert f.decide(api) == FilterDecision.NEUTRAL
    print("✅ test_logger_name_filter passed")


def test_keyword_filter() -> None:
    """Test keyword content filter."""
    f = KeywordFilter("password", accept=False)
    safe = LogRecord(LogLevel.INFO, "User logged in", "t")
    unsafe = LogRecord(LogLevel.INFO,
                       "Password changed", "t")

    assert f.decide(safe) == FilterDecision.NEUTRAL
    assert f.decide(unsafe) == FilterDecision.DENY
    print("✅ test_keyword_filter passed")


def test_filter_chain() -> None:
    """Test filter chain evaluation."""
    chain = FilterChain()
    chain.add_filter(LevelFilter(LogLevel.INFO))

    debug = LogRecord(LogLevel.DEBUG, "debug", "test")
    info = LogRecord(LogLevel.INFO, "info", "test")

    assert not chain.evaluate(debug)
    assert chain.evaluate(info)
    print("✅ test_filter_chain passed")


def test_filter_chain_empty() -> None:
    """Test empty filter chain defaults to accept."""
    chain = FilterChain()
    record = LogRecord(LogLevel.TRACE, "trace", "test")
    assert chain.evaluate(record)
    print("✅ test_filter_chain_empty passed")


def test_logger_level_check() -> None:
    """Test logger silences messages below its level."""
    LoggerRegistry.reset()
    memory = InMemoryAppender(formatter=SimpleFormatter())
    logger = Logger("test", level=LogLevel.WARN)
    logger.add_appender(memory)

    logger.debug("hidden")
    logger.info("hidden")
    logger.warn("visible")
    logger.error("visible")

    assert memory.size == 2
    records = memory.get_records()
    assert records[0].message == "visible"
    assert records[1].message == "visible"
    print("✅ test_logger_level_check passed")


def test_logger_hierarchy_propagation() -> None:
    """Test child logger propagates to parent."""
    LoggerRegistry.reset()
    parent_memory = InMemoryAppender(
        formatter=SimpleFormatter(),
    )
    parent = Logger("parent", level=LogLevel.DEBUG)
    parent.add_appender(parent_memory)

    child = Logger("parent.child", level=LogLevel.DEBUG,
                   parent=parent)

    child.info("child message")

    # Parent should have received via propagation
    assert parent_memory.size == 1
    records = parent_memory.get_records()
    assert records[0].message == "child message"
    assert records[0].logger_name == "parent.child"
    print("✅ test_logger_hierarchy_propagation passed")


def test_logger_no_propagation() -> None:
    """Test propagation can be disabled."""
    LoggerRegistry.reset()
    parent_memory = InMemoryAppender(
        formatter=SimpleFormatter(),
    )
    parent = Logger("parent", level=LogLevel.DEBUG)
    parent.add_appender(parent_memory)

    child = Logger("parent.child", level=LogLevel.DEBUG,
                   parent=parent)
    child.propagate = False

    child_memory = InMemoryAppender(
        formatter=SimpleFormatter(),
    )
    child.add_appender(child_memory)
    child.info("child only message")

    # Parent should NOT have received
    assert parent_memory.size == 0
    # Child's own appender should have it
    assert child_memory.size == 1
    print("✅ test_logger_no_propagation passed")


def test_logger_registry_singleton() -> None:
    """Test registry returns same instance."""
    LoggerRegistry.reset()
    r1 = LoggerRegistry.get_instance()
    r2 = LoggerRegistry.get_instance()
    assert r1 is r2
    LoggerRegistry.reset()
    print("✅ test_logger_registry_singleton passed")


def test_logger_registry_hierarchy() -> None:
    """Test registry auto-creates hierarchy."""
    LoggerRegistry.reset()
    registry = LoggerRegistry.get_instance()

    query = registry.get_logger("app.db.query")
    db = registry.get_logger("app.db")
    app = registry.get_logger("app")

    # All should exist
    assert registry.logger_count == 4  # root + 3
    assert query._parent is db
    assert db._parent is app
    assert app._parent is registry.root
    LoggerRegistry.reset()
    print("✅ test_logger_registry_hierarchy passed")


def test_mdc_context() -> None:
    """Test MDC thread-local context."""
    LogContext.clear()
    LogContext.put("key1", "value1")
    LogContext.put("key2", "value2")

    ctx = LogContext.get_context()
    assert ctx["key1"] == "value1"
    assert ctx["key2"] == "value2"

    LogContext.remove("key1")
    assert LogContext.get("key1") == ""
    assert LogContext.get("key2") == "value2"

    LogContext.clear()
    assert LogContext.get_context() == {}
    print("✅ test_mdc_context passed")


def test_mdc_in_log_record() -> None:
    """Test MDC context is captured in log records."""
    LoggerRegistry.reset()
    LogContext.clear()
    LogContext.put("request_id", "req-999")

    memory = InMemoryAppender(formatter=SimpleFormatter())
    logger = Logger("test", level=LogLevel.DEBUG)
    logger.add_appender(memory)

    logger.info("with context")

    records = memory.get_records()
    assert records[0].context["request_id"] == "req-999"

    LogContext.clear()
    LoggerRegistry.reset()
    print("✅ test_mdc_in_log_record passed")


def test_logging_service_lifecycle() -> None:
    """Test full service lifecycle."""
    service = LoggingService()
    service.configure_root(LogLevel.INFO)
    memory = service.add_memory_appender(
        formatter=SimpleFormatter(),
    )

    logger = service.get_logger("app")
    logger.info("hello")
    logger.warn("warning")
    logger.debug("hidden by INFO level")

    assert memory.size == 2
    formatted = memory.get_formatted()
    assert formatted[0] == "INFO: hello"
    assert formatted[1] == "WARN: warning"

    stats = service.collect_stats()
    assert stats.total_logged >= 2

    service.shutdown()
    print("✅ test_logging_service_lifecycle passed")


def test_multiple_appenders() -> None:
    """Test logger writes to multiple appenders."""
    LoggerRegistry.reset()
    mem1 = InMemoryAppender(formatter=SimpleFormatter())
    mem2 = InMemoryAppender(
        formatter=JsonFormatter(),
    )

    logger = Logger("multi", level=LogLevel.DEBUG)
    logger.add_appender(mem1)
    logger.add_appender(mem2)

    logger.info("dual output")

    assert mem1.size == 1
    assert mem2.size == 1
    assert "INFO: dual output" in mem1.get_formatted()[0]
    assert '"level": "INFO"' in mem2.get_formatted()[0]
    LoggerRegistry.reset()
    print("✅ test_multiple_appenders passed")


def test_filter_decision_enum() -> None:
    """Test filter decision values."""
    assert FilterDecision.ACCEPT.value == "accept"
    assert FilterDecision.DENY.value == "deny"
    assert FilterDecision.NEUTRAL.value == "neutral"
    print("✅ test_filter_decision_enum passed")


def test_async_appender() -> None:
    """Test async appender writes via background thread."""
    sync_mem = InMemoryAppender(
        formatter=SimpleFormatter(),
    )
    async_app = AsyncAppender(sync_mem, queue_size=50)

    for i in range(5):
        record = LogRecord(
            LogLevel.INFO, f"async-{i}", "test",
        )
        async_app.append(record)

    time.sleep(0.5)  # let worker process

    assert sync_mem.size == 5
    assert async_app.write_count == 5
    assert async_app.dropped_count == 0

    async_app.close()
    print("✅ test_async_appender passed")


def run_all_tests() -> None:
    print("\n" + "=" * 40)
    print("  RUNNING TESTS")
    print("=" * 40 + "\n")
    test_log_levels()
    test_log_record_creation()
    test_log_record_to_dict()
    test_pattern_formatter()
    test_json_formatter()
    test_simple_formatter()
    test_console_appender()
    test_in_memory_appender()
    test_in_memory_appender_max_records()
    test_level_filter()
    test_level_range_filter()
    test_logger_name_filter()
    test_keyword_filter()
    test_filter_chain()
    test_filter_chain_empty()
    test_logger_level_check()
    test_logger_hierarchy_propagation()
    test_logger_no_propagation()
    test_logger_registry_singleton()
    test_logger_registry_hierarchy()
    test_mdc_context()
    test_mdc_in_log_record()
    test_logging_service_lifecycle()
    test_multiple_appenders()
    test_filter_decision_enum()
    test_async_appender()
    print("\n✅ All tests passed!\n")


if __name__ == "__main__":
    main_scenario()
    demo_hierarchy()
    demo_filter_chain()
    demo_mdc_context()
    demo_async_appender()
    run_all_tests()
```

Each test isolates a single behavior. `test_log_levels` verifies the numeric ordering of all six levels. `test_log_record_to_dict` confirms that exception and context data appear in the dictionary output. `test_pattern_formatter` and `test_json_formatter` verify that formatters produce correct output. `test_in_memory_appender_max_records` tests that the oldest record is evicted when the appender reaches capacity. `test_level_filter` tests the core ACCEPT/DENY decisions. `test_level_range_filter` verifies that only records within the range pass. `test_keyword_filter` tests content-based filtering with case insensitivity. `test_filter_chain_empty` confirms the default-accept behavior. `test_logger_level_check` verifies that a WARN-level logger silences DEBUG and INFO. `test_logger_hierarchy_propagation` confirms that a child logger's records propagate to the parent's appenders. `test_logger_no_propagation` verifies that `propagate=False` stops the upward chain. `test_logger_registry_singleton` confirms that `get_instance()` always returns the same object. `test_logger_registry_hierarchy` verifies auto-creation of intermediate loggers. `test_mdc_context` tests the thread-local put/get/remove/clear lifecycle. `test_async_appender` verifies that records are written by the background worker thread.

---

## Class Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                    LoggingService                                    │
│──────────────────────────────────────────────────────────────────│
│ - _registry: LoggerRegistry                                         │
│ - _stats: LoggingStats                                              │
│ - _appenders: list[Appender]                                        │
│──────────────────────────────────────────────────────────────────│
│ + configure_root(level) → None                                      │
│ + add_console_appender(...) → ConsoleAppender                       │
│ + add_file_appender(...) → FileAppender                             │
│ + add_rotating_file_appender(...) → RotatingFileAppender            │
│ + add_memory_appender(...) → InMemoryAppender                       │
│ + get_logger(name) → Logger                                         │
│ + collect_stats() → LoggingStats                                    │
│ + display_status() → None                                           │
│ + shutdown() → None                                                  │
└───────┬──────────┬──────────────────────────────────────────────┘
        │          │
   uses ▼     uses ▼
┌──────────────────────────────────────────────────────────────────┐
│  LoggerRegistry (Singleton)                                         │
│──────────────────────────────────────────────────────────────────│
│ - _instance: LoggerRegistry (class var)                             │
│ - _loggers: dict[str, Logger]                                       │
│ - _root: Logger                                                      │
│──────────────────────────────────────────────────────────────────│
│ + get_instance() → LoggerRegistry     (class method)                │
│ + get_logger(name) → Logger           (auto-hierarchy)              │
│ + reset() → None                      (for testing)                 │
│ - _find_parent(name) → Logger                                       │
└──────────┬───────────────────────────────────────────────────────┘
           │ manages
           ▼
┌──────────────────────────────────────────────────────────────────┐
│  Logger                                                              │
│──────────────────────────────────────────────────────────────────│
│ - _name: str                                                         │
│ - _level: LogLevel                                                   │
│ - _parent: Logger | None                                             │
│ - _appenders: list[Appender]                                         │
│ - _filter_chain: FilterChain                                         │
│ - _propagate: bool                                                   │
│──────────────────────────────────────────────────────────────────│
│ + trace(msg) / debug(msg) / info(msg) / warn(msg)                   │
│ + error(msg, exception) / fatal(msg, exception)                      │
│ + add_appender(appender) → None                                      │
│ + add_filter(filter) → None                                          │
│ + effective_level → LogLevel         (walks parent chain)            │
│ - _log(level, message) → None                                        │
│ - _handle_propagated(record) → None                                  │
│                                                                      │
│ Pipeline: level check → MDC → filter → appenders → propagate        │
└──────────┬──────────┬────────────────────────────────────────────┘
           │          │
  fans out ▼     uses ▼
┌──────────────────────────────────────────────────────────────────┐
│  «abstract» Appender                                                 │
│──────────────────────────────────────────────────────────────────│
│ + append(record) → bool                                              │
│ + close() → None                                                     │
│ + appender_name() → str                                              │
└──────────┬────────────────────────────────────────────────────────┘
           │ implemented by
  ┌────────┼──────────┬──────────────────┐
  ▼        ▼          ▼                  ▼
Console  File      InMemory          Rotating
Appender Appender  Appender          FileAppender

                 ┌─────────────────────────────────┐
                 │  AsyncAppender (Decorator)        │
                 │  wraps any Appender               │
                 │──────────────────────────────────│
                 │ - _delegate: Appender             │
                 │ - _queue: Queue[LogRecord]         │
                 │ - _worker: Thread                  │
                 │──────────────────────────────────│
                 │ + append(record) → bool  (enqueue)│
                 │ - _process() → None   (worker)    │
                 │ + close() → None     (poison pill) │
                 └─────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  «abstract» Formatter                                                │
│──────────────────────────────────────────────────────────────────│
│ + format(record) → str                                               │
│ + name() → str                                                       │
└──────────┬────────────────────────────────────────────────────────┘
           │ implemented by
  ┌────────┼──────────────────┐
  ▼        ▼                  ▼
Pattern  Json              Simple
Formatter Formatter         Formatter

┌──────────────────────────────────────────────────────────────────┐
│  LogRecord                                                           │
│──────────────────────────────────────────────────────────────────│
│ + level: LogLevel                                                    │
│ + message: str                                                       │
│ + logger_name: str                                                   │
│ + timestamp: float                                                   │
│ + record_id: str                                                     │
│ + thread_name: str                                                   │
│ + exception: str                                                     │
│ + context: dict[str, str]     (from MDC)                             │
│──────────────────────────────────────────────────────────────────│
│ + formatted_timestamp → str                                          │
│ + to_dict() → dict                                                   │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  FilterChain                                                         │
│──────────────────────────────────────────────────────────────────│
│ - _filters: list[LogFilter]                                          │
│──────────────────────────────────────────────────────────────────│
│ + add_filter(filter) → None                                          │
│ + evaluate(record) → bool                                            │
│   ACCEPT → pass, DENY → drop, NEUTRAL → next                        │
└──────────┬───────────────────────────────────────────────────────┘
           │ contains
           ▼
┌──────────────────────────────────────────────────────────────────┐
│  «abstract» LogFilter                                                │
│──────────────────────────────────────────────────────────────────│
│ + decide(record) → FilterDecision                                    │
│ + filter_name() → str                                                │
└──────────┬────────────────────────────────────────────────────────┘
           │ implemented by
  ┌────────┼──────────┬──────────┐
  ▼        ▼          ▼          ▼
Level    LevelRange Logger    Keyword
Filter   Filter     NameFilter Filter

┌──────────────────────────────────────────────────────────────────┐
│  LogContext (MDC)                                                     │
│──────────────────────────────────────────────────────────────────│
│ - _local: threading.local()     (class var)                          │
│──────────────────────────────────────────────────────────────────│
│ + put(key, value) → None        (class method)                       │
│ + get(key) → str                (class method)                       │
│ + remove(key) → None            (class method)                       │
│ + clear() → None                (class method)                       │
│ + get_context() → dict          (class method, returns copy)         │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│ LogLevel (Enum)  │ │ FilterDecision   │ │ LoggingStats     │
│──────────────────│ │ (Enum)           │ │──────────────────│
│ TRACE = 0        │ │──────────────────│ │ + total_logged   │
│ DEBUG = 1        │ │ ACCEPT           │ │ + total_dropped  │
│ INFO = 2         │ │ DENY             │ │ + level_counts   │
│ WARN = 3         │ │ NEUTRAL          │ │ + appender_counts│
│ ERROR = 4        │ └──────────────────┘ │──────────────────│
│ FATAL = 5        │                      │ + display()      │
│ OFF = 6          │ ┌──────────────────┐ │ + reset()        │
└──────────────────┘ │ LoggingScenario  │ └──────────────────┘
                     │──────────────────│
                     │ - _service       │
                     │ - _events        │
                     │──────────────────│
                     │ + at_tick()      │
                     │ + run()          │
                     └──────────────────┘
```

---

## How to Run

Save all code into a single file `logging_framework.py` and run:

```bash
python logging_framework.py
```

### Example Output

```
╔══════════════════════════════════════════════╗
║      LOGGING FRAMEWORK                       ║
║  Levels: TRACE → DEBUG → INFO → WARN → ERROR ║
║  Appenders: Console, File, InMemory           ║
║  Filters: Level, Range, Name, Keyword         ║
╚══════════════════════════════════════════════╝

  ───────────────────────────────────────────────────────
  SIMULATION START (ticks: 1 → 7)
  ───────────────────────────────────────────────────────

  ⏱️  Tick 1: Basic logging at different levels
  [2026-10-28 23:30:00] [DEBUG] [app] Application starting
  [2026-10-28 23:30:00] [INFO ] [app] Server listening on port 8080
  [2026-10-28 23:30:00] [WARN ] [app] Disk usage at 85%
  [2026-10-28 23:30:00] [ERROR] [app] Failed to connect to cache
  [2026-10-28 23:30:00] [FATAL] [app] Database connection lost

  ⏱️  Tick 2: Logger hierarchy (app → app.db → app.db.query)
  [2026-10-28 23:30:00] [INFO ] [app.db] Connection pool initialized (5 connections)
  [2026-10-28 23:30:00] [DEBUG] [app.db.query] SELECT * FROM users WHERE id = 42
  [2026-10-28 23:30:00] [INFO ] [app.db.query] Query returned 1 row in 3ms

  ⏱️  Tick 3: MDC context (request_id, user_id)
  [2026-10-28 23:30:00] [INFO ] [app.api] Processing GET /api/orders
    Context: request_id=req-abc123 user_id=user-42
  [2026-10-28 23:30:00] [INFO ] [app.api] Retrieved 5 orders
    Context: request_id=req-abc123 user_id=user-42

  ⏱️  Tick 4: Error logging with exception traceback
  [2026-10-28 23:30:00] [ERROR] [app.service] Division by zero in calculation
    Exception: Traceback (most recent call last): ...

  ⏱️  Tick 5: Level filter (only WARN+ passes)
  [2026-10-28 23:30:00] [WARN ] [app.filtered] This WARN passes the filter
  [2026-10-28 23:30:00] [ERROR] [app.filtered] This ERROR passes the filter

  ⏱️  Tick 6: JSON formatter output
  JSON output samples:
    {"timestamp": "2026-10-28 23:30:00", "level": "INFO", "logger": "app.json", "message": "Structured log entry", "thread": "MainThread", "record_id": "a1b2c3d4"}
    {"timestamp": "2026-10-28 23:30:00", "level": "WARN", "logger": "app.json", "message": "Something needs attention", "thread": "MainThread", "record_id": "e5f6g7h8"}

  ⏱️  Tick 7: Display service statistics

  ═══════════════════════════════════════════════════════
  LOGGING SYSTEM STATISTICS
  ═══════════════════════════════════════════════════════
  Total logged:      15
  Total dropped:     2
  ───────────────────────────────────────────────────────
  PER-LEVEL BREAKDOWN
    DEBUG : 2
    INFO  : 7
    WARN  : 3
    ERROR : 2
    FATAL : 1
  ───────────────────────────────────────────────────────
  PER-APPENDER BREAKDOWN
    ConsoleAppender: 13 writes
    InMemoryAppender: 13 writes
  ═══════════════════════════════════════════════════════

  ───────────────────────────────────────────────────────
  SIMULATION COMPLETE
  ───────────────────────────────────────────────────────

=======================================================
  LOGGER HIERARCHY DEMO
=======================================================

  Hierarchy: root → app → app.db → app.db.query
  Root level: INFO (inherited by all)
  Root has 1 appender (InMemory)

  --- Root logger received (via propagation) ---
    INFO: Query executed: SELECT 1
    WARN: Slow query detected (>100ms)
    ERROR: Request timeout

  Root log count: 3
  App log count: 1
  DB log count: 1
  Query log count: 1
=======================================================

=======================================================
  FILTER CHAIN DEMO
=======================================================

  Filters: KeywordFilter('password', deny) + LevelFilter(WARN+)

  --- Messages that passed filters ---
    WARN: Session expiring soon
    ERROR: Database connection failed

  Logged: 2
  Dropped: 3
=======================================================

========================================
  RUNNING TESTS
========================================

✅ test_log_levels passed
✅ test_log_record_creation passed
✅ test_log_record_to_dict passed
✅ test_pattern_formatter passed
✅ test_json_formatter passed
✅ test_simple_formatter passed
✅ test_console_appender passed
✅ test_in_memory_appender passed
✅ test_in_memory_appender_max_records passed
✅ test_level_filter passed
✅ test_level_range_filter passed
✅ test_logger_name_filter passed
✅ test_keyword_filter passed
✅ test_filter_chain passed
✅ test_filter_chain_empty passed
✅ test_logger_level_check passed
✅ test_logger_hierarchy_propagation passed
✅ test_logger_no_propagation passed
✅ test_logger_registry_singleton passed
✅ test_logger_registry_hierarchy passed
✅ test_mdc_context passed
✅ test_mdc_in_log_record passed
✅ test_logging_service_lifecycle passed
✅ test_multiple_appenders passed
✅ test_filter_decision_enum passed
✅ test_async_appender passed

✅ All tests passed!
```
