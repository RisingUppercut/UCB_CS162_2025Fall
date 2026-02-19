# Project File System Design
Group X (replace X with your group number)

| Name | Autograder Login | Email |
| ---- | ---------------- | ----- |
|     WeiHan  |      Xia            |   dvd123@ruc.edu.cn   |
|  Nai    |   Loong               |  dvd123@ruc.edu.cn     |
|   Ha   |  JiMi                |  dvd123@ruc.edu.cn     |
|   North South   |       Green Bean           |  dvd123@ruc.edu.cn     |

----------

***See more details in code***

# Buffer Cache Design Document

## 1. Data Structures and Functions

### 1.1 Cache Control Structures

I implement a buffer cache with a maximum capacity of 64 disk blocks. The following structures are added to `filesys/filesys.c`:

```c
#define BUFFER_CACHE_SIZE 64
#define CACHE_FLUSH_FREQ (5 * TIMER_FREQ) 
#define READ_AHEAD_SIZE 32

/* Represents a single block in the buffer cache. */
struct cache_entry {
  bool dirty;                  /* True if data was modified and needs to be written back. */
  bool accessed;               /* Used by the clock algorithm to track recent access. */
  bool valid;                  /* True if the entry currently contains valid disk data. */
  bool is_loading;             /* True if the entry is undergoing an I/O operation. */
  block_sector_t sector;       /* The disk sector number currently stored. */
  block_sector_t target_sector;/* The sector number intended to be loaded (prevents race). */
  int pin;                     /* Reference count to prevent eviction while in use. */
  struct lock entry_lock;      /* Fine-grained lock protecting the data buffer. */
  struct condition is_loaded;  /* Condition variable to signal completion of I/O. */
  uint8_t data[BLOCK_SECTOR_SIZE]; /* 512-byte buffer for the sector data. */
};

/* Global cache state. */
static struct cache_entry buffer_cache[BUFFER_CACHE_SIZE]; /* The 64-sector cache. */
static struct lock cache_lock;      /* Global lock for metadata and eviction pointer. */
static int clock_hand;              /* Current hand position for the Clock algorithm. */

/* Read-ahead Queue. */
static block_sector_t ra_queue[READ_AHEAD_SIZE]; /* Circular buffer for read-ahead requests. */
static struct lock ra_lock;         /* Protects read-ahead queue pointers. */
static struct semaphore ra_sema;    /* Number of tasks for the read-ahead worker. */
static struct semaphore ra_free_slots; /* Available slots in the read-ahead queue. */
```

---

### 1.2 Function Prototypes

```c
static void cache_init(void);
/* Initializes locks, condition variables, and starts worker threads. */

struct cache_entry* cache_get_entry(block_sector_t sector);
/* Retrieves an entry from cache, fetching from disk if missing. */

static struct cache_entry* cache_evict(void);
/* Implements the Clock algorithm to find a victim for eviction. */

static void cache_flush(void);
/* Iterates through the cache and writes dirty blocks back to disk. */
```

---

## 2. Algorithms

### 2.1 Cache Lookup and Replacement (Clock Algorithm)

When a process requests a sector via `cache_get_entry`:

#### Lookup

I acquire `cache_lock` and scan the `buffer_cache` for the requested sector.

#### Hit

If found and valid:

- Increment the `pin` count and return.
- If the entry is `is_loading`, the thread waits on `is_loaded`.

#### Miss and Eviction

If the sector is not found, call `cache_evict`. This function uses the **Clock Algorithm**:

1. It iterates starting from `clock_hand`.
2. It skips entries that are pinned (`pin > 0`) or currently loading.
3. If `accessed` is true, it sets it to false and moves on.
4. The first entry found with `accessed == false` is chosen.

#### I/O Synchronization

After choosing a victim:

1. Set `is_loading = true`.
2. Release `cache_lock`.
3. Perform `block_write` (if dirty).
4. Perform `block_read` (for the new sector).
5. Update metadata.
6. Broadcast to any waiting threads.

---

### 2.2 Direct Data Transfer (Removing Bounce Buffers)

I replaced the original Pintos malloc-based bounce buffers in `inode.c`.

The new `cache_read` and `cache_write` functions directly use `memcpy` to transfer data between the caller's buffer and the `cache_entry->data` buffer while holding the `entry_lock`.

This ensures:

- No intermediate bounce buffer
- Reduced memory allocation overhead
- Direct data movement between cache and user/system buffer

---

### 2.3 Background Operations

#### Periodic Flush

I copy the non busy waiting timer sleep in proj2.

The `cache_flusher` thread:

- Sleeps using `timer_sleep`
- Wakes up every 5 seconds
- Flushes all dirty blocks that are not currently pinned

#### Asynchronous Read-Ahead

When `cache_read_ahead` is triggered:

1. It places the next sector into `ra_queue`.
2. A dedicated worker thread processes this queue.
3. If the queue is full, the request is dropped to avoid blocking the main execution path.

---

## 3. Synchronization

### 3.1 Global vs. Fine-Grained Locking

I utilize a hierarchical locking strategy to maximize concurrency.

#### Cache-Global `cache_lock`

- Protects metadata and the lookup process.
- Held only for short durations.
- Released before any disk I/O (`block_read`/`block_write`) occurs.

This ensures disk latency does not block other cache operations.

#### Per-entry `entry_lock`

- Each cache entry has its own lock.
- Allows multiple threads to operate on different blocks in parallel.

---

### 3.2 Race Conditions in Loading

To prevent duplicate loads of the same sector:

1. The first thread sets `is_loading = true` and assigns `target_sector`.
2. Other threads:
   - See matching `target_sector`
   - Detect `is_loading == true`
   - Wait on the condition variable
3. The first thread:
   - Completes I/O
   - Clears `is_loading`
   - Signals all waiting threads

This prevents the "thundering herd" problem.

---

### 3.3 Pinning for Eviction Safety

The `pin` variable prevents eviction of blocks in active use.

- No block with `pin > 0` can be chosen as a victim.
- Protects against eviction during `memcpy`.
- Ensures correctness during concurrent access.

---

## 4. Rationale

### 4.1 Choice of Approximation of MIN

We chose the **Clock Algorithm** because:

- It approximates LRU well.
- It avoids expensive list operations.
- Only requires setting a single bit (`accessed = true`).
- Minimizes locking overhead.

Compared to full LRU, this design significantly reduces contention on global structures.

---

### 4.2 Removing the Global File System Lock

I eliminate the global filesystem lock through:

- Fine-grained per-entry locks
- Releasing `cache_lock` during disk I/O
- Using reference counting (`pin`) to manage lifecycle

This improves throughput because:

- Disk I/O (milliseconds) does not block
- Cache hits (nanoseconds) can proceed in parallel

---

### 4.3 Reliability vs. Performance

I use a **write-back policy**:

- Coalesces multiple writes into a single disk operation
- Improves performance significantly

To reduce crash risk:

- A periodic flusher writes dirty blocks every 5 seconds

This balances:

- High performance
- Acceptable durability guarantees

---

### 4.4 Read-Ahead Strategy

Asynchronous read-ahead:

- Improves sequential read performance
- Hides disk latency
- Uses a producer-consumer queue
- Never blocks the main execution path

This ensures prefetching remains a background optimization rather than a performance bottleneck.

----------

# Extensible Files

## 1. Data Structures and Functions

### 1.1 Inode Structures

I modify the on-disk and in-memory inode structures to support indexed block mapping and synchronization:

```c
/* On-disk inode. Must be exactly 512 bytes. */
struct inode_disk {
    block_sector_t direct_blocks[12];   /* 12 direct pointers (6KB). */
    block_sector_t indirect;            /* One indirect pointer (64KB). */
    block_sector_t double_indirect;     /* One doubly indirect pointer (8MB). */
    off_t length;                       /* Current file size in bytes. */
    bool is_dir;                        /* Flag to distinguish directories. */
    unsigned magic;                     /* Safety check number. */
    uint32_t unused[111];               /* Padding to reach 512 bytes. */
};

/* In-memory inode. */
struct inode {
    struct list_elem elem;              /* Link for the global open_inodes list. */
    block_sector_t sector;              /* On-disk sector location of the inode_disk. */
    int open_cnt;                       /* Reference count for openers. */
    bool removed;                       /* Flag indicating the file is marked for deletion. */
    int deny_write_cnt;                 /* Counter to implement write-exclusion. */
    struct lock lock;                   /* Per-inode lock to synchronize R/W and growth. */
};
```

---

### 1.2 Global Synchronization

```c
static struct list open_inodes;         /* Shared list of all currently open inodes. */
static struct lock open_inodes_lock;    /* Lock protecting the open_inodes list and open_cnt. */
```

---

## 2. Algorithms

### 2.1 Indexed Inode Structure (FFS-style)

To support files up to 8 MiB and avoid external fragmentation, I use a three-level indexing scheme:

- **Direct Pointers (12)**  
  Points to 12 data blocks (12 × 512 bytes = 6 KiB).

- **Indirect Pointer (1)**  
  Points to a block containing 128 pointers to data blocks  
  (128 × 512 bytes = 64 KiB).

- **Doubly Indirect Pointer (1)**  
  Points to a block containing 128 indirect pointers,  
  each pointing to 128 data blocks  
  (128 × 128 × 512 bytes = 8 MiB).
---

### 2.2 Offset to Sector Translation

`inode_byte_to_sector` performs the translation:

- **Direct:**  
  Return `direct_blocks[index]`.

- **Indirect:**  
  Access indirect sector through buffer cache,  
  find the pointer at `index - 12`.

- **Doubly Indirect:**  
  Perform a two-level lookup:  
  1. Find the indirect block within the doubly indirect block.  
  2. Then find the data sector inside that indirect block.

I use `cache_pin_and_lock` to ensure metadata blocks remain in the buffer cache during translation.

---

### 2.3 File Growth and Sparse Files

File growth is triggered in `inode_write_at` if:

```
offset + size > length
```

#### `inode_grow`

This function:

- Iterates from the current sector count to the required sector count.
- Allocates new blocks via `free_map_allocate`.
- Populates the indexing structure accordingly.

#### Zero-padding

Newly allocated data blocks and index blocks are zeroed out using `cache_write` to:

- Ensure consistency
- Handle gaps (implicit zeros) when writing past EOF

#### Resource Exhaustion (Rollback)

If `free_map_allocate` fails due to a full disk:

- The function jumps to a rollback label.
- Calls `inode_shrink` to release any blocks allocated during the failed operation.
- Ensures file system consistency.
- Prevents space leakage.

---

## 3. Synchronization

### 3.1 Concurrent Access to Inodes

#### `inode->lock`

Each in-memory `struct inode` has a lock. It is acquired during:

- `inode_read_at`
- `inode_write_at`

This ensures:

- Multiple writers are serialized when extending the file.
- Reads do not occur while index structures are being modified.

#### Buffer Cache Pinning

When accessing index blocks (indirect/double indirect):

- I use the buffer cache’s pin mechanism.
- Ensures index blocks cannot be evicted during address translation.

---

### 3.2 Global Inode List

#### `open_inodes_lock`

This lock protects the `open_inodes` list.

It is held when:

- Opening a file (`inode_open`) to check if it is already in memory.
- Closing a file (`inode_close`) to safely decrement `open_cnt`.
- Removing an inode from the list when `open_cnt` reaches zero.

---

### 3.3 Deletion Safety

If a file is removed while still open (`inode_remove`):

- set `inode->removed = true`.

Actual disk blocks are freed only in `inode_close` when open_cnt == 0. Since `open_cnt` is protected by `open_inodes_lock`,  
race conditions between deletion and reopening are avoided.

---

## 4. Rationale

### 4.1 Indexed Structure vs. FAT

I chose an indexed structure (FFS-style) over a FAT design.

#### Random Access Speed

For large files (8 MiB):

- FAT requires traversing a linked list in the FAT table → O(N).
- Indexed structure provides:
  - O(1) for direct blocks
  - O(1) or O(2) for indirect levels

Performance is independent of file size depth.

#### Metadata Overhead

- Metadata is localized within inode/index blocks.
- Avoids maintaining a large global FAT table in memory.

---

### 4.2 Handling Growth and Failure

The rollback mechanism in `inode_grow` ensures robustness.

Using `inode_shrink` to undo partial allocations:

- Handles disk exhaustion gracefully.
- Maintains consistency.
- Prevents space leaks.

Zero-filling newly allocated blocks ensures:

- Gaps are zeroed when writing past EOF.
- No leakage of stale disk data.

---

### 4.3 Scalability

With:

- 12 direct
- 1 indirect
- 1 doubly indirect pointer

Maximum file size:

```
12 + 128 + (128 × 128) = 16,524 sectors
16,524 × 512 bytes = 8,460,288 bytes ≈ 8.06 MiB
```

This:

- Meets the 8 MiB requirement
- Fits within a single 512-byte `inode_disk`

---

### 4.4 Efficiency

I integrated the buffer cache into inode operations (`cache_pin_and_lock`).

Benefits:

- Frequently accessed metadata (e.g., doubly indirect block) stays in memory.
- Minimizes disk I/O during address translation.
- Improves overall filesystem performance.


----------

# Subdirectories

## 1. Data Structures and Functions

### 1.1 Directory Management

```c
/* A directory object in memory. */
struct dir {
    struct inode* inode; /* Backing store (the inode representing this directory). */
    off_t pos;           /* Current reading position for readdir. */
};

/* A single directory entry on disk. */
struct dir_entry {
    block_sector_t inode_sector; /* Sector number of the file/subdirectory inode. */
    char name[NAME_MAX + 1];      /* Null-terminated file name. */
    bool in_use;                  /* True if the entry is active, false if deleted. */
};

/* Process Control Block (PCB) addition in process.h */
struct pcb {
    ...
    struct dir* cwd;              /* Current working directory of the process. */
    ...
};
```

---

### 1.2 Path Resolution Functions

```c
bool dir_resolve_path(const char* path,
                      struct dir** target_dir,
                      char* file_name);
/*
 * Parses a string path (e.g., "/a/b/c").
 * Returns:
 *   - target_dir: directory object for "/a/b"
 *   - file_name:  "c"
 */

static int get_next_part(char part[NAME_MAX + 1],
                         const char** srcp);
/*
 * Tokenizes the path string by slashes.
 */
```

---

## 2. Algorithms

### 2.1 Path Resolution (Absolute vs. Relative)

Path resolution is handled by `dir_resolve_path`.

#### Starting Point

- If the path starts with `/`, begin at `dir_open_root()`.
- Otherwise, begin at `thread_current()->pcb->cwd`.

#### Traversal

- Use `get_next_part` to extract each path component.
- For each component except the last:
  1. Look up the name in the current directory using `dir_lookup`.
  2. Open the resulting inode.
  3. Convert it to a `struct dir`.

- Handle special cases:
  - `.`  → current directory
  - `..` → parent directory (stored during `dir_create`)

#### Output

The function returns:

- The `struct dir` of the immediate parent directory.
- The final component name separately.

This design simplifies system calls such as `mkdir` and `create`.

---

### 2.2 Directory Operations

#### Creation (`mkdir`)

- `dir_create` allocates a new extensible inode.
- Immediately inserts two entries:
  - `.`  → points to itself
  - `..` → points to parent directory's sector

This enables recursive upward traversal.

#### Deletion (`remove`)

- Call `dir_is_empty` before deletion.
- A directory is considered empty only if it contains no entries other than:
  - `.`
  - `..`

- Prevent removal of the root directory.

#### Expansion

- Directories are backed by inodes.
- They reuse the **Extensible Files** logic.
- If `dir_add` finds no empty slot:
  - `inode_write_at` automatically extends the directory file.

---

### 2.3 System Call Implementation

#### `chdir`

- Uses `dir_resolve_path` to locate target directory.
- Closes old `t->pcb->cwd`.
- Sets new working directory.

#### `readdir`

- Reads entries from directory inode.
- Explicitly skips:
  - `.`
  - `..`

User programs should not see these entries.

#### `open`

- Can open both files and directories.
- `sys_isdir` allows user programs to distinguish them via file descriptor.

---

## 3. Synchronization

### 3.1 Thread Safety

#### Per-Directory Locking

All directory operations:

- `dir_add`
- `dir_remove`
- `dir_lookup`
- `dir_readdir`

Internally call:

- `inode_read_at`
- `inode_write_at`

Since these acquire `inode->lock`, directory operations are thread-safe.

---

#### Path Traversal Races

If a directory in the middle of traversal is deleted:

- `dir_lookup` fails, or
- `inode_is_removed` check inside `dir_resolve_path` triggers.

In both cases, the operation fails gracefully.

---

#### Reference Counting

Using:

- `inode_open`
- `inode_close`

Ensures a directory's sector is not freed while:

- It is a process's `cwd`, or
- It has an open file descriptor.

---

### 3.2 CWD Inheritance

In `process_fork`:

```c
dir_reopen(parent->pcb->cwd);
```

This increments `open_cnt` of the inode.

---

## 4. Rationale

### 4.1 ".." Storage Strategy

We store `..` as a regular `dir_entry` created at directory creation time.

#### Alternative

- Compute parent dynamically using an inode back-pointer.

#### Why This Is Better

- No special-case logic in path traversal.
- `cd ..` behaves identically to `cd some_dir`.
- Reuses existing `dir_lookup` logic.
- Keeps implementation clean and uniform.

---

### 4.2 Handling the Root Directory

The root directory has no parent.

Implementation:

```c
dir_create(ROOT_DIR_SECTOR, ROOT_DIR_SECTOR);
```

This makes `..` point to itself.

Matches Unix behavior:

```
cd /
cd ..
```

Still results in `/`.

---

### 4.3 Path Parsing Logic

`get_next_part` + iterative `dir_resolve_path`:

- Avoids deep recursion.
- Important in Pintos (4KB kernel stack).
- Uses constant memory.

Space complexity:

```
O(1)
```

Regardless of path depth.

---

### 4.4 Consistency and Cleanup

Using `inode_is_removed` inside `dir_lookup` ensures:

- A process cannot create new files in a directory that has been:
  - Unlinked (removed)
  - But still open

Maintains logical consistency of the filesystem.



----------
# Concept check
1.  trivial

