# Linux Kernel: Page Fault Handling and Page Replacement

## Overview

Page fault handling and page replacement are core components of Linux virtual memory management. Page faults occur when a process tries to access memory that isn't currently in physical RAM, and the kernel must decide how to handle it. Page replacement determines which pages get evicted from memory when space is needed.

---

## Part 1: Page Fault Handling

### What is a Page Fault?

A page fault occurs when:
- A process accesses a virtual address that doesn't have a valid translation in the page table
- The memory page exists but isn't loaded in physical RAM
- Permission checks fail (e.g., write to read-only page)

### Page Fault Entry Point

**File:** `mm/memory.c`

The main entry point for page fault handling varies by architecture, but all architectures eventually call `handle_mm_fault()`:

```c
vm_fault_t handle_mm_fault(struct vm_area_struct *vma, unsigned long address,
                           unsigned int flags)
```

**Parameters:**
- `vma`: Virtual memory area structure containing memory region information
- `address`: The faulting virtual address
- `flags`: Flags indicating fault type (read, write, user/kernel, etc.)

### Fault Handling Flow

#### 1. **Initial Checks and Locking**

```c
// Lock the VMA using read-write lock
mmap_read_lock(mm);
```

The kernel acquires locks to protect page table modifications from concurrent access.

#### 2. **Handle PTE-level Faults**

The function `handle_pte_fault()` processes page table entry (PTE) level faults:

```c
static vm_fault_t handle_pte_fault(struct vm_fault *vmf)
```

**Key branches:**

**a) Entry is not present (`!pte_present(entry)`):**
   - Page is swapped to disk or never allocated
   - Call `do_swap_page()` to load from swap space
   - Or call `do_anonymous_page()` for new anonymous pages

**b) Entry is present (`pte_present(entry)`):**
   - Page is in memory but permission/access issue exists
   - Check for write faults on copy-on-write (CoW) pages
   - Call `do_wp_page()` for write-protect faults

**c) Entry has special handling:**
   - Call `do_pte_missing()` for special cases

#### 3. **Major Fault Types**

**A. Anonymous Page Fault (New page allocation)**

```c
static vm_fault_t do_anonymous_page(struct vm_fault *vmf)
```

- Allocates a new physical page from the page allocator
- Zeros the page (for security)
- Maps it to the page table entry
- Returns control to user process

**B. Swap-in Fault**

```c
static vm_fault_t do_swap_page(struct vm_fault *vmf)
```

- Reads page from swap space (disk)
- Allocates a new physical page
- Reads data from swap
- Updates page table
- Sets page as present in memory

**C. Copy-on-Write (CoW) Fault**

```c
static vm_fault_t do_wp_page(struct vm_fault *vmf)
```

- Triggered when process tries to write to a shared, read-only page
- Creates a copy of the page for the current process
- Maps the copy to the process's page table
- Original page remains for other processes

**D. File-backed Page Fault**

```c
static vm_fault_t do_fault(struct vm_fault *vmf)
```

- Maps pages from files (executables, memory-mapped files)
- Reads data from filesystem into page cache
- Updates page table to point to file page

### Page Fault Decision Tree

```
Page Fault
    ↓
Lock VMA and page tables
    ↓
Is entry present in page table?
    ├─→ YES: Permission violation?
    │        ├─→ YES: Copy-on-Write (do_wp_page)
    │        └─→ NO: (rare) return error
    │
    └─→ NO: What type of memory?
             ├─→ Anonymous: do_anonymous_page
             ├─→ Swap: do_swap_page
             ├─→ File-backed: do_fault
             └─→ Special: Handle specially
    ↓
Update page table
    ↓
Return to user code
```

### Fault Flags

The `flags` parameter indicates fault characteristics:

| Flag | Meaning |
|------|---------|
| `FAULT_FLAG_WRITE` | Write access attempted |
| `FAULT_FLAG_USER` | User-mode access |
| `FAULT_FLAG_INSTRUCTION` | Instruction fetch |
| `FAULT_FLAG_KILLABLE` | Can be interrupted |
| `FAULT_FLAG_RETRY_NOWAIT` | Retry without waiting |

### Return Values

Page fault handlers return `vm_fault_t` enum values:

```c
#define VM_FAULT_OK        0
#define VM_FAULT_SIGBUS    (1 << 0)  // Bus error signal
#define VM_FAULT_MAJOR     (1 << 1)  // Disk I/O needed
#define VM_FAULT_MINOR     0         // Page was in RAM
#define VM_FAULT_OOM       (1 << 3)  // Out of memory
```

**MAJOR vs MINOR faults:**
- **Major fault**: Requires disk I/O (slow, ~milliseconds)
- **Minor fault**: Page already in RAM (fast, ~microseconds)

---

## Part 2: Page Replacement

### What is Page Replacement?

Page replacement is the algorithm the kernel uses to decide **which page to evict from physical RAM** when memory is full and a new page needs to be allocated.

### Memory Pressure and Reclaim

When memory becomes scarce, the kernel initiates **page reclaim** to free up pages. This happens through:

1. **Direct reclaim**: Triggered by allocation failures
2. **Background reclaim**: Done by kswapd daemon

### The Page Reclaim Path

**File:** `mm/vmscan.c`

#### 1. **Shrinking Zones**

```c
static unsigned long shrink_zone(struct lruvec *lruvec, 
                                  struct scan_control *sc)
```

- Iterates through LRU (Least Recently Used) lists
- Scans and reclaims pages from multiple lists

#### 2. **Shrinking Folio Lists**

```c
static unsigned long shrink_folio_list(struct list_head *folio_list,
                                       struct pglist_data *pgdat,
                                       struct scan_control *sc)
```

**Process:**

**Step 1: Page Status Check**
```c
// Check if page is still referenced
if (folio_referenced(folio) > 0) {
    // Page is still in use, keep it
    continue;
}
```

**Step 2: Dirty Page Handling**
```c
// Check if page has been modified
if (folio_test_dirty(folio)) {
    // Write dirty page back to disk/swap
    if (!pageout(folio, mapping, sc)) {
        // I/O blocked, try later
        continue;
    }
}
```

**Step 3: Page Removal**
```c
// If all checks pass, remove the page
folio_list_del(folio);
folio_put(folio);  // Decrement reference count
```

### LRU (Least Recently Used) Lists

The kernel maintains **per-zone** LRU lists:

```c
struct lruvec {
    struct list_head lists[NR_LRU_LISTS];
};
```

**LRU List Types:**

| List | Purpose |
|------|---------|
| `LRU_INACTIVE_ANON` | Inactive anonymous pages (swap candidates) |
| `LRU_ACTIVE_ANON` | Active anonymous pages (recently used) |
| `LRU_INACTIVE_FILE` | Inactive file-backed pages (cache) |
| `LRU_ACTIVE_FILE` | Active file pages (working set) |
| `LRU_UNEVICTABLE` | Pages that can't be evicted (pinned, etc.) |

### LRU Page Movement

```
New page allocated
    ↓
Added to LRU_INACTIVE list
    ↓
Referenced (accessed)?
    ├─→ YES: Move to LRU_ACTIVE
    │        (more recently used)
    │
    └─→ NO: Stays INACTIVE
             (candidate for eviction)
    ↓
Time passes without access
    ↓
Moves from ACTIVE → INACTIVE
    ↓
Memory pressure?
    ├─→ YES: Evict from LRU_INACTIVE
    │
    └─→ NO: Keep in memory
```

### Page Replacement Algorithm

Linux uses a **modified LRU** algorithm with the following characteristics:

#### 1. **Two-list LRU System**

- **ACTIVE list**: Frequently accessed, working set
- **INACTIVE list**: Cold pages, reclaim candidates

This prevents one-time accesses from polluting the cache.

#### 2. **Second Chance**

When a page is about to be evicted:
```c
// Check if page was recently accessed
if (folio_referenced(folio) > 0) {
    // Give it another chance
    move_to_active_list(folio);
}
```

Pages accessed before eviction are given another chance.

#### 3. **Reference Bit Tracking**

```c
// Mark page as accessed
set_page_accessed(page);

// Check recent accesses
folio_referenced(folio);
```

The kernel tracks access patterns to inform LRU decisions.

#### 4. **Swap vs. File Cache Balancing**

```c
// Decide between evicting anonymous pages vs. file cache
if (anon_age > file_age) {
    // Swap out anonymous pages
    evict_anon_pages();
} else {
    // Reclaim file cache
    evict_file_pages();
}
```

The kernel balances between:
- **Anonymous pages** (must go to swap)
- **File-backed pages** (can be dropped if clean or written back)

### Swap Space

When anonymous pages are evicted, they go to **swap space**:

```c
// Write page to swap
int add_to_swap(struct folio *folio)
{
    struct swap_info_struct *si;
    swp_entry_t entry;
    
    // Allocate swap slot
    entry = get_swap_page(folio);
    
    // Write page to swap space
    write_to_swap(folio, entry);
}
```

**Swap characteristics:**
- Typically a disk partition or file
- Much slower than RAM (100-1000x slower)
- Used only when memory is scarce
- Swap-in triggers page faults

### Page Reclaim Zones

The kernel reclaims memory in **zones**:

```c
enum zone_type {
    ZONE_NORMAL,      // Regular 32-bit addressable RAM
    ZONE_DMA,         // DMA-safe memory for devices
    ZONE_HIGHMEM,     // High memory (32-bit systems only)
};
```

When a zone is under pressure, kswapd reclaims pages from that zone.

### kswapd Daemon

The kernel swap daemon `kswapd` runs continuously:

```c
// kswapd main loop (simplified)
while (!should_stop) {
    // Check if zones are under pressure
    if (zone_watermark_ok(zone)) {
        // Enough free pages, sleep
        sleep();
    } else {
        // Trigger reclaim
        shrink_zone(zone);
    }
}
```

**Watermarks:**
- **LOW**: Trigger kswapd (background reclaim)
- **MIN**: Trigger direct reclaim (blocking)
- **HIGH**: All zones satisfied

---

## Interaction: Fault + Replacement

Here's how page fault and replacement work together:

```
Process accesses unmapped page
    ↓
Page Fault Exception
    ↓
allocate_page() called
    ↓
Free pages > LOW watermark?
    ├─→ YES: Allocate from free list
    │
    └─→ NO: Trigger page replacement
             (shrink_zone)
    ↓
Replacement evicts cold pages
    ↓
Free memory available
    ↓
Page fault handler completes
    ↓
New page mapped to faulting address
    ↓
Process resumes execution
```

### Example Sequence

1. **Process reads from anonymous page that was swapped out**
   - Page fault triggered
   - `do_swap_page()` called
   - Needs free page to load swap data into

2. **No free pages available**
   - kswapd wakes up (or direct reclaim triggered)
   - LRU lists scanned
   - Least recently used pages evicted
   - Dirty pages written to swap/disk

3. **Free page becomes available**
   - Swap-in completes
   - Page table updated
   - Process continues

---

## Summary

### Page Fault Handling
- **Entry point**: `handle_mm_fault()`
- **Decision**: Determine fault type (anonymous, swap, file, CoW)
- **Action**: Allocate page, load from swap/disk, or create copy
- **Result**: Map page to virtual address, resume execution

### Page Replacement
- **Algorithm**: Modified LRU with active/inactive lists
- **Trigger**: Memory pressure (zones below watermarks)
- **Reclaim**: `kswapd` background daemon or direct reclaim
- **Selection**: Choose cold pages from LRU_INACTIVE lists
- **Destination**: Write to swap space or discard if clean cache

### Key Data Structures
- `vm_area_struct`: Virtual memory region
- `mm_struct`: Process memory management
- `lruvec`: Per-zone LRU lists
- `page` / `folio`: Physical page descriptors
- `swap_info_struct`: Swap space management

---

## Performance Implications

| Operation | Type | Typical Time |
|-----------|------|--------------|
| L1 cache hit | Minor fault | ~4 ns |
| L2/L3 cache hit | Minor fault | ~10 ns |
| RAM access | Minor fault | ~100 ns |
| Swap I/O | Major fault | ~10 ms |
| Disk I/O | Major fault | ~5-10 ms |

Excessive major faults indicate memory overcommitment and should be monitored.
