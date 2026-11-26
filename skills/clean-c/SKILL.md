---
name: clean-c
description: >-
  Clean Code principles tailored to modern C (C23), TigerStyle, and NASA Power of 10
  systems programming. Use when writing, reviewing, or refactoring C/H files to eliminate
  undefined behavior, oversized functions, duplicated truth, and structural slop.
---

# Clean Code in C (C23)

## P0 — Safety & Undefined Behavior (Blockers)
- **Strict aliasing, alignment & overlap:** Use `memcpy` for type punning and unaligned host-endian reads; never cast `const uint8_t *` to `const uint32_t *` and dereference. Use `memmove` when buffers may overlap. Decode wire integers with explicit endianness (`read_u32_le` / shifts), not host-endian `memcpy`.
- **Integer, shift & pointer arithmetic safety:** Prefer fixed-width types (`uint32_t`, `int64_t`, `size_t`, `ptrdiff_t`) over `int`/`long`. Use C23 `<stdckdint.h>` (`ckd_add`, `ckd_sub`, `ckd_mul`) before sizing, slicing, or allocating. Guard shift counts (`< bitwidth`), use unsigned shift LHS (`1u << 31`, `UINT64_C(1) << n`), and guard divide-by-zero and `INT_MIN / -1`. Check remaining bounds as `n > (size_t)(end - ptr)` (never `ptr + n > end`).
- **Explicit slices & safe strings:** Pass `(const uint8_t *ptr, size_t len)` or `struct byte_slice { const uint8_t *ptr; size_t len; }`. Always guard `len > 0` before `memcpy`/`memmove`/`memcmp`/`memset` (passing `nullptr` with `len == 0` is UB in C23 §7.26.1p2 and trips UBSan). Ban `strcpy`, `strcat`, `strncpy`, `sprintf`, `vsprintf`, `scanf("%s")`, `strtok`, and `atoi`/`atol`/`atof`. Cast `char` to `(unsigned char)` before `<ctype.h>` calls.
- **Bounded loops & no recursion:** Every loop walking external data, ring buffers, or linked lists must have an explicit upper bound (`depth < MAX_CHAIN_DEPTH`) and guaranteed forward progress, and must report overflow or cycles via an error return or invariant check — never silently truncate. Replace recursion with a fixed-capacity stack.
- **Contracts vs. untrusted input:** Use `assert()` (with zero side effects inside) for internal programmer invariants — or an always-on aborting check macro if builds define `NDEBUG`. Use explicit runtime checks and `[[nodiscard]]` error returns for external bytes, sizes, and fallible allocations. Use C23 `nullptr` (never `NULL`/`0`) and `static_assert` for compile-time struct/limit invariants.

## P1 — Architecture & Anti-Slop
- **Module & function budgets:** Keep files `< 300` lines (one cohesive responsibility) and functions `< 60` lines (one level of abstraction). *Escape hatch:* do not artificially shred flat `static const` lookup tables or pure enum-to-value `switch` tables.
- **Single source of truth & TU hygiene:** Put small compile-time constants in headers as `static constexpr`; put shared lookup tables in a `.c` file (`static const`, or `extern const` in `.h` + single definition in `.c`) to avoid duplicating tables across TUs. Deduplicate twin functions via a small `static` helper named for the shared operation; ban `bool` flag arguments that fork behavior.
- **Ownership, `{}` defaults & cleanup without RAII:** Prefer caller-owned buffers, static/linear arenas, and value-returning pure builders using C23 `{}` empty initializers. Note that returning structs by value leaves padding bytes unspecified (C23 §6.2.6.1p6) — explicitly `memset` output buffers or serialize field-by-field at IPC/wire/security boundaries. For multi-resource cleanup, use a single `goto cleanup;` label (explicit pragmatic exception to Power-of-10 Rule 1) or split init/deinit helpers if `goto` is forbidden by project policy.
- **Closed vs. open enum `switch`:** For *closed* internal enums, switch directly on the enum with no `default:` and place an `assert(false); return ...;` fallback right after the `switch` — this satisfies `-Wswitch`, `-Wswitch-enum`, GCC `-Wreturn-type`, and avoids UB on out-of-range integer values. For *open/external* protocol enums, define a C23 fixed underlying type (`enum wire_tag : uint32_t`) and cast `switch ((uint32_t)tag)` with an explicit `default:`.
- **Zero mutable globals:** Pass explicit context/arena pointers instead of reading hidden file-scope globals. Mark non-mutated parameters, pointers, locals, and tables `const`; treat `const`-casts as bugs.

## P2 — Naming, Layout, Comments & Macros
- **Meaningful names with units last:** `timeout_ms`, `offset_bytes`, `stride_u32`. Distinguish `index` (0-based), `count` (elements), and `size_bytes` (bytes). Ban numbered temps (`tmp1`, `buf2`) and noise suffixes (`_data`, `_obj`, `helper`, `_common`).
- **Vertical layout:** Place leaf `static` helpers at the top and the public API at the bottom to avoid forward-declaration noise. Declare locals at first use.
- **Comments explain *why*, never *what*:** Document invariants and spec quirks; extract 30-line "what" blocks into named functions. Delete `#if 0` and dead code.
- **Macro hygiene:** Prefer `static inline`, `constexpr`, and `_Generic` (eliminating double evaluation like `MAX(i++, j++)`). When a macro is unavoidable, wrap multi-statement bodies in `do { ... } while (0)`, parenthesize every parameter and expression, and never evaluate arguments more than once.

## Before / After C23 Patterns

### 1. Slice bounds + explicit LE decode + checked sizing
```c
// Slop: strict-aliasing UB, no bounds check, unchecked multiply, unchecked alloc
uint32_t count = *(const uint32_t *)packet_bytes;
uint8_t *out = arena_alloc(arena, count * sizeof(struct record));

// Clean: explicit slice bounds, endian-safe decode, <stdckdint.h>, checked alloc
static struct record *alloc_records(
    struct arena *arena, const uint8_t *packet_bytes, size_t packet_len) {
  if (arena == nullptr || packet_bytes == nullptr || packet_len < sizeof(uint32_t)) {
    return nullptr;
  }
  const uint32_t count = ((uint32_t)packet_bytes[0]) | ((uint32_t)packet_bytes[1] << 8) |
                         ((uint32_t)packet_bytes[2] << 16) | ((uint32_t)packet_bytes[3] << 24);
  size_t total_bytes = 0u;
  if (count == 0u || ckd_mul(&total_bytes, count, sizeof(struct record))) {
    return nullptr;
  }
  return arena_alloc(arena, total_bytes);
}
```

### 2. Bounded list traversal with cycle/overflow reporting
```c
// Slop: unbounded pointer chase that hangs on cycles
while (cur != NULL) {
  if (cur->kind == KIND_TARGET) { handle_target(cur); }
  cur = cur->next;
}

// Clean: Power-of-10 bound + explicit [[nodiscard]] cycle/overflow status
static constexpr uint32_t MAX_CHAIN_DEPTH = 256u;

[[nodiscard]] static bool visit_nodes(const struct node *head) {
  const struct node *cur = head;
  for (uint32_t depth = 0u; cur != nullptr && depth < MAX_CHAIN_DEPTH; depth++) {
    if (cur->kind == KIND_TARGET) {
      handle_target(cur);
    }
    cur = cur->next;
  }
  return cur == nullptr;
}
```

### 3. Value-returning struct normalization (`{}` defaults)
```c
// Slop: duplicated magic constant + multi-branch literal returns
static struct net_cfg normalize_net_cfg(const struct net_cfg *raw) {
  if (raw == NULL) { return (struct net_cfg){ .timeout_ms = 1000u }; }
  return (struct net_cfg){ .timeout_ms = raw->timeout_ms ? raw->timeout_ms : 1000u };
}

// Clean: single constant + C23 {} zero-init + single return
static constexpr uint32_t DEFAULT_TIMEOUT_MS = 1000u;

static struct net_cfg normalize_net_cfg(const struct net_cfg *raw) {
  struct net_cfg cfg = {};
  cfg.timeout_ms = DEFAULT_TIMEOUT_MS;
  if (raw != nullptr) {
    if (raw->timeout_ms > 0u) {
      cfg.timeout_ms = raw->timeout_ms;
    }
    cfg.retries = raw->retries;
  }
  return cfg;
}
```

### 4. Closed enum `switch` vs. open external enum
```c
// Closed internal enum: satisfies -Wswitch, -Wswitch-enum, and GCC -Wreturn-type
switch (codec_state) {
  case CODEC_IDLE:  return "idle";
  case CODEC_READY: return "ready";
}
assert(false);
return "unknown";

// Open external protocol enum: C23 fixed underlying type + explicit default
enum wire_tag : uint32_t { WIRE_TAG_PING = 1u };
switch ((uint32_t)wire_tag) {
  case WIRE_TAG_PING: return handle_ping(msg);
  default:            return ERR_UNSUPPORTED;
}
```

## Agent Modes & Verification
- **Audit mode (code review):** Report findings as `file:line | P0/P1/P2 | smell | fix`, sorted P0 first.
- **Refactor mode (edits):** Keep structural refactors in separate commits from behavioral changes.
- **Verify:** Compile with `clang -std=c23 -Wall -Wextra -Werror -Wpedantic -Wconversion -Wshadow -Wcast-qual -Wswitch-enum -Wformat=2 -Wundef`, run `clang-tidy --checks='bugprone-*,cert-*'` (or the project's `.clang-tidy`), and run tests under `-fsanitize=address,undefined`.
