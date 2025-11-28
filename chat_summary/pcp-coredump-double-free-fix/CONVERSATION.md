# Conversation Summary: PCP Coredump Double-Free Analysis and Fix

**Date:** November 28, 2025

**Issue:** PCP (Performance Co-Pilot) coredump with "double free or corruption" error

---

## Initial Problem Report

User provided a coredump stack trace showing a crash in `pmFreeResult()` called from `cache_read()` in `interp.c`:

```
#3  malloc_printerr (str="double free or corruption (out)")
#5  __pmFreeResultValueSets at freeresult.c:53
#6  pmFreeResult at freeresult.c:72
#7  cache_read at interp.c:216
#8  __pmLogFetchInterp at interp.c:1347
```

---

## Analysis Performed

### Files Investigated:
- `src/libpcp/src/interp.c` - Main interpolation logic with cache mechanism
- `src/libpcp/src/freeresult.c` - Result freeing with PDU buffer management
- `src/libpcp/src/pdubuf.c` - PDU buffer pin/unpin reference counting

### Root Cause Identified:

In `update_bounds()` function in `interp.c`, during value shuffle operations:

1. **"shuffle prior to next" (line ~730):**
   - Code unpins old `v_next.pval`
   - Copies `v_prior.pval` to `v_next.pval`
   - **BUG:** Does not pin the new `v_next.pval`

2. **"shuffle next to prior" (line ~770):**
   - Code unpins old `v_prior.pval`
   - Copies `v_next.pval` to `v_prior.pval`
   - **BUG:** Does not pin the new `v_prior.pval`

This causes PDU buffer reference count to become incorrect, leading to premature freeing and double-free.

---

## Q&A

### Q1: Does the upstream community have a patch for this?
**A:** No. Checked `performancecopilot/pcp` official repository - the same bug exists in the upstream code. The shuffle operations in `update_bounds()` (around lines 730 and 770) also miss the `__pmPinPDUBuf()` calls after copying pointers.

### Q2: How to quickly construct and verify this problem?
**A:** Use existing QA tests:

```bash
# Build PCP
./configure && make

# Run related tests
cd qa
./check 412    # Test interp wrapping
./check 962    # Test interp mode with <mark> records
./check 508    # pmlogreduce workout (includes interp.c bug check)
```

To trigger the bug, need frequent "shuffle prior to next" or "shuffle next to prior" operations, typically:
- Using `pmval -a` for time series interpolation on archives
- Reading across multiple volume/archive boundaries
- Processing archives containing `<mark>` records

---

## Fix Applied

Added `__pmPinPDUBuf()` calls after copying pointers in both shuffle operations:

```c
// Location 1: shuffle prior to next
if (icp->v_next.pval != NULL)
    __pmUnpinPDUBuf((void *)icp->v_next.pval);
icp->v_next.pval = icp->v_prior.pval;
if (icp->v_next.pval != NULL)                    // ADDED
    __pmPinPDUBuf((void *)icp->v_next.pval);     // ADDED

// Location 2: shuffle next to prior  
if (icp->v_prior.pval != NULL)
    __pmUnpinPDUBuf((void *)icp->v_prior.pval);
icp->v_prior.pval = icp->v_next.pval;
if (icp->v_prior.pval != NULL)                   // ADDED
    __pmPinPDUBuf((void *)icp->v_prior.pval);    // ADDED
```

---

## Files in This Directory

- `README.md` - Detailed analysis and documentation
- `0001-fix-double-free-pdu-buffer-pin.patch` - The patch file
- `CONVERSATION.md` - This conversation summary
