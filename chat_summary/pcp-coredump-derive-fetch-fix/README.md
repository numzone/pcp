# PCP Coredump 分析与修复：derive_fetch.c 崩溃问题

## 问题描述

用户遇到 PCP coredump 问题，崩溃堆栈如下：

```
#0  __pthread_kill_implementation (threadid=281473766084640, signo=signo@entry=6, no_tid=no_tid@entry=0) at pthread_kill.c:44
#1  0x0000ffffb75e1b84 in __pthread_kill_internal (signo=6, threadid=<optimized out>) at pthread_kill.c:78
#2  0x0000ffffb7599f2c in __GI_raise (sig=sig@entry=6) at ../sysdeps/posix/raise.c:26
#3  0x0000ffffb758703c in __GI_abort () at abort.c:79
#4  0x0000ffffb75d5c68 in __libc_message (fmt=fmt@entry=0xffffb76b1eb8 "%s\n") at ../sysdeps/posix/libc_fatal.c:150
#5  0x0000ffffb75ebb58 in malloc_printerr (str=str@entry=0xffffb76aef10 "munmap_chunk(): invalid pointer") at malloc.c:5765
#6  0x0000ffffb75ebe20 in munmap_chunk (p=<optimized out>) at malloc.c:3035
#7  0x0000ffffb75efdfc in __GI___libc_free (mem=<optimized out>) at malloc.c:3381
#8  0x0000ffffa9671144 in __pmFreeResultValueSets (ppvsend=<optimized out>, ppvstart=<optimized out>) at result.c:186
#9  __pmFreeResultValueSets (ppvstart=0xaaab04a7e678, ppvsend=<optimized out>) at result.c:161
#10 0x0000ffffa9671410 in __pmFreeResult (result=result@entry=0xaaab04a7e660) at result.c:225
#11 0x0000ffffa96b1308 in __dmpostfetch (ctxp=ctxp@entry=0xaaab049b54b0, result=result@entry=0xffffc5cd7840) at derive_fetch.c:2238
#12 0x0000ffffa966d9a0 in __pmFinishResult (ctxp=ctxp@entry=0xaaab049b54b0, count=count@entry=0, resultp=resultp@entry=0xffffc5cd7840) at fetch.c:87
#13 0x0000ffffa966ddb8 in __pmFetch (ctxp=<optimized out>, ctxp@entry=0x0, numpmid=<optimized out>, pmidlist=<optimized out>, result=result@entry=0xffffc5cd7840) at fetch.c:231
#14 0x0000ffffa966e1bc in pmFetch_ctx (result=0xffffc5cd7840, pmidlist=<optimized out>, numpmid=<optimized out>, ctxp=0x0) at fetch.c:262
#15 pmFetchHighRes (numpmid=<optimized out>, pmidlist=<optimized out>, result=result@entry=0xffffc5cd78b0) at fetch.c:289
#16 0x0000ffffa9670494 in pmFetchGroup (pmfg=0xaaab049bd3d0) at fetchgroup.c:1706
```

崩溃发生在 `__pmFreeResultValueSets` 函数中，错误信息为 "munmap_chunk(): invalid pointer"。

## 根因分析

通过分析代码，发现 `src/libpcp/src/derive_fetch.c` 中存在两个 bug：

### Bug 1: N_QUEST (三元运算符) 处理 PM_TYPE_STRING 时未复制 vlen

在 `eval_expr()` 函数中，处理 N_QUEST (三元运算符 `?:`) 的 PM_TYPE_STRING 类型时，只复制了 `value.cp` 指针，但没有复制 `vlen` 字段。这导致 `vlen` 保持未初始化状态（垃圾值），后续在 `__dmpostvalueset()` 中使用该垃圾值作为 malloc 大小，导致内存损坏和崩溃。

**问题代码位置**: `derive_fetch.c` 约第 1280 行

```c
case PM_TYPE_STRING:
    if (i < pick->data.info->numval)
        np->data.info->ivlist[i].value.cp = pick->data.info->ivlist[i].value.cp;
    else
        np->data.info->ivlist[i].value.cp = pick->data.info->ivlist[0].value.cp;
    break;
```

### Bug 2: PM_TYPE_DOUBLE 的 memcpy 使用了错误的源字段

在 `__dmpostvalueset()` 函数中，处理 PM_TYPE_DOUBLE 类型时，memcpy 从 `value.f`（float 类型）复制而不是从 `value.d`（double 类型）复制。

**问题代码位置**: `derive_fetch.c` 约第 2168 行

```c
memcpy((void *)vp->vbuf, (void *)&cp->mlist[m].expr->data.info->ivlist[i].value.f, sizeof(double));
```

## 修复方案

### 修复 Bug 1

在 PM_TYPE_STRING 情况下，同时复制 `value.cp` 和 `vlen`：

```c
case PM_TYPE_STRING:
    if (i < pick->data.info->numval) {
        np->data.info->ivlist[i].value.cp = pick->data.info->ivlist[i].value.cp;
        np->data.info->ivlist[i].vlen = pick->data.info->ivlist[i].vlen;
    }
    else {
        np->data.info->ivlist[i].value.cp = pick->data.info->ivlist[0].value.cp;
        np->data.info->ivlist[i].vlen = pick->data.info->ivlist[0].vlen;
    }
    break;
```

### 修复 Bug 2

将 `value.f` 改为 `value.d`：

```c
memcpy((void *)vp->vbuf, (void *)&cp->mlist[m].expr->data.info->ivlist[i].value.d, sizeof(double));
```

## 验证方法

1. **验证 N_QUEST STRING vlen bug**：创建一个使用三元运算符 (`?:`) 返回字符串类型的派生指标，然后使用 `pmFetchGroup` 多次获取值。如果崩溃带有 "munmap_chunk(): invalid pointer" 错误，则说明问题存在。

2. **验证修复**：应用本补丁后重新编译 libpcp，再次运行相同测试用例，应该不再崩溃。

3. **简单测试示例**：
   ```bash
   # 创建派生指标配置文件
   echo 'test.string = kernel.uname.nodename > 0 ? kernel.uname.nodename : "default"' > /tmp/derived.conf
   
   # 使用 pmval 测试 (需要应用补丁后编译的libpcp)
   PCP_DERIVED_CONFIG=/tmp/derived.conf pmval -s 5 test.string
   ```

## 上游状态

经检查，这两个 bug 在 performancecopilot/pcp 社区最新代码中**仍然存在**，尚未有相关补丁。建议将此修复提交到上游社区。

## 文件列表

- `fix.patch` - 修复补丁文件
- `README.md` - 本文档

## 相关信息

- **修复提交**: 0b3eab9
- **受影响文件**: `src/libpcp/src/derive_fetch.c`
- **平台**: aarch64 (从崩溃堆栈地址格式判断)
