# Design In-Memory File System – Low-Level Design Guide

![Bidur Sapkota](https://www.bidursapkota.com.np/images/gravatar.webp "Bidur Sapkota - Developer")&nbsp;[Bidur Sapkota](https://www.bidursapkota.com.np/)

## Table of Contents

1. [Requirements](#requirements)
2. [Key Design Patterns](#key-design-patterns)
3. [Core Enums & Constants](#core-enums--constants)
4. [File System Node (Composite Pattern)](#file-system-node-composite-pattern)
5. [File & Directory](#file--directory)
6. [Path Resolver](#path-resolver)
7. [File System Operations (Command Pattern)](#file-system-operations-command-pattern)
8. [Undo / Redo Manager](#undo--redo-manager)
9. [Search & Filtering (Strategy Pattern)](#search--filtering-strategy-pattern)
10. [File Watcher (Observer Pattern)](#file-watcher-observer-pattern)
11. [Permission System](#permission-system)
12. [File System Facade](#file-system-facade)
13. [Simulation Engine](#simulation-engine)
14. [Putting It All Together](#putting-it-all-together)
15. [Class Diagram](#class-diagram)
16. [How to Run](#how-to-run)

---

## Requirements

### Functional Requirements

| #  | Requirement                                                                                     |
| -- | ----------------------------------------------------------------------------------------------- |
| 1  | `mkdir(path)` creates a directory at the given path, including intermediate directories         |
| 2  | `touch(path)` creates an empty file at the given path                                           |
| 3  | `write(path, content)` writes content to a file, creating it if it does not exist               |
| 4  | `read(path)` returns the content of a file                                                      |
| 5  | `ls(path)` lists the contents of a directory                                                    |
| 6  | `rm(path)` removes a file or directory (recursive for directories)                              |
| 7  | `mv(src, dst)` moves a file or directory to a new location                                      |
| 8  | `cp(src, dst)` copies a file or directory to a new location                                     |
| 9  | `find(path, criteria)` searches for files matching a pattern, size, or extension                 |
| 10 | `undo() / redo()` undoes or redoes the last file system operation                               |
| 11 | Observer-based watcher — listeners react to create, modify, delete, and move events             |
| 12 | Permission system — read, write, execute permissions per node with ownership                     |
| 13 | Statistics — total files, directories, storage used, and tree depth                              |

### Non-Functional Requirements

- Clean OOP with SOLID principles.
- Extensible — new search strategies, commands, and event listeners can be added without modifying core logic.
- Simulatable — the system runs step-by-step for testing and demonstration.

---

## Key Design Patterns

| Pattern   | Where It Is Used                                  | Why                                                                                      |
| --------- | ------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| Composite | `FileNode` / `Directory` / `File` tree structure  | Directories and files share a common interface; directories contain children recursively  |
| Command   | `FSCommand` encapsulates each file system mutation | Operations are first-class objects — they can be executed, undone, and replayed           |
| Strategy  | `SearchStrategy` defines how `find` filters nodes | Swap between name glob, extension, size range, or custom filters without changing `find`  |
| Observer  | `FileWatcher` reacts to file system events         | Loggers, indexers, and sync agents react independently to mutations                      |
| Facade    | `FileSystem` exposes a unified API                 | Hides the complexity of path resolution, permissions, undo, events, and tree traversal   |

---

## Core Enums & Constants

```python
import time
import uuid
import copy
import fnmatch
from enum import Enum, Flag, auto
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from typing import Any, Callable
from collections import deque


class NodeType(Enum):
    FILE = "file"
    DIRECTORY = "directory"


class EventType(Enum):
    CREATED = "created"
    MODIFIED = "modified"
    DELETED = "deleted"
    MOVED = "moved"
    COPIED = "copied"
    PERMISSION_CHANGED = "permission_changed"


class Permission(Flag):
    NONE = 0
    READ = auto()                    # can read file content / list directory
    WRITE = auto()                   # can modify file / add children
    EXECUTE = auto()                 # can traverse directory / execute file
    READ_WRITE = READ | WRITE
    ALL = READ | WRITE | EXECUTE


# ------------------------------------------------------------------
# System constants
# ------------------------------------------------------------------

PATH_SEPARATOR = "/"
ROOT_PATH = "/"
DEFAULT_OWNER = "root"
DEFAULT_PERMISSION = Permission.ALL
MAX_UNDO_HISTORY = 100
STATS_DISPLAY_WIDTH = 55
```

`NodeType` distinguishes files from directories — every node in the tree is one or the other. `EventType` covers all mutations that the observer system can react to. `Permission` uses `Flag` instead of `Enum` so permissions can be combined with bitwise OR — `READ | WRITE` is a single value representing both. `auto()` assigns powers of 2 automatically (1, 2, 4), which is exactly what flags need. `MAX_UNDO_HISTORY` caps the undo stack at 100 to prevent unbounded memory growth.

---

## File System Node (Composite Pattern)

```python
class FileSystemNode(ABC):
    """
    Abstract base for all file system entries.

    Both files and directories implement this interface,
    enabling recursive tree operations. The composite pattern
    lets clients treat individual files and directory
    subtrees uniformly.

    Each node tracks:
        - name: the entry's base name (not the full path)
        - parent: reference to the containing directory
        - metadata: creation time, modification time, owner,
          permissions
    """

    def __init__(
        self,
        name: str,
        owner: str = DEFAULT_OWNER,
        permission: Permission = DEFAULT_PERMISSION,
    ) -> None:
        self.node_id: str = str(uuid.uuid4())[:8]
        self.name: str = name
        self.owner: str = owner
        self.permission: Permission = permission
        self.created_at: float = time.time()
        self.modified_at: float = self.created_at
        self.parent: "Directory | None" = None

    @property
    @abstractmethod
    def node_type(self) -> NodeType:
        ...

    @property
    @abstractmethod
    def size(self) -> int:
        """Size in bytes (file content length or recursive directory size)."""
        ...

    @property
    def path(self) -> str:
        """
        Compute the absolute path by walking up to the root.

        /root has no parent, so its path is "/".
        /root/home/user → "/home/user"
        """
        if self.parent is None:
            return ROOT_PATH
        parts: list[str] = []
        node: FileSystemNode = self
        while node.parent is not None:
            parts.append(node.name)
            node = node.parent
        return ROOT_PATH + "/".join(reversed(parts))

    @property
    def is_file(self) -> bool:
        return self.node_type == NodeType.FILE

    @property
    def is_directory(self) -> bool:
        return self.node_type == NodeType.DIRECTORY

    def touch(self) -> None:
        """Update the modification timestamp."""
        self.modified_at = time.time()

    @abstractmethod
    def deep_copy(self) -> "FileSystemNode":
        """Create a deep copy for cp operations."""
        ...

    def __repr__(self) -> str:
        return (f"{self.node_type.value}("
                f"name={self.name}, "
                f"size={self.size})")
```

`FileSystemNode` is the component in the composite pattern. `path` walks up via `parent` references to reconstruct the absolute path — this avoids storing redundant path strings that would desync during moves. `size` is abstract because files measure content length while directories sum their children recursively. `deep_copy` is needed for `cp` — we cannot share references between the original and the copy or mutations to one would affect the other. `touch` updates `modified_at`, mirroring Unix `touch` semantics.

---

## File & Directory

```python
class File(FileSystemNode):
    """
    A leaf node representing a file with content.

    Files store their content as a string. Size is the
    length of the content in bytes (UTF-8 encoded).
    Files have no children.
    """

    def __init__(
        self,
        name: str,
        content: str = "",
        owner: str = DEFAULT_OWNER,
        permission: Permission = DEFAULT_PERMISSION,
    ) -> None:
        super().__init__(name, owner, permission)
        self._content: str = content

    @property
    def node_type(self) -> NodeType:
        return NodeType.FILE

    @property
    def size(self) -> int:
        return len(self._content.encode("utf-8"))

    @property
    def content(self) -> str:
        return self._content

    @content.setter
    def content(self, value: str) -> None:
        self._content = value
        self.touch()

    def append(self, data: str) -> None:
        """Append data to the file content."""
        self._content += data
        self.touch()

    def truncate(self) -> None:
        """Clear the file content."""
        self._content = ""
        self.touch()

    def deep_copy(self) -> "File":
        new_file = File(
            name=self.name,
            content=self._content,
            owner=self.owner,
            permission=self.permission,
        )
        return new_file

    def __repr__(self) -> str:
        return (f"File(name={self.name}, "
                f"size={self.size}B)")


class Directory(FileSystemNode):
    """
    A composite node that contains children (files and
    other directories).

    Children are stored in a dict keyed by name for O(1)
    lookup. The directory's size is the sum of all
    descendants' sizes.
    """

    def __init__(
        self,
        name: str,
        owner: str = DEFAULT_OWNER,
        permission: Permission = DEFAULT_PERMISSION,
    ) -> None:
        super().__init__(name, owner, permission)
        self._children: dict[str, FileSystemNode] = {}

    @property
    def node_type(self) -> NodeType:
        return NodeType.DIRECTORY

    @property
    def size(self) -> int:
        return sum(child.size for child in self._children.values())

    @property
    def children(self) -> dict[str, FileSystemNode]:
        return dict(self._children)

    @property
    def child_count(self) -> int:
        return len(self._children)

    @property
    def is_empty(self) -> bool:
        return len(self._children) == 0

    def add_child(self, node: FileSystemNode) -> None:
        """
        Add a child node to this directory.

        Sets the child's parent reference to this directory.
        Overwrites if a child with the same name exists.
        """
        node.parent = self
        self._children[node.name] = node
        self.touch()

    def remove_child(self, name: str) -> FileSystemNode | None:
        """
        Remove and return a child by name.

        Returns None if no child with that name exists.
        Clears the child's parent reference.
        """
        child = self._children.pop(name, None)
        if child:
            child.parent = None
            self.touch()
        return child

    def get_child(self, name: str) -> FileSystemNode | None:
        return self._children.get(name)

    def has_child(self, name: str) -> bool:
        return name in self._children

    def list_names(self) -> list[str]:
        """Return sorted child names (like ls)."""
        return sorted(self._children.keys())

    def list_nodes(self) -> list[FileSystemNode]:
        """Return children sorted by name."""
        return [self._children[k]
                for k in sorted(self._children.keys())]

    def walk(self) -> list[FileSystemNode]:
        """
        Recursively yield all descendants (DFS).

        Returns a flat list of every node in the subtree
        rooted at this directory, including this directory.
        """
        result: list[FileSystemNode] = [self]
        for child in self._children.values():
            if child.is_directory:
                assert isinstance(child, Directory)
                result.extend(child.walk())
            else:
                result.append(child)
        return result

    @property
    def depth(self) -> int:
        """
        Maximum depth of the subtree rooted here.

        A leaf directory has depth 0. A directory containing
        only files has depth 0. A directory containing
        a subdirectory has depth >= 1.
        """
        max_child_depth = 0
        for child in self._children.values():
            if child.is_directory:
                assert isinstance(child, Directory)
                max_child_depth = max(
                    max_child_depth, child.depth + 1,
                )
        return max_child_depth

    def deep_copy(self) -> "Directory":
        new_dir = Directory(
            name=self.name,
            owner=self.owner,
            permission=self.permission,
        )
        for child in self._children.values():
            child_copy = child.deep_copy()
            new_dir.add_child(child_copy)
        return new_dir

    def tree(self, prefix: str = "", is_last: bool = True
             ) -> str:
        """
        Generate a tree visualization string.

        Produces output similar to the Unix `tree` command:
            /
            ├── home/
            │   └── user/
            │       └── hello.txt
            └── tmp/
        """
        connector = "└── " if is_last else "├── "
        if self.parent is None:
            line = f"{ROOT_PATH}\n"
        else:
            line = f"{prefix}{connector}{self.name}/\n"

        if self.parent is None:
            child_prefix = ""
        else:
            child_prefix = (prefix + ("    " if is_last
                                      else "│   "))

        names = sorted(self._children.keys())
        for i, name in enumerate(names):
            child = self._children[name]
            last = (i == len(names) - 1)
            if child.is_directory:
                assert isinstance(child, Directory)
                line += child.tree(child_prefix, last)
            else:
                conn = "└── " if last else "├── "
                line += f"{child_prefix}{conn}{name}\n"

        return line

    def __repr__(self) -> str:
        return (f"Directory(name={self.name}, "
                f"children={self.child_count})")
```

`File` is a leaf in the composite — it has content but no children. The `content` setter calls `touch()` automatically so every write updates `modified_at`. Size is computed as UTF-8 byte length, not character count, matching real file systems. `Directory` is the composite — it holds children in a `dict[str, FileSystemNode]` for O(1) name lookup. `add_child` sets the parent reference, maintaining the bidirectional link. `walk` performs DFS to flatten the subtree, useful for recursive operations like `rm -r` and `find`. `deep_copy` recursively clones the entire subtree — children are deep-copied and re-parented to the new directory. `tree` generates Unix-style tree output using box-drawing characters.

---

## Path Resolver

```python
class PathResolver:
    """
    Resolves string paths to FileSystemNode references.

    All paths are absolute (start with "/"). The resolver
    walks the tree from the root, following each path
    component through directory children.

    Path normalization:
        "/home//user/../user/./docs" → "/home/user/docs"
        Trailing slashes are stripped.
        ".." navigates to the parent directory.
        "." refers to the current directory.
    """

    def __init__(self, root: Directory) -> None:
        self._root = root

    def resolve(self, path: str) -> FileSystemNode | None:
        """
        Resolve a path to a node, or None if not found.

        Handles ".", "..", and normalizes multiple slashes.
        """
        parts = self._normalize(path)
        node: FileSystemNode = self._root

        for part in parts:
            if not node.is_directory:
                return None
            assert isinstance(node, Directory)
            child = node.get_child(part)
            if child is None:
                return None
            node = child

        return node

    def resolve_parent(self, path: str
                       ) -> tuple[Directory, str] | None:
        """
        Resolve the parent directory and the base name.

        Given "/home/user/file.txt", returns
        (Directory(/home/user), "file.txt").
        Returns None if the parent does not exist or
        is not a directory.
        """
        parts = self._normalize(path)
        if not parts:
            return None  # cannot get parent of root

        parent_parts = parts[:-1]
        base_name = parts[-1]

        node: FileSystemNode = self._root
        for part in parent_parts:
            if not node.is_directory:
                return None
            assert isinstance(node, Directory)
            child = node.get_child(part)
            if child is None:
                return None
            node = child

        if not node.is_directory:
            return None

        assert isinstance(node, Directory)
        return node, base_name

    def exists(self, path: str) -> bool:
        return self.resolve(path) is not None

    def _normalize(self, path: str) -> list[str]:
        """
        Normalize a path into a list of components.

        "/" → []
        "/home/user" → ["home", "user"]
        "/a/../b/./c" → ["b", "c"]
        """
        parts = path.strip(PATH_SEPARATOR).split(
            PATH_SEPARATOR
        )
        normalized: list[str] = []
        for part in parts:
            if part == "" or part == ".":
                continue
            elif part == "..":
                if normalized:
                    normalized.pop()
            else:
                normalized.append(part)
        return normalized

    def split_path(self, path: str) -> tuple[str, str]:
        """
        Split a path into (parent_path, base_name).

        "/home/user/file.txt" → ("/home/user", "file.txt")
        "/file.txt" → ("/", "file.txt")
        """
        parts = self._normalize(path)
        if not parts:
            return ROOT_PATH, ""
        base_name = parts[-1]
        if len(parts) == 1:
            parent_path = ROOT_PATH
        else:
            parent_path = (ROOT_PATH
                           + "/".join(parts[:-1]))
        return parent_path, base_name
```

`PathResolver` translates string paths into node references. `_normalize` handles `.` (current), `..` (parent), multiple slashes, and trailing slashes — the result is a clean list of path components. `resolve` walks the tree component by component, returning `None` on any miss. `resolve_parent` splits the path and resolves everything except the last component, returning `(parent_directory, base_name)` — this is used by `mkdir`, `touch`, and `mv` where we need the parent to insert or remove a child. `split_path` is a pure string operation that does not touch the tree, useful for constructing paths without resolution.

---

## File System Operations (Command Pattern)

```python
class FSCommand(ABC):
    """
    Encapsulates a file system mutation as a first-class object.

    Each command knows how to execute itself and undo itself.
    Commands are stored on the undo stack so the file system
    can reverse any sequence of operations.
    """

    @abstractmethod
    def execute(self) -> bool:
        """Execute the command. Returns True on success."""
        ...

    @abstractmethod
    def undo(self) -> bool:
        """Reverse the command. Returns True on success."""
        ...

    @abstractmethod
    def description(self) -> str:
        """Human-readable description of the command."""
        ...


class MkdirCommand(FSCommand):
    """
    Create a directory at the given path.

    Creates intermediate directories as needed (like mkdir -p).
    Undo removes only the directories that were actually
    created (not pre-existing ones).
    """

    def __init__(self, root: Directory,
                 resolver: PathResolver,
                 path: str) -> None:
        self._root = root
        self._resolver = resolver
        self._path = path
        self._created_parts: list[str] = []

    def execute(self) -> bool:
        parts = self._resolver._normalize(self._path)
        if not parts:
            return False  # root already exists

        node: FileSystemNode = self._root
        for part in parts:
            assert isinstance(node, Directory)
            child = node.get_child(part)
            if child is None:
                new_dir = Directory(name=part)
                node.add_child(new_dir)
                self._created_parts.append(part)
                node = new_dir
            elif child.is_directory:
                node = child
            else:
                return False  # path component is a file
        return True

    def undo(self) -> bool:
        if not self._created_parts:
            return False
        # Walk to the parent of the first created directory
        parts = self._resolver._normalize(self._path)
        start_idx = len(parts) - len(self._created_parts)
        node: FileSystemNode = self._root
        for part in parts[:start_idx]:
            assert isinstance(node, Directory)
            child = node.get_child(part)
            if child is None:
                return False
            node = child
        # Remove the first created directory (removes subtree)
        assert isinstance(node, Directory)
        first_created = self._created_parts[0]
        node.remove_child(first_created)
        self._created_parts.clear()
        return True

    def description(self) -> str:
        return f"mkdir {self._path}"


class TouchCommand(FSCommand):
    """
    Create an empty file at the given path.

    If the file already exists, updates its modification time.
    Undo removes the file if it was created, or restores
    the original modification time if it already existed.
    """

    def __init__(self, root: Directory,
                 resolver: PathResolver,
                 path: str) -> None:
        self._root = root
        self._resolver = resolver
        self._path = path
        self._was_created: bool = False
        self._old_modified_at: float = 0.0

    def execute(self) -> bool:
        existing = self._resolver.resolve(self._path)
        if existing is not None:
            if existing.is_file:
                self._old_modified_at = existing.modified_at
                existing.touch()
                self._was_created = False
                return True
            return False  # path exists as directory

        result = self._resolver.resolve_parent(self._path)
        if result is None:
            return False
        parent, name = result
        new_file = File(name=name)
        parent.add_child(new_file)
        self._was_created = True
        return True

    def undo(self) -> bool:
        if self._was_created:
            result = self._resolver.resolve_parent(
                self._path
            )
            if result is None:
                return False
            parent, name = result
            parent.remove_child(name)
            return True
        else:
            node = self._resolver.resolve(self._path)
            if node:
                node.modified_at = self._old_modified_at
                return True
            return False

    def description(self) -> str:
        return f"touch {self._path}"


class WriteCommand(FSCommand):
    """
    Write content to a file.

    Creates the file if it does not exist (like touch + write).
    Undo restores the previous content or removes the file
    if it was created by this command.
    """

    def __init__(self, root: Directory,
                 resolver: PathResolver,
                 path: str, content: str) -> None:
        self._root = root
        self._resolver = resolver
        self._path = path
        self._content = content
        self._old_content: str | None = None
        self._was_created: bool = False

    def execute(self) -> bool:
        node = self._resolver.resolve(self._path)
        if node is not None:
            if not node.is_file:
                return False
            assert isinstance(node, File)
            self._old_content = node.content
            node.content = self._content
            self._was_created = False
            return True

        # File does not exist — create it
        result = self._resolver.resolve_parent(self._path)
        if result is None:
            return False
        parent, name = result
        new_file = File(name=name, content=self._content)
        parent.add_child(new_file)
        self._was_created = True
        return True

    def undo(self) -> bool:
        if self._was_created:
            result = self._resolver.resolve_parent(
                self._path
            )
            if result is None:
                return False
            parent, name = result
            parent.remove_child(name)
            return True
        else:
            node = self._resolver.resolve(self._path)
            if node is None or not node.is_file:
                return False
            assert isinstance(node, File)
            if self._old_content is not None:
                node.content = self._old_content
            return True

    def description(self) -> str:
        preview = (self._content[:20] + "..."
                   if len(self._content) > 20
                   else self._content)
        return f"write {self._path} ({preview})"


class RemoveCommand(FSCommand):
    """
    Remove a file or directory (recursive).

    Stores the entire removed subtree so undo can
    restore it exactly, including all descendants.
    """

    def __init__(self, root: Directory,
                 resolver: PathResolver,
                 path: str) -> None:
        self._root = root
        self._resolver = resolver
        self._path = path
        self._removed_node: FileSystemNode | None = None
        self._parent_path: str = ""

    def execute(self) -> bool:
        node = self._resolver.resolve(self._path)
        if node is None or node is self._root:
            return False  # cannot remove root

        parent = node.parent
        if parent is None:
            return False

        self._parent_path = parent.path
        self._removed_node = node
        parent.remove_child(node.name)
        return True

    def undo(self) -> bool:
        if self._removed_node is None:
            return False
        parent_node = self._resolver.resolve(
            self._parent_path
        )
        if (parent_node is None
                or not parent_node.is_directory):
            return False
        assert isinstance(parent_node, Directory)
        parent_node.add_child(self._removed_node)
        self._removed_node = None
        return True

    def description(self) -> str:
        return f"rm {self._path}"


class MoveCommand(FSCommand):
    """
    Move a file or directory to a new location.

    Undo moves the node back to its original location.
    The node object is reused (not copied), so all
    references remain valid.
    """

    def __init__(self, root: Directory,
                 resolver: PathResolver,
                 src: str, dst: str) -> None:
        self._root = root
        self._resolver = resolver
        self._src = src
        self._dst = dst
        self._original_parent_path: str = ""
        self._original_name: str = ""

    def execute(self) -> bool:
        src_node = self._resolver.resolve(self._src)
        if src_node is None or src_node is self._root:
            return False

        src_parent = src_node.parent
        if src_parent is None:
            return False

        self._original_parent_path = src_parent.path
        self._original_name = src_node.name

        dst_result = self._resolver.resolve_parent(self._dst)
        if dst_result is None:
            return False
        dst_parent, dst_name = dst_result

        # Check if destination is inside source (would create cycle)
        if src_node.is_directory:
            check = dst_parent
            while check is not None:
                if check is src_node:
                    return False  # cycle
                check = check.parent

        src_parent.remove_child(src_node.name)
        src_node.name = dst_name
        dst_parent.add_child(src_node)
        return True

    def undo(self) -> bool:
        moved_node = self._resolver.resolve(self._dst)
        if moved_node is None:
            return False

        current_parent = moved_node.parent
        if current_parent is None:
            return False

        original_parent = self._resolver.resolve(
            self._original_parent_path
        )
        if (original_parent is None
                or not original_parent.is_directory):
            return False

        assert isinstance(original_parent, Directory)
        current_parent.remove_child(moved_node.name)
        moved_node.name = self._original_name
        original_parent.add_child(moved_node)
        return True

    def description(self) -> str:
        return f"mv {self._src} → {self._dst}"


class CopyCommand(FSCommand):
    """
    Copy a file or directory to a new location.

    Uses deep_copy to create an independent clone.
    Undo removes the copied node.
    """

    def __init__(self, root: Directory,
                 resolver: PathResolver,
                 src: str, dst: str) -> None:
        self._root = root
        self._resolver = resolver
        self._src = src
        self._dst = dst

    def execute(self) -> bool:
        src_node = self._resolver.resolve(self._src)
        if src_node is None:
            return False

        dst_result = self._resolver.resolve_parent(self._dst)
        if dst_result is None:
            return False
        dst_parent, dst_name = dst_result

        cloned = src_node.deep_copy()
        cloned.name = dst_name
        dst_parent.add_child(cloned)
        return True

    def undo(self) -> bool:
        result = self._resolver.resolve_parent(self._dst)
        if result is None:
            return False
        parent, name = result
        parent.remove_child(name)
        return True

    def description(self) -> str:
        return f"cp {self._src} → {self._dst}"
```

Each command encapsulates a single file system mutation. `MkdirCommand` tracks which directories it actually created (vs. already existing ones) so undo only removes what was added — this prevents undo from deleting a pre-existing `/home` just because the command traversed it. `WriteCommand` stores the old content before overwriting so undo can restore it byte-for-byte. `RemoveCommand` holds a reference to the removed subtree — since the node is just detached (not destroyed), undo can re-attach it. `MoveCommand` checks for cycles — moving `/a` into `/a/b` would create a self-referencing loop. It records the original parent path and name so undo can move the node back. `CopyCommand` uses `deep_copy` to create a fully independent clone; undo simply removes the copy.

---

## Undo / Redo Manager

```python
class UndoRedoManager:
    """
    Manages undo and redo stacks for file system commands.

    Every executed command is pushed onto the undo stack.
    Undoing pops from undo and pushes onto redo. Redoing
    pops from redo and pushes onto undo. A new command
    clears the redo stack (you cannot redo after a new
    mutation — the timeline has diverged).

    Stack capacity is capped at MAX_UNDO_HISTORY.
    """

    def __init__(self) -> None:
        self._undo_stack: deque[FSCommand] = deque(
            maxlen=MAX_UNDO_HISTORY,
        )
        self._redo_stack: deque[FSCommand] = deque(
            maxlen=MAX_UNDO_HISTORY,
        )

    def push(self, command: FSCommand) -> None:
        """Record an executed command and clear redo stack."""
        self._undo_stack.append(command)
        self._redo_stack.clear()

    def undo(self) -> FSCommand | None:
        """
        Undo the last command.

        Returns the undone command, or None if nothing to undo.
        """
        if not self._undo_stack:
            return None
        command = self._undo_stack.pop()
        if command.undo():
            self._redo_stack.append(command)
            return command
        # Undo failed — push back
        self._undo_stack.append(command)
        return None

    def redo(self) -> FSCommand | None:
        """
        Redo the last undone command.

        Returns the redone command, or None if nothing to redo.
        """
        if not self._redo_stack:
            return None
        command = self._redo_stack.pop()
        if command.execute():
            self._undo_stack.append(command)
            return command
        # Redo failed — push back
        self._redo_stack.append(command)
        return None

    @property
    def undo_count(self) -> int:
        return len(self._undo_stack)

    @property
    def redo_count(self) -> int:
        return len(self._redo_stack)

    @property
    def can_undo(self) -> bool:
        return len(self._undo_stack) > 0

    @property
    def can_redo(self) -> bool:
        return len(self._redo_stack) > 0

    def clear(self) -> None:
        self._undo_stack.clear()
        self._redo_stack.clear()

    def history(self) -> list[str]:
        """Return descriptions of all commands on the undo stack."""
        return [cmd.description() for cmd in self._undo_stack]

    def __repr__(self) -> str:
        return (f"UndoRedoManager("
                f"undo={len(self._undo_stack)}, "
                f"redo={len(self._redo_stack)})")
```

`UndoRedoManager` implements the classic dual-stack undo/redo model. `push` records a new command and clears the redo stack — once you make a new change, the old redo timeline is abandoned. This mirrors how undo works in text editors and IDEs. `deque(maxlen=MAX_UNDO_HISTORY)` automatically evicts the oldest entry when the stack exceeds capacity, preventing memory leaks from infinite undo history. If `undo()` fails (e.g., the file system state has changed in a way that prevents reversal), the command is pushed back and `None` is returned.

---

## Search & Filtering (Strategy Pattern)

```python
class SearchStrategy(ABC):
    """
    Defines how `find` filters file system nodes.

    Each strategy implements a `matches` predicate. The
    file system walks the tree and applies the strategy
    to each node. New search criteria (regex, content
    search, date range) can be added without modifying
    the tree traversal logic.
    """

    @abstractmethod
    def matches(self, node: FileSystemNode) -> bool:
        ...

    @abstractmethod
    def name(self) -> str:
        ...


class NameGlobStrategy(SearchStrategy):
    """
    Match nodes by name using glob patterns.

    Supports *, ?, and [seq] patterns via fnmatch.
    Example: "*.txt" matches all text files.
    """

    def __init__(self, pattern: str) -> None:
        self._pattern = pattern

    def matches(self, node: FileSystemNode) -> bool:
        return fnmatch.fnmatch(node.name, self._pattern)

    def name(self) -> str:
        return f"NameGlob({self._pattern})"


class ExtensionStrategy(SearchStrategy):
    """
    Match files by extension.

    Example: extension=".py" matches all Python files.
    Directories never match.
    """

    def __init__(self, extension: str) -> None:
        self._ext = (extension if extension.startswith(".")
                     else f".{extension}")

    def matches(self, node: FileSystemNode) -> bool:
        if not node.is_file:
            return False
        return node.name.endswith(self._ext)

    def name(self) -> str:
        return f"Extension({self._ext})"


class SizeRangeStrategy(SearchStrategy):
    """
    Match files within a size range (bytes).

    min_bytes=0 and max_bytes=inf matches all files.
    Directories never match.
    """

    def __init__(self, min_bytes: int = 0,
                 max_bytes: int = float("inf")) -> None:
        self._min = min_bytes
        self._max = max_bytes

    def matches(self, node: FileSystemNode) -> bool:
        if not node.is_file:
            return False
        return self._min <= node.size <= self._max

    def name(self) -> str:
        return (f"SizeRange({self._min}B–"
                f"{self._max}B)")


class NodeTypeStrategy(SearchStrategy):
    """
    Match only files or only directories.

    Useful for filtering `find` results to only one
    node type.
    """

    def __init__(self, node_type: NodeType) -> None:
        self._type = node_type

    def matches(self, node: FileSystemNode) -> bool:
        return node.node_type == self._type

    def name(self) -> str:
        return f"NodeType({self._type.value})"


class CompositeStrategy(SearchStrategy):
    """
    Combine multiple strategies with AND logic.

    A node must match ALL inner strategies to pass.
    """

    def __init__(self, *strategies: SearchStrategy) -> None:
        self._strategies = list(strategies)

    def matches(self, node: FileSystemNode) -> bool:
        return all(s.matches(node)
                   for s in self._strategies)

    def name(self) -> str:
        names = " AND ".join(s.name()
                             for s in self._strategies)
        return f"Composite({names})"
```

`SearchStrategy` is the strategy interface — each implementation defines a single `matches` predicate. `NameGlobStrategy` uses `fnmatch` for Unix-style wildcard matching (`*.txt`, `data_???.csv`). `ExtensionStrategy` filters by file extension, normalizing the dot prefix. `SizeRangeStrategy` filters by byte size range — useful for finding large files. `NodeTypeStrategy` filters by file vs. directory. `CompositeStrategy` combines multiple strategies with AND logic — for example, `CompositeStrategy(ExtensionStrategy(".log"), SizeRangeStrategy(min_bytes=1024))` finds log files larger than 1KB. Adding a new strategy (regex, content grep, date range) requires only implementing `matches` and `name`.

---

## File Watcher (Observer Pattern)

```python
@dataclass
class FileSystemEvent:
    """An event emitted when the file system changes."""
    event_id: str
    event_type: EventType
    path: str
    detail: str = ""
    timestamp: float = field(default_factory=time.time)


class FileEventListener(ABC):
    """
    Observer that reacts to file system events.

    Implementations receive all events and decide which
    ones to handle. Listeners are decoupled from the file
    system — they cannot affect mutations.
    """

    @abstractmethod
    def on_event(self, event: FileSystemEvent) -> None:
        ...

    @abstractmethod
    def listener_id(self) -> str:
        ...


class FileSystemLogger(FileEventListener):
    """Logs all file system events to the console."""

    def on_event(self, event: FileSystemEvent) -> None:
        emoji = {
            EventType.CREATED: "📄",
            EventType.MODIFIED: "✏️ ",
            EventType.DELETED: "🗑️ ",
            EventType.MOVED: "📦",
            EventType.COPIED: "📋",
            EventType.PERMISSION_CHANGED: "🔒",
        }
        icon = emoji.get(event.event_type, "📌")
        print(f"  {icon} [{event.event_type.value:>20}] "
              f"{event.path} {event.detail}")

    def listener_id(self) -> str:
        return "fs-logger"


class FileSystemMetrics(FileEventListener):
    """Collects operation counts per event type."""

    def __init__(self) -> None:
        self._counts: dict[EventType, int] = {
            et: 0 for et in EventType
        }

    def on_event(self, event: FileSystemEvent) -> None:
        self._counts[event.event_type] = (
            self._counts.get(event.event_type, 0) + 1
        )

    @property
    def total_operations(self) -> int:
        return sum(self._counts.values())

    def get_count(self, event_type: EventType) -> int:
        return self._counts.get(event_type, 0)

    def listener_id(self) -> str:
        return "fs-metrics"

    def display(self) -> None:
        print(f"\n  {'─'*STATS_DISPLAY_WIDTH}")
        print(f"  FILE SYSTEM METRICS")
        print(f"  {'─'*STATS_DISPLAY_WIDTH}")
        for et, count in self._counts.items():
            if count > 0:
                print(f"  {et.value:<22} {count}")
        print(f"  {'─'*STATS_DISPLAY_WIDTH}")
        print(f"  Total operations: "
              f"{self.total_operations}")
        print(f"  {'─'*STATS_DISPLAY_WIDTH}")


class FileSystemEventBus:
    """
    Central event dispatcher for file system events.

    Manages listener registration and broadcasts events
    to all registered listeners. Thread-safe for
    concurrent emission is not needed here since the
    file system is single-threaded, but the pattern
    remains consistent.
    """

    def __init__(self) -> None:
        self._listeners: list[FileEventListener] = []
        self._event_log: list[FileSystemEvent] = []

    def register(self, listener: FileEventListener) -> None:
        self._listeners.append(listener)

    def unregister(self, listener_id: str) -> None:
        self._listeners = [
            l for l in self._listeners
            if l.listener_id() != listener_id
        ]

    def emit(self, event: FileSystemEvent) -> None:
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
    def event_log(self) -> list[FileSystemEvent]:
        return list(self._event_log)
```

`FileEventListener` is the observer interface. `FileSystemLogger` prints every event with emoji-coded log lines — creates, writes, deletes, and moves are visually distinct. `FileSystemMetrics` tracks operation counts per event type for operational monitoring. `FileSystemEventBus` dispatches events to all registered listeners, isolating each listener from failures in others. Adding a new listener (e.g., a search indexer that rebuilds on file creation, or a sync agent that mirrors changes) requires only implementing `on_event` and `listener_id`.

---

## Permission System

```python
class PermissionManager:
    """
    Enforces read, write, and execute permissions on nodes.

    Each node stores its own Permission flags. The manager
    provides check methods that raise PermissionError on
    violations. The file system calls these before every
    read, write, or traversal operation.

    Permission semantics:
        READ: read file content, list directory
        WRITE: modify file, add/remove children
        EXECUTE: traverse into a directory (cd)
    """

    def check_read(self, node: FileSystemNode) -> bool:
        """Check if the node has read permission."""
        if Permission.READ not in node.permission:
            return False
        return True

    def check_write(self, node: FileSystemNode) -> bool:
        """Check if the node has write permission."""
        if Permission.WRITE not in node.permission:
            return False
        return True

    def check_execute(self, node: FileSystemNode) -> bool:
        """Check if the node has execute permission."""
        if Permission.EXECUTE not in node.permission:
            return False
        return True

    def set_permission(self, node: FileSystemNode,
                       permission: Permission) -> None:
        """Set the permission flags on a node."""
        node.permission = permission
        node.touch()

    def set_owner(self, node: FileSystemNode,
                  owner: str) -> None:
        """Change the owner of a node."""
        node.owner = owner
        node.touch()

    @staticmethod
    def permission_string(perm: Permission) -> str:
        """
        Convert permission flags to rwx string.

        ALL → "rwx"
        READ_WRITE → "rw-"
        READ → "r--"
        NONE → "---"
        """
        r = "r" if Permission.READ in perm else "-"
        w = "w" if Permission.WRITE in perm else "-"
        x = "x" if Permission.EXECUTE in perm else "-"
        return f"{r}{w}{x}"
```

`PermissionManager` keeps permission logic separate from the nodes themselves — nodes store the flags, the manager enforces them. `permission_string` converts the binary flags to the familiar Unix `rwx` format. The file system facade calls `check_read` before `read`, `check_write` before `write/mkdir/rm`, and `check_execute` before traversing directories. This separation means we can swap in a more complex permission model (ACLs, role-based) without changing node classes.

---

## File System Facade

```python
class FileSystemStats:
    """Aggregate file system statistics."""

    def __init__(self, root: Directory) -> None:
        self._root = root

    @property
    def total_files(self) -> int:
        return sum(1 for n in self._root.walk()
                   if n.is_file)

    @property
    def total_directories(self) -> int:
        return sum(1 for n in self._root.walk()
                   if n.is_directory)

    @property
    def total_size(self) -> int:
        return self._root.size

    @property
    def max_depth(self) -> int:
        return self._root.depth

    def display(self) -> None:
        print(f"\n  {'═'*STATS_DISPLAY_WIDTH}")
        print(f"  FILE SYSTEM STATISTICS")
        print(f"  {'═'*STATS_DISPLAY_WIDTH}")
        print(f"  Files:           {self.total_files}")
        print(f"  Directories:     {self.total_directories}")
        print(f"  Total size:      {self.total_size}B")
        print(f"  Max depth:       {self.max_depth}")
        print(f"  {'═'*STATS_DISPLAY_WIDTH}")


class FileSystem:
    """
    Facade that orchestrates the entire in-memory file system.

    Provides a simple API that hides the complexity of
    path resolution, permission checks, command execution,
    undo/redo, event emission, and tree management.
    """

    def __init__(self) -> None:
        self._root = Directory(name="")
        self._resolver = PathResolver(self._root)
        self._permissions = PermissionManager()
        self._undo_redo = UndoRedoManager()
        self._event_bus = FileSystemEventBus()
        self._logger = FileSystemLogger()
        self._metrics = FileSystemMetrics()
        self._event_bus.register(self._logger)
        self._event_bus.register(self._metrics)

    # ------------------------------------------------------------------
    # Core operations
    # ------------------------------------------------------------------

    def mkdir(self, path: str) -> bool:
        """
        Create a directory at the given path (mkdir -p).

        Creates all intermediate directories that do not
        exist. Returns True on success.
        """
        cmd = MkdirCommand(self._root, self._resolver,
                           path)
        success = cmd.execute()
        if success:
            self._undo_redo.push(cmd)
            self._emit(EventType.CREATED, path,
                       "directory created")
        return success

    def touch(self, path: str) -> bool:
        """
        Create an empty file or update modification time.

        Returns True on success.
        """
        cmd = TouchCommand(self._root, self._resolver,
                           path)
        success = cmd.execute()
        if success:
            self._undo_redo.push(cmd)
            if cmd._was_created:
                self._emit(EventType.CREATED, path,
                           "file created")
            else:
                self._emit(EventType.MODIFIED, path,
                           "timestamp updated")
        return success

    def write(self, path: str, content: str) -> bool:
        """
        Write content to a file (creates if needed).

        Returns True on success.
        """
        cmd = WriteCommand(self._root, self._resolver,
                           path, content)
        success = cmd.execute()
        if success:
            self._undo_redo.push(cmd)
            if cmd._was_created:
                self._emit(EventType.CREATED, path,
                           f"{len(content)}B written")
            else:
                self._emit(EventType.MODIFIED, path,
                           f"{len(content)}B written")
        return success

    def read(self, path: str) -> str | None:
        """
        Read the content of a file.

        Returns None if the path does not exist or
        is not a file.
        """
        node = self._resolver.resolve(path)
        if node is None or not node.is_file:
            return None
        if not self._permissions.check_read(node):
            print(f"  ⛔ Permission denied: "
                  f"read {path}")
            return None
        assert isinstance(node, File)
        return node.content

    def ls(self, path: str = ROOT_PATH
           ) -> list[str] | None:
        """
        List the contents of a directory.

        Returns sorted child names, or None if the
        path does not exist or is not a directory.
        """
        node = self._resolver.resolve(path)
        if node is None or not node.is_directory:
            return None
        if not self._permissions.check_read(node):
            print(f"  ⛔ Permission denied: "
                  f"ls {path}")
            return None
        assert isinstance(node, Directory)
        return node.list_names()

    def rm(self, path: str) -> bool:
        """
        Remove a file or directory (recursive).

        Returns True on success. Cannot remove root.
        """
        cmd = RemoveCommand(self._root, self._resolver,
                            path)
        success = cmd.execute()
        if success:
            self._undo_redo.push(cmd)
            self._emit(EventType.DELETED, path,
                       "removed")
        return success

    def mv(self, src: str, dst: str) -> bool:
        """
        Move a file or directory to a new location.

        Returns True on success.
        """
        cmd = MoveCommand(self._root, self._resolver,
                          src, dst)
        success = cmd.execute()
        if success:
            self._undo_redo.push(cmd)
            self._emit(EventType.MOVED, src,
                       f"→ {dst}")
        return success

    def cp(self, src: str, dst: str) -> bool:
        """
        Copy a file or directory (deep copy).

        Returns True on success.
        """
        cmd = CopyCommand(self._root, self._resolver,
                          src, dst)
        success = cmd.execute()
        if success:
            self._undo_redo.push(cmd)
            self._emit(EventType.COPIED, src,
                       f"→ {dst}")
        return success

    def find(self, path: str = ROOT_PATH,
             strategy: SearchStrategy | None = None,
             ) -> list[str]:
        """
        Search for nodes matching a strategy.

        Walks the subtree at `path` and returns paths
        of all matching nodes.
        """
        node = self._resolver.resolve(path)
        if node is None or not node.is_directory:
            return []
        assert isinstance(node, Directory)

        results: list[str] = []
        for descendant in node.walk():
            if descendant is node:
                continue  # skip the root of the search
            if strategy is None or strategy.matches(
                    descendant):
                results.append(descendant.path)
        return sorted(results)

    # ------------------------------------------------------------------
    # Undo / Redo
    # ------------------------------------------------------------------

    def undo(self) -> str | None:
        """
        Undo the last operation.

        Returns a description of the undone command,
        or None if nothing to undo.
        """
        cmd = self._undo_redo.undo()
        if cmd:
            return f"↩ undone: {cmd.description()}"
        return None

    def redo(self) -> str | None:
        """
        Redo the last undone operation.

        Returns a description of the redone command,
        or None if nothing to redo.
        """
        cmd = self._undo_redo.redo()
        if cmd:
            return f"↪ redone: {cmd.description()}"
        return None

    # ------------------------------------------------------------------
    # Permissions
    # ------------------------------------------------------------------

    def chmod(self, path: str,
              permission: Permission) -> bool:
        """Set permissions on a node."""
        node = self._resolver.resolve(path)
        if node is None:
            return False
        self._permissions.set_permission(node, permission)
        perm_str = PermissionManager.permission_string(
            permission
        )
        self._emit(EventType.PERMISSION_CHANGED, path,
                   f"→ {perm_str}")
        return True

    def chown(self, path: str, owner: str) -> bool:
        """Change the owner of a node."""
        node = self._resolver.resolve(path)
        if node is None:
            return False
        self._permissions.set_owner(node, owner)
        return True

    # ------------------------------------------------------------------
    # Inspection
    # ------------------------------------------------------------------

    def exists(self, path: str) -> bool:
        return self._resolver.exists(path)

    def is_file(self, path: str) -> bool:
        node = self._resolver.resolve(path)
        return node is not None and node.is_file

    def is_dir(self, path: str) -> bool:
        node = self._resolver.resolve(path)
        return node is not None and node.is_directory

    def size(self, path: str) -> int:
        node = self._resolver.resolve(path)
        if node is None:
            return 0
        return node.size

    def tree(self, path: str = ROOT_PATH) -> str:
        """Return a tree visualization of the file system."""
        node = self._resolver.resolve(path)
        if node is None or not node.is_directory:
            return ""
        assert isinstance(node, Directory)
        return node.tree()

    @property
    def stats(self) -> FileSystemStats:
        return FileSystemStats(self._root)

    @property
    def metrics(self) -> FileSystemMetrics:
        return self._metrics

    @property
    def event_bus(self) -> FileSystemEventBus:
        return self._event_bus

    @property
    def undo_redo(self) -> UndoRedoManager:
        return self._undo_redo

    def history(self) -> list[str]:
        """Return the command history."""
        return self._undo_redo.history()

    def display_status(self) -> None:
        self.stats.display()
        self._metrics.display()
        print(f"\n  Undo stack: "
              f"{self._undo_redo.undo_count} commands")
        print(f"  Redo stack: "
              f"{self._undo_redo.redo_count} commands")

    # ------------------------------------------------------------------
    # Internal helpers
    # ------------------------------------------------------------------

    def _emit(self, event_type: EventType,
              path: str, detail: str = "") -> None:
        event = FileSystemEvent(
            event_id=str(uuid.uuid4())[:8],
            event_type=event_type,
            path=path,
            detail=detail,
        )
        self._event_bus.emit(event)
```

`FileSystem` is the facade — it wires together the root directory, path resolver, permission manager, undo/redo manager, event bus, logger, and metrics collector. `mkdir` creates a `MkdirCommand`, executes it, pushes it onto the undo stack on success, and emits a `CREATED` event. Each operation follows this same pattern: create command → execute → record for undo → emit event. `read` and `ls` are queries, not mutations, so they don't create commands or emit events. `find` delegates to the tree's `walk` method and applies the search strategy as a filter. Users interact with `mkdir`, `touch`, `write`, `read`, `ls`, `rm`, `mv`, `cp`, `find`, `undo`, and `redo` without knowing about path resolution, command objects, or event dispatching.

---

## Simulation Engine

```python
class FSScenario:
    """
    Tick-based simulation engine for file system demos.

    Each tick represents a logical time step. Actions are
    registered at specific ticks and execute in order when
    the simulation runs.
    """

    def __init__(self, fs: FileSystem) -> None:
        self._fs = fs
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

`FSScenario` mirrors the simulation engine from other guides. Actions are registered at specific ticks and execute in order, making demos reproducible and readable.

---

## Putting It All Together

```python
def main_scenario() -> None:
    """Run the main in-memory file system demonstration."""

    print("\n╔══════════════════════════════════════════╗")
    print("║      IN-MEMORY FILE SYSTEM               ║")
    print("║  Composite + Command + Strategy + Observer║")
    print("╚══════════════════════════════════════════╝")

    fs = FileSystem()
    scenario = FSScenario(fs)

    # --- Create directory structure ---
    scenario.at_tick(1, "Create directory structure",
                     lambda: (
                         fs.mkdir("/home/user/docs"),
                         fs.mkdir("/home/user/pics"),
                         fs.mkdir("/var/log"),
                         fs.mkdir("/tmp"),
                     ))

    # --- Create and write files ---
    scenario.at_tick(2, "Create files with content",
                     lambda: (
                         fs.write("/home/user/docs/readme.txt",
                                  "Hello, File System!"),
                         fs.write("/home/user/docs/notes.md",
                                  "# My Notes\n- Item 1\n- Item 2"),
                         fs.write("/home/user/pics/photo.jpg",
                                  "<binary-data>"),
                         fs.write("/var/log/system.log",
                                  "2024-01-01 INFO Boot complete"),
                         fs.write("/tmp/scratch.txt",
                                  "temporary data"),
                     ))

    # --- List and read ---
    scenario.at_tick(3, "List /home/user/docs",
                     lambda: fs.ls("/home/user/docs"))

    scenario.at_tick(3, "Read readme.txt",
                     lambda: fs.read(
                         "/home/user/docs/readme.txt"))

    # --- Display tree ---
    scenario.at_tick(4, "Display file system tree",
                     lambda: print(fs.tree()))

    # --- Search files ---
    scenario.at_tick(5, "Find all .txt files",
                     lambda: fs.find(
                         "/",
                         ExtensionStrategy(".txt"),
                     ))

    scenario.at_tick(5, "Find all files > 10 bytes",
                     lambda: fs.find(
                         "/",
                         SizeRangeStrategy(min_bytes=10),
                     ))

    scenario.at_tick(5, "Find by name glob 'sys*'",
                     lambda: fs.find(
                         "/",
                         NameGlobStrategy("sys*"),
                     ))

    # --- Copy and move ---
    scenario.at_tick(6, "Copy readme.txt to /tmp/readme_backup.txt",
                     lambda: fs.cp(
                         "/home/user/docs/readme.txt",
                         "/tmp/readme_backup.txt",
                     ))

    scenario.at_tick(7, "Move scratch.txt to /home/user/docs/old.txt",
                     lambda: fs.mv(
                         "/tmp/scratch.txt",
                         "/home/user/docs/old.txt",
                     ))

    # --- Undo / Redo ---
    scenario.at_tick(8, "Undo last move",
                     lambda: fs.undo())

    scenario.at_tick(8, "Redo the move",
                     lambda: fs.redo())

    # --- Remove ---
    scenario.at_tick(9, "Remove /tmp directory",
                     lambda: fs.rm("/tmp"))

    scenario.at_tick(9, "Undo remove",
                     lambda: fs.undo())

    # --- Permissions ---
    scenario.at_tick(10, "Set readme.txt to read-only",
                      lambda: fs.chmod(
                          "/home/user/docs/readme.txt",
                          Permission.READ,
                      ))

    # --- Display stats ---
    scenario.at_tick(11, "Display final status",
                      lambda: fs.display_status())

    scenario.at_tick(11, "Display tree",
                      lambda: print(fs.tree()))

    scenario.at_tick(11, "Display command history",
                      lambda: print(
                          "\n  Command history:\n  " +
                          "\n  ".join(fs.history())))

    scenario.run()


def demo_undo_redo() -> None:
    """Demonstrate undo/redo functionality."""

    print("\n" + "=" * STATS_DISPLAY_WIDTH)
    print("  UNDO / REDO DEMO")
    print("=" * STATS_DISPLAY_WIDTH)

    fs = FileSystem()

    # Build up some state
    fs.mkdir("/project/src")
    fs.write("/project/src/main.py",
             "print('hello world')")
    fs.write("/project/src/utils.py",
             "def helper(): pass")

    print(f"\n  After initial setup:")
    print(fs.tree())

    # Undo the last write
    result = fs.undo()
    print(f"\n  {result}")
    print(f"  utils.py exists? "
          f"{fs.exists('/project/src/utils.py')}")

    # Undo the first write
    result = fs.undo()
    print(f"  {result}")
    print(f"  main.py exists? "
          f"{fs.exists('/project/src/main.py')}")

    # Redo both
    result = fs.redo()
    print(f"\n  {result}")
    result = fs.redo()
    print(f"  {result}")

    print(f"\n  After redo:")
    print(fs.tree())

    print("=" * STATS_DISPLAY_WIDTH)


def demo_search() -> None:
    """Demonstrate search strategies."""

    print("\n" + "=" * STATS_DISPLAY_WIDTH)
    print("  SEARCH STRATEGIES DEMO")
    print("=" * STATS_DISPLAY_WIDTH)

    fs = FileSystem()

    fs.mkdir("/project/src")
    fs.mkdir("/project/tests")
    fs.mkdir("/project/docs")
    fs.write("/project/src/app.py",
             "# main application")
    fs.write("/project/src/models.py",
             "# database models")
    fs.write("/project/tests/test_app.py",
             "# tests for app")
    fs.write("/project/docs/README.md",
             "# Project Documentation")
    fs.write("/project/docs/CHANGELOG.md",
             "# v1.0.0")
    fs.write("/project/config.json",
             '{"debug": true}')

    strategies = [
        ("Python files",
         ExtensionStrategy(".py")),
        ("Markdown files",
         ExtensionStrategy(".md")),
        ("Files with 'test' in name",
         NameGlobStrategy("test*")),
        ("Files 10–50 bytes",
         SizeRangeStrategy(10, 50)),
        ("Only directories",
         NodeTypeStrategy(NodeType.DIRECTORY)),
        ("Python files > 15 bytes (composite)",
         CompositeStrategy(
             ExtensionStrategy(".py"),
             SizeRangeStrategy(min_bytes=15),
         )),
    ]

    for label, strategy in strategies:
        results = fs.find("/", strategy)
        print(f"\n  {label} [{strategy.name()}]:")
        for r in results:
            print(f"    {r}")

    print("\n" + "=" * STATS_DISPLAY_WIDTH)


def demo_permissions() -> None:
    """Demonstrate the permission system."""

    print("\n" + "=" * STATS_DISPLAY_WIDTH)
    print("  PERMISSIONS DEMO")
    print("=" * STATS_DISPLAY_WIDTH)

    fs = FileSystem()

    fs.mkdir("/secure")
    fs.write("/secure/secret.txt",
             "top secret content")

    # Read before permission change
    content = fs.read("/secure/secret.txt")
    print(f"\n  Before chmod: {content}")

    # Remove read permission
    fs.chmod("/secure/secret.txt", Permission.NONE)

    # Try to read
    content = fs.read("/secure/secret.txt")
    print(f"  After chmod (NONE): {content}")

    # Restore read permission
    fs.chmod("/secure/secret.txt",
             Permission.READ_WRITE)
    content = fs.read("/secure/secret.txt")
    print(f"  After chmod (RW): {content}")

    print("\n" + "=" * STATS_DISPLAY_WIDTH)


# ------------------------------------------------------------------
# Unit Tests
# ------------------------------------------------------------------

def test_mkdir() -> None:
    """Test directory creation."""
    fs = FileSystem()
    assert fs.mkdir("/home/user/docs")
    assert fs.exists("/home")
    assert fs.exists("/home/user")
    assert fs.exists("/home/user/docs")
    assert fs.is_dir("/home/user/docs")
    print("✅ test_mkdir passed")


def test_touch() -> None:
    """Test file creation."""
    fs = FileSystem()
    fs.mkdir("/home")
    assert fs.touch("/home/file.txt")
    assert fs.exists("/home/file.txt")
    assert fs.is_file("/home/file.txt")
    print("✅ test_touch passed")


def test_write_and_read() -> None:
    """Test writing and reading file content."""
    fs = FileSystem()
    fs.mkdir("/data")
    assert fs.write("/data/hello.txt",
                    "Hello, World!")
    content = fs.read("/data/hello.txt")
    assert content == "Hello, World!"
    print("✅ test_write_and_read passed")


def test_write_creates_file() -> None:
    """Test that write creates a file if it does not exist."""
    fs = FileSystem()
    fs.mkdir("/data")
    assert fs.write("/data/new.txt", "new content")
    assert fs.exists("/data/new.txt")
    assert fs.read("/data/new.txt") == "new content"
    print("✅ test_write_creates_file passed")


def test_write_overwrites() -> None:
    """Test that write overwrites existing content."""
    fs = FileSystem()
    fs.mkdir("/data")
    fs.write("/data/file.txt", "old")
    fs.write("/data/file.txt", "new")
    assert fs.read("/data/file.txt") == "new"
    print("✅ test_write_overwrites passed")


def test_ls() -> None:
    """Test listing directory contents."""
    fs = FileSystem()
    fs.mkdir("/home")
    fs.touch("/home/a.txt")
    fs.touch("/home/b.txt")
    fs.mkdir("/home/sub")
    result = fs.ls("/home")
    assert result == ["a.txt", "b.txt", "sub"]
    print("✅ test_ls passed")


def test_rm_file() -> None:
    """Test removing a file."""
    fs = FileSystem()
    fs.mkdir("/data")
    fs.touch("/data/file.txt")
    assert fs.rm("/data/file.txt")
    assert not fs.exists("/data/file.txt")
    print("✅ test_rm_file passed")


def test_rm_directory() -> None:
    """Test removing a directory recursively."""
    fs = FileSystem()
    fs.mkdir("/data/sub/deep")
    fs.touch("/data/sub/file.txt")
    assert fs.rm("/data")
    assert not fs.exists("/data")
    assert not fs.exists("/data/sub")
    assert not fs.exists("/data/sub/file.txt")
    print("✅ test_rm_directory passed")


def test_rm_root_fails() -> None:
    """Test that removing root fails."""
    fs = FileSystem()
    assert not fs.rm("/")
    print("✅ test_rm_root_fails passed")


def test_mv() -> None:
    """Test moving a file."""
    fs = FileSystem()
    fs.mkdir("/src")
    fs.mkdir("/dst")
    fs.write("/src/file.txt", "data")
    assert fs.mv("/src/file.txt", "/dst/moved.txt")
    assert not fs.exists("/src/file.txt")
    assert fs.exists("/dst/moved.txt")
    assert fs.read("/dst/moved.txt") == "data"
    print("✅ test_mv passed")


def test_mv_directory() -> None:
    """Test moving a directory."""
    fs = FileSystem()
    fs.mkdir("/a/sub")
    fs.touch("/a/sub/file.txt")
    fs.mkdir("/b")
    assert fs.mv("/a/sub", "/b/sub")
    assert not fs.exists("/a/sub")
    assert fs.exists("/b/sub")
    assert fs.exists("/b/sub/file.txt")
    print("✅ test_mv_directory passed")


def test_cp() -> None:
    """Test copying a file."""
    fs = FileSystem()
    fs.mkdir("/src")
    fs.mkdir("/dst")
    fs.write("/src/file.txt", "copy me")
    assert fs.cp("/src/file.txt", "/dst/copy.txt")
    assert fs.exists("/src/file.txt")
    assert fs.exists("/dst/copy.txt")
    assert fs.read("/dst/copy.txt") == "copy me"
    # Verify deep copy (modifying original does not affect copy)
    fs.write("/src/file.txt", "modified")
    assert fs.read("/dst/copy.txt") == "copy me"
    print("✅ test_cp passed")


def test_cp_directory() -> None:
    """Test copying a directory recursively."""
    fs = FileSystem()
    fs.mkdir("/src/sub")
    fs.write("/src/sub/file.txt", "nested")
    fs.mkdir("/dst")
    assert fs.cp("/src", "/dst/src_copy")
    assert fs.exists("/dst/src_copy/sub/file.txt")
    assert fs.read("/dst/src_copy/sub/file.txt") == "nested"
    print("✅ test_cp_directory passed")


def test_find_by_extension() -> None:
    """Test finding files by extension."""
    fs = FileSystem()
    fs.mkdir("/project")
    fs.write("/project/app.py", "code")
    fs.write("/project/readme.md", "docs")
    fs.write("/project/style.css", "styles")
    results = fs.find("/", ExtensionStrategy(".py"))
    assert len(results) == 1
    assert "/project/app.py" in results
    print("✅ test_find_by_extension passed")


def test_find_by_name_glob() -> None:
    """Test finding files by name glob pattern."""
    fs = FileSystem()
    fs.mkdir("/logs")
    fs.write("/logs/app.log", "log1")
    fs.write("/logs/error.log", "log2")
    fs.write("/logs/debug.txt", "debug")
    results = fs.find("/", NameGlobStrategy("*.log"))
    assert len(results) == 2
    print("✅ test_find_by_name_glob passed")


def test_find_by_size() -> None:
    """Test finding files by size range."""
    fs = FileSystem()
    fs.mkdir("/data")
    fs.write("/data/small.txt", "hi")       # 2 bytes
    fs.write("/data/big.txt", "x" * 100)    # 100 bytes
    results = fs.find(
        "/", SizeRangeStrategy(min_bytes=50),
    )
    assert len(results) == 1
    assert "/data/big.txt" in results
    print("✅ test_find_by_size passed")


def test_find_composite() -> None:
    """Test composite search strategy."""
    fs = FileSystem()
    fs.mkdir("/project")
    fs.write("/project/app.py", "x" * 20)
    fs.write("/project/tiny.py", "x")
    fs.write("/project/readme.md", "x" * 30)
    strategy = CompositeStrategy(
        ExtensionStrategy(".py"),
        SizeRangeStrategy(min_bytes=10),
    )
    results = fs.find("/", strategy)
    assert len(results) == 1
    assert "/project/app.py" in results
    print("✅ test_find_composite passed")


def test_undo_mkdir() -> None:
    """Test undoing a mkdir."""
    fs = FileSystem()
    fs.mkdir("/new_dir")
    assert fs.exists("/new_dir")
    fs.undo()
    assert not fs.exists("/new_dir")
    print("✅ test_undo_mkdir passed")


def test_undo_write() -> None:
    """Test undoing a write restores old content."""
    fs = FileSystem()
    fs.mkdir("/data")
    fs.write("/data/file.txt", "original")
    fs.write("/data/file.txt", "modified")
    assert fs.read("/data/file.txt") == "modified"
    fs.undo()
    assert fs.read("/data/file.txt") == "original"
    print("✅ test_undo_write passed")


def test_undo_rm() -> None:
    """Test undoing a remove restores the node."""
    fs = FileSystem()
    fs.mkdir("/data")
    fs.write("/data/file.txt", "important")
    fs.rm("/data/file.txt")
    assert not fs.exists("/data/file.txt")
    fs.undo()
    assert fs.exists("/data/file.txt")
    assert fs.read("/data/file.txt") == "important"
    print("✅ test_undo_rm passed")


def test_undo_mv() -> None:
    """Test undoing a move."""
    fs = FileSystem()
    fs.mkdir("/a")
    fs.mkdir("/b")
    fs.write("/a/file.txt", "data")
    fs.mv("/a/file.txt", "/b/file.txt")
    assert not fs.exists("/a/file.txt")
    fs.undo()
    assert fs.exists("/a/file.txt")
    assert not fs.exists("/b/file.txt")
    print("✅ test_undo_mv passed")


def test_redo() -> None:
    """Test redo after undo."""
    fs = FileSystem()
    fs.mkdir("/data")
    fs.undo()
    assert not fs.exists("/data")
    fs.redo()
    assert fs.exists("/data")
    print("✅ test_redo passed")


def test_path_resolver_normalize() -> None:
    """Test path normalization."""
    root = Directory(name="")
    resolver = PathResolver(root)
    assert resolver._normalize("/") == []
    assert resolver._normalize("/home/user") == [
        "home", "user",
    ]
    assert resolver._normalize("/a/../b/./c") == [
        "b", "c",
    ]
    assert resolver._normalize("//a///b//") == ["a", "b"]
    print("✅ test_path_resolver_normalize passed")


def test_size_calculation() -> None:
    """Test file and directory size calculation."""
    fs = FileSystem()
    fs.mkdir("/data")
    fs.write("/data/a.txt", "hello")  # 5 bytes
    fs.write("/data/b.txt", "world")  # 5 bytes
    assert fs.size("/data/a.txt") == 5
    assert fs.size("/data") == 10
    print("✅ test_size_calculation passed")


def test_permissions() -> None:
    """Test permission enforcement."""
    fs = FileSystem()
    fs.mkdir("/secure")
    fs.write("/secure/secret.txt", "hidden")
    fs.chmod("/secure/secret.txt", Permission.NONE)
    assert fs.read("/secure/secret.txt") is None
    fs.chmod("/secure/secret.txt", Permission.ALL)
    assert fs.read("/secure/secret.txt") == "hidden"
    print("✅ test_permissions passed")


def test_tree_output() -> None:
    """Test tree visualization generates output."""
    fs = FileSystem()
    fs.mkdir("/a/b")
    fs.touch("/a/b/c.txt")
    tree = fs.tree()
    assert "a/" in tree
    assert "b/" in tree
    assert "c.txt" in tree
    print("✅ test_tree_output passed")


def test_deep_copy_isolation() -> None:
    """Test that deep_copy creates independent nodes."""
    fs = FileSystem()
    fs.mkdir("/src")
    fs.write("/src/file.txt", "original")
    fs.mkdir("/dst")
    fs.cp("/src/file.txt", "/dst/copy.txt")
    # Modify original
    fs.write("/src/file.txt", "changed")
    # Copy should be unaffected
    assert fs.read("/dst/copy.txt") == "original"
    print("✅ test_deep_copy_isolation passed")


def test_full_lifecycle() -> None:
    """Test full lifecycle: create, write, read, move, delete."""
    fs = FileSystem()
    fs.mkdir("/project/src")
    fs.write("/project/src/main.py",
             "print('hello')")
    assert fs.read("/project/src/main.py") == (
        "print('hello')")
    fs.mv("/project/src/main.py",
          "/project/src/app.py")
    assert fs.read("/project/src/app.py") == (
        "print('hello')")
    fs.rm("/project/src/app.py")
    assert not fs.exists("/project/src/app.py")
    # Undo the delete
    fs.undo()
    assert fs.exists("/project/src/app.py")
    print("✅ test_full_lifecycle passed")


def run_all_tests() -> None:
    print("\n" + "=" * 40)
    print("  RUNNING TESTS")
    print("=" * 40 + "\n")
    test_mkdir()
    test_touch()
    test_write_and_read()
    test_write_creates_file()
    test_write_overwrites()
    test_ls()
    test_rm_file()
    test_rm_directory()
    test_rm_root_fails()
    test_mv()
    test_mv_directory()
    test_cp()
    test_cp_directory()
    test_find_by_extension()
    test_find_by_name_glob()
    test_find_by_size()
    test_find_composite()
    test_undo_mkdir()
    test_undo_write()
    test_undo_rm()
    test_undo_mv()
    test_redo()
    test_path_resolver_normalize()
    test_size_calculation()
    test_permissions()
    test_tree_output()
    test_deep_copy_isolation()
    test_full_lifecycle()
    print("\n✅ All tests passed!\n")


if __name__ == "__main__":
    main_scenario()
    demo_undo_redo()
    demo_search()
    demo_permissions()
    run_all_tests()
```

Each test isolates a single behavior. `test_mkdir` verifies that intermediate directories are created automatically. `test_write_and_read` confirms content roundtrips correctly. `test_rm_directory` checks recursive deletion — removing `/data` also removes `/data/sub` and `/data/sub/file.txt`. `test_rm_root_fails` ensures the root cannot be deleted. `test_mv` verifies the node physically moves: absent at source, present at destination, with preserved content. `test_cp` verifies deep copy isolation — modifying the original after copy does not affect the copy. `test_find_composite` combines two strategies with AND logic. `test_undo_write` confirms that undoing a write restores the previous content, not just deleting the file. `test_undo_rm` restores a deleted node with all its content intact. `test_path_resolver_normalize` checks that `.`, `..`, multiple slashes, and trailing slashes are handled correctly. `test_full_lifecycle` runs a complete create → write → read → move → delete → undo sequence through the facade.

---

## Class Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                         FileSystem                                  │
│──────────────────────────────────────────────────────────────────│
│ - _root: Directory                                                  │
│ - _resolver: PathResolver                                           │
│ - _permissions: PermissionManager                                   │
│ - _undo_redo: UndoRedoManager                                       │
│ - _event_bus: FileSystemEventBus                                    │
│ - _metrics: FileSystemMetrics                                       │
│──────────────────────────────────────────────────────────────────│
│ + mkdir(path) → bool                                                │
│ + touch(path) → bool                                                │
│ + write(path, content) → bool                                       │
│ + read(path) → str | None                                           │
│ + ls(path) → list[str] | None                                       │
│ + rm(path) → bool                                                   │
│ + mv(src, dst) → bool                                               │
│ + cp(src, dst) → bool                                               │
│ + find(path, strategy) → list[str]                                  │
│ + undo() → str | None                                               │
│ + redo() → str | None                                               │
│ + chmod(path, perm) → bool                                          │
│ + tree(path) → str                                                  │
│ + display_status() → None                                           │
└───────┬──────────┬──────────┬────────────────────────────────────┘
        │          │          │
   uses ▼     uses ▼     uses ▼
┌──────────────────────────────────────────────────────────────────┐
│  «abstract» FileSystemNode (Composite)                              │
│──────────────────────────────────────────────────────────────────│
│ + node_id: str                                                      │
│ + name: str                                                         │
│ + owner: str                                                        │
│ + permission: Permission                                            │
│ + parent: Directory | None                                          │
│ + created_at: float                                                 │
│ + modified_at: float                                                │
│──────────────────────────────────────────────────────────────────│
│ + node_type → NodeType                (abstract)                    │
│ + size → int                          (abstract)                    │
│ + path → str                                                        │
│ + is_file → bool                                                    │
│ + is_directory → bool                                               │
│ + deep_copy() → FileSystemNode        (abstract)                    │
│ + touch() → None                                                    │
└──────────┬────────────────────────────────────────────────────────┘
           │ implemented by
  ┌────────┴──────────┐
  ▼                   ▼
┌──────────────────┐ ┌──────────────────────────────────────────────┐
│  File (Leaf)     │ │  Directory (Composite)                          │
│──────────────────│ │──────────────────────────────────────────────│
│ - _content: str  │ │ - _children: dict[str, FileSystemNode]         │
│──────────────────│ │──────────────────────────────────────────────│
│ + content → str  │ │ + add_child(node) → None                       │
│ + append(data)   │ │ + remove_child(name) → Node | None             │
│ + truncate()     │ │ + get_child(name) → Node | None                │
│ + size → int     │ │ + list_names() → list[str]                     │
│ + deep_copy()    │ │ + walk() → list[Node]                          │
└──────────────────┘ │ + depth → int                                   │
                     │ + tree() → str                                  │
                     │ + deep_copy() → Directory                       │
                     └──────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  PathResolver                                                       │
│──────────────────────────────────────────────────────────────────│
│ - _root: Directory                                                  │
│──────────────────────────────────────────────────────────────────│
│ + resolve(path) → Node | None                                       │
│ + resolve_parent(path) → (Directory, str) | None                    │
│ + exists(path) → bool                                               │
│ + split_path(path) → (str, str)                                     │
│ - _normalize(path) → list[str]                                      │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  «abstract» FSCommand (Command Pattern)                             │
│──────────────────────────────────────────────────────────────────│
│ + execute() → bool                                                  │
│ + undo() → bool                                                     │
│ + description() → str                                               │
└──────────┬───────────────────────────────────────────────────────┘
           │ implemented by
  ┌────────┼────────┬────────┬────────┬────────┐
  ▼        ▼        ▼        ▼        ▼        ▼
Mkdir    Touch    Write   Remove    Move     Copy
Command  Command  Command  Command  Command  Command

┌──────────────────────────────────────────────────────────────────┐
│  UndoRedoManager                                                    │
│──────────────────────────────────────────────────────────────────│
│ - _undo_stack: deque[FSCommand]                                     │
│ - _redo_stack: deque[FSCommand]                                     │
│──────────────────────────────────────────────────────────────────│
│ + push(command) → None                                              │
│ + undo() → FSCommand | None                                         │
│ + redo() → FSCommand | None                                         │
│ + history() → list[str]                                             │
│ + can_undo → bool                                                   │
│ + can_redo → bool                                                   │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  «abstract» SearchStrategy (Strategy Pattern)                       │
│──────────────────────────────────────────────────────────────────│
│ + matches(node) → bool                                              │
│ + name() → str                                                      │
└──────────┬───────────────────────────────────────────────────────┘
           │ implemented by
  ┌────────┼────────────┬────────────┬──────────────┐
  ▼        ▼            ▼            ▼              ▼
NameGlob  Extension   SizeRange   NodeType     Composite
Strategy  Strategy    Strategy    Strategy     Strategy

┌──────────────────────────────────────────────────────────────────┐
│  «abstract» FileEventListener (Observer Pattern)                    │
│──────────────────────────────────────────────────────────────────│
│ + on_event(event) → None                                            │
│ + listener_id() → str                                               │
└──────────┬────────────────────────────────────────────────────────┘
           │ implemented by
  ┌────────┴──────────┐
  ▼                   ▼
FileSystem         FileSystem
Logger             Metrics

┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│ FileSystemEvent  │ │ FileSystemStats  │ │ PermissionManager│
│──────────────────│ │──────────────────│ │──────────────────│
│ + event_id       │ │ + total_files    │ │ + check_read()   │
│ + event_type     │ │ + total_dirs     │ │ + check_write()  │
│ + path           │ │ + total_size     │ │ + check_execute()│
│ + detail         │ │ + max_depth      │ │ + set_permission │
│ + timestamp      │ │ + display()      │ │ + set_owner()    │
└──────────────────┘ └──────────────────┘ │ + perm_string()  │
                                          └──────────────────┘
┌──────────────────┐ ┌──────────────────┐
│ FileSystemEventBus│ │ FSScenario      │
│──────────────────│ │──────────────────│
│ - _listeners     │ │ - _fs           │
│ - _event_log     │ │ - _events       │
│──────────────────│ │──────────────────│
│ + register()     │ │ + at_tick()     │
│ + emit()         │ │ + run()         │
│ + unregister()   │ └──────────────────┘
└──────────────────┘

┌──────────────────┐ ┌──────────────────┐
│ NodeType (Enum)  │ │ EventType (Enum) │
│──────────────────│ │──────────────────│
│ FILE             │ │ CREATED          │
│ DIRECTORY        │ │ MODIFIED         │
└──────────────────┘ │ DELETED          │
                     │ MOVED            │
┌──────────────────┐ │ COPIED           │
│ Permission (Flag)│ │ PERMISSION_CHANGED│
│──────────────────│ └──────────────────┘
│ NONE = 0         │
│ READ = 1         │
│ WRITE = 2        │
│ EXECUTE = 4      │
│ READ_WRITE = 3   │
│ ALL = 7          │
└──────────────────┘
```

---

## How to Run

Save all code into a single file `file_system.py` and run:

```bash
python file_system.py
```

### Example Output

```
╔══════════════════════════════════════════╗
║      IN-MEMORY FILE SYSTEM               ║
║  Composite + Command + Strategy + Observer║
╚══════════════════════════════════════════╝

  ───────────────────────────────────────────────────────
  SIMULATION START (ticks: 1 → 11)
  ───────────────────────────────────────────────────────

  ⏱️  Tick 1: Create directory structure
  📄 [             created] /home/user/docs directory created
  📄 [             created] /home/user/pics directory created
  📄 [             created] /var/log directory created
  📄 [             created] /tmp directory created

  ⏱️  Tick 2: Create files with content
  📄 [             created] /home/user/docs/readme.txt 19B written
  📄 [             created] /home/user/docs/notes.md 29B written
  📄 [             created] /home/user/pics/photo.jpg 13B written
  📄 [             created] /var/log/system.log 31B written
  📄 [             created] /tmp/scratch.txt 14B written

  ⏱️  Tick 3: List /home/user/docs
       → ['notes.md', 'readme.txt']
  ⏱️  Tick 3: Read readme.txt
       → Hello, File System!

  ⏱️  Tick 4: Display file system tree
/
├── home/
│   └── user/
│       ├── docs/
│       │   ├── notes.md
│       │   └── readme.txt
│       └── pics/
│           └── photo.jpg
├── tmp/
│   └── scratch.txt
└── var/
    └── log/
        └── system.log

  ⏱️  Tick 5: Find all .txt files
       → ['/home/user/docs/readme.txt', '/tmp/scratch.txt']
  ⏱️  Tick 5: Find all files > 10 bytes
       → ['/home/user/docs/notes.md', '/home/user/docs/readme.txt',
          '/home/user/pics/photo.jpg', '/tmp/scratch.txt',
          '/var/log/system.log']
  ⏱️  Tick 5: Find by name glob 'sys*'
       → ['/var/log/system.log']

  ⏱️  Tick 6: Copy readme.txt to /tmp/readme_backup.txt
  📋 [              copied] /home/user/docs/readme.txt → /tmp/readme_backup.txt

  ⏱️  Tick 7: Move scratch.txt to /home/user/docs/old.txt
  📦 [               moved] /tmp/scratch.txt → /home/user/docs/old.txt

  ⏱️  Tick 8: Undo last move
       → ↩ undone: mv /tmp/scratch.txt → /home/user/docs/old.txt
  ⏱️  Tick 8: Redo the move
       → ↪ redone: mv /tmp/scratch.txt → /home/user/docs/old.txt

  ⏱️  Tick 9: Remove /tmp directory
  🗑️  [             deleted] /tmp removed
  ⏱️  Tick 9: Undo remove
       → ↩ undone: rm /tmp

  ⏱️  Tick 10: Set readme.txt to read-only
  🔒 [  permission_changed] /home/user/docs/readme.txt → r--

  ⏱️  Tick 11: Display final status

  ═══════════════════════════════════════════════════════
  FILE SYSTEM STATISTICS
  ═══════════════════════════════════════════════════════
  Files:           6
  Directories:     7
  Total size:      125B
  Max depth:       3
  ═══════════════════════════════════════════════════════

  ───────────────────────────────────────────────────────
  SIMULATION COMPLETE
  ───────────────────────────────────────────────────────

========================================
  RUNNING TESTS
========================================

✅ test_mkdir passed
✅ test_touch passed
✅ test_write_and_read passed
✅ test_write_creates_file passed
✅ test_write_overwrites passed
✅ test_ls passed
✅ test_rm_file passed
✅ test_rm_directory passed
✅ test_rm_root_fails passed
✅ test_mv passed
✅ test_mv_directory passed
✅ test_cp passed
✅ test_cp_directory passed
✅ test_find_by_extension passed
✅ test_find_by_name_glob passed
✅ test_find_by_size passed
✅ test_find_composite passed
✅ test_undo_mkdir passed
✅ test_undo_write passed
✅ test_undo_rm passed
✅ test_undo_mv passed
✅ test_redo passed
✅ test_path_resolver_normalize passed
✅ test_size_calculation passed
✅ test_permissions passed
✅ test_tree_output passed
✅ test_deep_copy_isolation passed
✅ test_full_lifecycle passed

✅ All tests passed!
```
