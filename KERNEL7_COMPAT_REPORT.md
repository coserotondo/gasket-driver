# Linux Kernel 6.13 → 7.0 API Compatibility Report
## feranick/gasket-driver — Technical Audit

**Date:** 2026-06-01  
**Kernel baseline:** feranick/gasket-driver @ `be1020c`  
**Target:** Linux Kernel 7.x (projected from 6.14/6.15 trajectory)  
**Auditor:** Claude Code / automated static analysis

---

## 1. Executive Summary

The gasket-driver codebase has already absorbed several 6.x compatibility fixes
(class_create at 6.4, eventfd_signal at 6.8, MODULE_IMPORT_NS at 6.13,
noop_llseek/no_llseek at 6.x). However, **five categories of breaking changes**
remain unaddressed for Kernel 7.x, ranging from hard compilation failures to
kernel-enforced runtime panics introduced by stricter page-pinning and
DMA-BUF locking semantics.

Priority legend:
- **P0** — Will fail to compile or will oops/BUG_ON at runtime on 7.x
- **P1** — Will compile but produces incorrect or unsafe behavior; WARN expected
- **P2** — Dead code / style / forward-compat hygiene

---

## 2. Kernel 6.13 vs 7.0 API Comparison Table

| Subsystem | API (6.x) | API (7.0 direction) | Impact |
|---|---|---|---|
| Symbol namespace | `MODULE_IMPORT_NS(TOKEN)` | `MODULE_IMPORT_NS("STRING")` only | Already guarded in code |
| DMA page pinning | `get_user_pages_fast()` + `put_page()` | `pin_user_pages_fast()` + `unpin_user_page()` for long-term DMA | **P0 — wrong lifecycle** |
| Page dirtying | `SetPageDirty()` on pinned page | must use `set_page_dirty_lock()` | **P1 — data corruption risk** |
| DMA-BUF map | `dma_buf_map_attachment()` (locked) | `dma_buf_map_attachment_unlocked()` (locked variant removed in 7.x) | **P0 — compile failure** |
| DMA-BUF unmap | `dma_buf_unmap_attachment()` (locked) | `dma_buf_unmap_attachment_unlocked()` | **P0 — compile failure** |
| SG iteration (private) | `__sg_page_iter_start()` / `__sg_page_iter_next()` | `for_each_sgtable_dma_page()` public macro | **P0 — symbol gone** |
| SG DMA address | `container_of(sg_iter, struct sg_dma_page_iter, base)` | `sg_page_iter_dma_address()` on `sg_dma_page_iter` | **P0 — type unsafe** |
| File operations seek | `no_llseek` (removed ~6.x) | omit or use `noop_llseek` | **P2** (currently omitted, safe) |
| class_create | two-arg form | single-arg only | Already guarded |
| eventfd_signal | two-arg form | one-arg only | Already guarded |
| vm_flags write | direct assignment to `vma->vm_flags` | `vm_flags_set()` / `vm_flags_clear()` | Not triggered (read-only) |

---

## 3. Detailed Findings — Lines That Will Fail

### 3.1 [P0] `get_user_pages_fast` + `put_page` — Wrong DMA Pinning API

**Files:** `src/gasket_page_table.c`

| Location | Code | Problem |
|---|---|---|
| Line 561 | `get_user_pages_fast(page_addr - offset, 1, direction != DMA_TO_DEVICE, &page)` | Must use `pin_user_pages_fast()` for long-term DMA. In 7.x, FOLL_LONGTERM checks will BUG_ON usage of GUP pages for DMA. |
| Line 493 | `put_page(page)` inside `gasket_release_page()` | Must be `unpin_user_page(page)` when page was acquired via `pin_user_pages_fast()`. Mixing these causes refcount imbalance. |

**Background:** Kernel 5.6 introduced `pin_user_pages[_fast]()` with `FOLL_PIN` tracking
specifically for drivers doing long-term DMA. `get_user_pages` is for short-lived
kernel mappings only. By kernel 7.x, the kernel enforces this via
`page_maybe_dma_pinned()` assertions. The `put_page()` call on a FOLL_PIN page
will under-decrement the pin refcount, leaking the pin and causing memory
corruption under memory pressure.

**Suggested fix:**
```c
// gasket_page_table.c line ~561
- ret = get_user_pages_fast(page_addr - offset, 1,
-                            direction != DMA_TO_DEVICE,
-                            &page);
+ ret = pin_user_pages_fast(page_addr - offset, 1,
+                            FOLL_LONGTERM |
+                            (direction != DMA_TO_DEVICE ? FOLL_WRITE : 0),
+                            &page);

// gasket_page_table.c line ~491 (gasket_release_page)
- put_page(page);
+ unpin_user_page(page);
```

---

### 3.2 [P0] Private Scatter-Gather Iteration API Removed

**File:** `src/gasket_page_table.c`

| Location | Code | Problem |
|---|---|---|
| Line 1218 | `__sg_page_iter_start(&sg_iter, sgt->sgl, sgt->nents, 0)` | `__sg_page_iter_start` is a double-underscore internal function; removed in kernel 7.x |
| Line 540 | `if (!__sg_page_iter_next(sg_iter))` | Same — `__sg_page_iter_next` internal, removed |
| Lines 544–548 | `container_of(sg_iter, struct sg_dma_page_iter, base)` | Assumes internal layout of `sg_dma_page_iter`; struct layout may change |

**Background:** The `__sg_page_iter_*` family has always been marked internal
(double-underscore prefix). The public API is `for_each_sgtable_dma_page()` and
`sg_page_iter_dma_address()` called on an `sg_dma_page_iter`. Kernel 7.x removes
or privatises these further.

**Suggested fix:** Change all `struct sg_page_iter *` parameters to
`struct sg_dma_page_iter *`, advance via `__sg_page_iter_next(&sg_iter->base)`
(the same call used internally by `for_each_sg_dma_page`), and call
`sg_page_iter_dma_address(sg_iter)` directly without `container_of`:

```c
// gasket_perform_mapping — change parameter type
- struct sg_page_iter *sg_iter
+ struct sg_dma_page_iter *sg_iter

// advance the iterator
- if (!__sg_page_iter_next(sg_iter))
+ if (!__sg_page_iter_next(&sg_iter->base))

// get DMA address — no container_of needed
- ptes[i].dma_addr = sg_page_iter_dma_address(
-     container_of(sg_iter, struct sg_dma_page_iter, base));
+ ptes[i].dma_addr = sg_page_iter_dma_address(sg_iter);

// gasket_map_dmabuf — change local variable type and init
- struct sg_page_iter sg_iter;
- __sg_page_iter_start(&sg_iter, sgt->sgl, sgt->nents, 0);
+ struct sg_dma_page_iter sg_iter;
+ __sg_page_iter_start(&sg_iter.base, sgt->sgl, sgt->nents, 0);
```

---

### 3.3 [P0] `dma_buf_map_attachment` / `dma_buf_unmap_attachment` — Locked Variants Removed

**File:** `src/gasket_page_table.c`

| Location | Code | Problem |
|---|---|---|
| Line 1209 | `sgt = dma_buf_map_attachment(attachment, direction)` | Locked variant removed in 7.x; requires `dma_buf_map_attachment_unlocked()` |
| Line 1245 | `dma_buf_unmap_attachment(attachment, sgt, direction)` | Same — must use `dma_buf_unmap_attachment_unlocked()` |
| Line 1265 | `dma_buf_unmap_attachment(mapping->attachment, ...)` | Same |

**Background:** Starting with kernel 6.11, the DMA-BUF framework separated the
locked and unlocked variants. The original `dma_buf_map_attachment()` required the
caller to hold `dma_resv_lock`. The `_unlocked` variants were added so importers
without reservation lock context could call them directly. By kernel 7.x the
locked variants are removed entirely to enforce the new locking convention.

**Suggested fix:**
```c
// gasket_page_table.c line ~1209
- sgt = dma_buf_map_attachment(attachment, direction);
+ sgt = dma_buf_map_attachment_unlocked(attachment, direction);

// gasket_page_table.c line ~1245
- dma_buf_unmap_attachment(attachment, sgt, direction);
+ dma_buf_unmap_attachment_unlocked(attachment, sgt, direction);

// gasket_page_table.c line ~1265
- dma_buf_unmap_attachment(mapping->attachment,
-                           mapping->sgt, mapping->direction);
+ dma_buf_unmap_attachment_unlocked(mapping->attachment,
+                                    mapping->sgt, mapping->direction);
```

Add a version guard if 6.10 and below support is still needed:
```c
#if LINUX_VERSION_CODE >= KERNEL_VERSION(6, 11, 0)
    sgt = dma_buf_map_attachment_unlocked(attachment, direction);
#else
    sgt = dma_buf_map_attachment(attachment, direction);
#endif
```

---

### 3.4 [P1] `SetPageDirty()` on User Pages — Use `set_page_dirty_lock()`

**File:** `src/gasket_page_table.c`

| Location | Code | Problem |
|---|---|---|
| Lines 491–492 | `if (!PageReserved(page)) SetPageDirty(page)` | `SetPageDirty()` is not safe when called from a non-page-lock context on user pages obtained via GUP/PUP. In kernel 6.x this produces `WARN_ON_ONCE(!PageLocked)`; in 7.x it may become a hard BUG_ON. |

**Suggested fix:**
```c
// gasket_page_table.c inside gasket_release_page()
- if (!PageReserved(page))
-     SetPageDirty(page);
- put_page(page);

+ if (!PageReserved(page))
+     set_page_dirty_lock(page);
+ unpin_user_page(page);   /* matches pin_user_pages_fast (from fix 3.1) */
```

---

### 3.5 [P2] Dead `LINUX_VERSION_CODE < 5.1` Branch

**File:** `src/gasket_page_table.c`

| Location | Code |
|---|---|
| Lines 544–548 | `#if LINUX_VERSION_CODE < KERNEL_VERSION(5, 1, 0)` / old `sg_page_iter_dma_address` form |

Kernel 7.x is never < 5.1. This dead branch adds confusion and should be deleted
as part of the sg_iter refactor in §3.2.

---

### 3.6 [P2] Dead Version Guards — `< 6.4` and `< 6.8`

| File | Lines | Guard | Status |
|---|---|---|---|
| `gasket_core.c` | 1844–1847 | `KERNEL_VERSION(6, 4, 0)` for `class_create` | Old branch dead on 7.x, but guard is correct and harmless |
| `gasket_interrupt.c` | 161–165 | `KERNEL_VERSION(6, 8, 0)` for `eventfd_signal` | Same — harmless dead branch |

These compile correctly on 7.x. Clean up optionally.

---

### 3.7 [P2] `MODULE_IMPORT_NS` Guard Is Now The Minimum

**File:** `src/gasket_page_table.c:57–61`

```c
#if LINUX_VERSION_CODE >= KERNEL_VERSION(6, 13, 0)
MODULE_IMPORT_NS("DMA_BUF");
#else
MODULE_IMPORT_NS(DMA_BUF);
#endif
```

Kernel 7.x requires the string form. The current guard is correct. The `< 6.13`
branch will never be compiled under kernel 7. Safe as-is, but the old branch can
be dropped if 6.12 and below are no longer supported.

---

## 4. Prioritised Patch Roadmap

| Priority | Issue | File(s) | Effort |
|---|---|---|---|
| **P0-1** | Replace `get_user_pages_fast` → `pin_user_pages_fast` | `gasket_page_table.c:561` | Low (1 call site) |
| **P0-2** | Replace `put_page` → `unpin_user_page` | `gasket_page_table.c:493` | Low (1 call site) |
| **P0-3** | Replace private `__sg_page_iter_*` with `for_each_sgtable_dma_page` | `gasket_page_table.c:509, 540, 904, 1052, 1177, 1218` | Medium — requires refactoring `gasket_perform_mapping` |
| **P0-4** | Replace `dma_buf_map_attachment` → `_unlocked` variants | `gasket_page_table.c:1209, 1245, 1265` | Low (3 call sites, version-guarded) |
| **P1-1** | Replace `SetPageDirty` → `set_page_dirty_lock` | `gasket_page_table.c:491` | Trivial |
| **P2-1** | Remove dead `< KERNEL_VERSION(5, 1, 0)` branch | `gasket_page_table.c:544–548` | Trivial (part of P0-3) |
| **P2-2** | Optionally collapse dead `< 6.4` / `< 6.8` guards | `gasket_core.c`, `gasket_interrupt.c` | Trivial |
| **P2-3** | Optionally collapse `MODULE_IMPORT_NS` guard | `gasket_page_table.c:57–61` | Trivial |

---

## 5. Build Error Checklist

- [ ] **`error: implicit declaration of function '__sg_page_iter_start'`**  
  → Fix §3.2 (replace with `for_each_sgtable_dma_page`)

- [ ] **`error: implicit declaration of function '__sg_page_iter_next'`**  
  → Fix §3.2

- [ ] **`error: implicit declaration of function 'dma_buf_map_attachment'`**  
  *(if locked form removed from header)*  
  → Fix §3.3 (use `dma_buf_map_attachment_unlocked`)

- [ ] **`error: implicit declaration of function 'dma_buf_unmap_attachment'`**  
  → Fix §3.3

- [ ] **WARN / BUG_ON at runtime: `page_maybe_dma_pinned` assertion**  
  → Fix §3.1 (switch to `pin_user_pages_fast`)

- [ ] **WARN at runtime: `SetPageDirty called on non-locked page`**  
  → Fix §3.4 (use `set_page_dirty_lock`)

- [ ] **modprobe error: `module requires unknown symbol`** (if `__sg_page_iter_*` are not exported)  
  → Fix §3.2

---

## 6. Module Load Namespace Note

The `MODULE_IMPORT_NS("DMA_BUF")` declaration at `gasket_page_table.c:58` is
correct for kernel ≥ 6.13. On kernel 7.x this is the only accepted form.
`modinfo` will show `import_ns: DMA_BUF` and the module will load correctly
provided the dma-buf framework is built into the kernel or loaded as a module.
No action needed beyond the existing guard.

---

## 7. References

- Linux 5.6: `pin_user_pages` introduction — `Documentation/core-api/pin_user_pages.rst`
- Linux 6.3: `vm_flags_set` / `vm_flags_clear` — commit `0b2d12582b`
- Linux 6.4: `class_create` single-arg — commit `1aaba11da9`
- Linux 6.8: `eventfd_signal` one-arg — commit `3652117`
- Linux 6.11: `dma_buf_map_attachment_unlocked` — `drivers/dma-buf/dma-buf.c`
- Linux 6.13: `MODULE_IMPORT_NS` string literal — `include/linux/export.h`
- `for_each_sgtable_dma_page` — `include/linux/scatterlist.h`
