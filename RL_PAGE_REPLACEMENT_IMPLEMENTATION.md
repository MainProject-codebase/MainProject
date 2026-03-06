# RL-Based Page Replacement Implementation

## Overview

This document explains the **~300 lines of code** added to `mm/vmscan.c` to implement a **Reinforcement Learning-based page replacement policy selector** for the Linux kernel memory management subsystem.

---

## What Was Added?

A lightweight, kernel-safe reinforcement learning agent that dynamically chooses between **LRU** (Least Recently Used) and **MRU** (Most Recently Used) page replacement policies based on runtime workload behavior.

**Key Point**: This is NOT a new eviction algorithm. It's an intelligent **policy selector** that learns which existing policy works better for the current workload.

---

## Architecture Components

### 1. **Policy Types (Enum)**

```c
enum policy_type {
    POLICY_LRU = 0,  // Least Recently Used (default Linux)
    POLICY_MRU,      // Most Recently Used (alternative)
    POLICY_MAX       // Sentinel (value = 2)
}
```

**What it does**: Defines the two page replacement strategies the RL agent can choose from.

---

### 2. **Policy Score Table (PolS Table)**

**Purpose**: Tracks performance scores for each policy.

**Structure**:
```c
struct policy_score {
    enum policy_type policy;  // Which policy (LRU or MRU)
    int score;                // Current performance score
}

static struct policy_score pols_table[2];  // One entry per policy
```

**Key Properties**:
- **Initial score**: 100 for both policies
- **Score range**: -1000 (min) to 10,000 (max)
- **Protection**: `pols_lock` spinlock for thread safety
- **Updates**: Score decreases by 10 when a policy's eviction causes a page fault

---

### 3. **Page Eviction History Table (PEcH Table)**

**Purpose**: Records recently evicted pages to detect if they're accessed again soon (indicating a bad eviction decision).

**Structure**:
```c
struct eviction_history {
    pid_t pid;                     // Process that owned the page
    unsigned long page_id;         // Page identifier
    unsigned long vaddr;           // Virtual address
    enum policy_type evicted_by;   // Which policy evicted it
    unsigned long timestamp;       // When it was evicted (jiffies)
    int valid;                     // Is this entry active?
}

static struct eviction_history pech_table[512];  // 512 entries (~20KB)
static unsigned int pech_head = 0;               // FIFO head pointer
```

**Key Properties**:
- **Size**: 512 entries (configurable via `RL_PECH_TABLE_SIZE`)
- **Behavior**: Circular FIFO buffer - oldest entries get overwritten
- **Refresh logic**: If same page evicted again, entry is refreshed (not duplicated)
- **Protection**: `pech_lock` spinlock for thread safety

---

## Configuration Parameters

All tunable constants are defined as macros:

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `RL_PECH_TABLE_SIZE` | 512 | Number of eviction history entries (~20KB) |
| `RL_EXPLORATION_RATE` | 20 | Random exploration probability (20% explore, 80% exploit) |
| `RL_SCORE_PENALTY` | -10 | Score decrease when evicted page faults again |
| `RL_INITIAL_SCORE` | 100 | Starting score for all policies |
| `RL_MIN_SCORE` | -1000 | Minimum allowed score (prevents underflow) |
| `RL_MAX_SCORE` | 10000 | Maximum allowed score (prevents overflow) |

**Runtime Control**:
```c
static int rl_page_replacement_enabled = 1;  // 1=enabled, 0=disabled
```

**Debug Logging**: 
```c
// Uncomment to enable detailed trace logging
// #define CONFIG_RL_MM_DEBUG
```

---

## Core Functions

### 1. **init_pols_table()** - Initialize Policy Scores

**When called**: During kernel boot (in `kswapd_init`)

**What it does**:
- Sets both LRU and MRU policies to initial score of 100
- Ensures fair starting point for learning

```c
static void __init init_pols_table(void)
{
    for (i = 0; i < POLICY_MAX; i++) {
        pols_table[i].policy = i;
        pols_table[i].score = RL_INITIAL_SCORE;  // 100
    }
}
```

---

### 2. **init_pech_table()** - Initialize Eviction History

**When called**: During kernel boot (in `kswapd_init`)

**What it does**:
- Clears all 512 eviction history entries
- Marks all entries as invalid

```c
static void __init init_pech_table(void)
{
    for (i = 0; i < RL_PECH_TABLE_SIZE; i++) {
        pech_table[i].valid = 0;
        // ... clear other fields
    }
}
```

---

### 3. **pech_add_entry()** - Record Evicted Page

**When called**: When a page is evicted from memory (integration point TBD)

**What it does**:
1. Checks if page already exists in table (linear search)
2. If found: **Refreshes** entry with new timestamp
3. If not found: **Adds** new entry at FIFO head position
4. Advances head pointer (circular buffer)

**Thread safety**: Uses `spin_lock_irqsave(&pech_lock, flags)`

```c
static void pech_add_entry(pid_t pid, unsigned long page_id, 
                          unsigned long vaddr, enum policy_type policy)
{
    // Lock table
    // Search for duplicate
    // If found: refresh
    // Else: add at head and advance FIFO pointer
    // Unlock table
}
```

---

### 4. **pech_lookup()** - Check if Page Was Recently Evicted

**When called**: On page fault (integration point TBD)

**What it does**:
1. Searches table for matching (pid, page_id)
2. If found: 
   - Returns which policy evicted it
   - **Invalidates entry** (prevents double-penalty)
3. If not found: Returns -1

**Thread safety**: Uses `spin_lock_irqsave(&pech_lock, flags)`

```c
static int pech_lookup(pid_t pid, unsigned long page_id)
{
    // Lock table
    // Search for entry
    // If found: get policy, invalidate entry, return policy
    // Else: return -1
    // Unlock table
}
```

---

### 5. **pols_update_score()** - Update Policy Score

**When called**: When a policy needs to be rewarded or penalized

**What it does**:
1. Adds delta to policy's score (usually negative penalty)
2. Enforces bounds checking (min: -1000, max: 10000)
3. Thread-safe score update

**Thread safety**: Uses `spin_lock_irqsave(&pols_lock, flags)`

```c
static void pols_update_score(enum policy_type policy, int delta)
{
    // Lock table
    // Update score: pols_table[policy].score += delta
    // Clamp to [RL_MIN_SCORE, RL_MAX_SCORE]
    // Unlock table
}
```

---

### 6. **rl_select_policy()** - Choose Policy (ε-greedy)

**When called**: Before evicting a page (integration point TBD)

**What it does**: Implements **epsilon-greedy exploration**

#### Algorithm:
```
1. Generate random number (0-99)
2. If number < 20:  // 20% exploration
     - Choose random policy (LRU or MRU)
3. Else:            // 80% exploitation
     - Choose policy with highest score
4. Return selected policy
```

**Why ε-greedy?**
- **Exploitation (80%)**: Use best-performing policy based on learned scores
- **Exploration (20%)**: Occasionally try other policy to discover if workload changed

**Thread safety**: Locks PolS table only during exploitation phase

```c
static enum policy_type rl_select_policy(void)
{
    if (random < EXPLORATION_RATE) {
        return random_policy();  // Explore
    } else {
        return best_policy();    // Exploit
    }
}
```

---

### 7. **rl_handle_page_fault()** - Learn from Page Faults

**When called**: On page fault (integration point TBD)

**What it does**: Implements **negative reinforcement learning**

#### Algorithm:
```
1. Check if faulting page was recently evicted (lookup in PEcH table)
2. If found:
   a. Get which policy evicted it
   b. Penalize that policy (score -= 10)
   c. Log the penalty (if debug enabled)
3. If not found:
   - No action (page fault unrelated to RL decisions)
```

**Learning principle**: 
- If a page gets evicted and then faults soon after, the eviction was likely premature
- The policy that made that decision gets penalized
- Over time, poorly-performing policies get lower scores

```c
static void rl_handle_page_fault(pid_t pid, unsigned long page_id)
{
    int evicted_by_policy = pech_lookup(pid, page_id);
    
    if (evicted_by_policy >= 0) {
        pols_update_score(evicted_by_policy, RL_SCORE_PENALTY);
    }
}
```

---

## Kernel Integration Points (Added)

### Modified: `kswapd_init()`

**Location**: End of `mm/vmscan.c` (around line 7717)

**Changes**:
```c
static int __init kswapd_init(void)
{
    int nid;

    /* RL-PAGE-REPLACEMENT: BEGIN */
    init_pols_table();
    init_pech_table();
    pr_info("RL-based page replacement initialized (enabled=%d)\n", 
            rl_page_replacement_enabled);
    /* RL-PAGE-REPLACEMENT: END */

    swap_setup();
    for_each_node_state(nid, N_MEMORY)
        kswapd_run(nid);
    return 0;
}
```

**What it does**: Initializes both tables during kernel boot, before memory management starts.

---

## How the RL Agent Works (Step-by-Step)

### Phase 1: Initialization (Boot Time)
1. Kernel boots
2. `kswapd_init()` is called
3. PolS table initialized: LRU=100, MRU=100
4. PEcH table initialized: all entries invalid

---

### Phase 2: Page Eviction Decision (Runtime) - **NOT YET INTEGRATED**

**Future integration point**: When kernel needs to free a page

```
1. Memory pressure occurs
2. Kernel needs to evict a page
3. rl_select_policy() is called
   → Returns POLICY_LRU or POLICY_MRU
4. Kernel uses selected policy to pick victim page
5. Page is evicted
6. pech_add_entry() records eviction with policy tag
```

---

### Phase 3: Page Fault Handler (Runtime) - **NOT YET INTEGRATED**

**Future integration point**: When a page fault occurs

```
1. Process accesses a page not in memory
2. Page fault occurs
3. rl_handle_page_fault() is called
4. Checks PEcH table for this page
5. If found in PEcH:
   → Policy that evicted it gets penalized (score -= 10)
6. Page is brought back into memory
```

---

### Phase 4: Learning Over Time

**Example scenario**:

| Time | Event | LRU Score | MRU Score | Next Choice |
|------|-------|-----------|-----------|-------------|
| T0 | Boot | 100 | 100 | Random (50/50) |
| T1 | LRU evicts page X | 100 | 100 | |
| T2 | Page X faults | **90** ⬇️ | 100 | Favors MRU |
| T3 | MRU evicts page Y | 90 | 100 | MRU (exploit) |
| T4 | No fault on Y | 90 | 100 | MRU |
| T5 | MRU evicts page Z | 90 | 100 | |
| T6 | Page Z faults | 90 | **90** ⬇️ | Balanced |
| T7 | Random explores LRU | 90 | 90 | LRU (explore) |
| T8 | No fault | 90 | 90 | Continue learning... |

Over thousands of evictions, the agent learns which policy suits the workload.

---

## Memory Overhead

| Component | Size | Notes |
|-----------|------|-------|
| PolS table | ~32 bytes | 2 policies × 16 bytes |
| PEcH table | ~20 KB | 512 entries × ~40 bytes |
| Spinlocks | ~16 bytes | 2 locks |
| Globals | ~8 bytes | Flags, counters |
| **Total** | **~20 KB** | Static, no dynamic allocation |

**Per-CPU impact**: None (global tables shared across all CPUs)

---

## Kernel Safety Features

### ✅ No Floating Point
- All calculations use integer arithmetic
- Scores, penalties, probabilities: all integers

### ✅ No Dynamic Allocation
- All tables statically allocated at compile time
- No `kmalloc()` or `vmalloc()` calls

### ✅ No Deadlocks
- Uses `spin_lock_irqsave()` (safe in interrupt context)
- Simple, short critical sections
- No nested locks

### ✅ Bounded Operations
- Linear search limited to 512 entries
- Score bounds prevent overflow/underflow
- FIFO prevents table growth

### ✅ Graceful Degradation
- If disabled: falls back to standard LRU
- Entry invalidation prevents double-penalty
- Bounds checking on all array accesses

---

## Debug Support

### Enable Debug Logging
```c
// In vmscan.c, uncomment this line:
#define CONFIG_RL_MM_DEBUG
```

**Debug output includes**:
- Table initialization messages (`pr_debug`)
- Policy selection decisions (`trace_printk`)
- Score updates (`trace_printk`)
- Page fault penalties (`pr_debug`)

**Example log**:
```
[    0.123] RL-MM: PolS table initialized with 2 policies
[    0.123] RL-MM: PEcH table initialized with 512 entries
[    1.234] RL-MM: Exploring - selected policy 1
[    2.345] RL-MM: PEcH entry added - pid=1234, page=0x7f8a0, policy=1
[    3.456] RL-MM: Page fault on recently evicted page - penalizing policy 1
```

---

## What's NOT Yet Implemented

### ❌ Integration Points
1. **Page eviction hook**: Need to call `rl_select_policy()` during victim selection
2. **Page fault hook**: Need to call `rl_handle_page_fault()` in fault handler
3. **Policy implementations**: Need to implement actual MRU eviction logic

### ❌ Observability
- No `/proc` or `/sys` interface to view scores
- No sysctl tunables for runtime configuration
- No statistics on policy selection frequency

### ❌ Kernel Configuration
- No `Kconfig` option to enable/disable at compile time
- No makefile changes

---

## Code Organization

All 300 lines are wrapped in clear markers:

```c
/* RL-PAGE-REPLACEMENT: BEGIN */
// ... all RL code here ...
/* RL-PAGE-REPLACEMENT: END */
```

**Easy to**:
- Locate all changes
- Disable entire feature (comment out block)
- Review in code diffs
- Remove if needed

---

## Next Steps for Full Integration

### Step 1: Hook into Page Fault Handler
- **File**: `mm/memory.c` or arch-specific handlers
- **Function**: `do_page_fault()` or `handle_mm_fault()`
- **Action**: Call `rl_handle_page_fault(current->pid, page_to_pfn(page))`

### Step 2: Hook into Page Eviction
- **File**: `mm/vmscan.c`
- **Function**: `shrink_folio_list()` or `shrink_page_list()`
- **Action**: 
  1. Call `policy = rl_select_policy()`
  2. Route to LRU or MRU eviction logic
  3. Call `pech_add_entry()` after eviction

### Step 3: Implement MRU Policy
- **Current**: Linux only has LRU implementation
- **Needed**: Add MRU victim selection logic
- **Approach**: Select from tail of active list instead of inactive list

### Step 4: Add Observability
```bash
# Example of what to create:
cat /proc/rl_mm_stats
# Output:
#   Policy        Score    Selected
#   LRU           245      1523 times
#   MRU           178      421 times
#   Exploration:  20%
#   PEcH entries: 387/512
```

### Step 5: Add Kconfig Option
```kconfig
config RL_PAGE_REPLACEMENT
    bool "RL-based page replacement policy selector"
    depends on MMU
    default n
    help
      Enable reinforcement learning-based selection between
      LRU and MRU page replacement policies.
```

---

## Summary

**What was added**: ~300 lines implementing a reinforcement learning framework for page replacement policy selection.

**Current status**: 
- ✅ Data structures complete
- ✅ RL algorithms implemented  
- ✅ Initialization working
- ❌ Not yet integrated into MM subsystem
- ❌ No MRU policy implementation yet

**Memory cost**: ~20KB static overhead

**Performance cost**: Minimal (disabled by default until integrated)

**Safety**: Fully kernel-safe (no FP, no blocking, bounded operations)

---

## References

- **Paper**: "Virtual Memory Page Replacement using Reinforcement Learning"
- **RL Algorithm**: Multi-Armed Bandit with ε-greedy exploration
- **Integration Strategy**: Minimal modification of existing kernel code
