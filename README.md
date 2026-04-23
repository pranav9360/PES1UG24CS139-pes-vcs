# PES-VCS Lab Report
**Name:** Gagan  
**SRN:** PES1UG24CS166  
**Repository:** [PES1UG24CS166-pes-vcs](https://github.com/Gagancreates/PES1UG24CS166-pes-vcs)

---

## Phase 1: Object Storage Foundation

### What I Implemented
- `object_write` — builds the full object (header + data), computes SHA-256, deduplicates, shards into `.pes/objects/XX/`, and writes atomically using temp-file + rename pattern.
- `object_read` — reads the object file, parses the header, recomputes the hash to verify integrity, and returns the data portion.

### Screenshot 1A — `./test_objects` passing
```
Stored blob with hash: d58213f5dbe0629b5c2fa28e5c7d4213ea09227ed0221bbe9db5e5c4b9aafc12
Object stored at: .pes/objects/d5/8213f5dbe0629b5c2fa28e5c7d4213ea09227ed0221bbe9db5e5c4b9aafc12
PASS: blob storage
PASS: deduplication
PASS: integrity check
All Phase 1 tests passed.
```
*(Insert screenshot here)*

### Screenshot 1B — Sharded object store
```
find .pes/objects -type f
```
*(Insert screenshot here)*

---

## Phase 2: Tree Objects

### What I Implemented
- `tree_from_index` — loads the staged index and recursively builds a tree hierarchy. Handles nested paths like `src/main.c` by grouping entries that share a directory prefix, recursively writing subtrees, and then writing the root tree. Uses a weak `index_load` stub so `test_tree` links without `index.o`.

### Screenshot 2A — `./test_tree` passing
```
Serialized tree: 139 bytes
PASS: tree serialize/parse roundtrip
PASS: tree deterministic serialization

All Phase 2 tests passed.
```
*(Insert screenshot here)*

### Screenshot 2B — Raw binary tree object
```
xxd .pes/objects/2c/f8d83d9ee29543b34a87727421fdecb7e3f3a183d337639025de576db9ebb4
00000000: 626c 6f62 2036 0068 656c 6c6f 0a         blob 6.hello.
```
*(Insert screenshot here)*

---

## Phase 3: The Index (Staging Area)

### What I Implemented
- `index_load` — reads `.pes/index` line by line, parsing `mode hash mtime size path` into the `Index` struct. Returns empty index if file doesn't exist.
- `index_save` — heap-allocates a sorted copy of the index (to avoid 5.6MB stack overflow), writes to a temp file, fsyncs, then renames atomically.
- `index_add` — reads the file, writes it as a blob, gets file metadata via `lstat`, then updates or inserts the index entry.

### Screenshot 3A — `pes init` → `pes add` → `pes status`
```
Initialized empty PES repository in .pes/
```
```
Staged changes:
  staged:     file1.txt
  staged:     file2.txt

Unstaged changes:
  (nothing to show)

Untracked files:
  ...
```
*(Insert screenshot here)*

### Screenshot 3B — `cat .pes/index`
```
100755 2cf8d83d9ee29543b34a87727421fdecb7e3f3a183d337639025de576db9ebb4 1776660272 6 file1.txt
100755 e00c50e16a2df38f8d6bf809e181ad0248da6e6719f35f9f7e65d6f606199f7f 1776660277 6 file2.txt
```
*(Insert screenshot here)*

---

## Phase 4: Commits and History

### What I Implemented
- `commit_create` — builds the root tree from the staged index via `tree_from_index`, reads the parent commit from HEAD (skips if first commit), fills a `Commit` struct with author, timestamp, and message, serializes and writes it as a commit object, then atomically updates HEAD via `head_update`.

### Screenshot 4A — `pes log` with three commits
```
commit fe2f9b01fb7cb9e37f1a95366e5e572b830b053bce2c65c4f6dc49f684039138
Author: PES User <pes@localhost>
Date:   1776660859

    Add farewell

commit e8cdc02f4a717155da4569fffc97da87fc4bd26b2fb7bac58bdc4beeee22a63d
Author: PES User <pes@localhost>
Date:   1776660846

    Add world

commit 00c70f693a05f9d91900e120992758900ce1689b6158bec87eb50ee563977833
Author: PES User <pes@localhost>
Date:   1776660832

    Initial commit
```
*(Insert screenshot here)*

### Screenshot 4B — `find .pes -type f | sort`
*(Insert screenshot here)*

### Screenshot 4C — HEAD and branch reference
```
cat .pes/refs/heads/main
fe2f9b01fb7cb9e37f1a95366e5e572b830b053bce2c65c4f6dc49f684039138

cat .pes/HEAD
ref: refs/heads/main
```
*(Insert screenshot here)*

### Final — Integration test
```
make test-integration
```
*(Insert screenshot here)*

---

## Phase 5: Analysis — Branching and Checkout

### Q5.1 — How would you implement `pes checkout <branch>`?

To implement `pes checkout <branch>`, the following steps are needed:

**Files that change in `.pes/`:**
- `.pes/HEAD` must be updated to `ref: refs/heads/<branch>` to point to the new branch.
- No other `.pes/` files change — the branch ref file itself already exists (or must be created for a new branch).

**What must happen to the working directory:**
1. Read the current HEAD commit and walk its tree to get the current file snapshot.
2. Read the target branch's commit and walk its tree to get the target file snapshot.
3. Diff the two trees — for each file that exists in the target but not the current, write it to disk. For each file that exists in the current but not the target, delete it. For files that exist in both but with different blob hashes, overwrite with the target version.

**What makes this complex:**
- Trees are recursive — you must walk subtrees depth-first to enumerate all files.
- You must restore file permissions (mode) correctly from the tree entries.
- You must handle the case where a path is a file in one branch and a directory in another.
- The entire working directory sync must be atomic enough that a crash mid-checkout doesn't leave the repo in a broken state.
- Untracked files must be preserved (don't delete files that aren't tracked in either branch).

### Q5.2 — Detecting dirty working directory conflict

The algorithm uses only the index and the object store:

1. For each file in the current index, compare its `mtime` and `size` against the actual file on disk using `stat()`. If either differs, the file is locally modified (dirty).

2. For each dirty file, check whether it differs between branches: read the blob hash for that file from the current HEAD's tree, and the blob hash from the target branch's tree. If the two hashes differ, then the file has changed between branches AND has local modifications — this is a conflict.

3. If any such conflict exists, refuse the checkout and print an error.

This is efficient because it uses the mtime/size fast-path (same as `git status`) to avoid re-hashing every file. Only dirty files need their blob hashes compared across trees.

### Q5.3 — Detached HEAD

When HEAD contains a commit hash directly instead of `ref: refs/heads/<branch>`, new commits are created and chained correctly, but no branch pointer is updated. The commits exist in the object store and are reachable from HEAD, but as soon as you switch to another branch, HEAD changes and the old commits become unreachable — no branch points to them.

**Recovery:** If you remember the commit hash (from terminal history or `pes log` output), you can create a new branch pointing to it:
```bash
# Create a branch at the detached commit before switching away
echo "<commit-hash>" > .pes/refs/heads/recovery-branch
```
If you've already switched away and lost the hash, the commits are still in the object store until garbage collection runs. You'd need to scan all objects in `.pes/objects/` and find commit objects with no incoming references — essentially a manual GC traversal in reverse.

---

## Phase 6: Analysis — Garbage Collection

### Q6.1 — Algorithm to find and delete unreachable objects

**Algorithm (Mark and Sweep):**

1. **Mark phase** — start from all branch refs in `.pes/refs/heads/`. For each ref, read the commit hash and do a BFS/DFS traversal:
   - Add the commit hash to a `reachable` set.
   - Read the commit object, add its tree hash to `reachable`.
   - Recursively walk the tree: add all blob hashes and subtree hashes to `reachable`.
   - Follow the parent pointer and repeat until no parent exists.
   - Also include HEAD itself as a starting point.

2. **Sweep phase** — enumerate all files under `.pes/objects/`. For each file, reconstruct the hash from its path (`XX` + `YYYY...`). If the hash is not in the `reachable` set, delete the file.

**Data structure:** A hash set (e.g. a hash table or sorted array of 32-byte hashes) for O(1) or O(log n) membership checks.

**Estimate for 100,000 commits and 50 branches:**
- Starting from 50 branch tips, each commit has 1 tree + ~N blobs. Assuming average 20 files per commit and 10% change rate between commits, roughly 10,000 unique trees and 50,000 unique blobs are reachable.
- Total objects to visit: ~160,000 (100,000 commits + 10,000 trees + 50,000 blobs).
- In the sweep phase, you'd enumerate all objects in the store — potentially more if there are unreachable ones.

### Q6.2 — GC race condition with concurrent commit

**The race condition:**

1. GC starts its mark phase and determines that blob `X` is unreachable (no branch points to it yet).
2. Concurrently, a `pes add` writes blob `X` to the object store and a `pes commit` begins building a tree that references `X`.
3. GC's sweep phase runs and deletes blob `X` before the commit finishes writing the commit object.
4. The commit object is written pointing to a tree that points to blob `X` — but blob `X` no longer exists. The repository is now corrupt.

**How Git avoids this:**

Git uses a grace period (default 2 weeks) — objects newer than the grace period are never deleted by GC, even if they appear unreachable. This means a concurrent commit always has time to complete and create references before GC can touch the new objects. Git also writes a `gc.log` lock file to prevent two GC processes from running simultaneously. Additionally, Git's `pack-refs` and ref locking mechanisms ensure that refs are read atomically, so the mark phase sees a consistent snapshot of all reachable objects.

---

## Summary

| Phase | Files Modified | Status |
|-------|---------------|--------|
| 1 | `object.c` | ✅ Complete |
| 2 | `tree.c` | ✅ Complete |
| 3 | `index.c` | ✅ Complete |
| 4 | `commit.c` | ✅ Complete |
| 5 & 6 | Analysis | ✅ Complete |
