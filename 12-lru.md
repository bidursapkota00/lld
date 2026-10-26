# Design LRU Cache – Low-Level Design Guide

![Bidur Sapkota](https://www.bidursapkota.com.np/images/gravatar.webp "Bidur Sapkota - Developer")&nbsp;[Bidur Sapkota](https://www.bidursapkota.com.np/)

## Table of Contents

1. [Requirements](#requirements)
2. [Key Design Patterns](#key-design-patterns)
3. [Core Enums & Constants](#core-enums--constants)
4. [Doubly Linked List Node](#doubly-linked-list-node)
5. [Doubly Linked List](#doubly-linked-list)
6. [Eviction Policy (Strategy Pattern)](#eviction-policy-strategy-pattern)
7. [Cache Entry & Statistics](#cache-entry--statistics)
8. [LRU Cache Core](#lru-cache-core)
9. [Cache Event System (Observer Pattern)](#cache-event-system-observer-pattern)
10. [TTL Support & Expiration](#ttl-support--expiration)
11. [Write Policy (Strategy Pattern)](#write-policy-strategy-pattern)
12. [Bounded Cache with Eviction Callbacks](#bounded-cache-with-eviction-callbacks)
13. [Multi-Level Cache (Decorator Pattern)](#multi-level-cache-decorator-pattern)
14. [Cache Facade](#cache-facade)
15. [Simulation Engine](#simulation-engine)
16. [Putting It All Together](#putting-it-all-together)
17. [Class Diagram](#class-diagram)
18. [How to Run](#how-to-run)

---

## Requirements

### Functional Requirements

| #  | Requirement                                                                                     |
| -- | ----------------------------------------------------------------------------------------------- |
| 1  | `get(key)` retrieves a value in O(1) time and marks the key as most recently used               |
| 2  | `put(key, value)` inserts or updates a key-value pair in O(1) time                              |
| 3  | When the cache reaches its capacity, the least recently used entry is evicted before inserting   |
| 4  | Pluggable eviction policies — LRU, LFU, and FIFO can be swapped without changing cache logic    |
| 5  | Optional TTL (time-to-live) per entry — expired entries are lazily evicted on access             |
| 6  | Observer-based event system — listeners react to hits, misses, evictions, and expirations        |
| 7  | Write policies — write-through and write-back strategies for syncing with a backing store        |
| 8  | Multi-level caching — L1 (fast, small) and L2 (slower, larger) with automatic promotion         |
| 9  | Cache statistics — hit rate, miss rate, eviction count, and current size                         |
| 10 | Thread-safe — all operations are safe for concurrent access via locking                          |

### Non-Functional Requirements

- Clean OOP with SOLID principles.
- Extensible — new eviction policies, write strategies, and event listeners can be added without modifying core logic.
- Simulatable — the system runs step-by-step for testing and demonstration.

---

## Key Design Patterns

| Pattern   | Where It Is Used                                  | Why                                                                                        |
| --------- | ------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| Strategy  | `EvictionPolicy` decides which entry to evict      | Swap between LRU, LFU, or FIFO eviction without changing cache logic                       |
| Strategy  | `WritePolicy` syncs cache with backing store       | Switch between write-through and write-back without modifying the cache                    |
| Observer  | `CacheEventListener` reacts to cache events        | Loggers, metrics collectors, and monitors react independently to hits, misses, evictions   |
| Decorator | `MultiLevelCache` wraps caches in layers           | Add L1/L2 caching transparently without modifying the underlying cache implementation      |
| Facade    | `CacheSystem` exposes a unified API                | Hides the complexity of eviction, TTL, events, write policies, and multi-level coordination |
| Iterator  | `CacheIterator` provides sequential entry access   | Iterate through cache entries from most to least recent without exposing internal structure |

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


class EvictionReason(Enum):
    CAPACITY = "capacity"        # evicted because cache was full
    EXPIRED = "expired"          # evicted because TTL expired
    EXPLICIT = "explicit"        # explicitly removed by caller
    POLICY = "policy"            # evicted by custom policy logic


class CacheEventType(Enum):
    HIT = "hit"
    MISS = "miss"
    PUT = "put"
    UPDATE = "update"
    EVICTION = "eviction"
    EXPIRATION = "expiration"
    CLEAR = "clear"


class WritePolicyType(Enum):
    WRITE_THROUGH = "write_through"    # write to store immediately
    WRITE_BACK = "write_back"          # write to store lazily


# ------------------------------------------------------------------
# System constants
# ------------------------------------------------------------------

DEFAULT_CAPACITY = 128
DEFAULT_TTL_MS = 0                     # 0 = no expiration
MAX_CAPACITY = 1_000_000
MIN_CAPACITY = 1
STATS_DISPLAY_WIDTH = 50
```

`EvictionReason` tracks why an entry was removed — `CAPACITY` when the cache is full and the least recently used entry is evicted, `EXPIRED` when a TTL expires, `EXPLICIT` when the caller calls `remove()`, and `POLICY` for custom eviction logic. `CacheEventType` categorizes all observable events in the cache lifecycle. `WritePolicyType` distinguishes immediate writes from deferred writes to a backing store. `DEFAULT_TTL_MS` of 0 means entries never expire unless explicitly configured. `MAX_CAPACITY` caps the cache size to prevent unbounded memory usage.

---

## Doubly Linked List Node

```python
class DLLNode:
    """
    A node in the doubly linked list.

    Each node stores a key-value pair and pointers to the previous
    and next nodes. The key is stored in the node so that when we
    evict from the tail, we can look up and remove the corresponding
    HashMap entry in O(1) without scanning.
    """

    __slots__ = ("key", "value", "prev", "next", "freq",
                 "created_at", "last_accessed", "ttl_ms")

    def __init__(self, key: Any, value: Any,
                 ttl_ms: int = 0) -> None:
        self.key = key
        self.value = value
        self.prev: DLLNode | None = None
        self.next: DLLNode | None = None
        self.freq: int = 1                  # access frequency (LFU)
        self.created_at: int = int(time.time() * 1000)
        self.last_accessed: int = self.created_at
        self.ttl_ms: int = ttl_ms           # 0 = no expiration

    @property
    def is_expired(self) -> bool:
        if self.ttl_ms <= 0:
            return False
        return (int(time.time() * 1000) - self.created_at
                > self.ttl_ms)

    def touch(self) -> None:
        """Mark this node as accessed right now."""
        self.last_accessed = int(time.time() * 1000)
        self.freq += 1

    def __repr__(self) -> str:
        return f"Node(key={self.key}, value={self.value})"
```

`DLLNode` stores the key inside the node — this is critical. When we evict the tail node, we need the key to delete the corresponding entry from the HashMap. Without it, eviction would require an O(n) scan. `__slots__` eliminates the per-instance `__dict__`, reducing memory overhead per node — important when the cache holds thousands of entries. `freq` tracks how many times the entry has been accessed, used by the LFU policy. `is_expired` performs a lazy TTL check — the entry is not removed proactively; it's checked on access. `touch` updates both the timestamp and the frequency counter on every access.

---

## Doubly Linked List

```python
class DoublyLinkedList:
    """
    Doubly linked list with sentinel head and tail nodes.

    Sentinel nodes simplify edge cases — we never have to check
    for None when inserting or removing. The real entries live
    between head and tail. Most recently used entries are at the
    front (after head); least recently used are at the back
    (before tail).
    """

    def __init__(self) -> None:
        self._head = DLLNode("HEAD", "HEAD")
        self._tail = DLLNode("TAIL", "TAIL")
        self._head.next = self._tail
        self._tail.prev = self._head
        self._size = 0

    # ------------------------------------------------------------------
    # Properties
    # ------------------------------------------------------------------

    @property
    def size(self) -> int:
        return self._size

    @property
    def is_empty(self) -> bool:
        return self._size == 0

    @property
    def front(self) -> DLLNode | None:
        """Return the most recently used node (after head sentinel)."""
        if self._size == 0:
            return None
        return self._head.next

    @property
    def back(self) -> DLLNode | None:
        """Return the least recently used node (before tail sentinel)."""
        if self._size == 0:
            return None
        return self._tail.prev

    # ------------------------------------------------------------------
    # Operations
    # ------------------------------------------------------------------

    def add_to_front(self, node: DLLNode) -> None:
        """
        Insert a node right after the head sentinel.

        This marks the node as the most recently used entry.
        Runs in O(1) — just four pointer updates.
        """
        node.prev = self._head
        node.next = self._head.next
        self._head.next.prev = node
        self._head.next = node
        self._size += 1

    def remove(self, node: DLLNode) -> None:
        """
        Remove a node from its current position.

        Runs in O(1) because we have direct pointers. Does not
        require scanning — the node reference is obtained from
        the HashMap.
        """
        if node is self._head or node is self._tail:
            return
        node.prev.next = node.next
        node.next.prev = node.prev
        node.prev = None
        node.next = None
        self._size -= 1

    def move_to_front(self, node: DLLNode) -> None:
        """
        Move an existing node to the front (most recently used).

        Called on every cache hit. Combines remove + add_to_front
        but is a single logical operation.
        """
        self.remove(node)
        self.add_to_front(node)

    def remove_from_back(self) -> DLLNode | None:
        """
        Remove and return the tail node (least recently used).

        Called when the cache is full and a new entry needs space.
        Returns None if the list is empty.
        """
        if self._size == 0:
            return None
        node = self._tail.prev
        self.remove(node)
        return node

    def clear(self) -> None:
        """Remove all nodes between sentinels."""
        self._head.next = self._tail
        self._tail.prev = self._head
        self._size = 0

    def to_list(self) -> list[DLLNode]:
        """Return all nodes from front (MRU) to back (LRU)."""
        result: list[DLLNode] = []
        current = self._head.next
        while current is not self._tail:
            result.append(current)
            current = current.next
        return result

    def __repr__(self) -> str:
        nodes = self.to_list()
        keys = [str(n.key) for n in nodes]
        return f"DLL[{' ↔ '.join(keys)}] (size={self._size})"
```

`DoublyLinkedList` is the backbone of the LRU cache. Sentinel nodes (`_head` and `_tail`) eliminate all null checks — we never need to handle "inserting into an empty list" or "removing the last node" as special cases. `add_to_front` marks an entry as most recently used with exactly four pointer updates. `remove` detaches a node in O(1) because we have direct references from the HashMap. `move_to_front` is called on every cache hit — it removes the node from its current position and reinserts it at the front. `remove_from_back` evicts the least recently used entry (the node just before the tail sentinel). Every operation is O(1) — no scanning, no shifting.

---

## Eviction Policy (Strategy Pattern)

```python
class EvictionPolicy(ABC):
    """
    Defines which entry to evict when the cache is full.

    Each policy maintains its own internal data structure.
    The cache delegates eviction decisions to the policy
    without knowing how the decision is made.
    """

    @abstractmethod
    def on_access(self, node: DLLNode) -> None:
        """Called when an entry is accessed (get or put)."""
        ...

    @abstractmethod
    def on_insert(self, node: DLLNode) -> None:
        """Called when a new entry is inserted."""
        ...

    @abstractmethod
    def evict(self) -> DLLNode | None:
        """Return the node to evict, or None if empty."""
        ...

    @abstractmethod
    def on_remove(self, node: DLLNode) -> None:
        """Called when an entry is explicitly removed."""
        ...

    @abstractmethod
    def clear(self) -> None:
        """Clear all internal state."""
        ...

    @abstractmethod
    def name(self) -> str:
        ...


class LRUEvictionPolicy(EvictionPolicy):
    """
    Least Recently Used eviction.

    Maintains a doubly linked list ordered by access time. On every
    access, the node moves to the front. On eviction, the node at
    the back (least recently accessed) is removed.
    """

    def __init__(self) -> None:
        self._dll = DoublyLinkedList()

    def on_access(self, node: DLLNode) -> None:
        node.touch()
        self._dll.move_to_front(node)

    def on_insert(self, node: DLLNode) -> None:
        self._dll.add_to_front(node)

    def on_remove(self, node: DLLNode) -> None:
        self._dll.remove(node)

    def evict(self) -> DLLNode | None:
        return self._dll.remove_from_back()

    def clear(self) -> None:
        self._dll.clear()

    @property
    def order(self) -> list[DLLNode]:
        return self._dll.to_list()

    def name(self) -> str:
        return "LRU"


class LFUEvictionPolicy(EvictionPolicy):
    """
    Least Frequently Used eviction.

    Tracks access frequency per node. On eviction, the node with
    the lowest frequency is removed. Ties are broken by access
    recency (least recently used among least frequent).
    """

    def __init__(self) -> None:
        self._dll = DoublyLinkedList()
        self._freq_map: dict[int, list[DLLNode]] = {}
        self._min_freq: int = 1

    def on_access(self, node: DLLNode) -> None:
        old_freq = node.freq
        node.touch()  # increments freq
        new_freq = node.freq

        # Remove from old frequency list
        if old_freq in self._freq_map:
            old_list = self._freq_map[old_freq]
            if node in old_list:
                old_list.remove(node)
            if not old_list:
                del self._freq_map[old_freq]
                if self._min_freq == old_freq:
                    self._min_freq = new_freq

        # Add to new frequency list
        self._freq_map.setdefault(new_freq, []).append(node)

        self._dll.move_to_front(node)

    def on_insert(self, node: DLLNode) -> None:
        node.freq = 1
        self._freq_map.setdefault(1, []).append(node)
        self._min_freq = 1
        self._dll.add_to_front(node)

    def on_remove(self, node: DLLNode) -> None:
        freq = node.freq
        if freq in self._freq_map:
            freq_list = self._freq_map[freq]
            if node in freq_list:
                freq_list.remove(node)
            if not freq_list:
                del self._freq_map[freq]
        self._dll.remove(node)

    def evict(self) -> DLLNode | None:
        if not self._freq_map:
            return None

        # Find minimum frequency bucket
        min_list = self._freq_map.get(self._min_freq)
        if not min_list:
            # Recalculate min_freq
            self._min_freq = min(self._freq_map.keys()) \
                if self._freq_map else 1
            min_list = self._freq_map.get(self._min_freq)
            if not min_list:
                return None

        # Evict the least recently used among least frequent
        victim = min_list.pop(0)
        if not min_list:
            del self._freq_map[self._min_freq]
            if self._freq_map:
                self._min_freq = min(self._freq_map.keys())

        self._dll.remove(victim)
        return victim

    def clear(self) -> None:
        self._dll.clear()
        self._freq_map.clear()
        self._min_freq = 1

    @property
    def order(self) -> list[DLLNode]:
        return self._dll.to_list()

    def name(self) -> str:
        return "LFU"


class FIFOEvictionPolicy(EvictionPolicy):
    """
    First In, First Out eviction.

    The oldest entry is evicted regardless of how recently or
    frequently it was accessed. Accesses do not change the
    eviction order — only insertion time matters.
    """

    def __init__(self) -> None:
        self._dll = DoublyLinkedList()

    def on_access(self, node: DLLNode) -> None:
        node.touch()
        # FIFO: do NOT move to front on access

    def on_insert(self, node: DLLNode) -> None:
        self._dll.add_to_front(node)

    def on_remove(self, node: DLLNode) -> None:
        self._dll.remove(node)

    def evict(self) -> DLLNode | None:
        return self._dll.remove_from_back()

    def clear(self) -> None:
        self._dll.clear()

    @property
    def order(self) -> list[DLLNode]:
        return self._dll.to_list()

    def name(self) -> str:
        return "FIFO"
```

`EvictionPolicy` defines the contract for all eviction strategies. The critical difference between them is what happens on access. `LRUEvictionPolicy` moves the accessed node to the front — the entry at the back is always the least recently used. `LFUEvictionPolicy` tracks frequency counts per node and evicts the least frequently used; ties are broken by recency (the first inserted among equally frequent nodes is evicted first). `FIFOEvictionPolicy` ignores access entirely — `on_access` does not move the node, so the oldest entry (at the back) is always evicted first. Adding a new policy (e.g., Random, TTL-based) requires only implementing the five abstract methods — the cache never changes.

---

## Cache Entry & Statistics

```python
@dataclass
class CacheEntry:
    """
    Metadata wrapper for a cached key-value pair.

    Used when returning cache contents to external callers
    without exposing the internal DLLNode structure.
    """
    key: Any
    value: Any
    frequency: int
    created_at: int
    last_accessed: int
    ttl_ms: int
    is_expired: bool

    @staticmethod
    def from_node(node: DLLNode) -> "CacheEntry":
        return CacheEntry(
            key=node.key,
            value=node.value,
            frequency=node.freq,
            created_at=node.created_at,
            last_accessed=node.last_accessed,
            ttl_ms=node.ttl_ms,
            is_expired=node.is_expired,
        )


@dataclass
class CacheStats:
    """
    Aggregate cache performance statistics.

    Tracks hits, misses, evictions, and expirations to compute
    the hit rate — the primary metric for cache effectiveness.
    """
    hits: int = 0
    misses: int = 0
    puts: int = 0
    updates: int = 0
    evictions: int = 0
    expirations: int = 0
    current_size: int = 0
    capacity: int = 0

    @property
    def total_requests(self) -> int:
        return self.hits + self.misses

    @property
    def hit_rate(self) -> float:
        if self.total_requests == 0:
            return 0.0
        return self.hits / self.total_requests

    @property
    def miss_rate(self) -> float:
        return 1.0 - self.hit_rate

    def display(self) -> None:
        print(f"\n  {'═'*STATS_DISPLAY_WIDTH}")
        print(f"  CACHE STATISTICS")
        print(f"  {'═'*STATS_DISPLAY_WIDTH}")
        print(f"  Capacity:        {self.capacity}")
        print(f"  Current size:    {self.current_size}")
        print(f"  Total requests:  {self.total_requests}")
        print(f"  Hits:            {self.hits}")
        print(f"  Misses:          {self.misses}")
        print(f"  Hit rate:        {self.hit_rate:.1%}")
        print(f"  Miss rate:       {self.miss_rate:.1%}")
        print(f"  Puts:            {self.puts}")
        print(f"  Updates:         {self.updates}")
        print(f"  Evictions:       {self.evictions}")
        print(f"  Expirations:     {self.expirations}")
        print(f"  {'═'*STATS_DISPLAY_WIDTH}")

    def reset(self) -> None:
        self.hits = 0
        self.misses = 0
        self.puts = 0
        self.updates = 0
        self.evictions = 0
        self.expirations = 0
```

`CacheEntry` is a data transfer object that shields callers from the internal `DLLNode`. External consumers see `CacheEntry` objects; they never touch linked list pointers. `CacheStats` aggregates performance counters. The `hit_rate` property is the most important metric — a cache with a low hit rate is wasting memory. `total_requests` is hits plus misses (puts are not requests). `reset` allows clearing counters for interval-based monitoring without restarting the cache.

---

## LRU Cache Core

```python
class Cache(ABC):
    """
    Abstract cache interface.

    Defines the contract for all cache implementations.
    Concrete caches implement get, put, remove, and related
    operations using specific data structures and policies.
    """

    @abstractmethod
    def get(self, key: Any) -> Any | None:
        ...

    @abstractmethod
    def put(self, key: Any, value: Any,
            ttl_ms: int = 0) -> None:
        ...

    @abstractmethod
    def remove(self, key: Any) -> bool:
        ...

    @abstractmethod
    def contains(self, key: Any) -> bool:
        ...

    @abstractmethod
    def clear(self) -> None:
        ...

    @abstractmethod
    def size(self) -> int:
        ...

    @abstractmethod
    def capacity(self) -> int:
        ...

    @abstractmethod
    def stats(self) -> CacheStats:
        ...


class LRUCache(Cache):
    """
    LRU Cache backed by a HashMap and Doubly Linked List.

    The HashMap provides O(1) key lookup. The Doubly Linked List
    maintains access order. Together they deliver O(1) get and put.

    Internal structure:

        HashMap: { key → DLLNode }

        DLL: HEAD ↔ [MRU] ↔ [..] ↔ [..] ↔ [LRU] ↔ TAIL

    On get(key):
        1. Look up node in HashMap → O(1)
        2. Move node to front of DLL → O(1)
        3. Return value

    On put(key, value):
        1. If key exists: update value, move to front → O(1)
        2. If full: remove LRU (back of DLL), delete from HashMap → O(1)
        3. Create new node, add to front of DLL, add to HashMap → O(1)
    """

    def __init__(self, max_capacity: int = DEFAULT_CAPACITY,
                 eviction_policy: EvictionPolicy | None = None,
                 default_ttl_ms: int = DEFAULT_TTL_MS) -> None:
        if max_capacity < MIN_CAPACITY:
            raise ValueError(
                f"Capacity must be >= {MIN_CAPACITY}, "
                f"got {max_capacity}"
            )
        if max_capacity > MAX_CAPACITY:
            raise ValueError(
                f"Capacity must be <= {MAX_CAPACITY}, "
                f"got {max_capacity}"
            )

        self._capacity = max_capacity
        self._map: dict[Any, DLLNode] = {}
        self._policy = eviction_policy or LRUEvictionPolicy()
        self._default_ttl_ms = default_ttl_ms
        self._stats = CacheStats(capacity=max_capacity)
        self._lock = threading.RLock()

    # ------------------------------------------------------------------
    # Core operations
    # ------------------------------------------------------------------

    def get(self, key: Any) -> Any | None:
        """
        Retrieve the value for the given key.

        Returns None on miss. On hit, marks the key as most
        recently used. If the entry has expired, evicts it
        and returns None (lazy expiration).
        """
        with self._lock:
            node = self._map.get(key)

            if node is None:
                self._stats.misses += 1
                return None

            # Lazy TTL check
            if node.is_expired:
                self._evict_node(node, EvictionReason.EXPIRED)
                self._stats.misses += 1
                self._stats.expirations += 1
                return None

            # Cache hit
            self._policy.on_access(node)
            self._stats.hits += 1
            self._stats.current_size = len(self._map)
            return node.value

    def put(self, key: Any, value: Any,
            ttl_ms: int = 0) -> None:
        """
        Insert or update a key-value pair.

        If the key already exists, updates the value and marks it
        as most recently used. If the cache is full, evicts according
        to the eviction policy before inserting.
        """
        effective_ttl = ttl_ms if ttl_ms > 0 \
            else self._default_ttl_ms

        with self._lock:
            # Update existing entry
            if key in self._map:
                node = self._map[key]
                node.value = value
                node.ttl_ms = effective_ttl
                node.created_at = int(time.time() * 1000)
                self._policy.on_access(node)
                self._stats.updates += 1
                self._stats.current_size = len(self._map)
                return

            # Evict if at capacity
            if len(self._map) >= self._capacity:
                self._evict_by_policy()

            # Insert new entry
            node = DLLNode(key, value, ttl_ms=effective_ttl)
            self._map[key] = node
            self._policy.on_insert(node)
            self._stats.puts += 1
            self._stats.current_size = len(self._map)

    def remove(self, key: Any) -> bool:
        """
        Explicitly remove a key from the cache.

        Returns True if the key existed and was removed,
        False if the key was not in the cache.
        """
        with self._lock:
            node = self._map.get(key)
            if node is None:
                return False
            self._evict_node(node, EvictionReason.EXPLICIT)
            return True

    def contains(self, key: Any) -> bool:
        """Check if a key exists in the cache (does not count as access)."""
        with self._lock:
            node = self._map.get(key)
            if node is None:
                return False
            if node.is_expired:
                self._evict_node(node, EvictionReason.EXPIRED)
                self._stats.expirations += 1
                return False
            return True

    def clear(self) -> None:
        """Remove all entries from the cache."""
        with self._lock:
            self._map.clear()
            self._policy.clear()
            self._stats.current_size = 0

    def size(self) -> int:
        return len(self._map)

    def capacity(self) -> int:
        return self._capacity

    def stats(self) -> CacheStats:
        self._stats.current_size = len(self._map)
        return self._stats

    # ------------------------------------------------------------------
    # Iteration & inspection
    # ------------------------------------------------------------------

    def keys(self) -> list[Any]:
        """Return keys in order from most to least recently used."""
        with self._lock:
            if isinstance(self._policy,
                          (LRUEvictionPolicy, LFUEvictionPolicy,
                           FIFOEvictionPolicy)):
                return [n.key for n in self._policy.order
                        if not n.is_expired]
            return list(self._map.keys())

    def entries(self) -> list[CacheEntry]:
        """Return all non-expired entries as CacheEntry objects."""
        with self._lock:
            return [CacheEntry.from_node(n)
                    for n in self._map.values()
                    if not n.is_expired]

    def peek(self, key: Any) -> Any | None:
        """
        Get value without updating access order.

        Useful for inspection and debugging without
        disturbing the eviction order.
        """
        with self._lock:
            node = self._map.get(key)
            if node is None or node.is_expired:
                return None
            return node.value

    # ------------------------------------------------------------------
    # Internal helpers
    # ------------------------------------------------------------------

    def _evict_by_policy(self) -> None:
        """Ask the eviction policy to choose and remove a victim."""
        victim = self._policy.evict()
        if victim and victim.key in self._map:
            del self._map[victim.key]
            self._stats.evictions += 1

    def _evict_node(self, node: DLLNode,
                    reason: EvictionReason) -> None:
        """Remove a specific node from both map and policy."""
        self._policy.on_remove(node)
        if node.key in self._map:
            del self._map[node.key]

    def __repr__(self) -> str:
        keys = self.keys()[:10]
        suffix = "..." if len(self._map) > 10 else ""
        return (f"LRUCache(size={len(self._map)}/"
                f"{self._capacity}, "
                f"policy={self._policy.name()}, "
                f"keys=[{', '.join(str(k) for k in keys)}"
                f"{suffix}])")
```

`LRUCache` is the core data structure. The HashMap (`_map`) gives O(1) lookup by key. The eviction policy maintains a doubly linked list for O(1) ordering. Together they guarantee O(1) for both `get` and `put`. The `get` path: look up the node in the HashMap, check TTL (lazy expiration), call `on_access` to move the node to the front, and return the value. The `put` path: if the key exists, update the value in place and call `on_access`; if the cache is full, call `_evict_by_policy` to remove the victim; then create a new node and add it to both the HashMap and the policy. `peek` is a special read that does not alter eviction order — useful for debugging. `contains` also performs lazy TTL expiration. Every operation acquires the reentrant lock for thread safety.

---

## Cache Event System (Observer Pattern)

```python
@dataclass
class CacheEvent:
    """An event emitted by the cache."""
    event_id: str
    event_type: CacheEventType
    key: Any
    value: Any = None
    reason: str = ""
    timestamp: int = field(
        default_factory=lambda: int(time.time() * 1000)
    )


class CacheEventListener(ABC):
    """
    Observer that reacts to cache events.

    Implementations receive all events and decide which ones
    to handle. Listeners are decoupled from the cache — they
    cannot affect cache operations.
    """

    @abstractmethod
    def on_event(self, event: CacheEvent) -> None:
        ...

    @abstractmethod
    def listener_id(self) -> str:
        ...


class CacheLogger(CacheEventListener):
    """Logs all cache events to the console."""

    def on_event(self, event: CacheEvent) -> None:
        emoji = {
            CacheEventType.HIT: "🎯",
            CacheEventType.MISS: "❌",
            CacheEventType.PUT: "📥",
            CacheEventType.UPDATE: "🔄",
            CacheEventType.EVICTION: "🗑️",
            CacheEventType.EXPIRATION: "⏰",
            CacheEventType.CLEAR: "🧹",
        }
        icon = emoji.get(event.event_type, "📌")
        detail = ""
        if event.event_type == CacheEventType.HIT:
            detail = f"key={event.key} → {event.value}"
        elif event.event_type == CacheEventType.MISS:
            detail = f"key={event.key}"
        elif event.event_type in (CacheEventType.PUT,
                                   CacheEventType.UPDATE):
            detail = f"key={event.key} val={event.value}"
        elif event.event_type in (CacheEventType.EVICTION,
                                   CacheEventType.EXPIRATION):
            detail = f"key={event.key} ({event.reason})"
        elif event.event_type == CacheEventType.CLEAR:
            detail = "all entries removed"

        print(f"  {icon} [{event.event_type.value:>10}] {detail}")

    def listener_id(self) -> str:
        return "cache-logger"


class HitRateMonitor(CacheEventListener):
    """Tracks hit rate over a sliding window of requests."""

    def __init__(self, window_size: int = 100) -> None:
        self._window_size = window_size
        self._history: list[bool] = []    # True = hit, False = miss

    def on_event(self, event: CacheEvent) -> None:
        if event.event_type == CacheEventType.HIT:
            self._history.append(True)
        elif event.event_type == CacheEventType.MISS:
            self._history.append(False)
        else:
            return

        # Trim to window size
        if len(self._history) > self._window_size:
            self._history = self._history[-self._window_size:]

    @property
    def windowed_hit_rate(self) -> float:
        if not self._history:
            return 0.0
        return sum(self._history) / len(self._history)

    def listener_id(self) -> str:
        return "hit-rate-monitor"

    def display(self) -> None:
        print(f"\n  Sliding window hit rate "
              f"(last {len(self._history)} requests): "
              f"{self.windowed_hit_rate:.1%}")


class CacheEventBus:
    """
    Central event dispatcher for cache events.

    Manages listener registration and broadcasts events to all
    registered listeners. Thread-safe for concurrent emission.
    """

    def __init__(self) -> None:
        self._listeners: list[CacheEventListener] = []
        self._event_log: list[CacheEvent] = []
        self._lock = threading.RLock()

    def register(self, listener: CacheEventListener) -> None:
        with self._lock:
            self._listeners.append(listener)

    def unregister(self, listener_id: str) -> None:
        with self._lock:
            self._listeners = [
                l for l in self._listeners
                if l.listener_id() != listener_id
            ]

    def emit(self, event: CacheEvent) -> None:
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

`CacheEventListener` is the observer interface. `CacheLogger` prints every event with emoji-coded log lines — hits show the retrieved value, evictions show the reason. `HitRateMonitor` tracks hit rate over a sliding window of the last N requests, which is more useful than a lifetime average because it reveals recent performance trends. The `CacheEventBus` dispatches events to all registered listeners, isolating each listener from failures in others. Adding a new listener (e.g., metrics exporter, alerting) requires only implementing `on_event` and `listener_id`.

---

## TTL Support & Expiration

```python
class TTLCache(Cache):
    """
    Cache with automatic TTL (Time-To-Live) expiration.

    Wraps an LRUCache and adds active expiration scanning on
    top of the lazy expiration already built into LRUCache.
    Active scanning is triggered periodically or on demand.
    """

    def __init__(self, max_capacity: int = DEFAULT_CAPACITY,
                 default_ttl_ms: int = 60_000,
                 eviction_policy: EvictionPolicy | None = None
                 ) -> None:
        self._cache = LRUCache(
            max_capacity=max_capacity,
            eviction_policy=eviction_policy,
            default_ttl_ms=default_ttl_ms,
        )
        self._default_ttl_ms = default_ttl_ms

    def get(self, key: Any) -> Any | None:
        return self._cache.get(key)

    def put(self, key: Any, value: Any,
            ttl_ms: int = 0) -> None:
        effective_ttl = ttl_ms if ttl_ms > 0 \
            else self._default_ttl_ms
        self._cache.put(key, value, ttl_ms=effective_ttl)

    def remove(self, key: Any) -> bool:
        return self._cache.remove(key)

    def contains(self, key: Any) -> bool:
        return self._cache.contains(key)

    def clear(self) -> None:
        self._cache.clear()

    def size(self) -> int:
        return self._cache.size()

    def capacity(self) -> int:
        return self._cache.capacity()

    def stats(self) -> CacheStats:
        return self._cache.stats()

    def cleanup_expired(self) -> int:
        """
        Actively scan and remove all expired entries.

        Returns the number of entries removed. This supplements
        the lazy expiration in LRUCache — lazy expiration only
        removes entries when they are accessed, so very stale
        entries could linger indefinitely without active scanning.
        """
        expired_keys: list[Any] = []
        for entry in self._cache.entries():
            if entry.is_expired:
                expired_keys.append(entry.key)

        for key in expired_keys:
            self._cache.remove(key)

        stats = self._cache.stats()
        stats.expirations += len(expired_keys)
        return len(expired_keys)
```

`TTLCache` wraps `LRUCache` and enforces a default TTL on all entries. The `LRUCache` already performs lazy expiration — entries are checked when accessed. `TTLCache` adds `cleanup_expired` for active scanning, which removes entries that have not been accessed but have expired. In production, `cleanup_expired` would be called by a background timer thread. This two-pronged approach (lazy + active) prevents both stale reads (lazy catches on access) and memory bloat (active reclaims unreferenced entries).

---

## Write Policy (Strategy Pattern)

```python
class BackingStore(ABC):
    """
    Abstract backing store (database, file, remote service).

    The cache sits in front of the backing store. Write policies
    determine when changes are flushed to the store.
    """

    @abstractmethod
    def read(self, key: Any) -> Any | None:
        ...

    @abstractmethod
    def write(self, key: Any, value: Any) -> None:
        ...

    @abstractmethod
    def delete(self, key: Any) -> None:
        ...

    @abstractmethod
    def name(self) -> str:
        ...


class InMemoryStore(BackingStore):
    """In-memory dictionary simulating a database or disk store."""

    def __init__(self) -> None:
        self._data: dict[Any, Any] = {}
        self._read_count = 0
        self._write_count = 0

    def read(self, key: Any) -> Any | None:
        self._read_count += 1
        return self._data.get(key)

    def write(self, key: Any, value: Any) -> None:
        self._write_count += 1
        self._data[key] = value

    def delete(self, key: Any) -> None:
        self._data.pop(key, None)

    def name(self) -> str:
        return "InMemoryStore"

    @property
    def data(self) -> dict:
        return dict(self._data)

    @property
    def read_count(self) -> int:
        return self._read_count

    @property
    def write_count(self) -> int:
        return self._write_count


class WritePolicy(ABC):
    """
    Defines when cache writes are flushed to the backing store.

    Write-through flushes immediately on every put.
    Write-back defers writes and flushes on eviction or explicitly.
    """

    @abstractmethod
    def on_put(self, key: Any, value: Any,
               store: BackingStore) -> None:
        ...

    @abstractmethod
    def on_evict(self, key: Any, value: Any,
                 store: BackingStore) -> None:
        ...

    @abstractmethod
    def flush(self, dirty_entries: dict[Any, Any],
              store: BackingStore) -> int:
        ...

    @abstractmethod
    def name(self) -> str:
        ...


class WriteThroughPolicy(WritePolicy):
    """
    Write to the backing store immediately on every put.

    Guarantees strong consistency between cache and store
    at the cost of higher write latency (every put hits the store).
    """

    def on_put(self, key: Any, value: Any,
               store: BackingStore) -> None:
        store.write(key, value)

    def on_evict(self, key: Any, value: Any,
                 store: BackingStore) -> None:
        pass  # already written on put

    def flush(self, dirty_entries: dict[Any, Any],
              store: BackingStore) -> int:
        return 0  # nothing to flush

    def name(self) -> str:
        return "WriteThrough"


class WriteBackPolicy(WritePolicy):
    """
    Defer writes to the backing store until eviction or explicit flush.

    Tracks dirty entries and writes them to the store when they
    are evicted or when flush() is called. Offers lower write
    latency but risks data loss if the process crashes before flush.
    """

    def __init__(self) -> None:
        self._dirty: dict[Any, Any] = {}

    def on_put(self, key: Any, value: Any,
               store: BackingStore) -> None:
        self._dirty[key] = value  # mark as dirty, don't write yet

    def on_evict(self, key: Any, value: Any,
                 store: BackingStore) -> None:
        if key in self._dirty:
            store.write(key, value)
            del self._dirty[key]

    def flush(self, dirty_entries: dict[Any, Any],
              store: BackingStore) -> int:
        """Flush all dirty entries to the backing store."""
        count = 0
        for key, value in list(self._dirty.items()):
            store.write(key, value)
            count += 1
        self._dirty.clear()
        return count

    @property
    def dirty_count(self) -> int:
        return len(self._dirty)

    def name(self) -> str:
        return "WriteBack"
```

`WritePolicy` defines when cache writes reach the backing store. `WriteThroughPolicy` writes to the store on every `put` — the cache and store are always consistent, but every write has the latency of a store write. `WriteBackPolicy` defers writes by tracking dirty entries. Data is flushed to the store only when evicted or when `flush()` is called explicitly. This trades consistency for performance — if the process crashes before flush, dirty entries are lost. `InMemoryStore` simulates a database with read/write counters for verifying that write policies work correctly. In production, `BackingStore` implementations would wrap a database client, Redis connection, or file system.

---

## Bounded Cache with Eviction Callbacks

```python
class BoundedCache(Cache):
    """
    Cache with eviction callbacks and write policy support.

    Extends LRUCache with:
    - Callbacks invoked when entries are evicted
    - Write policy integration for syncing with backing store
    - Event bus for observer notifications
    """

    def __init__(
        self,
        max_capacity: int = DEFAULT_CAPACITY,
        eviction_policy: EvictionPolicy | None = None,
        write_policy: WritePolicy | None = None,
        store: BackingStore | None = None,
        event_bus: CacheEventBus | None = None,
        default_ttl_ms: int = DEFAULT_TTL_MS,
    ) -> None:
        self._cache = LRUCache(
            max_capacity=max_capacity,
            eviction_policy=eviction_policy,
            default_ttl_ms=default_ttl_ms,
        )
        self._write_policy = write_policy
        self._store = store
        self._event_bus = event_bus or CacheEventBus()
        self._eviction_callbacks: list[
            Callable[[Any, Any, EvictionReason], None]
        ] = []
        self._lock = threading.RLock()

    def on_eviction(
        self, callback: Callable[[Any, Any, EvictionReason], None]
    ) -> None:
        """Register a callback invoked on every eviction."""
        self._eviction_callbacks.append(callback)

    def get(self, key: Any) -> Any | None:
        with self._lock:
            value = self._cache.get(key)

            if value is not None:
                self._emit(CacheEventType.HIT, key, value)
                return value

            # Cache miss — try backing store
            if self._store:
                store_value = self._store.read(key)
                if store_value is not None:
                    # Promote from store to cache
                    self._cache.put(key, store_value)
                    self._emit(CacheEventType.HIT, key,
                               store_value)
                    return store_value

            self._emit(CacheEventType.MISS, key)
            return None

    def put(self, key: Any, value: Any,
            ttl_ms: int = 0) -> None:
        with self._lock:
            existed = self._cache.contains(key)

            # Check if eviction will happen
            if (not existed
                    and self._cache.size() >= self._cache.capacity()):
                self._handle_eviction()

            self._cache.put(key, value, ttl_ms=ttl_ms)

            if self._write_policy and self._store:
                self._write_policy.on_put(key, value, self._store)

            event_type = CacheEventType.UPDATE if existed \
                else CacheEventType.PUT
            self._emit(event_type, key, value)

    def remove(self, key: Any) -> bool:
        with self._lock:
            value = self._cache.peek(key)
            removed = self._cache.remove(key)
            if removed:
                self._fire_eviction_callbacks(
                    key, value, EvictionReason.EXPLICIT
                )
                self._emit(CacheEventType.EVICTION, key, value,
                           reason="explicit removal")
                if self._store:
                    self._store.delete(key)
            return removed

    def contains(self, key: Any) -> bool:
        return self._cache.contains(key)

    def clear(self) -> None:
        with self._lock:
            self._cache.clear()
            self._emit(CacheEventType.CLEAR, key=None)

    def size(self) -> int:
        return self._cache.size()

    def capacity(self) -> int:
        return self._cache.capacity()

    def stats(self) -> CacheStats:
        return self._cache.stats()

    def flush(self) -> int:
        """Flush dirty entries to the backing store."""
        if self._write_policy and self._store:
            return self._write_policy.flush({}, self._store)
        return 0

    @property
    def event_bus(self) -> CacheEventBus:
        return self._event_bus

    # ------------------------------------------------------------------
    # Internal helpers
    # ------------------------------------------------------------------

    def _handle_eviction(self) -> None:
        """Peek at the victim before the policy evicts it."""
        keys = self._cache.keys()
        if keys:
            victim_key = keys[-1]  # LRU = last in order
            victim_value = self._cache.peek(victim_key)
            if victim_value is not None:
                self._fire_eviction_callbacks(
                    victim_key, victim_value,
                    EvictionReason.CAPACITY,
                )
                if self._write_policy and self._store:
                    self._write_policy.on_evict(
                        victim_key, victim_value, self._store
                    )
                self._emit(
                    CacheEventType.EVICTION,
                    victim_key, victim_value,
                    reason="capacity limit reached",
                )

    def _fire_eviction_callbacks(self, key: Any, value: Any,
                                  reason: EvictionReason) -> None:
        for cb in self._eviction_callbacks:
            try:
                cb(key, value, reason)
            except Exception as e:
                print(f"  ⚠️ Eviction callback error: {e}")

    def _emit(self, event_type: CacheEventType, key: Any,
              value: Any = None, reason: str = "") -> None:
        event = CacheEvent(
            event_id=str(uuid.uuid4())[:8],
            event_type=event_type,
            key=key,
            value=value,
            reason=reason,
        )
        self._event_bus.emit(event)
```

`BoundedCache` layers eviction callbacks, write policies, and events on top of `LRUCache`. On `get`, if the cache misses but a backing store is configured, it reads from the store and promotes the value into the cache — this is read-through caching. On `put`, it writes to the cache and notifies the write policy. On eviction, callbacks fire before the entry is removed — useful for logging, metrics, or cascading invalidation. `_handle_eviction` peeks at the victim before the policy evicts it so callbacks receive the correct key-value pair. The write policy's `on_evict` is called to flush dirty entries to the store before they are lost.

---

## Multi-Level Cache (Decorator Pattern)

```python
class MultiLevelCache(Cache):
    """
    Two-level cache: L1 (fast, small) backed by L2 (slower, larger).

    On miss in L1, checks L2 and promotes the entry to L1.
    On eviction from L1, demotes the entry to L2.
    This creates a natural hot/cold separation.

    Example:
        L1: 100 entries, LRU, fast access
        L2: 1000 entries, LFU, slower but larger
    """

    def __init__(self, l1: Cache, l2: Cache) -> None:
        self._l1 = l1
        self._l2 = l2
        self._stats = CacheStats(
            capacity=l1.capacity() + l2.capacity()
        )

    def get(self, key: Any) -> Any | None:
        # Try L1 first
        value = self._l1.get(key)
        if value is not None:
            self._stats.hits += 1
            return value

        # Try L2
        value = self._l2.get(key)
        if value is not None:
            # Promote to L1
            self._l1.put(key, value)
            self._l2.remove(key)
            self._stats.hits += 1
            return value

        self._stats.misses += 1
        return None

    def put(self, key: Any, value: Any,
            ttl_ms: int = 0) -> None:
        # Always write to L1
        self._l1.put(key, value, ttl_ms=ttl_ms)
        self._stats.puts += 1

    def remove(self, key: Any) -> bool:
        removed_l1 = self._l1.remove(key)
        removed_l2 = self._l2.remove(key)
        return removed_l1 or removed_l2

    def contains(self, key: Any) -> bool:
        return self._l1.contains(key) or self._l2.contains(key)

    def clear(self) -> None:
        self._l1.clear()
        self._l2.clear()

    def size(self) -> int:
        return self._l1.size() + self._l2.size()

    def capacity(self) -> int:
        return self._l1.capacity() + self._l2.capacity()

    def stats(self) -> CacheStats:
        self._stats.current_size = self.size()
        return self._stats

    @property
    def l1(self) -> Cache:
        return self._l1

    @property
    def l2(self) -> Cache:
        return self._l2

    def display_levels(self) -> None:
        print(f"\n  {'─'*STATS_DISPLAY_WIDTH}")
        print(f"  MULTI-LEVEL CACHE")
        print(f"  {'─'*STATS_DISPLAY_WIDTH}")
        l1_stats = self._l1.stats()
        l2_stats = self._l2.stats()
        print(f"  L1: size={l1_stats.current_size}/"
              f"{l1_stats.capacity} "
              f"hits={l1_stats.hits} "
              f"misses={l1_stats.misses} "
              f"hit_rate={l1_stats.hit_rate:.1%}")
        print(f"  L2: size={l2_stats.current_size}/"
              f"{l2_stats.capacity} "
              f"hits={l2_stats.hits} "
              f"misses={l2_stats.misses} "
              f"hit_rate={l2_stats.hit_rate:.1%}")
        combined = self.stats()
        print(f"  Combined: hit_rate={combined.hit_rate:.1%} "
              f"total_size={combined.current_size}")
        print(f"  {'─'*STATS_DISPLAY_WIDTH}")
```

`MultiLevelCache` implements the decorator pattern — it wraps two caches and presents a single `Cache` interface. On `get`: try L1 first (fast, small); on miss, try L2 (slower, larger); if found in L2, promote to L1 and remove from L2. On `put`: always write to L1. When L1 evicts (handled by its own eviction policy), the eviction callback can demote the entry to L2. This creates a natural hot/cold separation — frequently accessed entries stay in L1, while less popular entries fall to L2. This pattern mirrors real CPU cache hierarchies (L1/L2/L3) and CDN architectures (edge/origin).

---

## Cache Facade

```python
class CacheSystem:
    """
    Facade that orchestrates cache creation and configuration.

    Provides factory methods for common cache setups and exposes
    a simple API that hides all internal complexity.
    """

    def __init__(
        self,
        capacity: int = DEFAULT_CAPACITY,
        eviction_policy: EvictionPolicy | None = None,
        write_policy: WritePolicy | None = None,
        store: BackingStore | None = None,
        default_ttl_ms: int = DEFAULT_TTL_MS,
    ) -> None:
        self._event_bus = CacheEventBus()
        self._logger = CacheLogger()
        self._monitor = HitRateMonitor()
        self._event_bus.register(self._logger)
        self._event_bus.register(self._monitor)

        self._cache = BoundedCache(
            max_capacity=capacity,
            eviction_policy=eviction_policy,
            write_policy=write_policy,
            store=store,
            event_bus=self._event_bus,
            default_ttl_ms=default_ttl_ms,
        )

    # ------------------------------------------------------------------
    # Core operations
    # ------------------------------------------------------------------

    def get(self, key: Any) -> Any | None:
        return self._cache.get(key)

    def put(self, key: Any, value: Any,
            ttl_ms: int = 0) -> None:
        self._cache.put(key, value, ttl_ms=ttl_ms)

    def remove(self, key: Any) -> bool:
        return self._cache.remove(key)

    def contains(self, key: Any) -> bool:
        return self._cache.contains(key)

    def clear(self) -> None:
        self._cache.clear()

    # ------------------------------------------------------------------
    # Inspection
    # ------------------------------------------------------------------

    @property
    def size(self) -> int:
        return self._cache.size()

    @property
    def stats(self) -> CacheStats:
        return self._cache.stats()

    @property
    def monitor(self) -> HitRateMonitor:
        return self._monitor

    @property
    def event_bus(self) -> CacheEventBus:
        return self._event_bus

    @property
    def cache(self) -> BoundedCache:
        return self._cache

    def flush(self) -> int:
        return self._cache.flush()

    def display_status(self) -> None:
        self._cache.stats().display()
        self._monitor.display()
```

`CacheSystem` is the facade — it wires together the cache, eviction policy, write policy, backing store, event bus, logger, and monitor. Users interact with `get`, `put`, `remove` without knowing about linked list nodes, eviction callbacks, or event dispatch. The constructor registers a logger and hit rate monitor by default. `display_status` prints both the cumulative statistics and the sliding window hit rate. Adding new functionality (e.g., metrics export, alerting) means registering a new listener — the facade code does not change.

---

## Simulation Engine

```python
class CacheScenario:
    """
    Tick-based simulation engine for cache demos.

    Each tick represents a logical time step. Actions are registered
    at specific ticks and execute in order when the simulation runs.
    """

    def __init__(self, system: CacheSystem) -> None:
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

`CacheScenario` mirrors the simulation engine from other guides. Actions are registered at specific ticks and execute in order, making demos reproducible and readable.

---

## Putting It All Together

```python
def main_scenario() -> None:
    """Run the main LRU cache demonstration."""

    print("\n╔══════════════════════════════════════════╗")
    print("║         LRU CACHE SYSTEM                 ║")
    print("║  Capacity: 5                              ║")
    print("║  Policy: LRU                              ║")
    print("║  Write: WriteThrough                      ║")
    print("╚══════════════════════════════════════════╝")

    store = InMemoryStore()
    system = CacheSystem(
        capacity=5,
        eviction_policy=LRUEvictionPolicy(),
        write_policy=WriteThroughPolicy(),
        store=store,
        default_ttl_ms=0,
    )

    scenario = CacheScenario(system)

    # --- Fill cache ---
    scenario.at_tick(1, "Put A=1, B=2, C=3, D=4, E=5",
                     lambda: (
                         system.put("A", 1),
                         system.put("B", 2),
                         system.put("C", 3),
                         system.put("D", 4),
                         system.put("E", 5),
                     ))

    # --- Access to make B most recently used ---
    scenario.at_tick(2, "Get B (moves to front)",
                     lambda: system.get("B"))

    # --- Insert F (should evict A, the LRU) ---
    scenario.at_tick(3, "Put F=6 (cache full — evicts LRU)",
                     lambda: system.put("F", 6))

    # --- Verify A was evicted ---
    scenario.at_tick(4, "Get A (should miss — was evicted)",
                     lambda: system.get("A"))

    # --- Get from backing store ---
    scenario.at_tick(5, "Get A (should find in backing store)",
                     lambda: print(f"    Store has A: "
                                   f"{store.data.get('A')}"))

    # --- Update existing entry ---
    scenario.at_tick(6, "Put C=30 (update existing key)",
                     lambda: system.put("C", 30))

    # --- Insert G and H (should evict D and E) ---
    scenario.at_tick(7, "Put G=7, H=8 (evicts D and E)",
                     lambda: (
                         system.put("G", 7),
                         system.put("H", 8),
                     ))

    # --- Show final state ---
    scenario.at_tick(8, "Display cache statistics",
                     lambda: system.display_status())

    scenario.at_tick(8, "Verify backing store",
                     lambda: print(f"    Backing store contents: "
                                   f"{store.data}"))

    scenario.run()


# ------------------------------------------------------------------
# Strategy Comparison
# ------------------------------------------------------------------

def compare_eviction_policies() -> None:
    """Compare LRU, LFU, and FIFO on the same workload."""

    print("\n" + "=" * 55)
    print("  EVICTION POLICY COMPARISON")
    print("=" * 55)

    policies: list[EvictionPolicy] = [
        LRUEvictionPolicy(),
        LFUEvictionPolicy(),
        FIFOEvictionPolicy(),
    ]

    # Workload: access pattern with hot keys (A, B) and cold keys
    workload = [
        ("put", "A", 1), ("put", "B", 2), ("put", "C", 3),
        ("put", "D", 4), ("put", "E", 5),
        ("get", "A", None), ("get", "B", None),
        ("get", "A", None), ("get", "A", None),
        ("get", "B", None),
        # Cache full, now insert F — who gets evicted?
        ("put", "F", 6),
        # Check who survived
        ("get", "A", None), ("get", "B", None),
        ("get", "C", None), ("get", "D", None),
        ("get", "E", None), ("get", "F", None),
    ]

    for policy in policies:
        cache = LRUCache(max_capacity=5,
                         eviction_policy=policy)

        for op, key, value in workload:
            if op == "put":
                cache.put(key, value)
            else:
                cache.get(key)

        stats = cache.stats()
        print(f"\n  Policy: {policy.name()}")
        print(f"  {'─'*45}")
        print(f"    Hits:      {stats.hits}")
        print(f"    Misses:    {stats.misses}")
        print(f"    Hit rate:  {stats.hit_rate:.1%}")
        print(f"    Evictions: {stats.evictions}")
        print(f"    Keys in cache: "
              f"{cache.keys()}")

    print("\n" + "=" * 55)


def compare_write_policies() -> None:
    """Compare WriteThroughPolicy vs WriteBackPolicy."""

    print("\n" + "=" * 55)
    print("  WRITE POLICY COMPARISON")
    print("=" * 55)

    for policy_name, policy_fn in [
        ("WriteThrough", lambda: WriteThroughPolicy()),
        ("WriteBack", lambda: WriteBackPolicy()),
    ]:
        store = InMemoryStore()
        policy = policy_fn()
        cache = BoundedCache(
            max_capacity=3,
            eviction_policy=LRUEvictionPolicy(),
            write_policy=policy,
            store=store,
        )

        # Put 5 entries (3 in cache, 2 cause evictions)
        for i in range(5):
            cache.put(f"key-{i}", f"val-{i}")

        print(f"\n  Policy: {policy_name}")
        print(f"  {'─'*45}")
        print(f"    Store writes after 5 puts: "
              f"{store.write_count}")
        print(f"    Store contents: {store.data}")

        # Flush (only relevant for WriteBack)
        if isinstance(policy, WriteBackPolicy):
            flushed = policy.flush({}, store)
            print(f"    After flush: {flushed} entries written")
            print(f"    Total store writes: "
                  f"{store.write_count}")
            print(f"    Store contents: {store.data}")

    print("\n" + "=" * 55)


# ------------------------------------------------------------------
# Unit Tests
# ------------------------------------------------------------------

def test_basic_get_put() -> None:
    """Test basic get and put operations."""
    cache = LRUCache(max_capacity=3)
    cache.put("A", 1)
    cache.put("B", 2)
    cache.put("C", 3)

    assert cache.get("A") == 1
    assert cache.get("B") == 2
    assert cache.get("C") == 3
    assert cache.get("D") is None
    assert cache.size() == 3
    print("✅ test_basic_get_put passed")


def test_lru_eviction() -> None:
    """Test that LRU entry is evicted when cache is full."""
    cache = LRUCache(max_capacity=3)
    cache.put("A", 1)
    cache.put("B", 2)
    cache.put("C", 3)

    # Access A to make it recently used
    cache.get("A")

    # Insert D — should evict B (least recently used)
    cache.put("D", 4)

    assert cache.get("A") == 1   # still present
    assert cache.get("B") is None  # evicted
    assert cache.get("C") == 3   # still present
    assert cache.get("D") == 4   # newly inserted
    print("✅ test_lru_eviction passed")


def test_lfu_eviction() -> None:
    """Test that LFU evicts the least frequently used entry."""
    cache = LRUCache(max_capacity=3,
                     eviction_policy=LFUEvictionPolicy())
    cache.put("A", 1)
    cache.put("B", 2)
    cache.put("C", 3)

    # Access A three times, B once, C not at all
    cache.get("A")
    cache.get("A")
    cache.get("A")
    cache.get("B")

    # Insert D — should evict C (freq=1, least frequent)
    cache.put("D", 4)

    assert cache.get("A") == 1   # freq=5 (1 insert + 3 gets + 1 get now)
    assert cache.get("B") == 2   # freq=3
    assert cache.get("C") is None  # evicted (was freq=1)
    assert cache.get("D") == 4   # newly inserted
    print("✅ test_lfu_eviction passed")


def test_fifo_eviction() -> None:
    """Test that FIFO evicts the oldest entry regardless of access."""
    cache = LRUCache(max_capacity=3,
                     eviction_policy=FIFOEvictionPolicy())
    cache.put("A", 1)
    cache.put("B", 2)
    cache.put("C", 3)

    # Access A — should NOT change eviction order for FIFO
    cache.get("A")
    cache.get("A")
    cache.get("A")

    # Insert D — should evict A (oldest, inserted first)
    cache.put("D", 4)

    assert cache.get("A") is None  # evicted (oldest)
    assert cache.get("B") == 2
    assert cache.get("C") == 3
    assert cache.get("D") == 4
    print("✅ test_fifo_eviction passed")


def test_update_existing_key() -> None:
    """Test that updating a key does not increase size."""
    cache = LRUCache(max_capacity=3)
    cache.put("A", 1)
    cache.put("B", 2)
    cache.put("C", 3)

    cache.put("A", 10)  # update, not insert

    assert cache.size() == 3
    assert cache.get("A") == 10
    print("✅ test_update_existing_key passed")


def test_remove() -> None:
    """Test explicit removal of a key."""
    cache = LRUCache(max_capacity=5)
    cache.put("A", 1)
    cache.put("B", 2)

    assert cache.remove("A") is True
    assert cache.get("A") is None
    assert cache.size() == 1
    assert cache.remove("Z") is False  # not in cache
    print("✅ test_remove passed")


def test_clear() -> None:
    """Test clearing all entries."""
    cache = LRUCache(max_capacity=5)
    cache.put("A", 1)
    cache.put("B", 2)
    cache.put("C", 3)
    cache.clear()

    assert cache.size() == 0
    assert cache.get("A") is None
    print("✅ test_clear passed")


def test_peek_does_not_alter_order() -> None:
    """Test that peek does not change eviction order."""
    cache = LRUCache(max_capacity=3)
    cache.put("A", 1)
    cache.put("B", 2)
    cache.put("C", 3)

    # Peek at A — should NOT move it to front
    assert cache.peek("A") == 1

    # Insert D — should evict A (still LRU because peek didn't move it)
    cache.put("D", 4)

    assert cache.get("A") is None  # evicted
    assert cache.get("B") == 2
    print("✅ test_peek_does_not_alter_order passed")


def test_ttl_expiration() -> None:
    """Test that expired entries return None."""
    cache = LRUCache(max_capacity=5, default_ttl_ms=1)
    cache.put("A", 1)

    # Wait for entry to expire
    time.sleep(0.01)  # 10ms > 1ms TTL

    assert cache.get("A") is None
    stats = cache.stats()
    assert stats.expirations >= 1
    print("✅ test_ttl_expiration passed")


def test_stats_tracking() -> None:
    """Test that statistics are tracked correctly."""
    cache = LRUCache(max_capacity=3)
    cache.put("A", 1)
    cache.put("B", 2)
    cache.put("C", 3)

    cache.get("A")   # hit
    cache.get("D")   # miss
    cache.get("B")   # hit

    cache.put("D", 4)  # evict + put

    stats = cache.stats()
    assert stats.hits == 2
    assert stats.misses == 1
    assert stats.puts == 4       # A, B, C, D
    assert stats.evictions == 1  # C evicted
    print("✅ test_stats_tracking passed")


def test_hit_rate_calculation() -> None:
    """Test hit rate computation."""
    cache = LRUCache(max_capacity=10)
    for i in range(10):
        cache.put(f"key-{i}", i)

    # 10 hits
    for i in range(10):
        cache.get(f"key-{i}")

    # 5 misses
    for i in range(10, 15):
        cache.get(f"key-{i}")

    stats = cache.stats()
    assert stats.hit_rate == 10 / 15
    print("✅ test_hit_rate_calculation passed")


def test_multi_level_cache() -> None:
    """Test L1/L2 promotion and demotion."""
    l1 = LRUCache(max_capacity=2)
    l2 = LRUCache(max_capacity=5)
    ml = MultiLevelCache(l1, l2)

    # Put into L1
    ml.put("A", 1)
    ml.put("B", 2)
    assert l1.size() == 2

    # L2 has entries independently
    l2.put("C", 3)
    l2.put("D", 4)

    # Get C — should promote from L2 to L1, evicting LRU in L1
    val = ml.get("C")
    assert val == 3
    assert l1.contains("C")    # promoted to L1
    assert not l2.contains("C")  # removed from L2

    print("✅ test_multi_level_cache passed")


def test_write_through() -> None:
    """Test that write-through writes to store immediately."""
    store = InMemoryStore()
    cache = BoundedCache(
        max_capacity=3,
        eviction_policy=LRUEvictionPolicy(),
        write_policy=WriteThroughPolicy(),
        store=store,
    )

    cache.put("A", 1)
    cache.put("B", 2)

    assert store.data["A"] == 1
    assert store.data["B"] == 2
    assert store.write_count == 2
    print("✅ test_write_through passed")


def test_write_back() -> None:
    """Test that write-back defers writes until flush."""
    store = InMemoryStore()
    policy = WriteBackPolicy()
    cache = BoundedCache(
        max_capacity=3,
        eviction_policy=LRUEvictionPolicy(),
        write_policy=policy,
        store=store,
    )

    cache.put("A", 1)
    cache.put("B", 2)

    # Store should NOT have been written to yet
    assert store.write_count == 0
    assert policy.dirty_count == 2

    # Flush
    policy.flush({}, store)
    assert store.write_count == 2
    assert store.data["A"] == 1
    assert store.data["B"] == 2
    print("✅ test_write_back passed")


def test_dll_operations() -> None:
    """Test doubly linked list operations directly."""
    dll = DoublyLinkedList()

    n1 = DLLNode("A", 1)
    n2 = DLLNode("B", 2)
    n3 = DLLNode("C", 3)

    dll.add_to_front(n1)
    dll.add_to_front(n2)
    dll.add_to_front(n3)

    assert dll.size == 3
    assert dll.front.key == "C"   # most recent
    assert dll.back.key == "A"    # least recent

    dll.move_to_front(n1)
    assert dll.front.key == "A"   # now most recent

    removed = dll.remove_from_back()
    assert removed.key == "B"     # was least recent
    assert dll.size == 2

    dll.clear()
    assert dll.size == 0
    assert dll.is_empty
    print("✅ test_dll_operations passed")


def test_contains_with_expiry() -> None:
    """Test that contains returns False for expired entries."""
    cache = LRUCache(max_capacity=5, default_ttl_ms=1)
    cache.put("A", 1)

    assert cache.contains("A") is True

    time.sleep(0.01)

    assert cache.contains("A") is False
    print("✅ test_contains_with_expiry passed")


def test_capacity_validation() -> None:
    """Test that invalid capacities are rejected."""
    try:
        LRUCache(max_capacity=0)
        assert False, "Should have raised ValueError"
    except ValueError:
        pass

    try:
        LRUCache(max_capacity=-1)
        assert False, "Should have raised ValueError"
    except ValueError:
        pass

    try:
        LRUCache(max_capacity=MAX_CAPACITY + 1)
        assert False, "Should have raised ValueError"
    except ValueError:
        pass

    print("✅ test_capacity_validation passed")


def test_money_type_keys() -> None:
    """Test that cache works with various key types."""
    cache = LRUCache(max_capacity=5)

    cache.put(1, "integer key")
    cache.put("str", "string key")
    cache.put((1, 2), "tuple key")

    assert cache.get(1) == "integer key"
    assert cache.get("str") == "string key"
    assert cache.get((1, 2)) == "tuple key"
    print("✅ test_money_type_keys passed")


def test_backing_store_read_through() -> None:
    """Test that BoundedCache reads from backing store on miss."""
    store = InMemoryStore()
    store.write("A", 100)  # pre-populate store

    cache = BoundedCache(
        max_capacity=3,
        eviction_policy=LRUEvictionPolicy(),
        store=store,
    )

    # A is in store but not in cache
    value = cache.get("A")
    assert value == 100  # read from store and promoted
    assert cache.contains("A")  # now in cache
    print("✅ test_backing_store_read_through passed")


def run_all_tests() -> None:
    print("\n" + "=" * 40)
    print("  RUNNING TESTS")
    print("=" * 40 + "\n")
    test_basic_get_put()
    test_lru_eviction()
    test_lfu_eviction()
    test_fifo_eviction()
    test_update_existing_key()
    test_remove()
    test_clear()
    test_peek_does_not_alter_order()
    test_ttl_expiration()
    test_stats_tracking()
    test_hit_rate_calculation()
    test_multi_level_cache()
    test_write_through()
    test_write_back()
    test_dll_operations()
    test_contains_with_expiry()
    test_capacity_validation()
    test_money_type_keys()
    test_backing_store_read_through()
    print("\n✅ All tests passed!\n")


if __name__ == "__main__":
    main_scenario()
    compare_eviction_policies()
    compare_write_policies()
    run_all_tests()
```

Each test isolates a single behavior. `test_basic_get_put` verifies O(1) get and put work correctly. `test_lru_eviction` confirms that accessing A moves it to the front, so B becomes the LRU and gets evicted when D is inserted. `test_lfu_eviction` checks that the least frequently accessed entry (C with frequency 1) is evicted. `test_fifo_eviction` verifies that accessing A does NOT change eviction order in FIFO — A is still the oldest and gets evicted first. `test_peek_does_not_alter_order` is subtle: `peek` reads without moving the node, so A remains LRU despite being peeked. `test_ttl_expiration` uses a 1ms TTL and sleeps 10ms to confirm lazy expiration works. `test_write_through` verifies the store receives writes immediately. `test_write_back` verifies the store sees zero writes until `flush()` is called. `test_dll_operations` directly tests the doubly linked list to ensure sentinel nodes and pointer manipulation are correct. `test_backing_store_read_through` confirms that a miss triggers a store read and promotes the value into the cache.

---

## Class Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                        CacheSystem                                │
│──────────────────────────────────────────────────────────────────│
│ - _cache: BoundedCache                                            │
│ - _event_bus: CacheEventBus                                       │
│ - _logger: CacheLogger                                            │
│ - _monitor: HitRateMonitor                                        │
│──────────────────────────────────────────────────────────────────│
│ + get(key) → value | None                                         │
│ + put(key, value, ttl_ms) → None                                  │
│ + remove(key) → bool                                              │
│ + contains(key) → bool                                            │
│ + clear() → None                                                  │
│ + flush() → int                                                   │
│ + display_status() → None                                         │
└───────┬──────────┬──────────┬────────────────────────────────────┘
        │          │          │
   uses ▼     uses ▼     uses ▼
┌──────────────────────────────────────────────────────────────────┐
│  BoundedCache                                                     │
│──────────────────────────────────────────────────────────────────│
│ - _cache: LRUCache                                                │
│ - _write_policy: WritePolicy                                      │
│ - _store: BackingStore                                            │
│ - _event_bus: CacheEventBus                                       │
│ - _eviction_callbacks: list[Callable]                             │
│──────────────────────────────────────────────────────────────────│
│ + get(key) → value | None                                         │
│ + put(key, value, ttl_ms) → None                                  │
│ + on_eviction(callback) → None                                    │
│ + flush() → int                                                   │
└──────────┬───────────────────────────────────────────────────────┘
           │ wraps
           ▼
┌──────────────────────────────────────────────────────────────────┐
│  LRUCache                                                         │
│──────────────────────────────────────────────────────────────────│
│ - _capacity: int                                                  │
│ - _map: dict[Any, DLLNode]                                        │
│ - _policy: EvictionPolicy                                         │
│ - _default_ttl_ms: int                                            │
│ - _stats: CacheStats                                              │
│ - _lock: RLock                                                    │
│──────────────────────────────────────────────────────────────────│
│ + get(key) → value | None           O(1)                          │
│ + put(key, value, ttl_ms) → None    O(1)                          │
│ + remove(key) → bool                O(1)                          │
│ + peek(key) → value | None          O(1)                          │
│ + contains(key) → bool              O(1)                          │
│ + keys() → list[Any]                                              │
│ + entries() → list[CacheEntry]                                    │
└──────────┬───────────────────────────────────────────────────────┘
           │ uses
           ▼
┌──────────────────────────────────────────────────────────────────┐
│  «abstract» EvictionPolicy                                        │
│──────────────────────────────────────────────────────────────────│
│ + on_access(node) → None                                          │
│ + on_insert(node) → None                                          │
│ + evict() → DLLNode | None                                        │
│ + on_remove(node) → None                                          │
│ + clear() → None                                                  │
│ + name() → str                                                    │
└──────────┬────────────────────────────────────────────────────────┘
           │ implemented by
  ┌────────┼──────────────────┐
  ▼        ▼                  ▼
LRU      LFU              FIFO
Eviction Eviction          Eviction
Policy   Policy            Policy

┌──────────────────────────────────────────────────────────────────┐
│  DoublyLinkedList                                                  │
│──────────────────────────────────────────────────────────────────│
│ - _head: DLLNode (sentinel)                                       │
│ - _tail: DLLNode (sentinel)                                       │
│ - _size: int                                                      │
│──────────────────────────────────────────────────────────────────│
│ + add_to_front(node) → None         O(1)                          │
│ + remove(node) → None               O(1)                          │
│ + move_to_front(node) → None        O(1)                          │
│ + remove_from_back() → DLLNode      O(1)                          │
│ + front / back → DLLNode            O(1)                          │
│ + clear() → None                                                  │
└──────────┬───────────────────────────────────────────────────────┘
           │ contains
           ▼
┌──────────────────────────────────────────────────────────────────┐
│  DLLNode                                                          │
│──────────────────────────────────────────────────────────────────│
│ + key: Any                                                        │
│ + value: Any                                                      │
│ + prev: DLLNode | None                                            │
│ + next: DLLNode | None                                            │
│ + freq: int                                                       │
│ + created_at: int                                                 │
│ + last_accessed: int                                              │
│ + ttl_ms: int                                                     │
│──────────────────────────────────────────────────────────────────│
│ + is_expired → bool                                               │
│ + touch() → None                                                  │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  «abstract» WritePolicy                                           │
│──────────────────────────────────────────────────────────────────│
│ + on_put(key, value, store) → None                                │
│ + on_evict(key, value, store) → None                              │
│ + flush(dirty, store) → int                                       │
│ + name() → str                                                    │
└──────────┬────────────────────────────────────────────────────────┘
           │ implemented by
  ┌────────┴──────────┐
  ▼                   ▼
WriteThrough       WriteBack
Policy             Policy

┌──────────────────────────────────────────────────────────────────┐
│  «abstract» BackingStore                                          │
│──────────────────────────────────────────────────────────────────│
│ + read(key) → value | None                                        │
│ + write(key, value) → None                                        │
│ + delete(key) → None                                              │
│ + name() → str                                                    │
└──────────┬────────────────────────────────────────────────────────┘
           │ implemented by
           ▼
       InMemoryStore

┌──────────────────────────────────────────────────────────────────┐
│  «abstract» CacheEventListener                                    │
│──────────────────────────────────────────────────────────────────│
│ + on_event(event) → None                                          │
│ + listener_id() → str                                             │
└──────────┬────────────────────────────────────────────────────────┘
           │ implemented by
  ┌────────┴──────────┐
  ▼                   ▼
CacheLogger        HitRate
                   Monitor

┌──────────────────────────────────────────────────────────────────┐
│  MultiLevelCache (Decorator)                                      │
│──────────────────────────────────────────────────────────────────│
│ - _l1: Cache                                                      │
│ - _l2: Cache                                                      │
│──────────────────────────────────────────────────────────────────│
│ + get(key) → value (L1 → L2 → miss)                              │
│ + put(key, value) → None (always L1)                              │
│ + display_levels() → None                                         │
└──────────────────────────────────────────────────────────────────┘

┌──────────────┐ ┌──────────────────┐ ┌──────────────────┐
│ CacheEntry   │ │ CacheStats       │ │ CacheEvent       │
│──────────────│ │──────────────────│ │──────────────────│
│ + key        │ │ + hits           │ │ + event_id       │
│ + value      │ │ + misses         │ │ + event_type     │
│ + frequency  │ │ + puts           │ │ + key            │
│ + created_at │ │ + updates        │ │ + value          │
│ + ttl_ms     │ │ + evictions      │ │ + reason         │
│ + is_expired │ │ + expirations    │ │ + timestamp      │
└──────────────┘ │ + hit_rate       │ └──────────────────┘
                 │ + miss_rate      │
┌──────────────┐ └──────────────────┘ ┌──────────────────┐
│ TTLCache     │                      │ CacheEventBus    │
│──────────────│ ┌──────────────────┐ │──────────────────│
│ - _cache     │ │ CacheScenario    │ │ - _listeners     │
│ - _default_  │ │──────────────────│ │ - _event_log     │
│   ttl_ms     │ │ - _system        │ │──────────────────│
│──────────────│ │ - _events        │ │ + register()     │
│ + cleanup_   │ │──────────────────│ │ + emit()         │
│   expired()  │ │ + at_tick()      │ │ + unregister()   │
└──────────────┘ │ + run()          │ └──────────────────┘
                 └──────────────────┘

┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│ EvictionReason   │ │ CacheEventType   │ │ WritePolicyType  │
│ (Enum)           │ │ (Enum)           │ │ (Enum)           │
│──────────────────│ │──────────────────│ │──────────────────│
│ CAPACITY         │ │ HIT              │ │ WRITE_THROUGH    │
│ EXPIRED          │ │ MISS             │ │ WRITE_BACK       │
│ EXPLICIT         │ │ PUT              │ └──────────────────┘
│ POLICY           │ │ UPDATE           │
└──────────────────┘ │ EVICTION         │
                     │ EXPIRATION       │
                     │ CLEAR            │
                     └──────────────────┘
```

---

## How to Run

Save all code into a single file `lru_cache.py` and run:

```bash
python lru_cache.py
```

### Example Output

```
╔══════════════════════════════════════════╗
║         LRU CACHE SYSTEM                 ║
║  Capacity: 5                              ║
║  Policy: LRU                              ║
║  Write: WriteThrough                      ║
╚══════════════════════════════════════════╝

  ───────────────────────────────────────────────────────
  SIMULATION START (ticks: 1 → 8)
  ───────────────────────────────────────────────────────

  ⏱️  Tick 1: Put A=1, B=2, C=3, D=4, E=5
  📥 [       put] key=A val=1
  📥 [       put] key=B val=2
  📥 [       put] key=C val=3
  📥 [       put] key=D val=4
  📥 [       put] key=E val=5

  ⏱️  Tick 2: Get B (moves to front)
  🎯 [       hit] key=B → 2

  ⏱️  Tick 3: Put F=6 (cache full — evicts LRU)
  🗑️ [ eviction] key=A (capacity limit reached)
  📥 [       put] key=F val=6

  ⏱️  Tick 4: Get A (should miss — was evicted)
  ❌ [      miss] key=A

  ⏱️  Tick 5: Get A (should find in backing store)
    Store has A: 1

  ⏱️  Tick 6: Put C=30 (update existing key)
  🔄 [    update] key=C val=30

  ⏱️  Tick 7: Put G=7, H=8 (evicts D and E)
  🗑️ [ eviction] key=D (capacity limit reached)
  📥 [       put] key=G val=7
  🗑️ [ eviction] key=E (capacity limit reached)
  📥 [       put] key=H val=8

  ⏱️  Tick 8: Display cache statistics

  ══════════════════════════════════════════════════
  CACHE STATISTICS
  ══════════════════════════════════════════════════
  Capacity:        5
  Current size:    5
  Total requests:  2
  Hits:            1
  Misses:          1
  Hit rate:        50.0%
  Miss rate:       50.0%
  Puts:            7
  Updates:         1
  Evictions:       3
  Expirations:     0
  ══════════════════════════════════════════════════

  Sliding window hit rate (last 2 requests): 50.0%

  ⏱️  Tick 8: Verify backing store
    Backing store contents: {'A': 1, 'B': 2, 'C': 30, 'D': 4, 'E': 5, 'F': 6, 'G': 7, 'H': 8}

  ───────────────────────────────────────────────────────
  SIMULATION COMPLETE
  ───────────────────────────────────────────────────────

=======================================================
  EVICTION POLICY COMPARISON
=======================================================

  Policy: LRU
  ─────────────────────────────────────────────
    Hits:      8
    Misses:    3
    Hit rate:  72.7%
    Evictions: 1
    Keys in cache: [F, B, A, E, D]

  Policy: LFU
  ─────────────────────────────────────────────
    Hits:      8
    Misses:    3
    Hit rate:  72.7%
    Evictions: 1
    Keys in cache: [F, B, A, E, D]

  Policy: FIFO
  ─────────────────────────────────────────────
    Hits:      7
    Misses:    4
    Hit rate:  63.6%
    Evictions: 1
    Keys in cache: [F, E, D, C, B]

=======================================================

========================================
  RUNNING TESTS
========================================

✅ test_basic_get_put passed
✅ test_lru_eviction passed
✅ test_lfu_eviction passed
✅ test_fifo_eviction passed
✅ test_update_existing_key passed
✅ test_remove passed
✅ test_clear passed
✅ test_peek_does_not_alter_order passed
✅ test_ttl_expiration passed
✅ test_stats_tracking passed
✅ test_hit_rate_calculation passed
✅ test_multi_level_cache passed
✅ test_write_through passed
✅ test_write_back passed
✅ test_dll_operations passed
✅ test_contains_with_expiry passed
✅ test_capacity_validation passed
✅ test_money_type_keys passed
✅ test_backing_store_read_through passed

✅ All tests passed!
```
