# PCP Coredump Double-Free Fix Analysis

## Issue Description

A coredump was reported with the following stack trace showing a "double free or corruption" error:

```
#0  __GI_raise (sig=sig@entry=6) at ../sysdeps/unix/sysv/linux/raise.c:51
#1  0x00007f4b0a342101 in __GI_abort () at abort.c:79
#2  0x00007f4b0a3824ba in __libc_message (action=action@entry=do_abort, fmt=fmt@entry=0x7f4b0a48925e "%s\n") at ../sysdeps/posix/libc_fatal.c:182
#3  0x00007f4b0a388c1a in malloc_printerr (str=str@entry=0x7f4b0a48a760 "double free or corruption (out)") at malloc.c:5390
#4  0x00007f4b0a38a6f0 in _int_free (av=0x7f4b0a6b7aa0 <main_arena>, p=0x55bb0e14d9e0, have_lock=<optimized out>) at malloc.c:4314
#5  0x00007f4afc309aab in __pmFreeResultValueSets (ppvstart=<optimized out>, ppvsend=<optimized out>) at freeresult.c:53
#6  0x00007f4afc309c2a in pmFreeResult (result=0x55bb11677120) at freeresult.c:72
#7  0x00007f4afc32c3c3 in cache_read (ctxp=ctxp@entry=0x55bafed703b0, mode=mode@entry=2, rp=rp@entry=0x7ffe15187dd8) at interp.c:216
#8  0x00007f4afc32f6cf in __pmLogFetchInterp (ctxp=ctxp@entry=0x55bafed703b0, numpmid=numpmid@entry=10, pmidlist=pmidlist@entry=0x55bb0de7c700, result=result@entry=0x7ffe15188100) at interp.c:1347
#9  0x00007f4afc32a943 in __pmLogFetch (ctxp=ctxp@entry=0x55bafed703b0, numpmid=numpmid@entry=10, pmidlist=pmidlist@entry=0x55bb0de7c700, result=result@entry=0x7ffe15188100) at logutil.c:1913
#10 0x00007f4afc306b1d in pmFetch_ctx (ctxp=0x55bafed703b0, ctxp@entry=0x0, numpmid=10, pmidlist=0x55bb0de7c700, result=result@entry=0x7ffe15188100) at fetch.c:164
#11 0x00007f4afc30723f in pmFetch (numpmid=<optimized out>, pmidlist=<optimized out>, result=result@entry=0x7ffe15188100) at fetch.c:234
#12 0x00007f4afc309012 in pmFetchGroup (pmfg=0x55bafed71950) at fetchgroup.c:1628
...
```

## Root Cause Analysis

The crash occurs at `freeresult.c:53` when calling `pmFreeResult()` from `cache_read()` at `interp.c:216`.

### The Bug

In `update_bounds()` in `interp.c`, when shuffling values between `v_prior` and `v_next`, the code was:

1. Unpinning the old destination pointer
2. Copying the source pointer to destination
3. **Missing: Pinning the new destination pointer**

This caused the PDU buffer reference count to become incorrect:
- Source and destination now pointed to the same buffer (with pin count 1)
- When the source was later unpinned (to set a new value), the buffer's count reached 0 and was freed
- The destination still pointed to the now-freed buffer

When cached results containing these values were later evicted via `pmFreeResult()`:
1. `__pmUnpinPDUBuf()` was called on vsets in the freed buffer
2. The buffer wasn't found in the pool (already freed), returning 0
3. The code then tried to manually `free()` the vsets
4. This caused a double-free crash since the memory was already freed

### Affected Code Locations

Two locations in `src/libpcp/src/interp.c`:

**Location 1: "shuffle prior to next" (around line 730)**
```c
if (icp->v_next.pval != NULL)
    __pmUnpinPDUBuf((void *)icp->v_next.pval);
icp->v_next.pval = icp->v_prior.pval;
// Missing: __pmPinPDUBuf((void *)icp->v_next.pval);
```

**Location 2: "shuffle next to prior" (around line 770)**
```c
if (icp->v_prior.pval != NULL)
    __pmUnpinPDUBuf((void *)icp->v_prior.pval);
icp->v_prior.pval = icp->v_next.pval;
// Missing: __pmPinPDUBuf((void *)icp->v_prior.pval);
```

## The Fix

Added `__pmPinPDUBuf()` calls after copying pointers during the shuffle operations to maintain correct PDU buffer reference counts.

## Upstream Status

Verified that the same bug exists in the upstream `performancecopilot/pcp` repository. This fix is novel and should be contributed to the upstream project.

## Testing

To verify the fix, use existing QA tests:

```bash
# Build PCP
./configure && make

# Run related tests
cd qa
./check 412    # Test interp wrapping
./check 962    # Test interp mode with <mark> records
./check 508    # pmlogreduce workout (includes interp.c bug check)
```

The bug is typically triggered when:
- Using `pmval -a` for time series interpolation on archives
- Reading across multiple volume/archive boundaries
- Processing archives containing `<mark>` records
