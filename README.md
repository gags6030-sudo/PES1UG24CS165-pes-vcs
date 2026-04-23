# Building PES-VCS — A Version Control System from Scratch

**Objective:** Build a local version control system that tracks file changes, stores snapshots efficiently, and supports commit history. Every component maps directly to operating system and filesystem concepts.

**Platform:** Ubuntu 22.04

---

# Lab Report

This section contains the required screenshots and the written answers for the analysis-only questions from Phases 5 and 6.

## Implementation Summary

- **Phase 1** implements content-addressable object storage with hash verification.
- **Phase 2** implements tree serialization and root tree construction from staged files.
- **Phase 3** implements the text-based staging index with atomic saves.
- **Phase 4** implements commit creation, reference updates, and log traversal.

## Validation Summary

- `./test_objects` passes for object storage behavior.
- `./test_tree` passes for tree parsing and deterministic serialization.
- Manual `pes init`, `pes add`, `pes status`, `pes commit`, and `pes log` flows work.
- `make test-integration` completes successfully.

---

# Screenshots

## Phase 1

### Screenshot 1A — `./test_objects` output

<img width="1536" height="1024" alt="WhatsApp Image 2026-04-20 at 12 03 29" src="https://github.com/user-attachments/assets/a685aeb5-a14d-4a9e-b26b-75ff6c6b620d" />

This confirms successful blob storage, deduplication, and integrity checking in the object store implementation.

### Screenshot 1B — `find .pes/objects -type f`

![Screenshot 1B]<img width="2170" height="725" alt="image" src="https://github.com/user-attachments/assets/f850944e-1a43-48fd-aa94-61e785cdb250" />


The object files are sharded by the first two hexadecimal characters of the hash, which matches the intended content-addressable storage layout.

---

## Phase 2

### Screenshot 2A — `./test_tree` output

![Screenshot 2A]<img width="1536" height="1024" alt="WhatsApp Image 2026-04-20 at 12 11 26" src="https://github.com/user-attachments/assets/f27c5bff-ff0e-42ad-9724-06e72792046e" />


This shows that tree serialization and parsing are consistent and that the serialized ordering is deterministic.

### Screenshot 2B — raw tree object via `xxd`

![Screenshot 2B](images/2B_tree_xxd.jpg)

The raw dump shows the tree header followed by serialized entries containing mode, filename, and binary object hashes.

---

## Phase 3

### Screenshot 3A — `pes init -> pes add -> pes status`

![Screenshot 3A](images/3A_status.jpg)

The status output shows that staged files are tracked correctly and that no unstaged changes are reported immediately after adding them.

### Screenshot 3B — `cat .pes/index`

![Screenshot 3B](images/3B_index.jpg)

The index stores the file mode, blob hash, modification time, size, and path in the expected text format.

---

## Phase 4

### Screenshot 4A — `pes log`

![Screenshot 4A](images/4A_log.jpg)

The log confirms that commits are linked through parents and printed from newest to oldest with author, timestamp, and message metadata.

### Screenshot 4B — `find .pes -type f | sort`

![Screenshot 4B](images/4B_objects_growth.jpg)

This listing shows the growth of repository metadata after multiple commits, including refs, the index, and newly created objects.

### Screenshot 4C — `cat .pes/refs/heads/main` and `cat .pes/HEAD`

![Screenshot 4C](images/4C_refs_head.jpg)

The output demonstrates the expected reference chain: `HEAD` points to the branch name, and the branch file points to the latest commit hash.

---

# Final Integration Test

### Integration Screenshot — `make test-integration`

![Integration Screenshot 1](images/final_integration_1.jpg)
![Integration Screenshot 2](images/final_integration_2.jpg)
![Integration Screenshot 3](images/final_integration_3.jpg)

Together, these screenshots verify the full end-to-end workflow from repository initialization through staging, committing, history traversal, and reference updates.

---

# Analysis Answers

## Q5.1 Branching and Checkout

A branch in PES-VCS should be implemented as a file under `.pes/refs/heads/<branch>` that stores the target commit hash, exactly like the existing `main` branch file. A `pes checkout <branch>` operation would first verify that the branch file exists, then rewrite `.pes/HEAD` so that it contains `ref: refs/heads/<branch>`. After that, the working directory must be updated to match the tree pointed to by the branch's latest commit. That means reading the target commit object, reading its root tree, recursively materializing every file and directory into the working directory, and removing tracked files that exist in the current branch but not in the target branch. The complexity comes from the fact that checkout is not only a metadata change in `.pes/`; it also has to safely transform the user's actual filesystem state without overwriting uncommitted work or leaving the repository half-updated if an error occurs in the middle.

## Q5.2 Dirty Working Directory Conflict Detection

The conflict check can be done by comparing three states: the HEAD snapshot, the index, and the working directory. For each tracked path in the current branch, PES-VCS should determine whether the working directory still matches the staged version recorded in the index. The fast path is to compare `stat()` metadata such as modification time and size against the index entry. If those values no longer match, the safe path is to re-read the file, hash it, and compare that hash with the blob hash stored in the index or current HEAD tree. If a file has local modifications and the target branch also wants a different version of that same path, checkout must refuse because switching branches would overwrite user data. In short, PES-VCS should reject checkout when a tracked file is dirty in the working tree and the target branch would update or delete that path.

## Q5.3 Detached HEAD

In detached HEAD state, `.pes/HEAD` would contain a commit hash directly instead of a symbolic reference like `ref: refs/heads/main`. New commits made in this state still work: each new commit records the previous detached commit as its parent, so history continues normally, but no branch name moves forward to remember those commits. As a result, the commits become hard to find later unless the user notes the hash manually. Recovery is straightforward if the user still knows one of the detached commit hashes: create a new branch file under `.pes/refs/heads/` pointing to that commit, or update an existing branch to that hash. Once a reference points to the detached history again, the commits are no longer orphaned and can be reached through normal log traversal.

**Design assumption for Q5:** these checkout answers assume PES-VCS updates only tracked files and rebuilds the working directory from tree objects, rather than trying to preserve arbitrary untracked files automatically.

## Q6.1 Garbage Collection and Space Reclamation

Garbage collection should use a mark-and-sweep algorithm. The mark phase starts from every live reference, such as all files under `.pes/refs/heads/` and also `HEAD` if it stores a direct commit hash. For each referenced commit, PES-VCS recursively marks the commit itself, its parent commits, its root tree, every subtree reachable from that tree, and every blob reachable from those trees. A hash set is the right data structure for tracking reachability efficiently, because membership tests and inserts are close to `O(1)`. After marking, the sweep phase walks `.pes/objects/` and deletes any object whose hash is not in the reachable set. For a repository with 100,000 commits and 50 branches, the number of visited objects is on the order of the distinct reachable commits, trees, and blobs, not `100,000 × 50`, because many branches usually share large parts of history. In practice, GC would still need to visit a very large number of objects, likely hundreds of thousands or more in a moderately sized repo.

## Q6.2 Why Concurrent GC Is Dangerous

Running garbage collection concurrently with commit creation creates a race between “object becomes reachable” and “reference is updated.” During commit, PES-VCS first writes blobs, trees, and the new commit object, and only at the end updates the branch reference. If GC scans references in the middle of that sequence, it may not yet see the new commit from any branch, so it can conclude that the newly written tree or commit objects are unreachable and delete them. Immediately after that, the commit path might update `refs/heads/main` to point at a commit whose underlying objects were just removed, corrupting the repository. Real Git avoids this with coordination and conservative retention: reference updates are atomic, very new objects are protected, and GC is careful not to collect objects that may still become reachable moments later.

**Failure consequence for Q6:** the most visible symptom would be a branch ref that points to a commit hash which exists, but whose tree or blob dependencies have already been deleted, causing future reads or log/tree walks to fail.

---

# Getting Started

## Prerequisites

```bash
sudo apt update && sudo apt install -y gcc build-essential libssl-dev
```
Your project at commit A:          Your project at commit B:
                                   (only README changed)

    root/                              root/
    ├── README.md  ─────┐              ├── README.md  ─────┐
    ├── src/            │              ├── src/            │
    │   └── main.c ─────┼─┐            │   └── main.c ─────┼─┐
    └── Makefile ───────┼─┼─┐          └── Makefile ───────┼─┼─┐
                        │ │ │                              │ │ │
                        ▼ ▼ ▼                              ▼ ▼ ▼
    Object Store:       ┌─────────────────────────────────────────────┐
                        │  a1b2c3 (README v1)    ← only this is new   │
                        │  d4e5f6 (README v2)                         │
                        │  789abc (main.c)       ← shared by both!    │
                        │  fedcba (Makefile)     ← shared by both!    │
                        └─────────────────────────────────────────────┘
```
### The Three Object Types

#### 1. Blob (Binary Large Object)

A blob is just file contents. No filename, no permissions — just the raw bytes.

```
blob 16\0Hello, World!\n
     ↑    ↑
     │    └── The actual file content
     └─────── Size in bytes
```

The blob is stored at a path determined by its SHA-256 hash. If two files have identical contents, they share one blob.

#### 2. Tree

A tree represents a directory. It's a list of entries, each pointing to a blob (file) or another tree (subdirectory).

```
100644 blob a1b2c3d4... README.md
100755 blob e5f6a7b8... build.sh        ← executable file
040000 tree 9c0d1e2f... src             ← subdirectory
       ↑    ↑           ↑
       │    │           └── name
       │    └── hash of the object
       └─────── mode (permissions + type)
```

Mode values:
- `100644` — regular file, not executable
- `100755` — regular file, executable
- `040000` — directory (tree)

#### 3. Commit

A commit ties everything together. It points to a tree (the project snapshot) and contains metadata.

```
tree 9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d
parent a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0
author Alice <alice@example.com> 1699900000
committer Alice <alice@example.com> 1699900000

Add new feature
```

The parent pointer creates a linked list of history:

```
    C3 ──────► C2 ──────► C1 ──────► (no parent)
    │          │          │
    ▼          ▼          ▼
  Tree3      Tree2      Tree1
```

### How Objects Connect

```
                    ┌─────────────────────────────────┐
                    │           COMMIT                │
                    │  tree: 7a3f...                  │
                    │  parent: 4b2e...                │
                    │  author: Alice                  │
                    │  message: "Add feature"         │
                    └─────────────┬───────────────────┘
                                  │
                                  ▼
                    ┌─────────────────────────────────┐
                    │         TREE (root)             │
                    │  100644 blob f1a2... README.md  │
                    │  040000 tree 8b3c... src        │
                    │  100644 blob 9d4e... Makefile   │
                    └──────┬──────────┬───────────────┘
                           │          │
              ┌────────────┘          └────────────┐
              ▼                                    ▼
┌─────────────────────────┐          ┌─────────────────────────┐
│      TREE (src)         │          │     BLOB (README.md)    │
│ 100644 blob a5f6 main.c │          │  # My Project           │
└───────────┬─────────────┘          └─────────────────────────┘
            ▼
       ┌────────┐
       │ BLOB   │
       │main.c  │
       └────────┘
```

### References and HEAD

References are files that map human-readable names to commit hashes:

```
.pes/
├── HEAD                    # "ref: refs/heads/main"
└── refs/
    └── heads/
        └── main            # Contains: a1b2c3d4e5f6...
```

**HEAD** points to a branch name. The branch file contains the latest commit hash. When you commit:

1. Git creates the new commit object (pointing to parent)
2. Updates the branch file to contain the new commit's hash
3. HEAD still points to the branch, so it "follows" automatically

```
Before commit:                    After commit:

HEAD ─► main ─► C2 ─► C1         HEAD ─► main ─► C3 ─► C2 ─► C1
```
### Testing

```bash
make test_objects
./test_objects
```

The test program verifies:
- Blob storage and retrieval (write, read back, compare)
- Deduplication (same content → same hash → stored once)
- Integrity checking (detects corrupted objects)

**📸 Screenshot 1A:** Output of `./test_objects` showing all tests passing.

**📸 Screenshot 1B:** `find .pes/objects -type f` showing the sharded directory structure.

---

## Phase 2: Tree Objects

**Filesystem Concepts:** Directory representation, recursive structures, file modes and permissions

**Files:** `tree.h` (read), `tree.c` (implement all TODO functions)

### What to Implement

Open `tree.c`. Implement the function marked `// TODO`:

1. **`tree_from_index`** — Builds a tree hierarchy from the index.
   - Handles nested paths: `"src/main.c"` must create a `src` subtree
   - This is what `pes commit` uses to create the snapshot
   - Writes all tree objects to the object store and returns the root hash

### Testing

```bash
make test_tree
./test_tree
```

The test program verifies:
- Serialize → parse roundtrip preserves entries, modes, and hashes
- Deterministic serialization (same entries in any order → identical output)

**📸 Screenshot 2A:** Output of `./test_tree` showing all tests passing.

**📸 Screenshot 2B:** Pick a tree object from `find .pes/objects -type f` and run `xxd .pes/objects/XX/YYY... | head -20` to show the raw binary format.

---

## Phase 3: The Index (Staging Area)

**Filesystem Concepts:** File format design, atomic writes, change detection using metadata

**Files:** `index.h` (read), `index.c` (implement all TODO functions)

### What to Implement

Open `index.c`. Three functions are marked `// TODO`:

1. **`index_load`** — Reads the text-based `.pes/index` file into an `Index` struct.
   - If the file doesn't exist, initializes an empty index (this is not an error)
   - Parses each line: `<mode> <hash-hex> <mtime> <size> <path>`

2. **`index_save`** — Writes the index atomically (temp file + rename).
   - Sorts entries by path before writing
   - Uses `fsync()` on the temp file before renaming

3. **`index_add`** — Stages a file: reads it, writes blob to object store, updates index entry.
   - Use the provided `index_find` to check for an existing entry

`index_find` , `index_status` and `index_remove` are already implemented for you — read them to understand the index data structure before starting.

#### Expected Output of `pes status`

```
Staged changes:
  staged:     hello.txt
  staged:     src/main.c

Unstaged changes:
  modified:   README.md
  deleted:    old_file.txt

Untracked files:
  untracked:  notes.txt
```

If a section has no entries, print the header followed by `(nothing to show)`.

### Testing

```bash
make pes
./pes init
echo "hello" > file1.txt
echo "world" > file2.txt
./pes add file1.txt file2.txt
./pes status
cat .pes/index    # Human-readable text format
```

**📸 Screenshot 3A:** Run `./pes init`, `./pes add file1.txt file2.txt`, `./pes status` — show the output.

**📸 Screenshot 3B:** `cat .pes/index` showing the text-format index with your entries.

---

## Phase 4: Commits and History

**Filesystem Concepts:** Linked structures on disk, reference files, atomic pointer updates

**Files:** `commit.h` (read), `commit.c` (implement all TODO functions)

### What to Implement

Open `commit.c`. One function is marked `// TODO`:

1. **`commit_create`** — The main commit function:
   - Builds a tree from the index using `tree_from_index()` (**not** from the working directory — commits snapshot the staged state)
   - Reads current HEAD as the parent (may not exist for first commit)
   - Gets the author string from `pes_author()` (defined in `pes.h`)
   - Writes the commit object, then updates HEAD

`commit_parse`, `commit_serialize`, `commit_walk`, `head_read`, and `head_update` are already implemented — read them to understand the commit format before writing `commit_create`.

The commit text format is specified in the comment at the top of `commit.c`.

### Testing

```bash
./pes init
echo "Hello" > hello.txt
./pes add hello.txt
./pes commit -m "Initial commit"

echo "World" >> hello.txt
./pes add hello.txt
./pes commit -m "Add world"

echo "Goodbye" > bye.txt
./pes add bye.txt
./pes commit -m "Add farewell"

./pes log
```

You can also run the full integration test:

```bash
make test-integration
```
