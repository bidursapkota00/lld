# Design Kafka – Low-Level Design Guide

![Bidur Sapkota](https://www.bidursapkota.com.np/images/gravatar.webp "Bidur Sapkota - Developer")&nbsp;[Bidur Sapkota](https://www.bidursapkota.com.np/)

## Table of Contents

1. [Requirements](#requirements)
2. [Key Design Patterns](#key-design-patterns)
3. [Core Enums & Constants](#core-enums--constants)
4. [Message & Serialization](#message--serialization)
5. [Partition & Write-Ahead Log](#partition--write-ahead-log)
6. [Topic](#topic)
7. [Producer & Partitioning Strategy](#producer--partitioning-strategy)
8. [Consumer & Offset Tracking](#consumer--offset-tracking)
9. [Consumer Group & Rebalancing](#consumer-group--rebalancing)
10. [Broker](#broker)
11. [Cluster & Topic Manager](#cluster--topic-manager)
12. [Retention Policy (Strategy Pattern)](#retention-policy-strategy-pattern)
13. [Notification System (Observer Pattern)](#notification-system-observer-pattern)
14. [Metrics & Monitoring](#metrics--monitoring)
15. [Kafka System (Facade)](#kafka-system-facade)
16. [Simulation Engine](#simulation-engine)
17. [Putting It All Together](#putting-it-all-together)
18. [Class Diagram](#class-diagram)
19. [How to Run](#how-to-run)

---

## Requirements

### Functional Requirements

| #  | Requirement                                                                                  |
| -- | -------------------------------------------------------------------------------------------- |
| 1  | Producers publish messages to named topics                                                   |
| 2  | Topics are divided into configurable number of partitions                                    |
| 3  | Each partition maintains an ordered, append-only log with monotonically increasing offsets    |
| 4  | Messages are assigned to partitions via a pluggable partitioning strategy (round-robin, key-based, custom) |
| 5  | Consumers read messages from partitions starting at a committed or specified offset           |
| 6  | Consumer groups enable parallel consumption — each partition is assigned to exactly one consumer in the group |
| 7  | Consumer group rebalancing when consumers join or leave the group                            |
| 8  | Consumers commit offsets to track progress; uncommitted consumers can replay from any offset  |
| 9  | Configurable retention policies — time-based, size-based, or compaction-based                |
| 10 | Multiple brokers host partitions; topics are distributed across brokers                      |
| 11 | Observer-based notifications — listeners react to produce, consume, rebalance, and retention events |
| 12 | Metrics collection — throughput, lag, partition distribution, consumer group health           |

### Non-Functional Requirements

- Clean OOP with SOLID principles.
- Extensible — new partitioning strategies, retention policies, and serializers can be added without modifying core logic.
- Simulatable — the system runs step-by-step for testing and demonstration.

---

## Key Design Patterns

| Pattern  | Where It Is Used                                  | Why                                                                                        |
| -------- | ------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| Strategy | `PartitionStrategy` assigns messages to partitions | Swap between round-robin, key-hash, or custom partitioning without changing producer logic  |
| Strategy | `RetentionPolicy` determines which messages to purge | Add time-based, size-based, or log-compaction retention without modifying partition code    |
| Strategy | `Serializer` encodes/decodes message payloads      | Swap between JSON, string, or binary serialization transparently                           |
| Observer | `KafkaEventListener` observes system events        | Loggers, metrics collectors, and alerting react independently to produce/consume/rebalance  |
| Iterator | `PartitionIterator` provides sequential message access | Consumers iterate through partition logs from any offset without exposing internal storage  |
| Facade   | `KafkaSystem` exposes a unified API                | Hides the complexity of brokers, topics, partitions, consumers, and rebalancing             |

---

## Core Enums & Constants

```python
from enum import Enum


class AckMode(Enum):
    FIRE_AND_FORGET = "fire_and_forget"   # no acknowledgement
    LEADER_ONLY = "leader_only"           # ack after leader writes
    ALL_REPLICAS = "all_replicas"         # ack after all replicas write


class OffsetResetPolicy(Enum):
    EARLIEST = "earliest"     # start from beginning
    LATEST = "latest"         # start from latest offset


class ConsumerState(Enum):
    IDLE = "idle"
    SUBSCRIBED = "subscribed"
    CONSUMING = "consuming"
    PAUSED = "paused"
    CLOSED = "closed"


class RebalanceStrategy(Enum):
    RANGE = "range"
    ROUND_ROBIN = "round_robin"


class RetentionType(Enum):
    TIME_BASED = "time_based"
    SIZE_BASED = "size_based"
    LOG_COMPACTION = "log_compaction"


# ------------------------------------------------------------------
# System constants
# ------------------------------------------------------------------

DEFAULT_NUM_PARTITIONS = 3
DEFAULT_REPLICATION_FACTOR = 1
MAX_MESSAGE_SIZE_BYTES = 1_048_576       # 1 MB
DEFAULT_RETENTION_MS = 7 * 24 * 3600 * 1000   # 7 days
DEFAULT_RETENTION_BYTES = 1_073_741_824  # 1 GB per partition
MAX_POLL_RECORDS = 500                   # max records per poll
CONSUMER_SESSION_TIMEOUT_MS = 30_000     # 30 seconds
DEFAULT_BATCH_SIZE = 16_384              # 16 KB batch
LINGER_MS = 5                            # wait before sending batch
```

`AckMode` controls the durability guarantee a producer expects — `FIRE_AND_FORGET` gives maximum throughput but no guarantee, `LEADER_ONLY` balances throughput and safety, `ALL_REPLICAS` gives the strongest guarantee. `OffsetResetPolicy` decides where a new consumer starts: from the very first message or from the latest. `ConsumerState` tracks the lifecycle of a consumer from creation through subscription, active consumption, pause, and closure. `RebalanceStrategy` determines how partitions are distributed among consumers in a group — `RANGE` assigns contiguous chunks, `ROUND_ROBIN` distributes evenly. The constants define sensible defaults matching real Kafka's configuration.

---

## Message & Serialization

### Message

```python
import time
import json
import hashlib
from dataclasses import dataclass, field


@dataclass
class MessageHeader:
    """Key-value metadata attached to a message."""
    key: str
    value: str


@dataclass
class Message:
    """
    A single record in a Kafka partition log.

    Each message has a key (for partitioning), a value (payload),
    optional headers, a timestamp, and partition/offset assigned
    after being appended to a partition log.
    """
    key: str | None
    value: bytes
    headers: list[MessageHeader] = field(default_factory=list)
    timestamp: int = field(default_factory=lambda: int(time.time() * 1000))
    topic: str = ""
    partition: int = -1
    offset: int = -1

    @property
    def size_bytes(self) -> int:
        size = len(self.value)
        if self.key:
            size += len(self.key.encode())
        for h in self.headers:
            size += len(h.key.encode()) + len(h.value.encode())
        return size

    def __repr__(self) -> str:
        key_str = self.key or "null"
        val_preview = self.value[:50].decode(errors="replace")
        if len(self.value) > 50:
            val_preview += "..."
        return (
            f"Msg(key={key_str} partition={self.partition} "
            f"offset={self.offset} size={self.size_bytes}B "
            f"val={val_preview!r})"
        )
```

### Serializer (Strategy Pattern)

```python
from abc import ABC, abstractmethod


class Serializer(ABC):
    """Defines how message values are serialized and deserialized."""

    @abstractmethod
    def serialize(self, data: object) -> bytes:
        ...

    @abstractmethod
    def deserialize(self, raw: bytes) -> object:
        ...

    @abstractmethod
    def name(self) -> str:
        ...


class StringSerializer(Serializer):
    """Encodes/decodes plain strings to/from UTF-8 bytes."""

    def serialize(self, data: object) -> bytes:
        return str(data).encode("utf-8")

    def deserialize(self, raw: bytes) -> object:
        return raw.decode("utf-8")

    def name(self) -> str:
        return "StringSerializer"


class JsonSerializer(Serializer):
    """Encodes/decodes Python dicts to/from JSON bytes."""

    def serialize(self, data: object) -> bytes:
        return json.dumps(data).encode("utf-8")

    def deserialize(self, raw: bytes) -> object:
        return json.loads(raw.decode("utf-8"))

    def name(self) -> str:
        return "JsonSerializer"
```

`Message` mirrors a Kafka record — key (used for partitioning and compaction), value (the payload as raw bytes), headers (metadata), and a timestamp. `partition` and `offset` are assigned by the broker when the message is appended to a partition log; they are `-1` until then. `size_bytes` accounts for the full wire size. `Serializer` abstracts encoding — producers serialize before sending, consumers deserialize after receiving. `StringSerializer` handles plain text; `JsonSerializer` handles structured data. Adding Avro, Protobuf, or any custom format requires only a new `Serializer` subclass.

---

## Partition & Write-Ahead Log

```python
import threading


class Partition:
    """
    An ordered, append-only log of messages.

    Each partition maintains a monotonically increasing offset counter.
    Messages are appended at the end and read sequentially from any
    starting offset. Thread-safe via a reentrant lock.
    """

    def __init__(self, topic: str, partition_id: int) -> None:
        self._topic = topic
        self._partition_id = partition_id
        self._log: list[Message] = []
        self._next_offset: int = 0
        self._lock = threading.RLock()
        self._total_bytes: int = 0
        self._created_at: int = int(time.time() * 1000)

    # ------------------------------------------------------------------
    # Properties
    # ------------------------------------------------------------------

    @property
    def topic(self) -> str:
        return self._topic

    @property
    def partition_id(self) -> int:
        return self._partition_id

    @property
    def next_offset(self) -> int:
        return self._next_offset

    @property
    def log_size(self) -> int:
        return len(self._log)

    @property
    def total_bytes(self) -> int:
        return self._total_bytes

    @property
    def earliest_offset(self) -> int:
        if not self._log:
            return 0
        return self._log[0].offset

    @property
    def latest_offset(self) -> int:
        return self._next_offset

    # ------------------------------------------------------------------
    # Write
    # ------------------------------------------------------------------

    def append(self, message: Message) -> int:
        """
        Append a message to the log and return the assigned offset.

        Validates message size, assigns partition/offset metadata,
        and updates byte counters atomically.
        """
        if message.size_bytes > MAX_MESSAGE_SIZE_BYTES:
            raise ValueError(
                f"Message size {message.size_bytes}B exceeds "
                f"max {MAX_MESSAGE_SIZE_BYTES}B"
            )

        with self._lock:
            offset = self._next_offset
            message.topic = self._topic
            message.partition = self._partition_id
            message.offset = offset
            self._log.append(message)
            self._next_offset += 1
            self._total_bytes += message.size_bytes
            return offset

    # ------------------------------------------------------------------
    # Read
    # ------------------------------------------------------------------

    def read(self, offset: int, max_records: int = MAX_POLL_RECORDS
             ) -> list[Message]:
        """
        Read up to max_records messages starting from the given offset.

        Returns an empty list if offset is beyond the latest.
        """
        with self._lock:
            if not self._log:
                return []

            # Find start index relative to earliest offset
            base = self._log[0].offset
            start_idx = offset - base
            if start_idx < 0:
                start_idx = 0
            if start_idx >= len(self._log):
                return []

            end_idx = min(start_idx + max_records, len(self._log))
            return list(self._log[start_idx:end_idx])

    def read_at(self, offset: int) -> Message | None:
        """Read a single message at the exact offset."""
        messages = self.read(offset, 1)
        if messages and messages[0].offset == offset:
            return messages[0]
        return None

    # ------------------------------------------------------------------
    # Retention
    # ------------------------------------------------------------------

    def truncate_before(self, offset: int) -> int:
        """
        Remove all messages with offset < given offset.
        Returns the number of messages removed.
        """
        with self._lock:
            if not self._log:
                return 0

            base = self._log[0].offset
            cut_idx = offset - base
            if cut_idx <= 0:
                return 0
            cut_idx = min(cut_idx, len(self._log))

            removed = self._log[:cut_idx]
            self._log = self._log[cut_idx:]
            bytes_removed = sum(m.size_bytes for m in removed)
            self._total_bytes -= bytes_removed
            return len(removed)

    def truncate_to_size(self, max_bytes: int) -> int:
        """Remove oldest messages until total_bytes <= max_bytes."""
        with self._lock:
            removed_count = 0
            while self._log and self._total_bytes > max_bytes:
                msg = self._log.pop(0)
                self._total_bytes -= msg.size_bytes
                removed_count += 1
            return removed_count

    def compact(self) -> int:
        """
        Log compaction — keep only the latest message per key.
        Messages without keys are always retained.
        Returns the number of messages removed.
        """
        with self._lock:
            if not self._log:
                return 0

            # Build map of key -> latest message
            latest_by_key: dict[str, Message] = {}
            no_key: list[Message] = []

            for msg in self._log:
                if msg.key is None:
                    no_key.append(msg)
                else:
                    latest_by_key[msg.key] = msg

            compacted = sorted(
                list(latest_by_key.values()) + no_key,
                key=lambda m: m.offset
            )
            removed = len(self._log) - len(compacted)
            self._log = compacted
            self._total_bytes = sum(m.size_bytes for m in self._log)
            return removed

    # ------------------------------------------------------------------
    # Display
    # ------------------------------------------------------------------

    def __repr__(self) -> str:
        return (
            f"Partition({self._topic}-{self._partition_id} "
            f"offsets=[{self.earliest_offset}..{self.latest_offset}) "
            f"msgs={self.log_size} "
            f"bytes={self._total_bytes})"
        )

    def display(self) -> None:
        print(f"\n  {'─'*55}")
        print(f"  PARTITION {self._topic}-{self._partition_id}")
        print(f"  {'─'*55}")
        print(f"  Offsets:   [{self.earliest_offset}..{self.latest_offset})")
        print(f"  Messages:  {self.log_size}")
        print(f"  Size:      {self._total_bytes:,} bytes")
        for msg in self._log[-5:]:
            print(f"    {msg}")
        if self.log_size > 5:
            print(f"    ... ({self.log_size - 5} more)")
        print()
```

`Partition` is the fundamental storage unit — an ordered, append-only log. `append` assigns a monotonically increasing offset to each message, making the log's ordering guarantee explicit. The `_lock` ensures thread safety for concurrent producers writing to the same partition. `read` supports batch consumption from any starting offset — the consumer controls where it reads from. `truncate_before` and `truncate_to_size` implement time-based and size-based retention by removing the oldest messages. `compact` implements log compaction by keeping only the latest message per key — this is how Kafka maintains changelogs where only the most recent state matters. All three retention methods are invoked by the retention policy strategies.

---

## Topic

```python
class Topic:
    """
    A named feed of messages divided into partitions.

    A topic is created with a fixed number of partitions.
    Messages are distributed across partitions by the producer's
    partitioning strategy.
    """

    def __init__(self, name: str, num_partitions: int = DEFAULT_NUM_PARTITIONS,
                 replication_factor: int = DEFAULT_REPLICATION_FACTOR) -> None:
        self._name = name
        self._replication_factor = replication_factor
        self._partitions: list[Partition] = [
            Partition(name, i) for i in range(num_partitions)
        ]
        self._created_at: int = int(time.time() * 1000)

    # ------------------------------------------------------------------
    # Properties
    # ------------------------------------------------------------------

    @property
    def name(self) -> str:
        return self._name

    @property
    def num_partitions(self) -> int:
        return len(self._partitions)

    @property
    def replication_factor(self) -> int:
        return self._replication_factor

    @property
    def partitions(self) -> list[Partition]:
        return list(self._partitions)

    @property
    def total_messages(self) -> int:
        return sum(p.log_size for p in self._partitions)

    @property
    def total_bytes(self) -> int:
        return sum(p.total_bytes for p in self._partitions)

    # ------------------------------------------------------------------
    # Partition access
    # ------------------------------------------------------------------

    def get_partition(self, partition_id: int) -> Partition:
        if partition_id < 0 or partition_id >= len(self._partitions):
            raise ValueError(
                f"Partition {partition_id} does not exist in "
                f"topic '{self._name}' (0..{len(self._partitions)-1})"
            )
        return self._partitions[partition_id]

    # ------------------------------------------------------------------
    # Display
    # ------------------------------------------------------------------

    def __repr__(self) -> str:
        return (
            f"Topic({self._name} partitions={self.num_partitions} "
            f"msgs={self.total_messages} "
            f"bytes={self.total_bytes:,})"
        )

    def display(self) -> None:
        print(f"\n  {'═'*55}")
        print(f"  TOPIC — {self._name}")
        print(f"  {'═'*55}")
        print(f"  Partitions:          {self.num_partitions}")
        print(f"  Replication Factor:  {self._replication_factor}")
        print(f"  Total Messages:      {self.total_messages}")
        print(f"  Total Size:          {self.total_bytes:,} bytes")
        print(f"  {'─'*55}")
        for p in self._partitions:
            print(f"  {p}")
        print()
```

`Topic` is a logical grouping of partitions under a name. When a topic is created, it instantiates `num_partitions` independent `Partition` objects. Each partition is an independent ordered log — this is what enables parallelism. `total_messages` and `total_bytes` aggregate across all partitions for monitoring. The replication factor is stored for metadata but replication itself is handled at the broker level in a real system — here it tracks the configuration.

---

## Producer & Partitioning Strategy

### Partitioning Strategy

```python
class PartitionStrategy(ABC):
    """Determines which partition a message is sent to."""

    @abstractmethod
    def select_partition(self, key: str | None,
                         num_partitions: int) -> int:
        ...

    @abstractmethod
    def name(self) -> str:
        ...


class RoundRobinPartitioner(PartitionStrategy):
    """
    Distributes messages evenly across partitions in round-robin order.
    Ignores the message key. Maximises partition balance.
    """

    def __init__(self) -> None:
        self._counter: int = 0

    def select_partition(self, key: str | None,
                         num_partitions: int) -> int:
        partition = self._counter % num_partitions
        self._counter += 1
        return partition

    def name(self) -> str:
        return "RoundRobin"


class KeyHashPartitioner(PartitionStrategy):
    """
    Assigns messages to partitions based on a hash of the key.
    Messages with the same key always go to the same partition,
    guaranteeing per-key ordering.
    Null-key messages use round-robin fallback.
    """

    def __init__(self) -> None:
        self._fallback = RoundRobinPartitioner()

    def select_partition(self, key: str | None,
                         num_partitions: int) -> int:
        if key is None:
            return self._fallback.select_partition(key, num_partitions)

        key_hash = int(hashlib.md5(key.encode()).hexdigest(), 16)
        return key_hash % num_partitions

    def name(self) -> str:
        return "KeyHash"


class CustomPartitioner(PartitionStrategy):
    """
    Routes messages to a specific partition based on a user-defined
    function. Useful for geo-based or priority-based routing.
    """

    def __init__(self, partition_fn: callable) -> None:
        self._fn = partition_fn

    def select_partition(self, key: str | None,
                         num_partitions: int) -> int:
        return self._fn(key, num_partitions) % num_partitions

    def name(self) -> str:
        return "Custom"
```

### Producer Record & Result

```python
@dataclass
class ProducerRecord:
    """A record to be sent to a topic by a producer."""
    topic: str
    value: object
    key: str | None = None
    headers: list[MessageHeader] = field(default_factory=list)
    partition: int | None = None    # explicit partition override


@dataclass
class RecordMetadata:
    """Metadata returned after a message is successfully produced."""
    topic: str
    partition: int
    offset: int
    timestamp: int
    size_bytes: int

    def __repr__(self) -> str:
        return (
            f"RecordMetadata(topic={self.topic} "
            f"partition={self.partition} offset={self.offset} "
            f"size={self.size_bytes}B)"
        )
```

### Producer

```python
class Producer:
    """
    Publishes messages to Kafka topics.

    The producer serializes records, selects a partition using
    the configured strategy, and appends the message to the
    target partition's log.
    """

    _next_id: int = 0

    def __init__(self, serializer: Serializer | None = None,
                 partitioner: PartitionStrategy | None = None,
                 ack_mode: AckMode = AckMode.LEADER_ONLY) -> None:
        Producer._next_id += 1
        self._id = f"producer-{Producer._next_id}"
        self._serializer = serializer or StringSerializer()
        self._partitioner = partitioner or KeyHashPartitioner()
        self._ack_mode = ack_mode
        self._messages_sent: int = 0
        self._bytes_sent: int = 0
        self._closed: bool = False

    # ------------------------------------------------------------------
    # Properties
    # ------------------------------------------------------------------

    @property
    def id(self) -> str:
        return self._id

    @property
    def messages_sent(self) -> int:
        return self._messages_sent

    @property
    def bytes_sent(self) -> int:
        return self._bytes_sent

    @property
    def ack_mode(self) -> AckMode:
        return self._ack_mode

    # ------------------------------------------------------------------
    # Send
    # ------------------------------------------------------------------

    def send(self, record: ProducerRecord,
             topics: dict[str, Topic]) -> RecordMetadata | None:
        """
        Serialize and send a record to the target topic and partition.

        Steps:
        1. Validate the producer is open and the topic exists.
        2. Serialize the value using the configured serializer.
        3. Select a partition (explicit override or strategy).
        4. Append the message to the partition log.
        5. Return metadata on success.
        """
        if self._closed:
            print(f"  ✗ Producer {self._id} is closed")
            return None

        topic = topics.get(record.topic)
        if topic is None:
            print(f"  ✗ Topic '{record.topic}' does not exist")
            return None

        # Serialize
        value_bytes = self._serializer.serialize(record.value)

        # Build message
        message = Message(
            key=record.key,
            value=value_bytes,
            headers=record.headers,
        )

        # Validate size
        if message.size_bytes > MAX_MESSAGE_SIZE_BYTES:
            print(f"  ✗ Message too large: {message.size_bytes}B "
                  f"> {MAX_MESSAGE_SIZE_BYTES}B")
            return None

        # Select partition
        if record.partition is not None:
            partition_id = record.partition
        else:
            partition_id = self._partitioner.select_partition(
                record.key, topic.num_partitions
            )

        partition = topic.get_partition(partition_id)

        # Append
        offset = partition.append(message)

        # Update stats
        self._messages_sent += 1
        self._bytes_sent += message.size_bytes

        return RecordMetadata(
            topic=record.topic,
            partition=partition_id,
            offset=offset,
            timestamp=message.timestamp,
            size_bytes=message.size_bytes,
        )

    def close(self) -> None:
        self._closed = True

    def __repr__(self) -> str:
        return (
            f"Producer({self._id} {self._partitioner.name()} "
            f"sent={self._messages_sent} "
            f"bytes={self._bytes_sent:,})"
        )
```

`PartitionStrategy` decouples message routing from the producer. `RoundRobinPartitioner` cycles through partitions sequentially — best for even load distribution when key ordering does not matter. `KeyHashPartitioner` hashes the message key to guarantee same-key messages always land in the same partition — this is critical for maintaining per-key ordering (e.g., all events for user-123 go to partition 2). `CustomPartitioner` accepts a user-defined function for domain-specific routing. `Producer` orchestrates the full send pipeline: serialize → select partition → append to log. `ProducerRecord` separates the user's intent (topic, key, value) from the internal `Message` structure. `RecordMetadata` gives the producer confirmation of where the message landed.

---

## Consumer & Offset Tracking

### Offset Manager

```python
class OffsetManager:
    """
    Tracks committed offsets for a consumer across topic-partitions.

    Each consumer maintains a map of (topic, partition) -> committed_offset.
    The committed offset is the NEXT offset to be read, not the last
    offset that was read.
    """

    def __init__(self) -> None:
        self._committed: dict[tuple[str, int], int] = {}

    def commit(self, topic: str, partition: int, offset: int) -> None:
        """Commit the next offset to read."""
        self._committed[(topic, partition)] = offset

    def get_committed(self, topic: str, partition: int) -> int | None:
        """Get the committed offset, or None if never committed."""
        return self._committed.get((topic, partition))

    def reset(self, topic: str, partition: int) -> None:
        """Remove the committed offset for a partition."""
        self._committed.pop((topic, partition), None)

    def all_committed(self) -> dict[tuple[str, int], int]:
        return dict(self._committed)

    def __repr__(self) -> str:
        entries = [f"{t}-{p}:{o}" for (t, p), o in self._committed.items()]
        return f"Offsets({', '.join(entries)})"
```

### Consumer

```python
class Consumer:
    """
    Reads messages from assigned partitions.

    A consumer belongs to a consumer group. It is assigned partitions
    by the group coordinator and reads sequentially from each partition
    starting at the committed offset.
    """

    _next_id: int = 0

    def __init__(self, group_id: str,
                 deserializer: Serializer | None = None,
                 offset_reset: OffsetResetPolicy = OffsetResetPolicy.EARLIEST
                 ) -> None:
        Consumer._next_id += 1
        self._id = f"consumer-{Consumer._next_id}"
        self._group_id = group_id
        self._deserializer = deserializer or StringSerializer()
        self._offset_reset = offset_reset
        self._offset_manager = OffsetManager()
        self._assigned_partitions: list[tuple[str, int]] = []
        self._state = ConsumerState.IDLE
        self._messages_consumed: int = 0
        self._bytes_consumed: int = 0

    # ------------------------------------------------------------------
    # Properties
    # ------------------------------------------------------------------

    @property
    def id(self) -> str:
        return self._id

    @property
    def group_id(self) -> str:
        return self._group_id

    @property
    def state(self) -> ConsumerState:
        return self._state

    @property
    def assigned_partitions(self) -> list[tuple[str, int]]:
        return list(self._assigned_partitions)

    @property
    def offset_manager(self) -> OffsetManager:
        return self._offset_manager

    @property
    def messages_consumed(self) -> int:
        return self._messages_consumed

    @property
    def bytes_consumed(self) -> int:
        return self._bytes_consumed

    # ------------------------------------------------------------------
    # Subscription & assignment
    # ------------------------------------------------------------------

    def subscribe(self) -> None:
        """Mark the consumer as subscribed and ready for assignment."""
        if self._state == ConsumerState.CLOSED:
            print(f"  ✗ Consumer {self._id} is closed")
            return
        self._state = ConsumerState.SUBSCRIBED

    def assign_partitions(self, partitions: list[tuple[str, int]]) -> None:
        """Assign specific topic-partitions to this consumer."""
        self._assigned_partitions = partitions
        if partitions:
            self._state = ConsumerState.CONSUMING

    def revoke_partitions(self) -> None:
        """Remove all partition assignments (during rebalance)."""
        self._assigned_partitions = []
        if self._state != ConsumerState.CLOSED:
            self._state = ConsumerState.SUBSCRIBED

    # ------------------------------------------------------------------
    # Poll
    # ------------------------------------------------------------------

    def poll(self, topics: dict[str, Topic],
             max_records: int = MAX_POLL_RECORDS) -> list[Message]:
        """
        Fetch messages from all assigned partitions.

        For each assigned partition, reads from the committed offset
        (or the reset policy position if no offset is committed).
        Returns messages from all partitions combined.
        """
        if self._state not in (ConsumerState.CONSUMING,
                                ConsumerState.SUBSCRIBED):
            return []

        self._state = ConsumerState.CONSUMING
        all_messages: list[Message] = []
        records_per_partition = max(
            1, max_records // max(1, len(self._assigned_partitions))
        )

        for topic_name, partition_id in self._assigned_partitions:
            topic = topics.get(topic_name)
            if topic is None:
                continue

            partition = topic.get_partition(partition_id)

            # Determine start offset
            committed = self._offset_manager.get_committed(
                topic_name, partition_id
            )
            if committed is not None:
                start_offset = committed
            elif self._offset_reset == OffsetResetPolicy.EARLIEST:
                start_offset = partition.earliest_offset
            else:
                start_offset = partition.latest_offset

            messages = partition.read(start_offset, records_per_partition)
            all_messages.extend(messages)

        self._messages_consumed += len(all_messages)
        self._bytes_consumed += sum(m.size_bytes for m in all_messages)
        return all_messages

    # ------------------------------------------------------------------
    # Commit
    # ------------------------------------------------------------------

    def commit_offsets(self, messages: list[Message]) -> None:
        """
        Commit offsets for all consumed messages.

        For each (topic, partition), commits the offset of the last
        message + 1, which is the next offset to read.
        """
        # Group by (topic, partition) and find max offset
        max_offsets: dict[tuple[str, int], int] = {}
        for msg in messages:
            key = (msg.topic, msg.partition)
            if key not in max_offsets or msg.offset > max_offsets[key]:
                max_offsets[key] = msg.offset

        for (topic, partition), max_offset in max_offsets.items():
            self._offset_manager.commit(topic, partition, max_offset + 1)

    def commit_sync(self, topic: str, partition: int,
                    offset: int) -> None:
        """Manually commit a specific offset."""
        self._offset_manager.commit(topic, partition, offset)

    # ------------------------------------------------------------------
    # Lifecycle
    # ------------------------------------------------------------------

    def pause(self) -> None:
        if self._state == ConsumerState.CONSUMING:
            self._state = ConsumerState.PAUSED

    def resume(self) -> None:
        if self._state == ConsumerState.PAUSED:
            self._state = ConsumerState.CONSUMING

    def close(self) -> None:
        self._assigned_partitions = []
        self._state = ConsumerState.CLOSED

    # ------------------------------------------------------------------
    # Lag calculation
    # ------------------------------------------------------------------

    def compute_lag(self, topics: dict[str, Topic]) -> dict[tuple[str, int], int]:
        """
        Compute consumer lag — the number of unconsumed messages —
        for each assigned partition.
        """
        lag: dict[tuple[str, int], int] = {}
        for topic_name, partition_id in self._assigned_partitions:
            topic = topics.get(topic_name)
            if topic is None:
                continue
            partition = topic.get_partition(partition_id)
            committed = self._offset_manager.get_committed(
                topic_name, partition_id
            )
            if committed is None:
                committed = partition.earliest_offset
            lag[(topic_name, partition_id)] = (
                partition.latest_offset - committed
            )
        return lag

    # ------------------------------------------------------------------
    # Display
    # ------------------------------------------------------------------

    def __repr__(self) -> str:
        partitions_str = ", ".join(
            f"{t}-{p}" for t, p in self._assigned_partitions
        )
        return (
            f"Consumer({self._id} group={self._group_id} "
            f"[{self._state.value}] "
            f"partitions=[{partitions_str}] "
            f"consumed={self._messages_consumed})"
        )
```

`OffsetManager` tracks committed offsets per topic-partition. The committed offset is the *next* offset to read, not the last read — this is exactly how real Kafka works. `Consumer` fetches messages from its assigned partitions, starting at the committed offset or falling back to the offset reset policy (`EARLIEST` = replay everything, `LATEST` = only new messages). `poll` distributes the `max_records` budget across assigned partitions. `commit_offsets` computes the max offset per partition from the consumed batch and commits `max_offset + 1`. `compute_lag` calculates the gap between the committed offset and the partition's latest offset — this is the primary metric for monitoring consumer health.

---

## Consumer Group & Rebalancing

```python
class ConsumerGroup:
    """
    A group of consumers that collectively consume a topic.

    Each partition is assigned to exactly one consumer in the group.
    When consumers join or leave, a rebalance reassigns partitions.
    Supports range and round-robin assignment strategies.
    """

    def __init__(self, group_id: str,
                 strategy: RebalanceStrategy = RebalanceStrategy.RANGE
                 ) -> None:
        self._group_id = group_id
        self._strategy = strategy
        self._consumers: dict[str, Consumer] = {}
        self._subscriptions: set[str] = set()    # topic names
        self._generation: int = 0                 # rebalance generation

    # ------------------------------------------------------------------
    # Properties
    # ------------------------------------------------------------------

    @property
    def group_id(self) -> str:
        return self._group_id

    @property
    def generation(self) -> int:
        return self._generation

    @property
    def consumers(self) -> list[Consumer]:
        return list(self._consumers.values())

    @property
    def subscriptions(self) -> set[str]:
        return set(self._subscriptions)

    @property
    def active_consumers(self) -> list[Consumer]:
        return [c for c in self._consumers.values()
                if c.state != ConsumerState.CLOSED]

    # ------------------------------------------------------------------
    # Membership
    # ------------------------------------------------------------------

    def add_consumer(self, consumer: Consumer) -> None:
        """Add a consumer to the group and trigger rebalance."""
        self._consumers[consumer.id] = consumer
        consumer.subscribe()

    def remove_consumer(self, consumer_id: str) -> None:
        """Remove a consumer from the group and trigger rebalance."""
        consumer = self._consumers.pop(consumer_id, None)
        if consumer:
            consumer.revoke_partitions()
            consumer.close()

    def subscribe_to_topic(self, topic_name: str) -> None:
        """Subscribe the group to a topic."""
        self._subscriptions.add(topic_name)

    # ------------------------------------------------------------------
    # Rebalance
    # ------------------------------------------------------------------

    def rebalance(self, topics: dict[str, Topic]) -> None:
        """
        Reassign partitions to consumers using the configured strategy.

        Steps:
        1. Revoke all existing assignments.
        2. Collect all partitions from subscribed topics.
        3. Distribute partitions using range or round-robin.
        4. Increment the generation counter.
        """
        self._generation += 1
        active = self.active_consumers
        if not active:
            return

        # Revoke all
        for consumer in active:
            consumer.revoke_partitions()

        # Collect partitions
        all_partitions: list[tuple[str, int]] = []
        for topic_name in sorted(self._subscriptions):
            topic = topics.get(topic_name)
            if topic is None:
                continue
            for p in topic.partitions:
                all_partitions.append((topic_name, p.partition_id))

        if not all_partitions:
            return

        # Assign
        if self._strategy == RebalanceStrategy.ROUND_ROBIN:
            self._round_robin_assign(active, all_partitions)
        else:
            self._range_assign(active, all_partitions)

        print(f"  ⚡ Rebalance complete — group={self._group_id} "
              f"gen={self._generation} "
              f"consumers={len(active)} "
              f"partitions={len(all_partitions)}")

    def _range_assign(self, consumers: list[Consumer],
                      partitions: list[tuple[str, int]]) -> None:
        """
        Range assignment: divide partitions into contiguous chunks.

        Partitions are sorted by (topic, partition_id). Each consumer
        gets a contiguous range. If partitions don't divide evenly,
        the first consumers get one extra partition.
        """
        n_consumers = len(consumers)
        n_partitions = len(partitions)
        base = n_partitions // n_consumers
        remainder = n_partitions % n_consumers

        idx = 0
        for i, consumer in enumerate(consumers):
            count = base + (1 if i < remainder else 0)
            assigned = partitions[idx:idx + count]
            consumer.assign_partitions(assigned)
            idx += count

    def _round_robin_assign(self, consumers: list[Consumer],
                            partitions: list[tuple[str, int]]) -> None:
        """
        Round-robin assignment: distribute partitions one at a time
        across consumers in circular order.
        """
        assignments: dict[str, list[tuple[str, int]]] = {
            c.id: [] for c in consumers
        }
        for i, partition in enumerate(partitions):
            consumer = consumers[i % len(consumers)]
            assignments[consumer.id].append(partition)

        for consumer in consumers:
            consumer.assign_partitions(assignments[consumer.id])

    # ------------------------------------------------------------------
    # Lag
    # ------------------------------------------------------------------

    def total_lag(self, topics: dict[str, Topic]) -> int:
        """Compute total lag across all consumers in the group."""
        total = 0
        for consumer in self.active_consumers:
            lag = consumer.compute_lag(topics)
            total += sum(lag.values())
        return total

    # ------------------------------------------------------------------
    # Display
    # ------------------------------------------------------------------

    def __repr__(self) -> str:
        return (
            f"ConsumerGroup({self._group_id} "
            f"gen={self._generation} "
            f"consumers={len(self.active_consumers)} "
            f"topics={self._subscriptions})"
        )

    def display(self) -> None:
        print(f"\n  {'═'*55}")
        print(f"  CONSUMER GROUP — {self._group_id}")
        print(f"  {'═'*55}")
        print(f"  Strategy:    {self._strategy.value}")
        print(f"  Generation:  {self._generation}")
        print(f"  Topics:      {', '.join(sorted(self._subscriptions))}")
        print(f"  {'─'*55}")
        for consumer in self._consumers.values():
            print(f"  {consumer}")
        print()
```

`ConsumerGroup` is the coordination layer. Each partition is assigned to exactly one consumer in the group — this is the core guarantee that enables parallel, non-duplicate consumption. `rebalance` is triggered when consumers join or leave and follows a strict protocol: revoke all existing assignments, collect all partitions from subscribed topics, redistribute, and increment the generation counter. `_range_assign` gives contiguous chunks (partition 0–2 to consumer-1, 3–5 to consumer-2) — this minimises partition movement during rebalances. `_round_robin_assign` deals partitions one at a time for more even distribution. `total_lag` aggregates lag across all consumers — a non-zero value means the group is falling behind producers.

---

## Broker

```python
class Broker:
    """
    A Kafka broker that hosts topic partitions.

    Brokers store the actual partition data and handle
    produce/consume requests routed to their partitions.
    """

    _next_id: int = 0

    def __init__(self, host: str = "localhost",
                 port: int = 9092) -> None:
        Broker._next_id += 1
        self._id = f"broker-{Broker._next_id}"
        self._host = host
        self._port = port
        self._topics: dict[str, Topic] = {}
        self._started_at: int = int(time.time() * 1000)
        self._is_alive: bool = True

    # ------------------------------------------------------------------
    # Properties
    # ------------------------------------------------------------------

    @property
    def id(self) -> str:
        return self._id

    @property
    def host(self) -> str:
        return self._host

    @property
    def port(self) -> int:
        return self._port

    @property
    def address(self) -> str:
        return f"{self._host}:{self._port}"

    @property
    def is_alive(self) -> bool:
        return self._is_alive

    @property
    def topics(self) -> dict[str, Topic]:
        return dict(self._topics)

    @property
    def total_partitions(self) -> int:
        return sum(t.num_partitions for t in self._topics.values())

    @property
    def total_messages(self) -> int:
        return sum(t.total_messages for t in self._topics.values())

    # ------------------------------------------------------------------
    # Topic management
    # ------------------------------------------------------------------

    def create_topic(self, name: str,
                     num_partitions: int = DEFAULT_NUM_PARTITIONS,
                     replication_factor: int = DEFAULT_REPLICATION_FACTOR
                     ) -> Topic:
        """Create a new topic on this broker."""
        if name in self._topics:
            raise ValueError(f"Topic '{name}' already exists on "
                             f"broker {self._id}")
        topic = Topic(name, num_partitions, replication_factor)
        self._topics[name] = topic
        print(f"  📦 Topic '{name}' created on {self._id} "
              f"({num_partitions} partitions)")
        return topic

    def get_topic(self, name: str) -> Topic | None:
        return self._topics.get(name)

    def delete_topic(self, name: str) -> bool:
        if name in self._topics:
            del self._topics[name]
            print(f"  🗑 Topic '{name}' deleted from {self._id}")
            return True
        return False

    # ------------------------------------------------------------------
    # Lifecycle
    # ------------------------------------------------------------------

    def shutdown(self) -> None:
        self._is_alive = False
        print(f"  ⚫ Broker {self._id} shut down")

    def start(self) -> None:
        self._is_alive = True
        print(f"  🟢 Broker {self._id} started at {self.address}")

    # ------------------------------------------------------------------
    # Display
    # ------------------------------------------------------------------

    def __repr__(self) -> str:
        status = "🟢" if self._is_alive else "⚫"
        return (
            f"Broker({self._id} {status} {self.address} "
            f"topics={len(self._topics)} "
            f"partitions={self.total_partitions} "
            f"msgs={self.total_messages})"
        )

    def display(self) -> None:
        status = "ONLINE" if self._is_alive else "OFFLINE"
        print(f"\n  {'═'*55}")
        print(f"  BROKER — {self._id} [{status}]")
        print(f"  {'═'*55}")
        print(f"  Address:     {self.address}")
        print(f"  Topics:      {len(self._topics)}")
        print(f"  Partitions:  {self.total_partitions}")
        print(f"  Messages:    {self.total_messages}")
        print(f"  {'─'*55}")
        for topic in self._topics.values():
            print(f"  {topic}")
        print()
```

`Broker` is the physical node that hosts partition data. In a real Kafka cluster, each broker stores a subset of the total partitions. `create_topic` instantiates a topic with its partitions on this broker. `shutdown`/`start` model broker lifecycle — when a broker goes offline, its partitions become unavailable. The broker tracks aggregate metrics (total partitions, total messages) for cluster-level monitoring.

---

## Cluster & Topic Manager

```python
class Cluster:
    """
    Manages a cluster of Kafka brokers and topic distribution.

    Handles topic creation across brokers, topic-level metadata,
    and provides a unified topic registry for producers and consumers.
    """

    def __init__(self) -> None:
        self._brokers: dict[str, Broker] = {}
        self._topic_broker_map: dict[str, str] = {}   # topic -> broker_id
        self._all_topics: dict[str, Topic] = {}

    # ------------------------------------------------------------------
    # Broker management
    # ------------------------------------------------------------------

    def add_broker(self, broker: Broker) -> None:
        self._brokers[broker.id] = broker

    def remove_broker(self, broker_id: str) -> None:
        broker = self._brokers.pop(broker_id, None)
        if broker:
            broker.shutdown()

    def get_broker(self, broker_id: str) -> Broker | None:
        return self._brokers.get(broker_id)

    @property
    def brokers(self) -> list[Broker]:
        return list(self._brokers.values())

    @property
    def alive_brokers(self) -> list[Broker]:
        return [b for b in self._brokers.values() if b.is_alive]

    # ------------------------------------------------------------------
    # Topic management
    # ------------------------------------------------------------------

    def create_topic(self, name: str,
                     num_partitions: int = DEFAULT_NUM_PARTITIONS,
                     replication_factor: int = DEFAULT_REPLICATION_FACTOR
                     ) -> Topic | None:
        """
        Create a topic on the broker with the fewest partitions.
        Returns the created topic or None if no brokers available.
        """
        if name in self._all_topics:
            print(f"  ✗ Topic '{name}' already exists")
            return self._all_topics[name]

        alive = self.alive_brokers
        if not alive:
            print(f"  ✗ No alive brokers to create topic '{name}'")
            return None

        # Pick broker with fewest partitions
        target = min(alive, key=lambda b: b.total_partitions)
        topic = target.create_topic(name, num_partitions,
                                    replication_factor)
        self._topic_broker_map[name] = target.id
        self._all_topics[name] = topic
        return topic

    def get_topic(self, name: str) -> Topic | None:
        return self._all_topics.get(name)

    def delete_topic(self, name: str) -> bool:
        broker_id = self._topic_broker_map.pop(name, None)
        if broker_id:
            broker = self._brokers.get(broker_id)
            if broker:
                broker.delete_topic(name)
        return self._all_topics.pop(name, None) is not None

    @property
    def all_topics(self) -> dict[str, Topic]:
        return dict(self._all_topics)

    # ------------------------------------------------------------------
    # Display
    # ------------------------------------------------------------------

    def display(self) -> None:
        print(f"\n  {'═'*55}")
        print(f"  CLUSTER STATUS")
        print(f"  {'═'*55}")
        print(f"  Brokers:    {len(self._brokers)} "
              f"({len(self.alive_brokers)} alive)")
        print(f"  Topics:     {len(self._all_topics)}")
        print(f"  {'─'*55}")
        for broker in self._brokers.values():
            print(f"  {broker}")
        print()
```

`Cluster` manages the multi-broker topology. `create_topic` places the topic on the broker with the fewest partitions — a simple load-balancing heuristic. `_topic_broker_map` tracks which broker owns which topic, and `_all_topics` provides a unified registry that producers and consumers use for lookups. This separation means producers and consumers never talk directly to brokers — they go through the cluster's topic registry.

---

## Retention Policy (Strategy Pattern)

```python
class RetentionPolicy(ABC):
    """Defines how old messages are purged from partitions."""

    @abstractmethod
    def enforce(self, partition: Partition) -> int:
        """
        Apply the retention policy to the partition.
        Returns the number of messages removed.
        """
        ...

    @abstractmethod
    def name(self) -> str:
        ...


class TimeBasedRetention(RetentionPolicy):
    """
    Remove messages older than a configurable duration.
    Default: 7 days.
    """

    def __init__(self, retention_ms: int = DEFAULT_RETENTION_MS) -> None:
        self._retention_ms = retention_ms

    def enforce(self, partition: Partition) -> int:
        cutoff = int(time.time() * 1000) - self._retention_ms
        messages = partition.read(partition.earliest_offset,
                                  partition.log_size)
        # Find the first message newer than cutoff
        cutoff_offset = partition.latest_offset
        for msg in messages:
            if msg.timestamp >= cutoff:
                cutoff_offset = msg.offset
                break

        return partition.truncate_before(cutoff_offset)

    def name(self) -> str:
        hours = self._retention_ms / (3600 * 1000)
        return f"TimeBased({hours:.0f}h)"


class SizeBasedRetention(RetentionPolicy):
    """
    Remove oldest messages when partition exceeds a size limit.
    Default: 1 GB.
    """

    def __init__(self, max_bytes: int = DEFAULT_RETENTION_BYTES) -> None:
        self._max_bytes = max_bytes

    def enforce(self, partition: Partition) -> int:
        return partition.truncate_to_size(self._max_bytes)

    def name(self) -> str:
        mb = self._max_bytes / (1024 * 1024)
        return f"SizeBased({mb:.0f}MB)"


class LogCompactionRetention(RetentionPolicy):
    """
    Keep only the latest message per key.
    Used for changelog topics where only the latest state matters.
    """

    def enforce(self, partition: Partition) -> int:
        return partition.compact()

    def name(self) -> str:
        return "LogCompaction"
```

`RetentionPolicy` strategies determine when and how messages are purged. `TimeBasedRetention` removes messages older than the configured duration — Kafka's default 7-day retention. `SizeBasedRetention` caps each partition's size by removing the oldest messages first — useful for resource-constrained environments. `LogCompactionRetention` keeps only the latest message per key — this is how Kafka's `__consumer_offsets` topic and changelog topics work. The partition already implements the low-level truncation and compaction operations; the strategies decide *when* and *how much* to purge.

---

## Notification System (Observer Pattern)

### Listener Interface

```python
class KafkaEventListener(ABC):
    """Observer interface for Kafka system events."""

    @abstractmethod
    def on_message_produced(self, metadata: RecordMetadata) -> None:
        ...

    @abstractmethod
    def on_message_consumed(self, consumer_id: str,
                            messages: list[Message]) -> None:
        ...

    @abstractmethod
    def on_offsets_committed(self, consumer_id: str,
                             offsets: dict[tuple[str, int], int]) -> None:
        ...

    @abstractmethod
    def on_topic_created(self, topic_name: str,
                         num_partitions: int) -> None:
        ...

    @abstractmethod
    def on_rebalance(self, group_id: str, generation: int,
                     assignments: dict[str, list[tuple[str, int]]]) -> None:
        ...

    @abstractmethod
    def on_retention_applied(self, topic: str, partition: int,
                              removed: int) -> None:
        ...
```

### Event Logger

```python
class KafkaEventLogger(KafkaEventListener):
    """Logs all Kafka events to console."""

    def __init__(self) -> None:
        self._log: list[str] = []

    def _log_event(self, entry: str) -> None:
        self._log.append(entry)
        print(entry)

    def on_message_produced(self, metadata: RecordMetadata) -> None:
        self._log_event(
            f"  📤 PRODUCED: {metadata.topic}-{metadata.partition} "
            f"offset={metadata.offset} size={metadata.size_bytes}B"
        )

    def on_message_consumed(self, consumer_id: str,
                            messages: list[Message]) -> None:
        if messages:
            topics = set(m.topic for m in messages)
            self._log_event(
                f"  📥 CONSUMED: {consumer_id} "
                f"read {len(messages)} msgs from "
                f"{', '.join(topics)}"
            )

    def on_offsets_committed(self, consumer_id: str,
                             offsets: dict[tuple[str, int], int]) -> None:
        offsets_str = ", ".join(
            f"{t}-{p}:{o}" for (t, p), o in offsets.items()
        )
        self._log_event(
            f"  ✅ COMMITTED: {consumer_id} [{offsets_str}]"
        )

    def on_topic_created(self, topic_name: str,
                         num_partitions: int) -> None:
        self._log_event(
            f"  📦 TOPIC CREATED: {topic_name} "
            f"({num_partitions} partitions)"
        )

    def on_rebalance(self, group_id: str, generation: int,
                     assignments: dict[str, list[tuple[str, int]]]) -> None:
        self._log_event(
            f"  ⚡ REBALANCE: group={group_id} gen={generation}"
        )
        for consumer_id, parts in assignments.items():
            parts_str = ", ".join(f"{t}-{p}" for t, p in parts)
            self._log_event(
                f"    {consumer_id} → [{parts_str}]"
            )

    def on_retention_applied(self, topic: str, partition: int,
                              removed: int) -> None:
        if removed > 0:
            self._log_event(
                f"  🗑 RETENTION: {topic}-{partition} "
                f"removed {removed} messages"
            )

    def print_log(self, last_n: int = 20) -> None:
        print(f"\n  Event Log (last {last_n}):")
        for entry in self._log[-last_n:]:
            print(entry)

    def clear(self) -> None:
        self._log.clear()
```

### Metrics Listener

```python
class MetricsCollector(KafkaEventListener):
    """Collects operational metrics from Kafka events."""

    def __init__(self) -> None:
        self._produced_count: int = 0
        self._produced_bytes: int = 0
        self._consumed_count: int = 0
        self._commits: int = 0
        self._rebalances: int = 0
        self._retention_removed: int = 0

    def on_message_produced(self, metadata: RecordMetadata) -> None:
        self._produced_count += 1
        self._produced_bytes += metadata.size_bytes

    def on_message_consumed(self, consumer_id: str,
                            messages: list[Message]) -> None:
        self._consumed_count += len(messages)

    def on_offsets_committed(self, consumer_id: str,
                             offsets: dict[tuple[str, int], int]) -> None:
        self._commits += 1

    def on_topic_created(self, topic_name: str,
                         num_partitions: int) -> None:
        pass

    def on_rebalance(self, group_id: str, generation: int,
                     assignments: dict[str, list[tuple[str, int]]]) -> None:
        self._rebalances += 1

    def on_retention_applied(self, topic: str, partition: int,
                              removed: int) -> None:
        self._retention_removed += removed

    def display(self) -> None:
        print(f"\n  {'═'*55}")
        print(f"  KAFKA METRICS")
        print(f"  {'═'*55}")
        print(f"  Messages produced:  {self._produced_count:,}")
        print(f"  Bytes produced:     {self._produced_bytes:,}")
        print(f"  Messages consumed:  {self._consumed_count:,}")
        print(f"  Offset commits:     {self._commits}")
        print(f"  Rebalances:         {self._rebalances}")
        print(f"  Retention removed:  {self._retention_removed}")
        print(f"  {'═'*55}\n")
```

`KafkaEventListener` defines hooks for six event types covering the full Kafka lifecycle. `KafkaEventLogger` provides an audit trail with emoji markers for quick visual scanning. `MetricsCollector` aggregates counters for operational monitoring — total produced/consumed counts, commit frequency, rebalance count, and retention purge totals. The observer pattern means adding an alerting system, a time-series exporter, or a dead-letter-queue handler requires zero changes to existing code.

---

## Metrics & Monitoring

```python
class KafkaMetrics:
    """
    Platform-wide metrics and health monitoring.
    """

    def __init__(self) -> None:
        self._produce_events: list[RecordMetadata] = []
        self._consume_events: list[tuple[str, int]] = []  # (consumer, count)

    def record_produce(self, metadata: RecordMetadata) -> None:
        self._produce_events.append(metadata)

    def record_consume(self, consumer_id: str, count: int) -> None:
        self._consume_events.append((consumer_id, count))

    @property
    def total_produced(self) -> int:
        return len(self._produce_events)

    @property
    def total_consumed(self) -> int:
        return sum(c for _, c in self._consume_events)

    @property
    def throughput_by_topic(self) -> dict[str, int]:
        counts: dict[str, int] = {}
        for meta in self._produce_events:
            counts[meta.topic] = counts.get(meta.topic, 0) + 1
        return counts

    @property
    def throughput_by_partition(self) -> dict[tuple[str, int], int]:
        counts: dict[tuple[str, int], int] = {}
        for meta in self._produce_events:
            key = (meta.topic, meta.partition)
            counts[key] = counts.get(key, 0) + 1
        return counts

    def display(self) -> None:
        print(f"\n  {'═'*55}")
        print(f"  KAFKA PLATFORM METRICS")
        print(f"  {'═'*55}")
        print(f"  Total produced:    {self.total_produced:,}")
        print(f"  Total consumed:    {self.total_consumed:,}")
        print(f"  {'─'*55}")
        print(f"  Throughput by topic:")
        for topic, count in self.throughput_by_topic.items():
            print(f"    {topic}: {count:,} messages")
        print(f"  {'─'*55}")
        print(f"  Throughput by partition:")
        for (topic, part), count in self.throughput_by_partition.items():
            print(f"    {topic}-{part}: {count:,} messages")
        print(f"  {'═'*55}\n")
```

`KafkaMetrics` maintains a detailed event log of all produce and consume operations. `throughput_by_topic` and `throughput_by_partition` provide breakdowns for identifying hot partitions — a common operational concern in real Kafka deployments. The separation of metrics from the core system keeps the monitoring concern cleanly isolated.

---

## Kafka System (Facade)

The `KafkaSystem` ties everything together. It manages brokers, topics, producers, consumers, consumer groups, retention, and notifications through a single unified API.

```python
class KafkaSystem:
    """
    Central Kafka platform.
    Acts as a Facade over brokers, topics, producers, consumers,
    consumer groups, retention, and notifications.
    """

    def __init__(self,
                 retention_policy: RetentionPolicy | None = None
                 ) -> None:
        self._cluster = Cluster()
        self._consumer_groups: dict[str, ConsumerGroup] = {}
        self._producers: dict[str, Producer] = {}
        self._retention = retention_policy or TimeBasedRetention()
        self._listeners: list[KafkaEventListener] = []
        self._metrics = KafkaMetrics()
        self._tick: int = 0

    # ------------------------------------------------------------------
    # Properties
    # ------------------------------------------------------------------

    @property
    def tick(self) -> int:
        return self._tick

    @property
    def cluster(self) -> Cluster:
        return self._cluster

    @property
    def metrics(self) -> KafkaMetrics:
        return self._metrics

    @property
    def topics(self) -> dict[str, Topic]:
        return self._cluster.all_topics

    # ------------------------------------------------------------------
    # Setup
    # ------------------------------------------------------------------

    def add_broker(self, broker: Broker) -> None:
        self._cluster.add_broker(broker)
        broker.start()

    def add_listener(self, listener: KafkaEventListener) -> None:
        self._listeners.append(listener)

    # ------------------------------------------------------------------
    # Topic management
    # ------------------------------------------------------------------

    def create_topic(self, name: str,
                     num_partitions: int = DEFAULT_NUM_PARTITIONS,
                     replication_factor: int = DEFAULT_REPLICATION_FACTOR
                     ) -> Topic | None:
        """Create a new topic distributed across the cluster."""
        self._tick += 1
        topic = self._cluster.create_topic(
            name, num_partitions, replication_factor
        )
        if topic:
            self._notify_topic_created(name, num_partitions)
        return topic

    def delete_topic(self, name: str) -> bool:
        return self._cluster.delete_topic(name)

    def get_topic(self, name: str) -> Topic | None:
        return self._cluster.get_topic(name)

    # ------------------------------------------------------------------
    # Producer management
    # ------------------------------------------------------------------

    def register_producer(self, producer: Producer) -> None:
        self._producers[producer.id] = producer

    def produce(self, producer_id: str,
                record: ProducerRecord) -> RecordMetadata | None:
        """
        Produce a message through a registered producer.

        Steps:
        1. Look up the producer.
        2. Delegate to producer.send() with the topic registry.
        3. Record metrics and notify listeners.
        """
        self._tick += 1
        producer = self._producers.get(producer_id)
        if producer is None:
            print(f"  ✗ Producer '{producer_id}' not registered")
            return None

        metadata = producer.send(record, self._cluster.all_topics)
        if metadata:
            self._metrics.record_produce(metadata)
            self._notify_produced(metadata)
        return metadata

    def produce_direct(self, record: ProducerRecord,
                       serializer: Serializer | None = None,
                       partitioner: PartitionStrategy | None = None
                       ) -> RecordMetadata | None:
        """Produce a message without a pre-registered producer."""
        self._tick += 1
        producer = Producer(serializer=serializer,
                            partitioner=partitioner)
        metadata = producer.send(record, self._cluster.all_topics)
        if metadata:
            self._metrics.record_produce(metadata)
            self._notify_produced(metadata)
        return metadata

    # ------------------------------------------------------------------
    # Consumer group management
    # ------------------------------------------------------------------

    def create_consumer_group(self, group_id: str,
                              strategy: RebalanceStrategy = RebalanceStrategy.RANGE
                              ) -> ConsumerGroup:
        """Create or return an existing consumer group."""
        if group_id not in self._consumer_groups:
            self._consumer_groups[group_id] = ConsumerGroup(
                group_id, strategy
            )
        return self._consumer_groups[group_id]

    def get_consumer_group(self, group_id: str) -> ConsumerGroup | None:
        return self._consumer_groups.get(group_id)

    def add_consumer_to_group(self, group_id: str,
                               consumer: Consumer) -> None:
        """Add a consumer to a group, subscribe to topics, and rebalance."""
        group = self._consumer_groups.get(group_id)
        if group is None:
            group = self.create_consumer_group(group_id)

        group.add_consumer(consumer)

    def subscribe_group(self, group_id: str, topic_name: str) -> None:
        """Subscribe a consumer group to a topic."""
        group = self._consumer_groups.get(group_id)
        if group:
            group.subscribe_to_topic(topic_name)

    def rebalance_group(self, group_id: str) -> None:
        """Trigger a rebalance for a consumer group."""
        group = self._consumer_groups.get(group_id)
        if group is None:
            print(f"  ✗ Consumer group '{group_id}' not found")
            return

        group.rebalance(self._cluster.all_topics)

        # Notify
        assignments = {
            c.id: c.assigned_partitions
            for c in group.active_consumers
        }
        self._notify_rebalance(group_id, group.generation, assignments)

    def remove_consumer_from_group(self, group_id: str,
                                    consumer_id: str) -> None:
        """Remove a consumer and trigger rebalance."""
        group = self._consumer_groups.get(group_id)
        if group:
            group.remove_consumer(consumer_id)
            self.rebalance_group(group_id)

    # ------------------------------------------------------------------
    # Consumer operations
    # ------------------------------------------------------------------

    def consume(self, consumer: Consumer,
                max_records: int = MAX_POLL_RECORDS) -> list[Message]:
        """
        Poll messages for a consumer from assigned partitions.
        """
        self._tick += 1
        messages = consumer.poll(self._cluster.all_topics, max_records)
        if messages:
            self._metrics.record_consume(consumer.id, len(messages))
            self._notify_consumed(consumer.id, messages)
        return messages

    def commit(self, consumer: Consumer,
               messages: list[Message]) -> None:
        """Commit offsets for consumed messages."""
        consumer.commit_offsets(messages)
        offsets = consumer.offset_manager.all_committed()
        self._notify_committed(consumer.id, offsets)

    # ------------------------------------------------------------------
    # Retention
    # ------------------------------------------------------------------

    def apply_retention(self) -> int:
        """
        Apply the retention policy to all partitions in all topics.
        Returns total messages removed.
        """
        total_removed = 0
        for topic_name, topic in self._cluster.all_topics.items():
            for partition in topic.partitions:
                removed = self._retention.enforce(partition)
                if removed > 0:
                    total_removed += removed
                    self._notify_retention(
                        topic_name, partition.partition_id, removed
                    )
        return total_removed

    # ------------------------------------------------------------------
    # Lag & health
    # ------------------------------------------------------------------

    def consumer_lag(self, group_id: str) -> dict[str, dict[tuple[str, int], int]]:
        """Get per-consumer lag for a group."""
        group = self._consumer_groups.get(group_id)
        if group is None:
            return {}
        result: dict[str, dict[tuple[str, int], int]] = {}
        for consumer in group.active_consumers:
            result[consumer.id] = consumer.compute_lag(
                self._cluster.all_topics
            )
        return result

    def total_lag(self, group_id: str) -> int:
        """Get total lag for a consumer group."""
        group = self._consumer_groups.get(group_id)
        if group is None:
            return 0
        return group.total_lag(self._cluster.all_topics)

    # ------------------------------------------------------------------
    # Notifications
    # ------------------------------------------------------------------

    def _notify_produced(self, metadata: RecordMetadata) -> None:
        for listener in self._listeners:
            listener.on_message_produced(metadata)

    def _notify_consumed(self, consumer_id: str,
                         messages: list[Message]) -> None:
        for listener in self._listeners:
            listener.on_message_consumed(consumer_id, messages)

    def _notify_committed(self, consumer_id: str,
                          offsets: dict[tuple[str, int], int]) -> None:
        for listener in self._listeners:
            listener.on_offsets_committed(consumer_id, offsets)

    def _notify_topic_created(self, topic_name: str,
                               num_partitions: int) -> None:
        for listener in self._listeners:
            listener.on_topic_created(topic_name, num_partitions)

    def _notify_rebalance(self, group_id: str, generation: int,
                          assignments: dict[str, list[tuple[str, int]]]
                          ) -> None:
        for listener in self._listeners:
            listener.on_rebalance(group_id, generation, assignments)

    def _notify_retention(self, topic: str, partition: int,
                          removed: int) -> None:
        for listener in self._listeners:
            listener.on_retention_applied(topic, partition, removed)

    # ------------------------------------------------------------------
    # Display
    # ------------------------------------------------------------------

    def display_status(self) -> None:
        print(f"\n  {'═'*60}")
        print(f"  KAFKA SYSTEM — Tick {self._tick}")
        print(f"  Retention: {self._retention.name()}")
        print(f"  {'═'*60}")

        print(f"\n  BROKERS:")
        print(f"  {'─'*55}")
        for broker in self._cluster.brokers:
            print(f"  {broker}")

        print(f"\n  TOPICS:")
        print(f"  {'─'*55}")
        for topic in self._cluster.all_topics.values():
            print(f"  {topic}")

        print(f"\n  PRODUCERS:")
        print(f"  {'─'*55}")
        for producer in self._producers.values():
            print(f"  {producer}")

        print(f"\n  CONSUMER GROUPS:")
        print(f"  {'─'*55}")
        for group in self._consumer_groups.values():
            print(f"  {group}")
            for consumer in group.consumers:
                lag = consumer.compute_lag(self._cluster.all_topics)
                total_lag = sum(lag.values()) if lag else 0
                print(f"    {consumer} lag={total_lag}")

        print()
```

`KafkaSystem` is the primary entry point. `produce` validates the producer, delegates to `producer.send()`, records metrics, and notifies listeners — a clean pipeline. `consume` and `commit` handle the consumer side. `rebalance_group` triggers reassignment and notifies listeners with the new assignments. `apply_retention` iterates over every partition in every topic and enforces the configured retention policy. `consumer_lag` and `total_lag` provide health monitoring. The facade keeps all subsystems (cluster, groups, retention, metrics, observers) coordinated without exposing their internal complexity.

---

## Simulation Engine

```python
from typing import Callable


class KafkaScenario:
    """A sequence of timed Kafka actions for simulation."""

    def __init__(self, system: KafkaSystem) -> None:
        self._system = system
        self._events: list[tuple[int, str, Callable]] = []

    def at_tick(self, tick: int, description: str,
                action: Callable) -> "KafkaScenario":
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
        print("  KAFKA SIMULATION START")
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
        print("  KAFKA SIMULATION END")
        print("═" * 62)
```

`KafkaScenario` lets you script events at specific ticks: "at tick 1, create a topic", "at tick 3, produce messages", "at tick 5, a consumer joins". `run` executes the simulation in order, firing events at the right ticks and displaying the system state at configurable intervals.

---

## Putting It All Together

### Basic Simulation

```python
def main() -> None:
    # Reset counters
    Producer._next_id = 0
    Consumer._next_id = 0
    Broker._next_id = 0

    # Create the system
    system = KafkaSystem(
        retention_policy=TimeBasedRetention()
    )

    # Attach observers
    logger = KafkaEventLogger()
    metrics_collector = MetricsCollector()
    system.add_listener(logger)
    system.add_listener(metrics_collector)

    # --- Add brokers ---
    broker1 = Broker("kafka-node-1", 9092)
    broker2 = Broker("kafka-node-2", 9093)
    system.add_broker(broker1)
    system.add_broker(broker2)

    # ╔══════════════════════════════════════════╗
    # ║           KAFKA SYSTEM                   ║
    # ╚══════════════════════════════════════════╝

    print("╔══════════════════════════════════════════╗")
    print("║           KAFKA SYSTEM                   ║")
    print("║  Brokers: 2                              ║")
    print("║  Retention: Time-Based (7 days)          ║")
    print("╚══════════════════════════════════════════╝")

    # --- Create topics ---
    print("\n  --- Creating topics ---")
    system.create_topic("orders", num_partitions=3)
    system.create_topic("payments", num_partitions=2)

    # --- Register producers ---
    order_producer = Producer(
        serializer=JsonSerializer(),
        partitioner=KeyHashPartitioner(),
        ack_mode=AckMode.ALL_REPLICAS,
    )
    payment_producer = Producer(
        serializer=JsonSerializer(),
        partitioner=KeyHashPartitioner(),
    )
    system.register_producer(order_producer)
    system.register_producer(payment_producer)

    # --- Produce order events ---
    print("\n  --- Producing order events ---")
    orders = [
        {"order_id": "ORD-001", "item": "Laptop", "amount": 999.99},
        {"order_id": "ORD-002", "item": "Phone", "amount": 699.99},
        {"order_id": "ORD-003", "item": "Tablet", "amount": 449.99},
        {"order_id": "ORD-004", "item": "Watch", "amount": 299.99},
        {"order_id": "ORD-005", "item": "Headphones", "amount": 149.99},
    ]

    for order in orders:
        metadata = system.produce(
            order_producer.id,
            ProducerRecord(
                topic="orders",
                key=order["order_id"],
                value=order,
            )
        )

    # --- Produce payment events ---
    print("\n  --- Producing payment events ---")
    payments = [
        {"payment_id": "PAY-001", "order_id": "ORD-001", "status": "completed"},
        {"payment_id": "PAY-002", "order_id": "ORD-002", "status": "pending"},
        {"payment_id": "PAY-003", "order_id": "ORD-003", "status": "completed"},
    ]

    for payment in payments:
        system.produce(
            payment_producer.id,
            ProducerRecord(
                topic="payments",
                key=payment["order_id"],
                value=payment,
            )
        )

    # --- Create consumer group ---
    print("\n  --- Setting up consumer group ---")
    group = system.create_consumer_group(
        "order-processors", RebalanceStrategy.ROUND_ROBIN
    )
    system.subscribe_group("order-processors", "orders")

    consumer1 = Consumer("order-processors",
                         deserializer=JsonSerializer(),
                         offset_reset=OffsetResetPolicy.EARLIEST)
    consumer2 = Consumer("order-processors",
                         deserializer=JsonSerializer(),
                         offset_reset=OffsetResetPolicy.EARLIEST)

    system.add_consumer_to_group("order-processors", consumer1)
    system.add_consumer_to_group("order-processors", consumer2)
    system.rebalance_group("order-processors")

    # --- Consume messages ---
    print("\n  --- Consuming messages ---")
    for consumer in [consumer1, consumer2]:
        messages = system.consume(consumer)
        if messages:
            print(f"\n  {consumer.id} received {len(messages)} messages:")
            for msg in messages:
                print(f"    {msg}")
            system.commit(consumer, messages)

    # --- Check lag ---
    print(f"\n  --- Consumer lag ---")
    lag = system.consumer_lag("order-processors")
    for consumer_id, partitions in lag.items():
        for (topic, part), lag_value in partitions.items():
            print(f"  {consumer_id}: {topic}-{part} lag={lag_value}")

    total_lag = system.total_lag("order-processors")
    print(f"  Total group lag: {total_lag}")

    # --- Display final state ---
    system.display_status()
    metrics_collector.display()


if __name__ == "__main__":
    main()
```

### Scripted Scenario

```python
def main_scenario() -> None:
    Producer._next_id = 0
    Consumer._next_id = 0
    Broker._next_id = 0

    system = KafkaSystem()
    logger = KafkaEventLogger()
    system.add_listener(logger)

    # Setup brokers
    system.add_broker(Broker("node-1", 9092))
    system.add_broker(Broker("node-2", 9093))

    scenario = KafkaScenario(system)

    # Create topics
    scenario.at_tick(1, "Create 'user-events' topic",
                     lambda: system.create_topic("user-events", 4))

    scenario.at_tick(1, "Create 'notifications' topic",
                     lambda: system.create_topic("notifications", 2))

    # Register producer
    producer = Producer(
        serializer=JsonSerializer(),
        partitioner=KeyHashPartitioner(),
    )
    scenario.at_tick(2, "Register producer",
                     lambda: system.register_producer(producer))

    # Produce user events
    for i in range(1, 9):
        user_id = f"user-{(i % 3) + 1}"
        event = {"user": user_id, "action": f"action-{i}",
                 "timestamp": int(time.time() * 1000)}
        scenario.at_tick(3, f"Produce event for {user_id}",
                         lambda e=event, u=user_id: system.produce(
                             producer.id,
                             ProducerRecord(
                                 topic="user-events",
                                 key=u,
                                 value=e)))

    # Setup consumer group with 2 consumers
    c1 = Consumer("analytics-group",
                  deserializer=JsonSerializer(),
                  offset_reset=OffsetResetPolicy.EARLIEST)
    c2 = Consumer("analytics-group",
                  deserializer=JsonSerializer(),
                  offset_reset=OffsetResetPolicy.EARLIEST)

    scenario.at_tick(4, "Create consumer group and subscribe",
                     lambda: (
                         system.create_consumer_group(
                             "analytics-group",
                             RebalanceStrategy.ROUND_ROBIN),
                         system.subscribe_group(
                             "analytics-group", "user-events")))

    scenario.at_tick(5, "Add consumer-1",
                     lambda: system.add_consumer_to_group(
                         "analytics-group", c1))

    scenario.at_tick(5, "Add consumer-2",
                     lambda: system.add_consumer_to_group(
                         "analytics-group", c2))

    scenario.at_tick(6, "Rebalance group",
                     lambda: system.rebalance_group("analytics-group"))

    # Consumers poll
    scenario.at_tick(7, "Consumer-1 polls",
                     lambda: (
                         msgs := system.consume(c1),
                         print(f"    consumer-1 got {len(msgs)} msgs"),
                         system.commit(c1, msgs) if msgs else None))

    scenario.at_tick(7, "Consumer-2 polls",
                     lambda: (
                         msgs := system.consume(c2),
                         print(f"    consumer-2 got {len(msgs)} msgs"),
                         system.commit(c2, msgs) if msgs else None))

    # Add a third consumer (triggers rebalance)
    c3 = Consumer("analytics-group",
                  deserializer=JsonSerializer(),
                  offset_reset=OffsetResetPolicy.EARLIEST)

    scenario.at_tick(8, "Add consumer-3 (triggers rebalance)",
                     lambda: (
                         system.add_consumer_to_group(
                             "analytics-group", c3),
                         system.rebalance_group("analytics-group")))

    # Check lag after rebalance
    scenario.at_tick(9, "Check group lag",
                     lambda: print(
                         f"    Total lag: "
                         f"{system.total_lag('analytics-group')}"))

    # Remove a consumer (triggers rebalance)
    scenario.at_tick(10, "Remove consumer-2 (triggers rebalance)",
                     lambda: system.remove_consumer_from_group(
                         "analytics-group", c2.id))

    scenario.run(display_interval=5)


if __name__ == "__main__":
    main_scenario()
```

### Strategy Comparison

```python
def compare_strategies() -> None:
    """Compare partitioning strategies on the same workload."""

    print("\n" + "=" * 55)
    print("  PARTITIONING STRATEGY COMPARISON")
    print("=" * 55)

    strategies: list[PartitionStrategy] = [
        RoundRobinPartitioner(),
        KeyHashPartitioner(),
    ]

    for strategy in strategies:
        Producer._next_id = 0
        Broker._next_id = 0

        system = KafkaSystem()
        system.add_broker(Broker("node-1", 9092))
        system.create_topic("test-topic", num_partitions=4)

        producer = Producer(
            serializer=StringSerializer(),
            partitioner=strategy,
        )
        system.register_producer(producer)

        # Produce 12 messages with 3 distinct keys
        for i in range(12):
            key = f"key-{i % 3}"
            system.produce(
                producer.id,
                ProducerRecord(
                    topic="test-topic",
                    key=key,
                    value=f"message-{i}",
                )
            )

        topic = system.get_topic("test-topic")
        print(f"\n  Strategy: {strategy.name()}")
        print(f"  {'─'*45}")
        for p in topic.partitions:
            keys = set(m.key for m in p.read(0, 100) if m.key)
            print(f"    Partition {p.partition_id}: "
                  f"{p.log_size} msgs, "
                  f"keys={keys or 'mixed'}")

    # --- Retention strategy comparison ---
    print("\n" + "=" * 55)
    print("  RETENTION STRATEGY COMPARISON")
    print("=" * 55)

    retention_strategies: list[RetentionPolicy] = [
        SizeBasedRetention(max_bytes=100),  # very small for demo
        LogCompactionRetention(),
    ]

    for retention in retention_strategies:
        Broker._next_id = 0
        Producer._next_id = 0

        system = KafkaSystem(retention_policy=retention)
        system.add_broker(Broker("node-1", 9092))
        system.create_topic("compact-topic", num_partitions=1)

        producer = Producer(serializer=StringSerializer(),
                            partitioner=RoundRobinPartitioner())
        system.register_producer(producer)

        # Produce messages: multiple updates per key
        for key in ["user-1", "user-2", "user-1", "user-3",
                     "user-2", "user-1"]:
            system.produce(
                producer.id,
                ProducerRecord(
                    topic="compact-topic",
                    key=key,
                    value=f"latest-value-for-{key}",
                )
            )

        topic = system.get_topic("compact-topic")
        partition = topic.get_partition(0)
        before = partition.log_size

        removed = system.apply_retention()

        print(f"\n  Retention: {retention.name()}")
        print(f"    Before: {before} messages")
        print(f"    Removed: {removed} messages")
        print(f"    After: {partition.log_size} messages")
        remaining = partition.read(partition.earliest_offset, 100)
        for msg in remaining:
            print(f"      offset={msg.offset} key={msg.key} "
                  f"val={msg.value.decode()}")

    print("\n" + "=" * 55)
```

This comparison highlights strategy differences. For partitioning: `RoundRobinPartitioner` distributes messages evenly regardless of key — partition sizes stay balanced but same-key messages scatter across partitions. `KeyHashPartitioner` groups same-key messages in the same partition — partition sizes may be uneven but per-key ordering is guaranteed. For retention: `SizeBasedRetention` removes the oldest messages when the partition exceeds the byte limit. `LogCompactionRetention` removes older entries for duplicate keys, keeping only the latest value per key.

### Unit Tests

```python
def test_produce_and_consume() -> None:
    """Test basic produce-consume flow."""
    Producer._next_id = 0
    Consumer._next_id = 0
    Broker._next_id = 0

    system = KafkaSystem()
    system.add_broker(Broker("node", 9092))
    system.create_topic("test", num_partitions=2)

    producer = Producer(serializer=StringSerializer(),
                        partitioner=RoundRobinPartitioner())
    system.register_producer(producer)

    for i in range(4):
        system.produce(
            producer.id,
            ProducerRecord(topic="test", value=f"msg-{i}")
        )

    consumer = Consumer("test-group",
                        offset_reset=OffsetResetPolicy.EARLIEST)
    system.add_consumer_to_group("test-group", consumer)
    system.subscribe_group("test-group", "test")
    system.rebalance_group("test-group")

    messages = system.consume(consumer)
    assert len(messages) == 4
    print("✅ test_produce_and_consume passed")


def test_key_based_partitioning() -> None:
    """Test that same-key messages go to the same partition."""
    Producer._next_id = 0
    Broker._next_id = 0

    system = KafkaSystem()
    system.add_broker(Broker("node", 9092))
    system.create_topic("keyed", num_partitions=4)

    producer = Producer(serializer=StringSerializer(),
                        partitioner=KeyHashPartitioner())
    system.register_producer(producer)

    # Produce 3 messages with the same key
    partitions_used = set()
    for i in range(3):
        meta = system.produce(
            producer.id,
            ProducerRecord(
                topic="keyed",
                key="same-key",
                value=f"val-{i}",
            )
        )
        partitions_used.add(meta.partition)

    # All should go to the same partition
    assert len(partitions_used) == 1
    print("✅ test_key_based_partitioning passed")


def test_consumer_group_rebalance() -> None:
    """Test that rebalance distributes partitions evenly."""
    Producer._next_id = 0
    Consumer._next_id = 0
    Broker._next_id = 0

    system = KafkaSystem()
    system.add_broker(Broker("node", 9092))
    system.create_topic("rebal", num_partitions=4)

    system.create_consumer_group("grp", RebalanceStrategy.ROUND_ROBIN)
    system.subscribe_group("grp", "rebal")

    c1 = Consumer("grp")
    c2 = Consumer("grp")
    system.add_consumer_to_group("grp", c1)
    system.add_consumer_to_group("grp", c2)
    system.rebalance_group("grp")

    # 4 partitions, 2 consumers => 2 each
    assert len(c1.assigned_partitions) == 2
    assert len(c2.assigned_partitions) == 2

    # No overlap
    all_assigned = set(c1.assigned_partitions + c2.assigned_partitions)
    assert len(all_assigned) == 4
    print("✅ test_consumer_group_rebalance passed")


def test_offset_commit_and_resume() -> None:
    """Test that consumers resume from committed offset."""
    Producer._next_id = 0
    Consumer._next_id = 0
    Broker._next_id = 0

    system = KafkaSystem()
    system.add_broker(Broker("node", 9092))
    system.create_topic("offsets", num_partitions=1)

    producer = Producer(serializer=StringSerializer(),
                        partitioner=RoundRobinPartitioner())
    system.register_producer(producer)

    # Produce 5 messages
    for i in range(5):
        system.produce(
            producer.id,
            ProducerRecord(topic="offsets", value=f"msg-{i}")
        )

    consumer = Consumer("grp",
                        offset_reset=OffsetResetPolicy.EARLIEST)
    system.add_consumer_to_group("grp", consumer)
    system.subscribe_group("grp", "offsets")
    system.rebalance_group("grp")

    # First poll: get all 5
    msgs = system.consume(consumer)
    assert len(msgs) == 5
    # Commit only first 3 (offset 0,1,2 -> commit 3)
    system.commit(consumer, msgs[:3])

    # Second poll: should get 2 remaining (offsets 3,4)
    msgs2 = system.consume(consumer)
    assert len(msgs2) == 2
    assert msgs2[0].offset == 3
    assert msgs2[1].offset == 4
    print("✅ test_offset_commit_and_resume passed")


def test_consumer_lag() -> None:
    """Test consumer lag calculation."""
    Producer._next_id = 0
    Consumer._next_id = 0
    Broker._next_id = 0

    system = KafkaSystem()
    system.add_broker(Broker("node", 9092))
    system.create_topic("lag-test", num_partitions=1)

    producer = Producer(serializer=StringSerializer(),
                        partitioner=RoundRobinPartitioner())
    system.register_producer(producer)

    for i in range(10):
        system.produce(
            producer.id,
            ProducerRecord(topic="lag-test", value=f"msg-{i}")
        )

    consumer = Consumer("grp",
                        offset_reset=OffsetResetPolicy.EARLIEST)
    system.add_consumer_to_group("grp", consumer)
    system.subscribe_group("grp", "lag-test")
    system.rebalance_group("grp")

    # Before consuming: lag = 10
    assert system.total_lag("grp") == 10

    # Consume and commit all
    msgs = system.consume(consumer)
    system.commit(consumer, msgs)

    # After: lag = 0
    assert system.total_lag("grp") == 0
    print("✅ test_consumer_lag passed")


def test_log_compaction() -> None:
    """Test that log compaction keeps only latest per key."""
    Broker._next_id = 0
    Producer._next_id = 0

    system = KafkaSystem(
        retention_policy=LogCompactionRetention()
    )
    system.add_broker(Broker("node", 9092))
    system.create_topic("compact", num_partitions=1)

    producer = Producer(serializer=StringSerializer(),
                        partitioner=RoundRobinPartitioner())
    system.register_producer(producer)

    # Produce: key-A x3, key-B x2
    for key, val in [("A", "v1"), ("B", "v1"), ("A", "v2"),
                     ("B", "v2"), ("A", "v3")]:
        system.produce(
            producer.id,
            ProducerRecord(topic="compact", key=key, value=val)
        )

    topic = system.get_topic("compact")
    assert topic.get_partition(0).log_size == 5

    system.apply_retention()

    # After compaction: 2 messages (latest A and latest B)
    partition = topic.get_partition(0)
    assert partition.log_size == 2

    msgs = partition.read(partition.earliest_offset, 10)
    keys = {m.key: m.value.decode() for m in msgs}
    assert keys["A"] == "v3"
    assert keys["B"] == "v2"
    print("✅ test_log_compaction passed")


def test_rebalance_on_consumer_leave() -> None:
    """Test that partitions are reassigned when a consumer leaves."""
    Producer._next_id = 0
    Consumer._next_id = 0
    Broker._next_id = 0

    system = KafkaSystem()
    system.add_broker(Broker("node", 9092))
    system.create_topic("rebal2", num_partitions=3)

    system.create_consumer_group("grp2", RebalanceStrategy.RANGE)
    system.subscribe_group("grp2", "rebal2")

    c1 = Consumer("grp2")
    c2 = Consumer("grp2")
    c3 = Consumer("grp2")
    system.add_consumer_to_group("grp2", c1)
    system.add_consumer_to_group("grp2", c2)
    system.add_consumer_to_group("grp2", c3)
    system.rebalance_group("grp2")

    # 3 partitions, 3 consumers => 1 each
    assert len(c1.assigned_partitions) == 1
    assert len(c2.assigned_partitions) == 1
    assert len(c3.assigned_partitions) == 1

    # Remove consumer-2
    system.remove_consumer_from_group("grp2", c2.id)

    # 3 partitions, 2 consumers
    group = system.get_consumer_group("grp2")
    active = group.active_consumers
    total_assigned = sum(
        len(c.assigned_partitions) for c in active
    )
    assert total_assigned == 3  # all partitions reassigned
    print("✅ test_rebalance_on_consumer_leave passed")


def test_multiple_topics() -> None:
    """Test consuming from multiple topics."""
    Producer._next_id = 0
    Consumer._next_id = 0
    Broker._next_id = 0

    system = KafkaSystem()
    system.add_broker(Broker("node", 9092))
    system.create_topic("topic-a", num_partitions=1)
    system.create_topic("topic-b", num_partitions=1)

    producer = Producer(serializer=StringSerializer(),
                        partitioner=RoundRobinPartitioner())
    system.register_producer(producer)

    system.produce(producer.id,
                   ProducerRecord(topic="topic-a", value="a-msg"))
    system.produce(producer.id,
                   ProducerRecord(topic="topic-b", value="b-msg"))

    consumer = Consumer("multi-grp",
                        offset_reset=OffsetResetPolicy.EARLIEST)
    system.add_consumer_to_group("multi-grp", consumer)
    system.subscribe_group("multi-grp", "topic-a")
    system.subscribe_group("multi-grp", "topic-b")
    system.rebalance_group("multi-grp")

    msgs = system.consume(consumer)
    topics_consumed = set(m.topic for m in msgs)
    assert "topic-a" in topics_consumed
    assert "topic-b" in topics_consumed
    print("✅ test_multiple_topics passed")


def test_round_robin_partitioner() -> None:
    """Test that round-robin distributes evenly."""
    Producer._next_id = 0
    Broker._next_id = 0

    system = KafkaSystem()
    system.add_broker(Broker("node", 9092))
    system.create_topic("rr-test", num_partitions=3)

    producer = Producer(serializer=StringSerializer(),
                        partitioner=RoundRobinPartitioner())
    system.register_producer(producer)

    partitions_hit: list[int] = []
    for i in range(9):
        meta = system.produce(
            producer.id,
            ProducerRecord(topic="rr-test", value=f"msg-{i}")
        )
        partitions_hit.append(meta.partition)

    # Each partition should get exactly 3 messages
    for p in range(3):
        assert partitions_hit.count(p) == 3
    print("✅ test_round_robin_partitioner passed")


def run_all_tests() -> None:
    print("\n" + "=" * 40)
    print("  RUNNING TESTS")
    print("=" * 40 + "\n")
    test_produce_and_consume()
    test_key_based_partitioning()
    test_consumer_group_rebalance()
    test_offset_commit_and_resume()
    test_consumer_lag()
    test_log_compaction()
    test_rebalance_on_consumer_leave()
    test_multiple_topics()
    test_round_robin_partitioner()
    print("\n✅ All tests passed!\n")
```

Each test resets the ID counters for isolation. `test_produce_and_consume` verifies the basic pipeline end-to-end. `test_key_based_partitioning` confirms same-key messages land in the same partition. `test_consumer_group_rebalance` checks that 4 partitions distribute evenly across 2 consumers with no overlap. `test_offset_commit_and_resume` validates that after committing offset 3, the next poll starts from offset 3. `test_consumer_lag` verifies lag drops to zero after consuming and committing everything. `test_log_compaction` produces 5 messages for 2 keys and confirms compaction reduces to 2 messages with the correct latest values. `test_rebalance_on_consumer_leave` ensures all partitions get reassigned when a consumer drops. `test_multiple_topics` confirms a consumer group can consume from multiple topics simultaneously.

---

## Class Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                       KafkaSystem                                 │
│──────────────────────────────────────────────────────────────────│
│ - _cluster: Cluster                                               │
│ - _consumer_groups: dict[str, ConsumerGroup]                      │
│ - _producers: dict[str, Producer]                                 │
│ - _retention: RetentionPolicy                                     │
│ - _listeners: list[KafkaEventListener]                            │
│ - _metrics: KafkaMetrics                                          │
│ - _tick: int                                                      │
│──────────────────────────────────────────────────────────────────│
│ + create_topic(name, partitions, repl) -> Topic                   │
│ + produce(producer_id, record) -> RecordMetadata                  │
│ + consume(consumer, max_records) -> list[Message]                 │
│ + commit(consumer, messages) -> None                              │
│ + create_consumer_group(id, strategy) -> ConsumerGroup            │
│ + rebalance_group(group_id) -> None                               │
│ + apply_retention() -> int                                        │
│ + consumer_lag(group_id) -> dict                                  │
│ + display_status() -> None                                        │
└──────┬──────────┬──────────┬──────────┬───────────┬──────────────┘
       │          │          │          │           │
  uses ▼     uses ▼     uses ▼    uses ▼      uses ▼
┌──────────────────────────────────────────────────────────────────┐
│  Cluster                                                          │
│──────────────────────────────────────────────────────────────────│
│ - _brokers: dict[str, Broker]                                     │
│ - _all_topics: dict[str, Topic]                                   │
│ - _topic_broker_map: dict[str, str]                               │
│──────────────────────────────────────────────────────────────────│
│ + add_broker(broker) -> None                                      │
│ + create_topic(name, partitions, repl) -> Topic                   │
│ + all_topics -> dict[str, Topic]                                  │
└──────────┬──────────────────────────────────────────────────────┘
           │ contains
           ▼
┌──────────────────────────────────────────────────────────────────┐
│  Broker                                                           │
│──────────────────────────────────────────────────────────────────│
│ - _id: str                                                        │
│ - _host: str                                                      │
│ - _port: int                                                      │
│ - _topics: dict[str, Topic]                                       │
│ - _is_alive: bool                                                 │
│──────────────────────────────────────────────────────────────────│
│ + create_topic(name, partitions) -> Topic                         │
│ + shutdown() / start() -> None                                    │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  Topic                                                            │
│──────────────────────────────────────────────────────────────────│
│ - _name: str                                                      │
│ - _partitions: list[Partition]                                    │
│ - _replication_factor: int                                        │
│──────────────────────────────────────────────────────────────────│
│ + get_partition(id) -> Partition                                   │
│ + num_partitions -> int                                            │
│ + total_messages -> int                                            │
└──────────┬──────────────────────────────────────────────────────┘
           │ contains
           ▼
┌──────────────────────────────────────────────────────────────────┐
│  Partition                                                        │
│──────────────────────────────────────────────────────────────────│
│ - _topic: str                                                     │
│ - _partition_id: int                                              │
│ - _log: list[Message]                                             │
│ - _next_offset: int                                               │
│ - _total_bytes: int                                               │
│ - _lock: RLock                                                    │
│──────────────────────────────────────────────────────────────────│
│ + append(message) -> int (offset)                                 │
│ + read(offset, max) -> list[Message]                              │
│ + truncate_before(offset) -> int                                  │
│ + truncate_to_size(bytes) -> int                                  │
│ + compact() -> int                                                │
│ + earliest_offset / latest_offset -> int                          │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  «abstract» PartitionStrategy                                     │
│──────────────────────────────────────────────────────────────────│
│ + select_partition(key, num_partitions) -> int                     │
│ + name() -> str                                                   │
└──────────┬────────────────────────────────────────────────────────┘
           │ implemented by
  ┌────────┼──────────────────┐
  ▼        ▼                  ▼
RoundRobin KeyHash          Custom
Partitioner Partitioner     Partitioner

┌──────────────────────────────────────────────────────────────────┐
│  «abstract» RetentionPolicy                                       │
│──────────────────────────────────────────────────────────────────│
│ + enforce(partition) -> int                                        │
│ + name() -> str                                                   │
└──────────┬────────────────────────────────────────────────────────┘
           │ implemented by
  ┌────────┼──────────────────┐
  ▼        ▼                  ▼
TimeBased  SizeBased        LogCompaction
Retention  Retention        Retention

┌──────────────────────────────────────────────────────────────────┐
│  «abstract» Serializer                                            │
│──────────────────────────────────────────────────────────────────│
│ + serialize(data) -> bytes                                         │
│ + deserialize(raw) -> object                                       │
│ + name() -> str                                                   │
└──────────┬────────────────────────────────────────────────────────┘
           │ implemented by
  ┌────────┴──────────┐
  ▼                   ▼
String             Json
Serializer         Serializer

┌──────────────────────────────────────────────────────────────────┐
│  «abstract» KafkaEventListener                                    │
│──────────────────────────────────────────────────────────────────│
│ + on_message_produced(metadata)                                    │
│ + on_message_consumed(consumer_id, messages)                       │
│ + on_offsets_committed(consumer_id, offsets)                        │
│ + on_topic_created(topic_name, num_partitions)                     │
│ + on_rebalance(group_id, generation, assignments)                  │
│ + on_retention_applied(topic, partition, removed)                  │
└──────────┬──────────────────────────────────────────────────────┘
           │ implemented by
  ┌────────┴──────────┐
  ▼                   ▼
KafkaEvent         Metrics
Logger             Collector

┌──────────────┐ ┌──────────────────┐
│  Producer    │ │  Consumer         │
│──────────────│ │──────────────────│
│ - _id        │ │ - _id             │
│ - _serializer│ │ - _group_id       │
│ - _partitioner││ - _deserializer   │
│ - _ack_mode  │ │ - _offset_reset   │
│ - _msgs_sent │ │ - _offset_manager │
│ - _bytes_sent│ │ - _assigned_parts │
│──────────────│ │ - _state          │
│ + send()     │ │ - _msgs_consumed  │
│ + close()    │ │──────────────────│
└──────────────┘ │ + subscribe()     │
                 │ + poll()          │
┌──────────────┐ │ + commit_offsets()│
│ Message      │ │ + compute_lag()   │
│──────────────│ │ + pause/resume()  │
│ + key        │ │ + close()         │
│ + value      │ └──────────────────┘
│ + headers    │
│ + timestamp  │ ┌──────────────────┐
│ + topic      │ │  ConsumerGroup    │
│ + partition  │ │──────────────────│
│ + offset     │ │ - _group_id       │
│ + size_bytes │ │ - _strategy       │
└──────────────┘ │ - _consumers      │
                 │ - _subscriptions  │
┌──────────────┐ │ - _generation     │
│ ProducerRecord││──────────────────│
│──────────────│ │ + add_consumer()  │
│ + topic      │ │ + remove_consumer │
│ + value      │ │ + rebalance()     │
│ + key        │ │ + total_lag()     │
│ + headers    │ └──────────────────┘
│ + partition  │
└──────────────┘ ┌──────────────────┐
                 │ OffsetManager     │
┌──────────────┐ │──────────────────│
│ RecordMetadata││ - _committed      │
│──────────────│ │──────────────────│
│ + topic      │ │ + commit()        │
│ + partition  │ │ + get_committed() │
│ + offset     │ │ + reset()         │
│ + timestamp  │ └──────────────────┘
│ + size_bytes │
└──────────────┘ ┌──────────────────┐
                 │ KafkaMetrics      │
┌──────────────┐ │──────────────────│
│ MessageHeader│ │ - _produce_events │
│──────────────│ │ - _consume_events │
│ + key        │ │──────────────────│
│ + value      │ │ + throughput_by_  │
└──────────────┘ │   topic/partition │
                 │ + display()       │
                 └──────────────────┘

┌────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│ AckMode        │ │ OffsetResetPolicy│ │ ConsumerState     │
│ (Enum)         │ │ (Enum)           │ │ (Enum)            │
│────────────────│ │──────────────────│ │──────────────────│
│ FIRE_AND_FORGET│ │ EARLIEST         │ │ IDLE              │
│ LEADER_ONLY    │ │ LATEST           │ │ SUBSCRIBED        │
│ ALL_REPLICAS   │ └──────────────────┘ │ CONSUMING         │
└────────────────┘                      │ PAUSED            │
                 ┌──────────────────┐   │ CLOSED            │
                 │ RebalanceStrategy│   └──────────────────┘
                 │ (Enum)           │
                 │──────────────────│   ┌──────────────────┐
                 │ RANGE            │   │ RetentionType     │
                 │ ROUND_ROBIN      │   │ (Enum)            │
                 └──────────────────┘   │──────────────────│
                                        │ TIME_BASED        │
                 ┌──────────────────┐   │ SIZE_BASED        │
                 │ KafkaScenario    │   │ LOG_COMPACTION     │
                 │──────────────────│   └──────────────────┘
                 │ - _system        │
                 │ - _events        │
                 │──────────────────│
                 │ + at_tick()       │
                 │ + run()           │
                 └──────────────────┘
```

---

## How to Run

Save all code into a single file `kafka.py` and run:

```bash
python kafka.py
```

### Example Output

```
╔══════════════════════════════════════════╗
║           KAFKA SYSTEM                   ║
║  Brokers: 2                              ║
║  Retention: Time-Based (7 days)          ║
╚══════════════════════════════════════════╝

  --- Creating topics ---
  🟢 Broker broker-1 started at kafka-node-1:9092
  🟢 Broker broker-2 started at kafka-node-2:9093
  📦 Topic 'orders' created on broker-1 (3 partitions)
  📦 TOPIC CREATED: orders (3 partitions)
  📦 Topic 'payments' created on broker-2 (2 partitions)
  📦 TOPIC CREATED: payments (2 partitions)

  --- Producing order events ---
  📤 PRODUCED: orders-1 offset=0 size=54B
  📤 PRODUCED: orders-2 offset=0 size=53B
  📤 PRODUCED: orders-0 offset=0 size=55B
  📤 PRODUCED: orders-1 offset=1 size=54B
  📤 PRODUCED: orders-0 offset=1 size=58B

  --- Producing payment events ---
  📤 PRODUCED: payments-0 offset=0 size=64B
  📤 PRODUCED: payments-1 offset=0 size=62B
  📤 PRODUCED: payments-0 offset=1 size=64B

  --- Setting up consumer group ---
  ⚡ Rebalance complete — group=order-processors gen=1 consumers=2 partitions=3
  ⚡ REBALANCE: group=order-processors gen=1
    consumer-1 → [orders-0, orders-2]
    consumer-2 → [orders-1]

  --- Consuming messages ---

  consumer-1 received 3 messages:
    Msg(key=ORD-003 partition=0 offset=0 ...)
    Msg(key=ORD-005 partition=0 offset=1 ...)
    Msg(key=ORD-002 partition=2 offset=0 ...)
  📥 CONSUMED: consumer-1 read 3 msgs from orders
  ✅ COMMITTED: consumer-1 [orders-0:2, orders-2:1]

  consumer-2 received 2 messages:
    Msg(key=ORD-001 partition=1 offset=0 ...)
    Msg(key=ORD-004 partition=1 offset=1 ...)
  📥 CONSUMED: consumer-2 read 2 msgs from orders
  ✅ COMMITTED: consumer-2 [orders-1:2]

  --- Consumer lag ---
  consumer-1: orders-0 lag=0
  consumer-1: orders-2 lag=0
  consumer-2: orders-1 lag=0
  Total group lag: 0

  ═══════════════════════════════════════════════════════════
  KAFKA METRICS
  ═══════════════════════════════════════════════════════════
  Messages produced:  8
  Bytes produced:     464
  Messages consumed:  5
  Offset commits:     2
  Rebalances:         1
  Retention removed:  0
  ═══════════════════════════════════════════════════════════
```
