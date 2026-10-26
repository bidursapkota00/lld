# Design Task Scheduler – Low-Level Design Guide

![Bidur Sapkota](https://www.bidursapkota.com.np/images/gravatar.webp "Bidur Sapkota - Developer")&nbsp;[Bidur Sapkota](https://www.bidursapkota.com.np/)

## Table of Contents

1. [Requirements](#requirements)
2. [Key Design Patterns](#key-design-patterns)
3. [Core Enums & Constants](#core-enums--constants)
4. [Task & Task Result](#task--task-result)
5. [Priority Queue (Heap-Based)](#priority-queue-heap-based)
6. [Scheduling Policy (Strategy Pattern)](#scheduling-policy-strategy-pattern)
7. [Task Executor](#task-executor)
8. [Delayed Job Scheduler](#delayed-job-scheduler)
9. [Recurring Job Support](#recurring-job-support)
10. [Task Event System (Observer Pattern)](#task-event-system-observer-pattern)
11. [Worker Pool](#worker-pool)
12. [Task Retry & Dead Letter Queue](#task-retry--dead-letter-queue)
13. [Task Dependencies (DAG Execution)](#task-dependencies-dag-execution)
14. [Scheduler Facade](#scheduler-facade)
15. [Simulation Engine](#simulation-engine)
16. [Putting It All Together](#putting-it-all-together)
17. [Class Diagram](#class-diagram)
18. [How to Run](#how-to-run)

---

## Requirements

### Functional Requirements

| #  | Requirement                                                                                       |
| -- | ------------------------------------------------------------------------------------------------- |
| 1  | `submit(task)` enqueues a task for execution based on its priority                                |
| 2  | `schedule(task, delay)` enqueues a task to run after a specified delay                            |
| 3  | Pluggable scheduling policies — Priority, FIFO, Shortest Job First can be swapped                |
| 4  | Recurring jobs — tasks can be scheduled to run at fixed intervals                                 |
| 5  | Task dependencies — a task can depend on other tasks (DAG-based execution order)                  |
| 6  | Observer-based event system — listeners react to submitted, started, completed, and failed events |
| 7  | Worker pool — configurable number of workers execute tasks concurrently                           |
| 8  | Retry with backoff — failed tasks are retried with exponential backoff up to a max retry count    |
| 9  | Dead letter queue — tasks that exhaust all retries are moved to a DLQ for inspection              |
| 10 | Statistics — total submitted, completed, failed, average execution time, and queue depth          |

### Non-Functional Requirements

- Clean OOP with SOLID principles.
- Extensible — new scheduling policies, task types, and event listeners can be added without modifying core logic.
- Simulatable — the system runs step-by-step for testing and demonstration.

---

## Key Design Patterns

| Pattern   | Where It Is Used                                   | Why                                                                                        |
| --------- | -------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| Strategy  | `SchedulingPolicy` decides task execution order    | Swap between Priority, FIFO, or SJF without changing scheduler logic                       |
| Observer  | `TaskEventListener` reacts to task lifecycle events | Loggers, metrics, and alerting react independently to task state changes                   |
| Command   | `Task` encapsulates work as an object               | Tasks are created, queued, serialized, retried, and inspected as first-class objects       |
| Facade    | `TaskSchedulerSystem` exposes a unified API         | Hides the complexity of queues, workers, retries, events, and dependency resolution        |

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
from collections import defaultdict, deque


class TaskStatus(Enum):
    PENDING = "pending"              # waiting in queue
    SCHEDULED = "scheduled"          # delayed, waiting for trigger time
    RUNNING = "running"              # currently executing
    COMPLETED = "completed"          # finished successfully
    FAILED = "failed"                # execution raised an exception
    RETRYING = "retrying"            # waiting for retry
    DEAD = "dead"                    # exhausted all retries (in DLQ)
    CANCELLED = "cancelled"          # cancelled before execution


class TaskPriority(Enum):
    CRITICAL = 0                     # highest priority (lowest number)
    HIGH = 1
    MEDIUM = 2
    LOW = 3
    BACKGROUND = 4                   # lowest priority


class TaskEventType(Enum):
    SUBMITTED = "submitted"
    STARTED = "started"
    COMPLETED = "completed"
    FAILED = "failed"
    RETRYING = "retrying"
    DEAD_LETTERED = "dead_lettered"
    CANCELLED = "cancelled"
    DELAYED = "delayed"
    RECURRING = "recurring"
    DEPENDENCY_MET = "dependency_met"


# ------------------------------------------------------------------
# System constants
# ------------------------------------------------------------------

DEFAULT_MAX_WORKERS = 4
DEFAULT_MAX_RETRIES = 3
DEFAULT_RETRY_BACKOFF = 1.0        # base seconds for exponential backoff
DEFAULT_QUEUE_CAPACITY = 10_000
STATS_DISPLAY_WIDTH = 50
```

`TaskStatus` tracks every state a task can be in. `PENDING` means the task is in the queue waiting for a worker. `SCHEDULED` means it has a future trigger time and is not yet eligible. `RETRYING` means it failed but will be retried after a backoff delay. `DEAD` means all retries are exhausted. `TaskPriority` uses integer values where lower is higher priority — this maps directly to heap ordering. `CRITICAL` (0) always executes before `BACKGROUND` (4). `DEFAULT_RETRY_BACKOFF` of 1.0 means the first retry waits 1 second, the second waits 2 seconds, the third waits 4 seconds (exponential backoff with base 2).

---

## Task & Task Result

```python
@dataclass
class TaskResult:
    """
    The outcome of a task execution.

    Created when a task completes (successfully or with an error).
    Contains the return value or the exception, plus timing data.
    """
    task_id: str
    success: bool
    value: Any = None
    error: str = ""
    execution_time_ms: float = 0.0
    attempts: int = 1
    completed_at: float = field(default_factory=time.time)


class Task:
    """
    A unit of work submitted to the scheduler.

    Encapsulates the callable to execute, its priority, optional
    delay, retry policy, and execution metadata. Tasks are
    first-class objects — they can be queued, inspected,
    retried, cancelled, and compared.

    Ordering:
        Tasks are compared by (priority, created_at) so that
        higher priority tasks run first. Among equal priorities,
        the oldest task runs first (FIFO within priority).
    """

    def __init__(
        self,
        name: str,
        action: Callable[[], Any],
        priority: TaskPriority = TaskPriority.MEDIUM,
        delay_seconds: float = 0.0,
        max_retries: int = DEFAULT_MAX_RETRIES,
        estimated_duration_ms: float = 0.0,
        depends_on: list[str] | None = None,
        interval_seconds: float = 0.0,
    ) -> None:
        self.task_id: str = str(uuid.uuid4())[:8]
        self.name: str = name
        self.action: Callable[[], Any] = action
        self.priority: TaskPriority = priority
        self.status: TaskStatus = TaskStatus.PENDING
        self.delay_seconds: float = delay_seconds
        self.max_retries: int = max_retries
        self.retry_count: int = 0
        self.estimated_duration_ms: float = estimated_duration_ms
        self.depends_on: list[str] = depends_on or []
        self.interval_seconds: float = interval_seconds
        self.created_at: float = time.time()
        self.started_at: float = 0.0
        self.completed_at: float = 0.0
        self.trigger_at: float = (
            self.created_at + delay_seconds
            if delay_seconds > 0 else 0.0
        )
        self.result: TaskResult | None = None
        self.error_history: list[str] = []

        if delay_seconds > 0:
            self.status = TaskStatus.SCHEDULED

    @property
    def is_ready(self) -> bool:
        """Check if task is ready to execute (delay has passed)."""
        if self.trigger_at <= 0:
            return True
        return time.time() >= self.trigger_at

    @property
    def is_recurring(self) -> bool:
        return self.interval_seconds > 0

    @property
    def is_terminal(self) -> bool:
        """Check if task is in a final state."""
        return self.status in (
            TaskStatus.COMPLETED,
            TaskStatus.DEAD,
            TaskStatus.CANCELLED,
        )

    @property
    def can_retry(self) -> bool:
        return self.retry_count < self.max_retries

    def execute(self) -> TaskResult:
        """
        Run the task's action and return the result.

        Captures execution time and any exceptions. Does not
        handle retries — that is the executor's responsibility.
        """
        self.status = TaskStatus.RUNNING
        self.started_at = time.time()

        try:
            value = self.action()
            elapsed = (time.time() - self.started_at) * 1000
            self.status = TaskStatus.COMPLETED
            self.completed_at = time.time()
            self.result = TaskResult(
                task_id=self.task_id,
                success=True,
                value=value,
                execution_time_ms=elapsed,
                attempts=self.retry_count + 1,
            )
            return self.result
        except Exception as e:
            elapsed = (time.time() - self.started_at) * 1000
            self.status = TaskStatus.FAILED
            self.error_history.append(str(e))
            self.result = TaskResult(
                task_id=self.task_id,
                success=False,
                error=str(e),
                execution_time_ms=elapsed,
                attempts=self.retry_count + 1,
            )
            return self.result

    def __lt__(self, other: "Task") -> bool:
        """Priority comparison for heap ordering."""
        if self.priority.value != other.priority.value:
            return self.priority.value < other.priority.value
        return self.created_at < other.created_at

    def __repr__(self) -> str:
        return (f"Task(id={self.task_id}, "
                f"name={self.name}, "
                f"priority={self.priority.name}, "
                f"status={self.status.value})")
```

`Task` is the command object — it encapsulates work (the `action` callable), scheduling metadata (priority, delay, retries), and execution results. `__lt__` implements heap ordering: lower priority value runs first (`CRITICAL=0` before `BACKGROUND=4`); ties are broken by creation time (FIFO among same priority). `trigger_at` is the Unix timestamp when a delayed task becomes eligible — before that time, the task sits in the delayed queue. `is_ready` checks if the delay has elapsed. `execute` runs the action, captures timing, and wraps the outcome in a `TaskResult`. Exceptions are caught and stored in `error_history` so the retry system has visibility into what went wrong. `is_terminal` checks if the task has reached a final state where no further action is possible.

---

## Priority Queue (Heap-Based)

```python
class TaskQueue:
    """
    Min-heap priority queue for tasks.

    Uses Python's heapq module. Tasks are ordered by
    (priority_value, created_at) so higher priority tasks
    (lower value) are dequeued first. Within the same
    priority, older tasks are dequeued first (FIFO).

    Internal structure:
        heap: [(task1), (task2), ...]
        Task.__lt__ drives the ordering.
    """

    def __init__(self, capacity: int = DEFAULT_QUEUE_CAPACITY
                 ) -> None:
        self._heap: list[Task] = []
        self._capacity = capacity
        self._lock = threading.RLock()

    def push(self, task: Task) -> bool:
        """
        Add a task to the queue.

        Returns False if the queue is at capacity.
        """
        with self._lock:
            if len(self._heap) >= self._capacity:
                return False
            heapq.heappush(self._heap, task)
            return True

    def pop(self) -> Task | None:
        """Remove and return the highest-priority task."""
        with self._lock:
            if not self._heap:
                return None
            return heapq.heappop(self._heap)

    def peek(self) -> Task | None:
        """Return the highest-priority task without removing it."""
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

    def to_list(self) -> list[Task]:
        """Return tasks in priority order (does not modify queue)."""
        with self._lock:
            return sorted(self._heap)

    def __repr__(self) -> str:
        return (f"TaskQueue(size={len(self._heap)}, "
                f"capacity={self._capacity})")
```

`TaskQueue` wraps Python's `heapq` to provide a thread-safe min-heap. `heapq.heappush` and `heapq.heappop` maintain the heap invariant in O(log n). The ordering is driven by `Task.__lt__` — `CRITICAL` tasks float to the top. `capacity` prevents unbounded queue growth; `push` returns `False` when full, giving the caller a chance to reject or backpressure. `peek` returns the next task without removing it, useful for the delayed scheduler to check if the top task's trigger time has arrived. `to_list` returns a sorted copy for inspection without disturbing the heap.

---

## Scheduling Policy (Strategy Pattern)

```python
class SchedulingPolicy(ABC):
    """
    Defines which task to execute next from the queue.

    Each policy determines the ordering of tasks. The scheduler
    delegates the "which task next?" decision to the policy
    without knowing how it's made.
    """

    @abstractmethod
    def enqueue(self, task: Task, queue: TaskQueue) -> bool:
        """Add a task to the queue using policy-specific logic."""
        ...

    @abstractmethod
    def dequeue(self, queue: TaskQueue) -> Task | None:
        """Remove and return the next task to execute."""
        ...

    @abstractmethod
    def name(self) -> str:
        ...


class PrioritySchedulingPolicy(SchedulingPolicy):
    """
    Execute tasks in priority order.

    The heap naturally orders by (priority, created_at).
    CRITICAL tasks always run before LOW tasks. Within
    the same priority, FIFO order is preserved.
    """

    def enqueue(self, task: Task, queue: TaskQueue) -> bool:
        return queue.push(task)

    def dequeue(self, queue: TaskQueue) -> Task | None:
        return queue.pop()

    def name(self) -> str:
        return "Priority"


class FIFOSchedulingPolicy(SchedulingPolicy):
    """
    Execute tasks in submission order, ignoring priority.

    Overrides the task's priority to MEDIUM before enqueuing
    so all tasks have equal priority and the heap degenerates
    to FIFO (ordered by created_at).
    """

    def enqueue(self, task: Task, queue: TaskQueue) -> bool:
        task.priority = TaskPriority.MEDIUM
        return queue.push(task)

    def dequeue(self, queue: TaskQueue) -> Task | None:
        return queue.pop()

    def name(self) -> str:
        return "FIFO"


class ShortestJobFirstPolicy(SchedulingPolicy):
    """
    Execute the shortest estimated task first.

    Wraps tasks with a comparison key based on estimated
    duration. Tasks with shorter estimated_duration_ms
    run first. If no estimate is provided, falls back
    to priority ordering.
    """

    def __init__(self) -> None:
        self._sjf_queue: list[tuple[float, float, Task]] = []
        self._lock = threading.RLock()

    def enqueue(self, task: Task, queue: TaskQueue) -> bool:
        with self._lock:
            heapq.heappush(
                self._sjf_queue,
                (task.estimated_duration_ms,
                 task.created_at, task),
            )
            return True

    def dequeue(self, queue: TaskQueue) -> Task | None:
        with self._lock:
            if not self._sjf_queue:
                return None
            _, _, task = heapq.heappop(self._sjf_queue)
            return task

    def name(self) -> str:
        return "ShortestJobFirst"
```

`SchedulingPolicy` defines the contract for all scheduling strategies. `PrioritySchedulingPolicy` uses the heap's natural ordering — `Task.__lt__` compares by `(priority.value, created_at)`. `FIFOSchedulingPolicy` forces all tasks to `MEDIUM` priority so the heap degenerates to creation-time order. `ShortestJobFirstPolicy` maintains its own heap keyed by `estimated_duration_ms` — tasks with shorter estimates run first, minimizing average wait time. Adding a new policy (e.g., Round Robin, Fair Share) requires only implementing `enqueue`, `dequeue`, and `name` — the scheduler never changes.

---

## Task Executor

```python
class TaskExecutor:
    """
    Executes a single task with retry support.

    Handles the execute → check result → retry loop. On failure,
    applies exponential backoff before retrying. If all retries
    are exhausted, the task is marked as DEAD.

    Backoff formula: wait = base × 2^(retry_count - 1)
    Retry 1: 1.0s, Retry 2: 2.0s, Retry 3: 4.0s
    """

    def __init__(
        self,
        retry_backoff: float = DEFAULT_RETRY_BACKOFF,
    ) -> None:
        self._retry_backoff = retry_backoff

    def execute(self, task: Task) -> TaskResult:
        """
        Execute the task, retrying on failure.

        Returns the final TaskResult after either success
        or exhaustion of all retries.
        """
        while True:
            result = task.execute()

            if result.success:
                return result

            # Task failed — check retry eligibility
            if task.can_retry:
                task.retry_count += 1
                task.status = TaskStatus.RETRYING
                backoff = (self._retry_backoff
                           * (2 ** (task.retry_count - 1)))
                time.sleep(backoff)
                task.status = TaskStatus.PENDING
            else:
                task.status = TaskStatus.DEAD
                return result

    def execute_no_retry(self, task: Task) -> TaskResult:
        """Execute once without retrying (for simulation)."""
        result = task.execute()
        if not result.success and not task.can_retry:
            task.status = TaskStatus.DEAD
        return result
```

`TaskExecutor` handles the retry loop. On failure, it checks `task.can_retry` (retry_count < max_retries), increments the counter, computes exponential backoff (`base × 2^(attempt-1)`), sleeps, then re-executes. The backoff sequence with `base=1.0` is: 1s, 2s, 4s, 8s — each retry doubles. If all retries fail, the task is marked `DEAD`. `execute_no_retry` is a convenience method for simulation where we don't want actual sleeps.

---

## Delayed Job Scheduler

```python
class DelayedJobScheduler:
    """
    Manages tasks with future trigger times.

    Delayed tasks are stored in a separate min-heap ordered
    by trigger_at. The scheduler periodically checks if any
    delayed tasks are ready and promotes them to the main queue.

    Internal structure:
        delayed_heap: [(trigger_at, task), ...]

    A task with delay_seconds=30 has trigger_at = now + 30.
    When current time >= trigger_at, the task is moved to
    the main execution queue.
    """

    def __init__(self) -> None:
        self._delayed: list[tuple[float, Task]] = []
        self._lock = threading.RLock()

    def schedule(self, task: Task) -> None:
        """Add a task to the delayed queue."""
        with self._lock:
            task.status = TaskStatus.SCHEDULED
            heapq.heappush(
                self._delayed,
                (task.trigger_at, task),
            )

    def get_ready_tasks(self) -> list[Task]:
        """
        Return all tasks whose trigger time has passed.

        Pops tasks from the delayed heap until the top
        task's trigger time is in the future.
        """
        ready: list[Task] = []
        with self._lock:
            now = time.time()
            while (self._delayed
                   and self._delayed[0][0] <= now):
                _, task = heapq.heappop(self._delayed)
                task.status = TaskStatus.PENDING
                ready.append(task)
        return ready

    @property
    def size(self) -> int:
        return len(self._delayed)

    @property
    def next_trigger(self) -> float | None:
        """Return the trigger time of the next delayed task."""
        with self._lock:
            if not self._delayed:
                return None
            return self._delayed[0][0]

    def clear(self) -> None:
        with self._lock:
            self._delayed.clear()

    def __repr__(self) -> str:
        return (f"DelayedJobScheduler("
                f"pending={len(self._delayed)})")
```

`DelayedJobScheduler` uses a min-heap ordered by `trigger_at`. `schedule` pushes the task with its trigger timestamp. `get_ready_tasks` pops all tasks whose trigger time has passed — this is called periodically by the main scheduler. The heap ensures we only check the earliest task: if it's not ready, nothing behind it is either, so the check is O(1) in the common case. `next_trigger` returns the soonest trigger time, useful for computing how long to sleep before the next check.

---

## Recurring Job Support

```python
class RecurringJobManager:
    """
    Manages tasks that repeat at fixed intervals.

    When a recurring task completes, a new copy is created
    with trigger_at = now + interval_seconds and submitted
    back to the delayed scheduler.

    Example: interval_seconds=60
    → Task runs, completes at T
    → New task created with trigger_at = T + 60
    → Runs again at T + 60, new copy at T + 120, etc.
    """

    def __init__(self) -> None:
        self._recurring: dict[str, Task] = {}
        self._lock = threading.RLock()

    def register(self, task: Task) -> None:
        """Mark a task as recurring."""
        with self._lock:
            self._recurring[task.task_id] = task

    def on_completion(
        self, task: Task,
        submit_fn: Callable[[Task], None],
    ) -> Task | None:
        """
        Handle a recurring task's completion.

        If the task is recurring, create a new copy with
        a fresh trigger time and submit it.
        """
        with self._lock:
            if (task.task_id not in self._recurring
                    or not task.is_recurring):
                return None

            # Create next occurrence
            next_task = Task(
                name=f"{task.name}",
                action=task.action,
                priority=task.priority,
                delay_seconds=task.interval_seconds,
                max_retries=task.max_retries,
                estimated_duration_ms=task.estimated_duration_ms,
                interval_seconds=task.interval_seconds,
            )
            self._recurring[next_task.task_id] = next_task
            del self._recurring[task.task_id]
            submit_fn(next_task)
            return next_task

    def cancel(self, task_id: str) -> bool:
        """Stop a recurring task from scheduling future runs."""
        with self._lock:
            if task_id in self._recurring:
                del self._recurring[task_id]
                return True
            return False

    @property
    def active_count(self) -> int:
        return len(self._recurring)

    def __repr__(self) -> str:
        return (f"RecurringJobManager("
                f"active={len(self._recurring)})")
```

`RecurringJobManager` tracks tasks that should repeat. When a recurring task completes, `on_completion` creates a fresh `Task` with `delay_seconds=interval_seconds`, effectively scheduling the next run after the interval. The old task ID is removed and the new one is registered. `cancel` stops future runs. Creating a new `Task` instead of reusing the old one ensures clean state — execution metadata (result, timing, retry count) starts fresh for each occurrence.

---

## Task Event System (Observer Pattern)

```python
@dataclass
class TaskEvent:
    """An event emitted during the task lifecycle."""
    event_id: str
    event_type: TaskEventType
    task_id: str
    task_name: str
    detail: str = ""
    timestamp: float = field(default_factory=time.time)


class TaskEventListener(ABC):
    """
    Observer that reacts to task lifecycle events.

    Implementations receive all events and decide which ones
    to handle. Listeners are decoupled from the scheduler —
    they cannot affect task execution.
    """

    @abstractmethod
    def on_event(self, event: TaskEvent) -> None:
        ...

    @abstractmethod
    def listener_id(self) -> str:
        ...


class TaskLogger(TaskEventListener):
    """Logs all task events to the console."""

    def on_event(self, event: TaskEvent) -> None:
        emoji = {
            TaskEventType.SUBMITTED: "📥",
            TaskEventType.STARTED: "▶️ ",
            TaskEventType.COMPLETED: "✅",
            TaskEventType.FAILED: "❌",
            TaskEventType.RETRYING: "🔄",
            TaskEventType.DEAD_LETTERED: "💀",
            TaskEventType.CANCELLED: "🚫",
            TaskEventType.DELAYED: "⏳",
            TaskEventType.RECURRING: "🔁",
            TaskEventType.DEPENDENCY_MET: "🔗",
        }
        icon = emoji.get(event.event_type, "📌")
        print(f"  {icon} [{event.event_type.value:>15}] "
              f"{event.task_name} {event.detail}")

    def listener_id(self) -> str:
        return "task-logger"


class TaskMetricsCollector(TaskEventListener):
    """Collects execution time metrics per task."""

    def __init__(self) -> None:
        self._execution_times: list[float] = []
        self._failure_count = 0
        self._success_count = 0

    def on_event(self, event: TaskEvent) -> None:
        if event.event_type == TaskEventType.COMPLETED:
            self._success_count += 1
            # Extract execution time from detail if available
            if "ms" in event.detail:
                try:
                    ms_str = event.detail.split("in ")[1].split("ms")[0]
                    self._execution_times.append(float(ms_str))
                except (IndexError, ValueError):
                    pass
        elif event.event_type == TaskEventType.FAILED:
            self._failure_count += 1

    @property
    def avg_execution_time_ms(self) -> float:
        if not self._execution_times:
            return 0.0
        return sum(self._execution_times) / len(self._execution_times)

    @property
    def success_rate(self) -> float:
        total = self._success_count + self._failure_count
        if total == 0:
            return 0.0
        return self._success_count / total

    def listener_id(self) -> str:
        return "task-metrics"

    def display(self) -> None:
        print(f"\n  Avg execution time: "
              f"{self.avg_execution_time_ms:.1f}ms")
        print(f"  Success rate: "
              f"{self.success_rate:.1%} "
              f"({self._success_count} ok, "
              f"{self._failure_count} failed)")


class TaskEventBus:
    """
    Central event dispatcher for task events.

    Manages listener registration and broadcasts events to all
    registered listeners. Thread-safe for concurrent emission.
    """

    def __init__(self) -> None:
        self._listeners: list[TaskEventListener] = []
        self._event_log: list[TaskEvent] = []
        self._lock = threading.RLock()

    def register(self, listener: TaskEventListener) -> None:
        with self._lock:
            self._listeners.append(listener)

    def unregister(self, listener_id: str) -> None:
        with self._lock:
            self._listeners = [
                l for l in self._listeners
                if l.listener_id() != listener_id
            ]

    def emit(self, event: TaskEvent) -> None:
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

`TaskEventListener` is the observer interface. `TaskLogger` prints every event with emoji-coded log lines — submissions, starts, completions, and failures are visually distinct. `TaskMetricsCollector` tracks execution times and success rates for operational monitoring. The `TaskEventBus` dispatches events to all registered listeners, isolating each listener from failures in others. Adding a new listener (e.g., Slack alerting on failures, Prometheus histogram for execution times) requires only implementing `on_event` and `listener_id`.

---

## Worker Pool

```python
class Worker:
    """
    A single worker thread that processes tasks.

    Each worker pulls tasks from the queue, executes them,
    and reports results. Workers run continuously until
    stopped.
    """

    def __init__(self, worker_id: int,
                 executor: TaskExecutor) -> None:
        self.worker_id = worker_id
        self._executor = executor
        self._tasks_completed = 0
        self._is_busy = False
        self._current_task: Task | None = None

    def execute(self, task: Task) -> TaskResult:
        """Execute a task on this worker."""
        self._is_busy = True
        self._current_task = task
        result = self._executor.execute_no_retry(task)
        self._tasks_completed += 1
        self._is_busy = False
        self._current_task = None
        return result

    @property
    def is_busy(self) -> bool:
        return self._is_busy

    @property
    def tasks_completed(self) -> int:
        return self._tasks_completed

    def __repr__(self) -> str:
        status = "busy" if self._is_busy else "idle"
        return (f"Worker(id={self.worker_id}, "
                f"status={status}, "
                f"completed={self._tasks_completed})")


class WorkerPool:
    """
    Pool of workers for concurrent task execution.

    Manages a fixed number of workers. Tasks are assigned
    to the first available worker. If all workers are busy,
    the task waits in the queue.

    For simulation purposes, workers execute synchronously
    one task at a time. In production, each worker would
    be a thread or process.
    """

    def __init__(self, max_workers: int = DEFAULT_MAX_WORKERS,
                 executor: TaskExecutor | None = None) -> None:
        self._executor = executor or TaskExecutor()
        self._workers = [
            Worker(i, self._executor)
            for i in range(max_workers)
        ]
        self._lock = threading.RLock()

    def assign(self, task: Task) -> tuple[Worker, TaskResult] | None:
        """
        Assign a task to an available worker.

        Returns the (worker, result) tuple, or None if no
        worker is available.
        """
        with self._lock:
            for worker in self._workers:
                if not worker.is_busy:
                    result = worker.execute(task)
                    return worker, result
            return None

    def get_available_worker(self) -> Worker | None:
        """Return the first idle worker, or None."""
        with self._lock:
            for worker in self._workers:
                if not worker.is_busy:
                    return worker
            return None

    @property
    def size(self) -> int:
        return len(self._workers)

    @property
    def available_count(self) -> int:
        return sum(1 for w in self._workers if not w.is_busy)

    @property
    def busy_count(self) -> int:
        return sum(1 for w in self._workers if w.is_busy)

    def display(self) -> None:
        print(f"\n  {'─'*STATS_DISPLAY_WIDTH}")
        print(f"  WORKER POOL")
        print(f"  {'─'*STATS_DISPLAY_WIDTH}")
        for w in self._workers:
            print(f"  {w}")
        total = sum(w.tasks_completed for w in self._workers)
        print(f"  Total completed: {total}")
        print(f"  {'─'*STATS_DISPLAY_WIDTH}")

    def __repr__(self) -> str:
        return (f"WorkerPool(size={len(self._workers)}, "
                f"available={self.available_count})")
```

`Worker` represents a single execution unit. It tracks whether it's busy, how many tasks it has completed, and which task it's currently running. `WorkerPool` manages a fixed set of workers. `assign` finds the first idle worker and hands it the task — in simulation this is synchronous; in production each worker would be a thread pulling from a shared queue. `available_count` and `busy_count` provide pool utilization metrics. The pool is fixed-size by design — unbounded worker creation would lead to thread thrashing.

---

## Task Retry & Dead Letter Queue

```python
class DeadLetterQueue:
    """
    Stores tasks that have exhausted all retries.

    Tasks in the DLQ can be inspected, replayed (resubmitted
    with fresh retry counts), or purged. The DLQ is the last
    stop for a failed task — it prevents silent data loss.
    """

    def __init__(self) -> None:
        self._tasks: list[Task] = []
        self._lock = threading.RLock()

    def add(self, task: Task) -> None:
        """Move a dead task to the DLQ."""
        with self._lock:
            task.status = TaskStatus.DEAD
            self._tasks.append(task)

    def replay(self, task_id: str,
               submit_fn: Callable[[Task], None]) -> bool:
        """
        Resubmit a DLQ task with fresh retry state.

        Resets the retry count and status, then submits
        the task back to the scheduler.
        """
        with self._lock:
            for i, task in enumerate(self._tasks):
                if task.task_id == task_id:
                    task.retry_count = 0
                    task.status = TaskStatus.PENDING
                    task.error_history.clear()
                    self._tasks.pop(i)
                    submit_fn(task)
                    return True
            return False

    def peek(self) -> list[Task]:
        """Return all tasks in the DLQ for inspection."""
        with self._lock:
            return list(self._tasks)

    @property
    def size(self) -> int:
        return len(self._tasks)

    def clear(self) -> None:
        with self._lock:
            self._tasks.clear()

    def display(self) -> None:
        print(f"\n  {'─'*STATS_DISPLAY_WIDTH}")
        print(f"  DEAD LETTER QUEUE ({len(self._tasks)} tasks)")
        print(f"  {'─'*STATS_DISPLAY_WIDTH}")
        for task in self._tasks:
            errors = "; ".join(task.error_history[-3:])
            print(f"  💀 {task.name} "
                  f"(retries={task.retry_count}, "
                  f"errors=[{errors}])")
        print(f"  {'─'*STATS_DISPLAY_WIDTH}")

    def __repr__(self) -> str:
        return f"DeadLetterQueue(size={len(self._tasks)})"


class RetryManager:
    """
    Handles retry logic for failed tasks.

    On failure, checks if the task can retry. If yes, applies
    exponential backoff and resubmits. If no, moves the task
    to the Dead Letter Queue.

    Backoff: base × 2^(retry - 1)
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
        self, task: Task,
        resubmit_fn: Callable[[Task], None],
    ) -> bool:
        """
        Handle a failed task.

        Returns True if the task will be retried,
        False if it was sent to the DLQ.
        """
        if task.can_retry:
            task.retry_count += 1
            task.status = TaskStatus.RETRYING
            backoff = (self._backoff_base
                       * (2 ** (task.retry_count - 1)))
            # Schedule retry as a delayed task
            task.trigger_at = time.time() + backoff
            task.delay_seconds = backoff
            task.status = TaskStatus.SCHEDULED
            resubmit_fn(task)
            return True
        else:
            self._dlq.add(task)
            return False

    @property
    def dlq(self) -> DeadLetterQueue:
        return self._dlq
```

`DeadLetterQueue` is the final destination for tasks that exhaust all retries. Without a DLQ, failed tasks would silently disappear — the DLQ preserves them for debugging, alerting, and manual replay. `replay` resets a DLQ task's state and resubmits it to the scheduler. `RetryManager` orchestrates the retry-or-DLQ decision. On failure, it checks `can_retry`, increments the counter, computes backoff (`base × 2^(retry-1)`), sets a new `trigger_at`, and resubmits the task as a delayed job. The backoff prevents retry storms — if a downstream service is down, hammering it with retries every millisecond makes things worse.

---

## Task Dependencies (DAG Execution)

```python
class DependencyResolver:
    """
    Resolves task dependencies using a DAG.

    Tasks can depend on other tasks by ID. A task is only
    eligible for execution when all its dependencies have
    completed successfully. Circular dependencies are detected.

    Internal structure:
        _dependencies: { task_id → set of dependency task_ids }
        _completed: set of completed task_ids
        _waiting: { task_id → Task }
    """

    def __init__(self) -> None:
        self._dependencies: dict[str, set[str]] = {}
        self._completed: set[str] = set()
        self._waiting: dict[str, Task] = {}
        self._lock = threading.RLock()

    def register(self, task: Task) -> None:
        """
        Register a task and its dependencies.

        If the task has no unmet dependencies, it is
        immediately eligible. Otherwise, it waits.
        """
        with self._lock:
            if not task.depends_on:
                return

            unmet = set(task.depends_on) - self._completed
            if not unmet:
                return  # all deps already met

            self._dependencies[task.task_id] = unmet
            self._waiting[task.task_id] = task

    def mark_completed(self, task_id: str) -> list[Task]:
        """
        Mark a task as completed and return newly unblocked tasks.

        When a task completes, check all waiting tasks to see
        if any now have all dependencies met.
        """
        with self._lock:
            self._completed.add(task_id)
            unblocked: list[Task] = []

            for tid in list(self._dependencies.keys()):
                self._dependencies[tid].discard(task_id)
                if not self._dependencies[tid]:
                    # All dependencies met
                    task = self._waiting.pop(tid, None)
                    del self._dependencies[tid]
                    if task:
                        task.status = TaskStatus.PENDING
                        unblocked.append(task)

            return unblocked

    def has_unmet_dependencies(self, task: Task) -> bool:
        """Check if a task has any unmet dependencies."""
        with self._lock:
            if not task.depends_on:
                return False
            unmet = set(task.depends_on) - self._completed
            return len(unmet) > 0

    def has_circular_dependency(self, task: Task) -> bool:
        """
        Detect circular dependencies using DFS.

        Returns True if adding this task would create a cycle.
        """
        with self._lock:
            visited: set[str] = set()
            stack: set[str] = set()

            def dfs(tid: str) -> bool:
                if tid in stack:
                    return True  # cycle found
                if tid in visited:
                    return False
                visited.add(tid)
                stack.add(tid)
                for dep in self._dependencies.get(tid, set()):
                    if dfs(dep):
                        return True
                stack.discard(tid)
                return False

            # Temporarily add the task's dependencies
            self._dependencies[task.task_id] = set(task.depends_on)
            result = dfs(task.task_id)
            if result:
                del self._dependencies[task.task_id]
            return result

    @property
    def waiting_count(self) -> int:
        return len(self._waiting)

    def __repr__(self) -> str:
        return (f"DependencyResolver("
                f"waiting={len(self._waiting)}, "
                f"completed={len(self._completed)})")
```

`DependencyResolver` enforces task dependencies as a directed acyclic graph (DAG). `register` records a task's dependencies and parks it in the waiting set. `mark_completed` is called when a task finishes — it removes the completed task ID from every waiting task's dependency set. If a waiting task's set becomes empty, all dependencies are met and the task is unblocked. `has_circular_dependency` uses DFS with a recursion stack to detect cycles — if adding a task would create a cycle, it's rejected. This prevents deadlocks where A depends on B and B depends on A.

---

## Scheduler Facade

```python
class SchedulerStats:
    """Aggregate scheduler performance statistics."""

    def __init__(self) -> None:
        self.submitted: int = 0
        self.completed: int = 0
        self.failed: int = 0
        self.retried: int = 0
        self.dead_lettered: int = 0
        self.cancelled: int = 0
        self.total_execution_ms: float = 0.0

    @property
    def total_processed(self) -> int:
        return self.completed + self.failed

    @property
    def success_rate(self) -> float:
        if self.total_processed == 0:
            return 0.0
        return self.completed / self.total_processed

    @property
    def avg_execution_ms(self) -> float:
        if self.completed == 0:
            return 0.0
        return self.total_execution_ms / self.completed

    def display(self) -> None:
        print(f"\n  {'═'*STATS_DISPLAY_WIDTH}")
        print(f"  SCHEDULER STATISTICS")
        print(f"  {'═'*STATS_DISPLAY_WIDTH}")
        print(f"  Submitted:         {self.submitted}")
        print(f"  Completed:         {self.completed}")
        print(f"  Failed:            {self.failed}")
        print(f"  Retried:           {self.retried}")
        print(f"  Dead-lettered:     {self.dead_lettered}")
        print(f"  Cancelled:         {self.cancelled}")
        print(f"  Success rate:      {self.success_rate:.1%}")
        print(f"  Avg execution:     {self.avg_execution_ms:.1f}ms")
        print(f"  {'═'*STATS_DISPLAY_WIDTH}")

    def reset(self) -> None:
        self.submitted = 0
        self.completed = 0
        self.failed = 0
        self.retried = 0
        self.dead_lettered = 0
        self.cancelled = 0
        self.total_execution_ms = 0.0


class TaskSchedulerSystem:
    """
    Facade that orchestrates the entire task scheduler.

    Provides a simple API that hides the complexity of
    queues, workers, policies, retries, events, dependencies,
    delayed jobs, and recurring tasks.
    """

    def __init__(
        self,
        policy: SchedulingPolicy | None = None,
        max_workers: int = DEFAULT_MAX_WORKERS,
        max_retries: int = DEFAULT_MAX_RETRIES,
        retry_backoff: float = DEFAULT_RETRY_BACKOFF,
    ) -> None:
        self._event_bus = TaskEventBus()
        self._logger = TaskLogger()
        self._metrics = TaskMetricsCollector()
        self._event_bus.register(self._logger)
        self._event_bus.register(self._metrics)

        self._policy = policy or PrioritySchedulingPolicy()
        self._queue = TaskQueue()
        self._executor = TaskExecutor(retry_backoff=retry_backoff)
        self._worker_pool = WorkerPool(
            max_workers=max_workers, executor=self._executor,
        )
        self._dlq = DeadLetterQueue()
        self._retry_manager = RetryManager(
            self._dlq, backoff_base=retry_backoff,
        )
        self._delayed = DelayedJobScheduler()
        self._recurring = RecurringJobManager()
        self._dependency = DependencyResolver()
        self._stats = SchedulerStats()
        self._all_tasks: dict[str, Task] = {}
        self._lock = threading.RLock()

    # ------------------------------------------------------------------
    # Core operations
    # ------------------------------------------------------------------

    def submit(self, task: Task) -> str:
        """
        Submit a task for execution.

        Handles delayed tasks, dependencies, and immediate
        scheduling based on task configuration.
        """
        with self._lock:
            self._all_tasks[task.task_id] = task
            self._stats.submitted += 1

            # Check for circular dependencies
            if (task.depends_on
                    and self._dependency.has_circular_dependency(task)):
                task.status = TaskStatus.CANCELLED
                self._stats.cancelled += 1
                self._emit(TaskEventType.CANCELLED, task,
                           "circular dependency detected")
                return task.task_id

            # Register recurring task
            if task.is_recurring:
                self._recurring.register(task)
                self._emit(TaskEventType.RECURRING, task,
                           f"interval={task.interval_seconds}s")

            # Handle dependencies
            if task.depends_on:
                if self._dependency.has_unmet_dependencies(task):
                    self._dependency.register(task)
                    self._emit(TaskEventType.SUBMITTED, task,
                               f"waiting on {task.depends_on}")
                    return task.task_id

            # Handle delayed tasks
            if task.delay_seconds > 0 and not task.is_ready:
                self._delayed.schedule(task)
                self._emit(TaskEventType.DELAYED, task,
                           f"triggers in "
                           f"{task.delay_seconds:.1f}s")
                return task.task_id

            # Enqueue for immediate execution
            self._policy.enqueue(task, self._queue)
            self._emit(TaskEventType.SUBMITTED, task,
                       f"priority={task.priority.name}")
            return task.task_id

    def run_next(self) -> TaskResult | None:
        """
        Execute the next task in the queue.

        Promotes delayed tasks, resolves dependencies,
        then dequeues and executes the highest-priority task.
        """
        with self._lock:
            # Promote delayed tasks
            ready = self._delayed.get_ready_tasks()
            for task in ready:
                self._policy.enqueue(task, self._queue)
                self._emit(TaskEventType.SUBMITTED, task,
                           "delay elapsed — now queued")

            # Dequeue next task
            task = self._policy.dequeue(self._queue)
            if task is None:
                return None

            # Execute
            self._emit(TaskEventType.STARTED, task,
                       f"worker assigned")

            assignment = self._worker_pool.assign(task)
            if assignment is None:
                # No worker available — re-enqueue
                self._policy.enqueue(task, self._queue)
                return None

            worker, result = assignment
            return self._handle_result(task, result)

    def run_all(self) -> list[TaskResult]:
        """Execute all tasks currently in the queue."""
        results: list[TaskResult] = []
        while not self._queue.is_empty:
            result = self.run_next()
            if result:
                results.append(result)
        return results

    def cancel(self, task_id: str) -> bool:
        """Cancel a pending task."""
        with self._lock:
            task = self._all_tasks.get(task_id)
            if (task and task.status in
                    (TaskStatus.PENDING, TaskStatus.SCHEDULED)):
                task.status = TaskStatus.CANCELLED
                self._stats.cancelled += 1
                self._emit(TaskEventType.CANCELLED, task, "")
                return True
            return False

    # ------------------------------------------------------------------
    # Inspection
    # ------------------------------------------------------------------

    @property
    def queue_size(self) -> int:
        return self._queue.size

    @property
    def stats(self) -> SchedulerStats:
        return self._stats

    @property
    def dlq(self) -> DeadLetterQueue:
        return self._dlq

    @property
    def worker_pool(self) -> WorkerPool:
        return self._worker_pool

    @property
    def event_bus(self) -> TaskEventBus:
        return self._event_bus

    @property
    def metrics(self) -> TaskMetricsCollector:
        return self._metrics

    def get_task(self, task_id: str) -> Task | None:
        return self._all_tasks.get(task_id)

    def display_status(self) -> None:
        self._stats.display()
        self._worker_pool.display()
        if self._dlq.size > 0:
            self._dlq.display()
        self._metrics.display()

    # ------------------------------------------------------------------
    # Internal helpers
    # ------------------------------------------------------------------

    def _handle_result(self, task: Task,
                       result: TaskResult) -> TaskResult:
        """Process a task execution result."""
        if result.success:
            self._stats.completed += 1
            self._stats.total_execution_ms += (
                result.execution_time_ms
            )
            self._emit(TaskEventType.COMPLETED, task,
                       f"in {result.execution_time_ms:.1f}ms")

            # Handle recurring tasks
            if task.is_recurring:
                next_task = self._recurring.on_completion(
                    task, self.submit,
                )
                if next_task:
                    self._emit(TaskEventType.RECURRING, next_task,
                               f"next in "
                               f"{task.interval_seconds}s")

            # Resolve dependencies
            unblocked = self._dependency.mark_completed(
                task.task_id
            )
            for ut in unblocked:
                self._emit(TaskEventType.DEPENDENCY_MET, ut,
                           f"deps resolved")
                self.submit(ut)

        else:
            self._stats.failed += 1
            self._emit(TaskEventType.FAILED, task,
                       f"error={result.error}")

            will_retry = self._retry_manager.handle_failure(
                task,
                lambda t: self._delayed.schedule(t),
            )
            if will_retry:
                self._stats.retried += 1
                self._emit(TaskEventType.RETRYING, task,
                           f"attempt "
                           f"{task.retry_count}/"
                           f"{task.max_retries}")
            else:
                self._stats.dead_lettered += 1
                self._emit(TaskEventType.DEAD_LETTERED, task,
                           f"moved to DLQ after "
                           f"{task.retry_count} retries")

        return result

    def _emit(self, event_type: TaskEventType,
              task: Task, detail: str = "") -> None:
        event = TaskEvent(
            event_id=str(uuid.uuid4())[:8],
            event_type=event_type,
            task_id=task.task_id,
            task_name=task.name,
            detail=detail,
        )
        self._event_bus.emit(event)
```

`TaskSchedulerSystem` is the facade — it wires together the scheduling policy, task queue, worker pool, executor, retry manager, dead letter queue, delayed scheduler, recurring job manager, dependency resolver, event bus, logger, and metrics collector. `submit` handles all task types: delayed tasks go to the delayed scheduler, tasks with unmet dependencies are parked in the resolver, recurring tasks are registered, and immediate tasks are enqueued. `run_next` promotes delayed tasks, dequeues the highest-priority task, assigns it to a worker, and processes the result. `_handle_result` handles success (stats, recurring, dependencies) and failure (retry or DLQ). Users interact with `submit`, `run_next`, `run_all`, and `cancel` without knowing about any internal complexity.

---

## Simulation Engine

```python
class SchedulerScenario:
    """
    Tick-based simulation engine for scheduler demos.

    Each tick represents a logical time step. Actions are registered
    at specific ticks and execute in order when the simulation runs.
    """

    def __init__(self, system: TaskSchedulerSystem) -> None:
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

`SchedulerScenario` mirrors the simulation engine from other guides. Actions are registered at specific ticks and execute in order, making demos reproducible and readable.

---

## Putting It All Together

```python
def main_scenario() -> None:
    """Run the main task scheduler demonstration."""

    print("\n╔══════════════════════════════════════════╗")
    print("║      TASK SCHEDULER SYSTEM               ║")
    print("║  Policy: Priority                         ║")
    print("║  Workers: 4                               ║")
    print("║  Max Retries: 3                           ║")
    print("╚══════════════════════════════════════════╝")

    system = TaskSchedulerSystem(
        policy=PrioritySchedulingPolicy(),
        max_workers=4,
        max_retries=3,
    )

    scenario = SchedulerScenario(system)

    # --- Submit tasks with different priorities ---
    scenario.at_tick(1, "Submit tasks with varying priorities",
                     lambda: (
                         system.submit(Task(
                             name="low-task",
                             action=lambda: "low done",
                             priority=TaskPriority.LOW,
                         )),
                         system.submit(Task(
                             name="critical-task",
                             action=lambda: "critical done",
                             priority=TaskPriority.CRITICAL,
                         )),
                         system.submit(Task(
                             name="medium-task",
                             action=lambda: "medium done",
                             priority=TaskPriority.MEDIUM,
                         )),
                         system.submit(Task(
                             name="high-task",
                             action=lambda: "high done",
                             priority=TaskPriority.HIGH,
                         )),
                     ))

    # --- Execute all (should run in priority order) ---
    scenario.at_tick(2, "Execute all tasks (priority order)",
                     lambda: system.run_all())

    # --- Submit a failing task ---
    call_count = {"n": 0}

    def flaky_action():
        call_count["n"] += 1
        if call_count["n"] < 3:
            raise RuntimeError("transient failure")
        return "finally succeeded"

    scenario.at_tick(3, "Submit a flaky task (fails twice)",
                     lambda: system.submit(Task(
                         name="flaky-task",
                         action=flaky_action,
                         priority=TaskPriority.HIGH,
                         max_retries=3,
                     )))

    scenario.at_tick(4, "Execute flaky task (1st attempt fails)",
                     lambda: system.run_next())

    # --- Submit a task that always fails ---
    scenario.at_tick(5, "Submit a doomed task (always fails)",
                     lambda: system.submit(Task(
                         name="doomed-task",
                         action=lambda: (_ for _ in ()).throw(
                             ValueError("permanent error")),
                         priority=TaskPriority.MEDIUM,
                         max_retries=0,
                     )))

    scenario.at_tick(6, "Execute doomed task → DLQ",
                     lambda: system.run_next())

    # --- Display statistics ---
    scenario.at_tick(7, "Display scheduler statistics",
                     lambda: system.display_status())

    scenario.run()


# ------------------------------------------------------------------
# Policy Comparison
# ------------------------------------------------------------------

def compare_scheduling_policies() -> None:
    """Compare Priority, FIFO, and SJF on the same workload."""

    print("\n" + "=" * 55)
    print("  SCHEDULING POLICY COMPARISON")
    print("=" * 55)

    tasks_data = [
        ("task-A", TaskPriority.LOW, 100.0),
        ("task-B", TaskPriority.CRITICAL, 50.0),
        ("task-C", TaskPriority.HIGH, 10.0),
        ("task-D", TaskPriority.MEDIUM, 200.0),
        ("task-E", TaskPriority.BACKGROUND, 5.0),
    ]

    policies: list[tuple[str, SchedulingPolicy]] = [
        ("Priority", PrioritySchedulingPolicy()),
        ("FIFO", FIFOSchedulingPolicy()),
        ("ShortestJobFirst", ShortestJobFirstPolicy()),
    ]

    for policy_name, policy in policies:
        queue = TaskQueue()
        execution_order: list[str] = []

        for name, priority, est in tasks_data:
            task = Task(
                name=name,
                action=lambda: "done",
                priority=priority,
                estimated_duration_ms=est,
            )
            time.sleep(0.001)  # ensure distinct created_at
            policy.enqueue(task, queue)

        while True:
            task = policy.dequeue(queue)
            if task is None:
                break
            execution_order.append(task.name)

        print(f"\n  Policy: {policy_name}")
        print(f"  {'─'*45}")
        print(f"    Execution order: "
              f"{' → '.join(execution_order)}")

    print("\n" + "=" * 55)


def demo_delayed_jobs() -> None:
    """Demonstrate delayed job scheduling."""

    print("\n" + "=" * 55)
    print("  DELAYED JOBS DEMO")
    print("=" * 55)

    system = TaskSchedulerSystem(max_workers=2)

    # Submit a delayed task
    system.submit(Task(
        name="delayed-cleanup",
        action=lambda: "cleanup done",
        priority=TaskPriority.LOW,
        delay_seconds=0.5,
    ))

    # Submit an immediate task
    system.submit(Task(
        name="immediate-work",
        action=lambda: "work done",
        priority=TaskPriority.HIGH,
    ))

    print("\n  Running immediate tasks...")
    system.run_all()

    print(f"\n  Waiting for delayed task "
          f"(0.5s delay)...")
    time.sleep(0.6)

    print("\n  Running delayed tasks...")
    system.run_next()

    system.stats.display()
    print("\n" + "=" * 55)


def demo_dependencies() -> None:
    """Demonstrate task dependency resolution."""

    print("\n" + "=" * 55)
    print("  TASK DEPENDENCIES DEMO")
    print("=" * 55)

    system = TaskSchedulerSystem(max_workers=2)

    # Task B depends on Task A
    task_a = Task(
        name="fetch-data",
        action=lambda: "data fetched",
        priority=TaskPriority.HIGH,
    )
    task_b = Task(
        name="process-data",
        action=lambda: "data processed",
        priority=TaskPriority.MEDIUM,
        depends_on=[task_a.task_id],
    )
    task_c = Task(
        name="send-report",
        action=lambda: "report sent",
        priority=TaskPriority.LOW,
        depends_on=[task_b.task_id],
    )

    # Submit in reverse order
    print("\n  Submitting tasks (reverse dependency order):")
    system.submit(task_c)  # depends on B
    system.submit(task_b)  # depends on A
    system.submit(task_a)  # no dependencies

    print("\n  Executing dependency chain:")
    # Run A → unblocks B → unblocks C
    system.run_next()  # A
    system.run_next()  # B (unblocked by A)
    system.run_next()  # C (unblocked by B)

    system.stats.display()
    print("\n" + "=" * 55)


def demo_dead_letter_queue() -> None:
    """Demonstrate retry exhaustion and DLQ."""

    print("\n" + "=" * 55)
    print("  DEAD LETTER QUEUE DEMO")
    print("=" * 55)

    system = TaskSchedulerSystem(
        max_workers=2, max_retries=2,
    )

    # Task that always fails
    system.submit(Task(
        name="always-fails",
        action=lambda: (_ for _ in ()).throw(
            RuntimeError("permanent error")),
        priority=TaskPriority.HIGH,
        max_retries=2,
    ))

    # Execute (fails, retries, DLQ)
    system.run_next()

    print(f"\n  DLQ size: {system.dlq.size}")
    system.dlq.display()

    # Replay from DLQ
    dlq_tasks = system.dlq.peek()
    if dlq_tasks:
        print("\n  Replaying task from DLQ "
              "(will fail again)...")
        system.dlq.replay(
            dlq_tasks[0].task_id, system.submit,
        )
        system.run_next()

    system.stats.display()
    print("\n" + "=" * 55)


# ------------------------------------------------------------------
# Unit Tests
# ------------------------------------------------------------------

def test_task_creation() -> None:
    """Test task creation with defaults."""
    task = Task(
        name="test-task",
        action=lambda: 42,
    )
    assert task.name == "test-task"
    assert task.priority == TaskPriority.MEDIUM
    assert task.status == TaskStatus.PENDING
    assert task.retry_count == 0
    assert not task.is_terminal
    print("✅ test_task_creation passed")


def test_task_execution() -> None:
    """Test successful task execution."""
    task = Task(
        name="compute",
        action=lambda: 2 + 2,
    )
    result = task.execute()

    assert result.success
    assert result.value == 4
    assert result.execution_time_ms >= 0
    assert task.status == TaskStatus.COMPLETED
    print("✅ test_task_execution passed")


def test_task_failure() -> None:
    """Test task execution with exception."""
    task = Task(
        name="fail-task",
        action=lambda: (_ for _ in ()).throw(
            ValueError("boom")),
    )
    result = task.execute()

    assert not result.success
    assert "boom" in result.error
    assert task.status == TaskStatus.FAILED
    assert len(task.error_history) == 1
    print("✅ test_task_failure passed")


def test_priority_ordering() -> None:
    """Test that tasks are ordered by priority."""
    queue = TaskQueue()

    low = Task("low", lambda: None, TaskPriority.LOW)
    critical = Task("critical", lambda: None,
                     TaskPriority.CRITICAL)
    high = Task("high", lambda: None, TaskPriority.HIGH)

    queue.push(low)
    queue.push(critical)
    queue.push(high)

    t1 = queue.pop()
    t2 = queue.pop()
    t3 = queue.pop()

    assert t1.name == "critical"
    assert t2.name == "high"
    assert t3.name == "low"
    print("✅ test_priority_ordering passed")


def test_fifo_within_priority() -> None:
    """Test FIFO ordering among tasks with same priority."""
    queue = TaskQueue()

    t1 = Task("first", lambda: None, TaskPriority.HIGH)
    time.sleep(0.01)
    t2 = Task("second", lambda: None, TaskPriority.HIGH)
    time.sleep(0.01)
    t3 = Task("third", lambda: None, TaskPriority.HIGH)

    queue.push(t1)
    queue.push(t2)
    queue.push(t3)

    assert queue.pop().name == "first"
    assert queue.pop().name == "second"
    assert queue.pop().name == "third"
    print("✅ test_fifo_within_priority passed")


def test_queue_capacity() -> None:
    """Test queue rejects when full."""
    queue = TaskQueue(capacity=2)

    assert queue.push(Task("a", lambda: None))
    assert queue.push(Task("b", lambda: None))
    assert not queue.push(Task("c", lambda: None))
    assert queue.size == 2
    print("✅ test_queue_capacity passed")


def test_delayed_task() -> None:
    """Test that delayed tasks wait for trigger time."""
    delayed = DelayedJobScheduler()

    task = Task(
        name="delayed",
        action=lambda: "done",
        delay_seconds=0.1,
    )
    delayed.schedule(task)

    # Not ready yet
    ready = delayed.get_ready_tasks()
    assert len(ready) == 0

    time.sleep(0.15)

    ready = delayed.get_ready_tasks()
    assert len(ready) == 1
    assert ready[0].name == "delayed"
    print("✅ test_delayed_task passed")


def test_worker_pool() -> None:
    """Test worker pool assigns and executes tasks."""
    pool = WorkerPool(max_workers=2)

    task = Task("work", action=lambda: "result")
    assignment = pool.assign(task)

    assert assignment is not None
    worker, result = assignment
    assert result.success
    assert result.value == "result"
    assert worker.tasks_completed == 1
    print("✅ test_worker_pool passed")


def test_dead_letter_queue() -> None:
    """Test DLQ stores and replays tasks."""
    dlq = DeadLetterQueue()

    task = Task(
        name="dead-task",
        action=lambda: None,
        max_retries=0,
    )
    task.status = TaskStatus.DEAD
    task.error_history.append("error1")
    dlq.add(task)

    assert dlq.size == 1

    replayed = []
    dlq.replay(task.task_id,
               lambda t: replayed.append(t))

    assert dlq.size == 0
    assert len(replayed) == 1
    assert replayed[0].retry_count == 0
    print("✅ test_dead_letter_queue passed")


def test_retry_manager() -> None:
    """Test retry manager retries or DLQs tasks."""
    dlq = DeadLetterQueue()
    manager = RetryManager(dlq, backoff_base=0.01)

    # Task with retries left
    task = Task(
        name="retry-me",
        action=lambda: None,
        max_retries=2,
    )
    resubmitted = []
    will_retry = manager.handle_failure(
        task, lambda t: resubmitted.append(t),
    )
    assert will_retry
    assert task.retry_count == 1
    assert len(resubmitted) == 1

    # Task with no retries left
    task2 = Task(
        name="no-retry",
        action=lambda: None,
        max_retries=0,
    )
    will_retry = manager.handle_failure(
        task2, lambda t: resubmitted.append(t),
    )
    assert not will_retry
    assert dlq.size == 1
    print("✅ test_retry_manager passed")


def test_dependency_resolution() -> None:
    """Test tasks blocked until dependencies complete."""
    resolver = DependencyResolver()

    task_a = Task("A", lambda: None)
    task_b = Task("B", lambda: None,
                  depends_on=[task_a.task_id])

    resolver.register(task_b)
    assert resolver.waiting_count == 1

    unblocked = resolver.mark_completed(task_a.task_id)
    assert len(unblocked) == 1
    assert unblocked[0].name == "B"
    assert resolver.waiting_count == 0
    print("✅ test_dependency_resolution passed")


def test_dependency_chain() -> None:
    """Test multi-level dependency chain."""
    resolver = DependencyResolver()

    task_a = Task("A", lambda: None)
    task_b = Task("B", lambda: None,
                  depends_on=[task_a.task_id])
    task_c = Task("C", lambda: None,
                  depends_on=[task_b.task_id])

    resolver.register(task_b)
    resolver.register(task_c)
    assert resolver.waiting_count == 2

    # Complete A → unblocks B
    unblocked = resolver.mark_completed(task_a.task_id)
    assert len(unblocked) == 1
    assert unblocked[0].name == "B"

    # Complete B → unblocks C
    unblocked = resolver.mark_completed(task_b.task_id)
    assert len(unblocked) == 1
    assert unblocked[0].name == "C"
    print("✅ test_dependency_chain passed")


def test_fifo_policy() -> None:
    """Test FIFO policy ignores priority."""
    policy = FIFOSchedulingPolicy()
    queue = TaskQueue()

    t1 = Task("first", lambda: None, TaskPriority.LOW)
    time.sleep(0.01)
    t2 = Task("second", lambda: None, TaskPriority.CRITICAL)

    policy.enqueue(t1, queue)
    policy.enqueue(t2, queue)

    r1 = policy.dequeue(queue)
    r2 = policy.dequeue(queue)

    assert r1.name == "first"
    assert r2.name == "second"
    print("✅ test_fifo_policy passed")


def test_sjf_policy() -> None:
    """Test SJF executes shortest task first."""
    policy = ShortestJobFirstPolicy()
    queue = TaskQueue()

    long = Task("long", lambda: None,
                estimated_duration_ms=100.0)
    short = Task("short", lambda: None,
                 estimated_duration_ms=10.0)
    medium = Task("medium", lambda: None,
                  estimated_duration_ms=50.0)

    policy.enqueue(long, queue)
    policy.enqueue(short, queue)
    policy.enqueue(medium, queue)

    assert policy.dequeue(queue).name == "short"
    assert policy.dequeue(queue).name == "medium"
    assert policy.dequeue(queue).name == "long"
    print("✅ test_sjf_policy passed")


def test_scheduler_stats() -> None:
    """Test scheduler statistics tracking."""
    system = TaskSchedulerSystem(max_workers=2)

    system.submit(Task("ok-1", lambda: "a"))
    system.submit(Task("ok-2", lambda: "b"))
    system.run_all()

    stats = system.stats
    assert stats.submitted == 2
    assert stats.completed == 2
    assert stats.success_rate == 1.0
    print("✅ test_scheduler_stats passed")


def test_task_cancel() -> None:
    """Test cancelling a pending task."""
    system = TaskSchedulerSystem(max_workers=1)

    task = Task("cancel-me", lambda: "nope")
    tid = system.submit(task)

    assert system.cancel(tid)
    assert task.status == TaskStatus.CANCELLED
    print("✅ test_task_cancel passed")


def test_recurring_task() -> None:
    """Test recurring task registration."""
    manager = RecurringJobManager()
    task = Task(
        name="recurring",
        action=lambda: "done",
        interval_seconds=5.0,
    )
    manager.register(task)
    assert manager.active_count == 1

    cancelled = manager.cancel(task.task_id)
    assert cancelled
    assert manager.active_count == 0
    print("✅ test_recurring_task passed")


def test_task_result() -> None:
    """Test task result properties."""
    result = TaskResult(
        task_id="abc",
        success=True,
        value=42,
        execution_time_ms=5.5,
    )
    assert result.success
    assert result.value == 42
    assert result.execution_time_ms == 5.5
    print("✅ test_task_result passed")


def test_full_lifecycle() -> None:
    """Test full task lifecycle: submit → execute → complete."""
    system = TaskSchedulerSystem(
        policy=PrioritySchedulingPolicy(),
        max_workers=2,
    )

    system.submit(Task(
        name="lifecycle-task",
        action=lambda: "lifecycle complete",
        priority=TaskPriority.HIGH,
    ))

    results = system.run_all()
    assert len(results) == 1
    assert results[0].success
    assert results[0].value == "lifecycle complete"
    assert system.stats.completed == 1
    print("✅ test_full_lifecycle passed")


def run_all_tests() -> None:
    print("\n" + "=" * 40)
    print("  RUNNING TESTS")
    print("=" * 40 + "\n")
    test_task_creation()
    test_task_execution()
    test_task_failure()
    test_priority_ordering()
    test_fifo_within_priority()
    test_queue_capacity()
    test_delayed_task()
    test_worker_pool()
    test_dead_letter_queue()
    test_retry_manager()
    test_dependency_resolution()
    test_dependency_chain()
    test_fifo_policy()
    test_sjf_policy()
    test_scheduler_stats()
    test_task_cancel()
    test_recurring_task()
    test_task_result()
    test_full_lifecycle()
    print("\n✅ All tests passed!\n")


if __name__ == "__main__":
    main_scenario()
    compare_scheduling_policies()
    demo_delayed_jobs()
    demo_dependencies()
    demo_dead_letter_queue()
    run_all_tests()
```

Each test isolates a single behavior. `test_task_creation` verifies default values are applied correctly. `test_task_execution` confirms a successful task returns its value and transitions to `COMPLETED`. `test_task_failure` checks that exceptions are captured in `error_history` and the status becomes `FAILED`. `test_priority_ordering` is the core test — it verifies that `CRITICAL` tasks dequeue before `HIGH` before `LOW`. `test_fifo_within_priority` uses `time.sleep(0.01)` to ensure distinct creation timestamps, verifying FIFO among same-priority tasks. `test_delayed_task` creates a 0.1s delayed task, verifies it's not ready immediately, waits 0.15s, and confirms it becomes ready. `test_dependency_resolution` verifies that completing task A unblocks task B. `test_dependency_chain` tests multi-level chains: A → B → C. `test_sjf_policy` confirms shortest estimated duration runs first. `test_retry_manager` verifies retries happen when retries remain and DLQ happens when they don't. `test_full_lifecycle` runs a complete submit → execute → complete cycle through the facade.

---

## Class Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                    TaskSchedulerSystem                              │
│──────────────────────────────────────────────────────────────────│
│ - _policy: SchedulingPolicy                                        │
│ - _queue: TaskQueue                                                │
│ - _worker_pool: WorkerPool                                         │
│ - _executor: TaskExecutor                                          │
│ - _dlq: DeadLetterQueue                                            │
│ - _retry_manager: RetryManager                                     │
│ - _delayed: DelayedJobScheduler                                    │
│ - _recurring: RecurringJobManager                                  │
│ - _dependency: DependencyResolver                                  │
│ - _event_bus: TaskEventBus                                         │
│ - _stats: SchedulerStats                                           │
│──────────────────────────────────────────────────────────────────│
│ + submit(task) → task_id                                           │
│ + run_next() → TaskResult | None                                   │
│ + run_all() → list[TaskResult]                                     │
│ + cancel(task_id) → bool                                           │
│ + display_status() → None                                          │
└───────┬──────────┬──────────┬────────────────────────────────────┘
        │          │          │
   uses ▼     uses ▼     uses ▼
┌──────────────────────────────────────────────────────────────────┐
│  «abstract» SchedulingPolicy                                       │
│──────────────────────────────────────────────────────────────────│
│ + enqueue(task, queue) → bool                                      │
│ + dequeue(queue) → Task | None                                     │
│ + name() → str                                                     │
└──────────┬────────────────────────────────────────────────────────┘
           │ implemented by
  ┌────────┼──────────────────┐
  ▼        ▼                  ▼
Priority  FIFO             Shortest
Scheduling Scheduling       Job First
Policy    Policy            Policy

┌──────────────────────────────────────────────────────────────────┐
│  TaskQueue (Heap-Based)                                            │
│──────────────────────────────────────────────────────────────────│
│ - _heap: list[Task]                                                │
│ - _capacity: int                                                   │
│──────────────────────────────────────────────────────────────────│
│ + push(task) → bool                  O(log n)                      │
│ + pop() → Task | None                O(log n)                      │
│ + peek() → Task | None               O(1)                          │
│ + size → int                          O(1)                          │
│ + is_empty → bool                                                  │
│ + is_full → bool                                                   │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  Task (Command Pattern)                                            │
│──────────────────────────────────────────────────────────────────│
│ + task_id: str                                                     │
│ + name: str                                                        │
│ + action: Callable[[], Any]                                        │
│ + priority: TaskPriority                                           │
│ + status: TaskStatus                                               │
│ + delay_seconds: float                                             │
│ + max_retries: int                                                 │
│ + retry_count: int                                                 │
│ + estimated_duration_ms: float                                     │
│ + depends_on: list[str]                                            │
│ + interval_seconds: float                                          │
│ + result: TaskResult | None                                        │
│──────────────────────────────────────────────────────────────────│
│ + execute() → TaskResult                                           │
│ + is_ready → bool                                                  │
│ + is_recurring → bool                                              │
│ + is_terminal → bool                                               │
│ + can_retry → bool                                                 │
│ + __lt__(other) → bool   (heap ordering)                           │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  WorkerPool                                                        │
│──────────────────────────────────────────────────────────────────│
│ - _workers: list[Worker]                                           │
│ - _executor: TaskExecutor                                          │
│──────────────────────────────────────────────────────────────────│
│ + assign(task) → (Worker, TaskResult) | None                       │
│ + available_count → int                                            │
│ + busy_count → int                                                 │
│ + display() → None                                                 │
└──────────┬───────────────────────────────────────────────────────┘
           │ contains
           ▼
┌──────────────────────────────────────────────────────────────────┐
│  Worker                                                            │
│──────────────────────────────────────────────────────────────────│
│ + worker_id: int                                                   │
│ + is_busy → bool                                                   │
│ + tasks_completed → int                                            │
│ + execute(task) → TaskResult                                       │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  TaskExecutor                                                      │
│──────────────────────────────────────────────────────────────────│
│ - _retry_backoff: float                                            │
│──────────────────────────────────────────────────────────────────│
│ + execute(task) → TaskResult        (with retries)                 │
│ + execute_no_retry(task) → TaskResult                              │
│   backoff = base × 2^(retry - 1)                                  │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  RetryManager                                                      │
│──────────────────────────────────────────────────────────────────│
│ - _dlq: DeadLetterQueue                                            │
│ - _backoff_base: float                                             │
│──────────────────────────────────────────────────────────────────│
│ + handle_failure(task, resubmit_fn) → bool                         │
│   → True: task will be retried                                     │
│   → False: task sent to DLQ                                        │
└──────────┬───────────────────────────────────────────────────────┘
           │ uses
           ▼
┌──────────────────────────────────────────────────────────────────┐
│  DeadLetterQueue                                                   │
│──────────────────────────────────────────────────────────────────│
│ - _tasks: list[Task]                                               │
│──────────────────────────────────────────────────────────────────│
│ + add(task) → None                                                 │
│ + replay(task_id, submit_fn) → bool                                │
│ + peek() → list[Task]                                              │
│ + display() → None                                                 │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  DelayedJobScheduler                                               │
│──────────────────────────────────────────────────────────────────│
│ - _delayed: list[(trigger_at, Task)]    min-heap                   │
│──────────────────────────────────────────────────────────────────│
│ + schedule(task) → None                                            │
│ + get_ready_tasks() → list[Task]                                   │
│ + next_trigger → float | None                                      │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  RecurringJobManager                                               │
│──────────────────────────────────────────────────────────────────│
│ - _recurring: dict[str, Task]                                      │
│──────────────────────────────────────────────────────────────────│
│ + register(task) → None                                            │
│ + on_completion(task, submit_fn) → Task | None                     │
│ + cancel(task_id) → bool                                           │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  DependencyResolver (DAG)                                          │
│──────────────────────────────────────────────────────────────────│
│ - _dependencies: dict[str, set[str]]                               │
│ - _completed: set[str]                                             │
│ - _waiting: dict[str, Task]                                        │
│──────────────────────────────────────────────────────────────────│
│ + register(task) → None                                            │
│ + mark_completed(task_id) → list[Task]                             │
│ + has_unmet_dependencies(task) → bool                              │
│ + has_circular_dependency(task) → bool                             │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  «abstract» TaskEventListener                                      │
│──────────────────────────────────────────────────────────────────│
│ + on_event(event) → None                                           │
│ + listener_id() → str                                              │
└──────────┬────────────────────────────────────────────────────────┘
           │ implemented by
  ┌────────┴──────────┐
  ▼                   ▼
TaskLogger         TaskMetrics
                   Collector

┌──────────────┐ ┌──────────────────┐ ┌──────────────────┐
│ TaskResult   │ │ SchedulerStats   │ │ TaskEvent        │
│──────────────│ │──────────────────│ │──────────────────│
│ + task_id    │ │ + submitted      │ │ + event_id       │
│ + success    │ │ + completed      │ │ + event_type     │
│ + value      │ │ + failed         │ │ + task_id        │
│ + error      │ │ + retried        │ │ + task_name      │
│ + exec_time  │ │ + dead_lettered  │ │ + detail         │
│ + attempts   │ │ + cancelled      │ │ + timestamp      │
└──────────────┘ │ + success_rate   │ └──────────────────┘
                 │ + avg_exec_ms    │
                 └──────────────────┘ ┌──────────────────┐
                                      │ TaskEventBus     │
┌──────────────────┐                  │──────────────────│
│ SchedulerScenario│                  │ - _listeners     │
│──────────────────│                  │ - _event_log     │
│ - _system        │                  │──────────────────│
│ - _events        │                  │ + register()     │
│──────────────────│                  │ + emit()         │
│ + at_tick()      │                  │ + unregister()   │
│ + run()          │                  └──────────────────┘
└──────────────────┘

┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│ TaskStatus       │ │ TaskPriority     │ │ TaskEventType    │
│ (Enum)           │ │ (Enum)           │ │ (Enum)           │
│──────────────────│ │──────────────────│ │──────────────────│
│ PENDING          │ │ CRITICAL = 0     │ │ SUBMITTED        │
│ SCHEDULED        │ │ HIGH = 1         │ │ STARTED          │
│ RUNNING          │ │ MEDIUM = 2       │ │ COMPLETED        │
│ COMPLETED        │ │ LOW = 3          │ │ FAILED           │
│ FAILED           │ │ BACKGROUND = 4   │ │ RETRYING         │
│ RETRYING         │ └──────────────────┘ │ DEAD_LETTERED    │
│ DEAD             │                      │ CANCELLED        │
│ CANCELLED        │                      │ DELAYED          │
└──────────────────┘                      │ RECURRING        │
                                          │ DEPENDENCY_MET   │
                                          └──────────────────┘
```

---

## How to Run

Save all code into a single file `task_scheduler.py` and run:

```bash
python task_scheduler.py
```

### Example Output

```
╔══════════════════════════════════════════╗
║      TASK SCHEDULER SYSTEM               ║
║  Policy: Priority                         ║
║  Workers: 4                               ║
║  Max Retries: 3                           ║
╚══════════════════════════════════════════╝

  ───────────────────────────────────────────────────────
  SIMULATION START (ticks: 1 → 7)
  ───────────────────────────────────────────────────────

  ⏱️  Tick 1: Submit tasks with varying priorities
  📥 [      submitted] low-task priority=LOW
  📥 [      submitted] critical-task priority=CRITICAL
  📥 [      submitted] medium-task priority=MEDIUM
  📥 [      submitted] high-task priority=HIGH

  ⏱️  Tick 2: Execute all tasks (priority order)
  ▶️  [        started] critical-task worker assigned
  ✅ [      completed] critical-task in 0.0ms
  ▶️  [        started] high-task worker assigned
  ✅ [      completed] high-task in 0.0ms
  ▶️  [        started] medium-task worker assigned
  ✅ [      completed] medium-task in 0.0ms
  ▶️  [        started] low-task worker assigned
  ✅ [      completed] low-task in 0.0ms

  ⏱️  Tick 3: Submit a flaky task (fails twice)
  📥 [      submitted] flaky-task priority=HIGH

  ⏱️  Tick 4: Execute flaky task (1st attempt fails)
  ▶️  [        started] flaky-task worker assigned
  ❌ [         failed] flaky-task error=transient failure
  🔄 [       retrying] flaky-task attempt 1/3

  ⏱️  Tick 5: Submit a doomed task (always fails)
  📥 [      submitted] doomed-task priority=MEDIUM

  ⏱️  Tick 6: Execute doomed task → DLQ
  ▶️  [        started] doomed-task worker assigned
  ❌ [         failed] doomed-task error=permanent error
  💀 [  dead_lettered] doomed-task moved to DLQ after 0 retries

  ⏱️  Tick 7: Display scheduler statistics

  ══════════════════════════════════════════════════
  SCHEDULER STATISTICS
  ══════════════════════════════════════════════════
  Submitted:         6
  Completed:         4
  Failed:            2
  Retried:           1
  Dead-lettered:     1
  Cancelled:         0
  Success rate:      66.7%
  Avg execution:     0.0ms
  ══════════════════════════════════════════════════

  ──────────────────────────────────────────────────
  WORKER POOL
  ──────────────────────────────────────────────────
  Worker(id=0, status=idle, completed=4)
  Worker(id=1, status=idle, completed=1)
  Worker(id=2, status=idle, completed=1)
  Worker(id=3, status=idle, completed=0)
  Total completed: 6
  ──────────────────────────────────────────────────

  ──────────────────────────────────────────────────
  DEAD LETTER QUEUE (1 tasks)
  ──────────────────────────────────────────────────
  💀 doomed-task (retries=0, errors=[permanent error])
  ──────────────────────────────────────────────────

  Avg execution time: 0.0ms
  Success rate: 66.7% (4 ok, 2 failed)

  ───────────────────────────────────────────────────────
  SIMULATION COMPLETE
  ───────────────────────────────────────────────────────

=======================================================
  SCHEDULING POLICY COMPARISON
=======================================================

  Policy: Priority
  ─────────────────────────────────────────────
    Execution order: task-B → task-C → task-D → task-A → task-E

  Policy: FIFO
  ─────────────────────────────────────────────
    Execution order: task-A → task-B → task-C → task-D → task-E

  Policy: ShortestJobFirst
  ─────────────────────────────────────────────
    Execution order: task-E → task-C → task-B → task-A → task-D

=======================================================

========================================
  RUNNING TESTS
========================================

✅ test_task_creation passed
✅ test_task_execution passed
✅ test_task_failure passed
✅ test_priority_ordering passed
✅ test_fifo_within_priority passed
✅ test_queue_capacity passed
✅ test_delayed_task passed
✅ test_worker_pool passed
✅ test_dead_letter_queue passed
✅ test_retry_manager passed
✅ test_dependency_resolution passed
✅ test_dependency_chain passed
✅ test_fifo_policy passed
✅ test_sjf_policy passed
✅ test_scheduler_stats passed
✅ test_task_cancel passed
✅ test_recurring_task passed
✅ test_task_result passed
✅ test_full_lifecycle passed

✅ All tests passed!
```
