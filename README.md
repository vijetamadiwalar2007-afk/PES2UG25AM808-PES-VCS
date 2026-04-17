# PES-VCS Lab Submission

**Name:** Vijeta Madiwalar 
**SRN:** PES2UG25AM808

---

## Additional Files

### .env
This file contains the `PES_AUTHOR` environment variable required by the assignment:
```
PES_AUTHOR="Vijeta Madiwalar <PES2UG25AM808>"
```

**Purpose:** The README specifies that PES-VCS reads the author name from the `PES_AUTHOR` environment variable. This file provides a convenient way to set this variable before running commands.

**Usage:** Run `source .env` before executing any `pes` commands to ensure commits are attributed correctly.

---

## Phase 1: Object Storage Foundation

### Implementation Summary
Implemented content-addressable object store with SHA-256 hashing, directory sharding, and atomic writes.

### Screenshots

**Screenshot 1A: test_objects output**
<!-- Attach: Screenshots/1A_test_objects.png -->
![Screenshot 1A](Screenshots/1A_test_objects.png)

**Screenshot 1B: Object directory structure**
<!-- Attach: Screenshots/1B_objects_structure.png -->
![Screenshot 1B](Screenshots/1B_objects_structure.png)

---

## Phase 2: Tree Objects

### Implementation Summary
Implemented tree serialization and construction from index, supporting nested directory structures.

### Screenshots

**Screenshot 2A: test_tree output**
<!-- Attach: Screenshots/2A_test_tree.png -->
![Screenshot 2A](Screenshots/2A_test_tree.png)

**Screenshot 2B: Raw tree object format**
<!-- Attach: Screenshots/2B_tree_raw.png -->
![Screenshot 2B](Screenshots/2B_tree_raw.png)

---

## Phase 3: The Index (Staging Area)

### Implementation Summary
Implemented text-based staging area with atomic writes and change detection.

### Screenshots

**Screenshot 3A: pes add and status**
<!-- Attach: Screenshots/3A_add_status.png -->
![Screenshot 3A](Screenshots/3A_add_status.png)

**Screenshot 3B: Index file contents**
<!-- Attach: Screenshots/3B_index_contents.png -->
![Screenshot 3B](Screenshots/3B_index_contents.png)

---

## Phase 4: Commits and History

### Implementation Summary
Implemented commit creation with parent linking and history traversal.

### Screenshots

**Screenshot 4A: pes log output**
<!-- Attach: Screenshots/4A_log.png -->
![Screenshot 4A](Screenshots/4A_log.png)

**Screenshot 4B: Object store growth**
<!-- Attach: Screenshots/4B_objects_growth.png -->
![Screenshot 4B](Screenshots/4B_objects_growth.png)

**Screenshot 4C: HEAD and refs**
<!-- Attach: Screenshots/4C_head_refs.png -->
![Screenshot 4C](Screenshots/4C_head_refs.png)

---

## Final Integration Test

**Screenshot: Full integration test (Part 1)**
<!-- Attach: Screenshots/final_integration_part1.png -->
![Final Integration Part 1](Screenshots/final_integration_part1.png)

**Screenshot: Full integration test (Part 2)**
<!-- Attach: Screenshots/final_integration_part2.png -->
![Final Integration Part 2](Screenshots/final_integration_part2.png)

---

## Phase 5: Branching and Checkout (Analysis)

### Q5.1: Implementing `pes checkout <branch>`

To implement `pes checkout <branch>`, the following changes are needed:

**Files to change in `.pes/`:**
1. `.pes/HEAD` - Update to point to the target branch reference (e.g., `ref: refs/heads/<branch>`)
2. No changes to `.pes/refs/heads/<branch>` - it already contains the commit hash

**Working directory updates:**
1. Read the commit hash from `.pes/refs/heads/<branch>`
2. Load the commit object and extract its tree hash
3. Recursively walk the tree structure
4. For each file in the tree:
   - If it exists in working directory with different content, update it
   - If it doesn't exist, create it
5. Remove files that exist in working directory but not in the target tree

**What makes this complex:**
- Must handle conflicts when working directory has uncommitted changes
- Need to preserve untracked files (don't delete them)
- Must handle nested directories (create/remove as needed)
- Requires careful ordering: can't delete a directory before removing its files
- Must be atomic: if checkout fails midway, working directory could be in inconsistent state
- Performance: for large repositories, updating thousands of files is expensive

### Q5.2: Detecting "dirty working directory" conflicts

To detect conflicts when switching branches with uncommitted changes:

**Algorithm:**
1. Load the current index (staged files)
2. Load the current HEAD commit's tree
3. Load the target branch's commit tree
4. For each file that differs between current and target trees:
   - Check if the file in working directory matches the index entry (compare mtime/size, or rehash)
   - If working directory differs from index: **uncommitted change detected**
   - Check if the index entry differs from HEAD tree: **staged change detected**
   - If either exists AND the file differs in target tree: **conflict - refuse checkout**

**Using only index and object store:**
- Index provides: staged file hashes, mtimes, sizes
- Object store provides: HEAD tree's file hashes, target tree's file hashes
- Compare working file's mtime/size against index to detect modifications
- Compare index hash against HEAD tree hash to detect staged changes
- Compare HEAD tree hash against target tree hash to detect branch differences
- Conflict exists when: (working ≠ index OR index ≠ HEAD) AND (HEAD ≠ target)

### Q5.3: Detached HEAD state

**What happens in detached HEAD:**
When HEAD contains a commit hash directly (e.g., `a1b2c3d4...`) instead of a branch reference (e.g., `ref: refs/heads/main`), you're in "detached HEAD" state.

**Making commits in this state:**
- Commits work normally: new commit points to current HEAD as parent
- HEAD updates to point to the new commit hash
- **Problem:** No branch reference is updated
- The commit chain exists but is "floating" - not reachable from any branch

**Example:**
```
Before: HEAD → a1b2c3 → a0b1c2 → ...
        main → a1b2c3

Commit in detached state:
        HEAD → b2c3d4 → a1b2c3 → a0b1c2 → ...
        main → a1b2c3
        (b2c3d4 is orphaned)
```

**Recovery:**
1. **If you remember the commit hash:** Create a branch pointing to it:
   ```bash
   pes branch recover-branch b2c3d4
   ```

2. **If you forgot the hash:** Check the reflog (if implemented):
   ```bash
   git reflog  # Shows recent HEAD movements
   ```
   Find the commit hash and create a branch

3. **Without reflog:** The commits become unreachable and will be garbage collected eventually. They're effectively lost unless you can find the hash in terminal history or other logs.

**Prevention:** Git warns you when entering detached HEAD state and reminds you to create a branch if you want to keep commits.

---

## Phase 6: Garbage Collection (Analysis)

### Q6.1: Finding and deleting unreachable objects

**Algorithm to find unreachable objects:**

1. **Mark phase** - Find all reachable objects:
   ```
   reachable = new HashSet()
   
   for each branch in .pes/refs/heads/*:
       commit_hash = read_file(branch)
       mark_reachable(commit_hash, reachable)
   
   function mark_reachable(hash, reachable):
       if hash in reachable:
           return  # Already visited
       
       reachable.add(hash)
       obj = read_object(hash)
       
       if obj.type == COMMIT:
           mark_reachable(obj.tree_hash, reachable)
           if obj.has_parent:
               mark_reachable(obj.parent_hash, reachable)
       
       elif obj.type == TREE:
           for entry in obj.entries:
               mark_reachable(entry.hash, reachable)
       
       # Blobs have no references, so we're done
   ```

2. **Sweep phase** - Delete unreachable objects:
   ```
   for each file in .pes/objects/*/*:
       hash = extract_hash_from_path(file)
       if hash not in reachable:
           delete(file)
   ```

**Data structure:** HashSet (or hash table) for O(1) lookup of reachable hashes.

**Complexity estimation for 100,000 commits and 50 branches:**
- Assume average 10 files per commit, 5 commits per branch = 250 commits reachable from branch tips
- Assume average depth of 100 commits per branch
- Total reachable commits: ~5,000 (with overlap between branches)
- Total reachable trees: ~5,000 (one per commit)
- Total reachable blobs: ~50,000 (10 files × 5,000 commits, with deduplication)
- **Total objects to visit: ~60,000**

The HashSet would need to store ~60,000 hashes (32 bytes each) = ~1.9 MB memory.

### Q6.2: Race condition in concurrent GC

**Dangerous race condition:**

```
Time  | Commit Process                | GC Process
------|-------------------------------|---------------------------
T0    | Create blob for new file      |
      | (blob hash: abc123)           |
T1    |                               | Start mark phase
T2    |                               | Mark all reachable objects
T3    |                               | (abc123 not yet referenced)
T4    | Create tree referencing       |
      | abc123                        |
T5    |                               | Sweep phase: DELETE abc123
T6    | Create commit referencing     |
      | tree (which references        |
      | abc123 - now deleted!)        |
T7    | Repository is corrupted!      |
```

**The problem:** GC marks objects as reachable based on current refs, but a concurrent commit is creating new objects that will soon be referenced. The new blob exists but isn't yet reachable from any branch, so GC deletes it. When the commit completes, it references a deleted object.

**How Git avoids this:**

1. **Grace period:** Git's GC doesn't delete objects immediately. Objects must be unreachable AND older than a threshold (default: 2 weeks for loose objects). This makes it extremely unlikely that a just-created object gets deleted.

2. **Object creation timestamps:** Git checks the mtime of object files. Recently created objects are never deleted, even if unreachable.

3. **Locking:** Git uses lock files (`.git/gc.pid`) to prevent multiple GC processes from running simultaneously. Commit operations check for this lock.

4. **Two-phase deletion:** 
   - Phase 1: Move unreachable objects to a "quarantine" area
   - Phase 2: After a delay, delete quarantined objects
   - If an object becomes reachable during the delay, it's restored

5. **Atomic reference updates:** Branch updates are atomic (via rename). GC re-checks refs after marking but before sweeping to catch any new commits.

**Best practice:** Run GC only when no other Git operations are active, or implement proper locking and grace periods.

---

## Commit History

This repository contains detailed commits for each phase of development:
- Phase 1: Minimum 5 commits for object storage implementation
- Phase 2: Minimum 5 commits for tree object implementation  
- Phase 3: Minimum 5 commits for index implementation
- Phase 4: Minimum 5 commits for commit creation implementation

Each commit message clearly describes the changes made and follows best practices.

---

## Repository Link

GitHub Repository: https://github.com/SunOrangeBurger/PES2UG24AM126-pes-vcs

---

## Important: Running Commands

All `pes` commands require the `PES_AUTHOR` environment variable to be set. Before running any commands:

```bash
source .env
```

Or run commands with the environment variable inline:
```bash
export PES_AUTHOR="Arun Hariharan <PES2UG24AM126>" && ./pes [command]
```

---

## Declaration

I declare that this submission is my own work and I have properly attributed any external resources used.

**Signature:** Arun Hariharan  
**Date:** 17/4/2026
