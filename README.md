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

![Screenshot 1A](images/1A_test_objects.jpg)

This confirms successful blob storage, deduplication, and integrity checking in the object store implementation.

### Screenshot 1B — `find .pes/objects -type f`

![Screenshot 1B](images/1B_objects_structure.jpg)

The object files are sharded by the first two hexadecimal characters of the hash, which matches the intended content-addressable storage layout.

---

## Phase 2

### Screenshot 2A — `./test_tree` output

![Screenshot 2A](images/2A_test_tree.jpg)

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
