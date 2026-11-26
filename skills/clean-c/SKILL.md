---
name: clean-c
description: >-
  Clean Code principles tailored to modern C (C23), TigerStyle, and NASA Power of 10
  systems programming. Use when writing, reviewing, or refactoring C/H files to eliminate
  undefined behavior, oversized functions, duplicated truth, and structural slop.
---

# Clean Code in C (C23)

## P0 — Safety & Undefined Behavior (Blockers)
- **Strict aliasing & alignment:** Use `memcpy` for type punning and unaligned reads; never cast `const uint8_t *` to `const uint32_t *` and dereference.
- **Integer & arithmetic safety:** Prefer fixed-width types (`uint32_t`, `int64_t`, `size_t`, `ptrdiff_t`) over `int`/`long`. Use C23 `<stdckdint.h>` (`ckd_add`, `ckd_mul`) before sizing or allocating buffers. Use unsigned shift operands (`1u << 31`, never `1 << 31`).
- **Explicit slices over raw NUL strings:** Pass `(const uint8_t *data, size_t len)` or a slice struct (`struct byte_slice { const uint8_t *ptr; size_t len; }`). Ban `strcpy`, `strcat`, `sprintf`, `gets`, and `atoi`.
- **Bounded loops & no recursion:** Every loop walking external data, ring buffers, or linked lists must have an explicit upper bound (`depth < MAX_DEPTH`) and guaranteed forward progress. Replace recursion with a fixed-capacity stack.
- **Contracts vs. untrusted input:** Use `assert()` for internal programmer invariants; use explicit runtime checks and error returns for external bytes, sizes, and user input. Never ignore fallible return values (`[[nodiscard]]`).

## P1 — Architecture & Anti-Slop
- **Module & function budgets:** Keep files `< 300` lines (one cohesive responsibility) and functions `< 60` lines (one level of abstraction). *Escape hatch:* do not artificially shred flat `static const` lookup tables or pure enum-to-value `switch` tables.
- **Single source of truth:** Put shared limits, flags, and mappings in one `static const` table or `static constexpr` constant. Deduplicate twin functions via a small `_common` helper; ban `bool` flag arguments that fork behavior.
- **Ownership & cleanup without RAII:** Prefer caller-owned buffers, static/linear arenas, and value-returning pure builders (`{ 0 }` safe defaults; zero output buffers to prevent buffer bleeds). When a function acquires multiple fallible resources, use a single `goto cleanup;` label (or separate init/deinit helpers if `goto` is forbidden by project policy) — never duplicate teardown across 5 early returns.
- **Closed vs. open enum `switch`:** Switch directly on *closed* internal enums with no `default:` (so `-Wswitch-enum` catches missing variants at compile time). Cast `switch ((uint32_t)code)` only for *open/external* protocol enums.
- **Zero mutable globals:** Pass explicit context/arena pointers instead of reading hidden file-scope globals. Mark non-mutated parameters, pointers, locals, and tables `const`; treat `const`-casts as bugs.

## P2 — Naming, Layout, Comments & Macros
- **Meaningful names with units last:** `timeout_ms`, `offset_bytes`, `stride_u32`. Distinguish `index` (0-based), `count` (elements), and `size_bytes` (bytes). Ban numbered temps (`tmp1`, `buf2`) and noise suffixes (`_data`, `_obj`, `helper`).
- **Vertical layout:** Place leaf `static` helpers at the top and the public API at the bottom to avoid forward-declaration noise. Declare locals at first use.
- **Comments explain *why*, never *what*:** Document invariants and spec quirks; extract 30-line "what" blocks into named functions. Delete `#if 0` and dead code.
- **Macro hygiene:** Prefer `static inline`, `constexpr`, and `_Generic`. When a macro is unavoidable, wrap multi-statement bodies in `do { ... } while (0)` and parenthesize every parameter and expression.

## Before / After C23 Patterns

### 1. Aliasing-safe unaligned read + checked sizing
```c
// Slop: strict-aliasing UB + unaligned deref + unchecked multiply
uint32_t count = *(const uint32_t *)packet_bytes;
uint8_t *out = arena_alloc(arena, count * sizeof(struct record));

// Clean: memcpy + C23 <stdckdint.h>
uint32_t count = 0u;
memcpy(&count, packet_bytes, sizeof(count));
size_t total_bytes = 0u;
if (ckd_mul(&total_bytes, count, sizeof(struct record))) {
  return nullptr;
}
uint8_t *const out = arena_alloc(arena, total_bytes);
```

### 2. Bounded list traversal + guard clauses
```c
// Slop: arrow nesting + unbounded pointer chase
if (head != NULL) {
  struct node *cur = head;
  while (cur != NULL) {
    if (cur->kind == KIND_TARGET) { ... }
    cur = cur->next;
  }
}

// Clean: flat guard + Power-of-10 bound
static constexpr uint32_t MAX_CHAIN_DEPTH = 256u;

static void visit_nodes(const struct node *head) {
  const struct node *cur = head;
  for (uint32_t depth = 0u; cur != nullptr && depth < MAX_CHAIN_DEPTH; depth++) {
    if (cur->kind == KIND_TARGET) {
      handle_target(cur);
    }
    cur = cur->next;
  }
}
```

### 3. Value-returning struct normalization (`{ 0 }` defaults)
```c
// Slop: memset + 30 lines of field mutation in caller
struct net_cfg cfg;
memset(&cfg, 0, sizeof(cfg));
cfg.timeout_ms = raw ? raw->timeout_ms : 1000u;

// Clean: pure builder returning normalized value
static struct net_cfg normalize_net_cfg(const struct net_cfg *raw) {
  if (raw == nullptr) {
    return (struct net_cfg){ .timeout_ms = 1000u };
  }
  return (struct net_cfg){
    .timeout_ms = raw->timeout_ms > 0u ? raw->timeout_ms : 1000u,
    .retries = raw->retries,
  };
}
```

### 4. Closed enum `switch` vs. open external enum
```c
// Closed internal enum: let -Wswitch-enum catch missing variants
switch (codec_state) {
  case CODEC_IDLE:  return "idle";
  case CODEC_READY: return "ready";
}

// Open external protocol enum: cast + explicit default
switch ((uint32_t)wire_tag) {
  case WIRE_TAG_PING: return handle_ping(msg);
  default:            return ERR_UNSUPPORTED;
}
```

## Agent Modes & Verification
- **Audit mode:** Report findings as `file:line | P0/P1/P2 | smell | fix`, sorted P0 first.
- **Refactor mode:** Keep structural refactors in separate commits from behavioral changes.
- **Verify:** Compile with `clang -std=c23 -Wall -Wextra -Werror -Wpedantic -Wconversion -Wshadow -Wcast-qual -Wswitch-enum`, run `clang-tidy`, and run tests under `-fsanitize=address,undefined`.
