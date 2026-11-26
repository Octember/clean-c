---
name: clean-c
description: >-
  Dense Clean Code principles tailored to modern C (C23) and TigerStyle / NASA Power of 10
  systems programming. Use when writing, reviewing, or refactoring C/H files to eliminate
  slop, oversized functions, duplicated truth, unsafe loops, or leaky abstractions.
---

# Clean Code in C (C23)

## 1. Meaningful Names
- **Intention-revealing names:** Every identifier must state *what* it represents and *why* it exists without requiring a comment. Prefer `elapsed_time_ms` over `d`, `max_pnext_depth` over `n`, `buffer_size_bytes` over `sz`.
- **Units and representation in names:** In C, types don't carry units. Put units last so related names align: `timeout_ms`, `offset_bytes`, `stride_u32`, `align_bytes`. Distinguish `index` (0-based position), `count` (number of elements), and `size_bytes` (byte length) to prevent off-by-one and byte-vs-element bugs.
- **Avoid disinformation & noise words:** Never name an array `account_list` if it is a flat array, or append meaningless suffixes like `_info`, `_data`, `_obj`, `tmp1`, `val2`, `helper`. Every word in a name must carry distinct information.
- **Consistent vocabulary:** Pick one verb per concept across the whole codebase and stick to it. Don't mix `get`/`fetch`/`query`/`read`, or `create`/`alloc`/`make`/`new`, or `destroy`/`free`/`release`/`delete` for identical lifecycles.
- **Scope-length rule:** Short loop counters (`i`, `j`, `k`) are fine only in 3–5 line loops. Longer scopes demand longer, more descriptive names. Conversely, narrow `static` helpers can have short names while exported symbols need a consistent module prefix (`vkr_`, `wgsl_`, `sqlite3_`).
- **Searchable constants:** Replace every magic number with a `static constexpr` or `enum` constant (`VKR_QUEUE_FAMILY_COUNT`, `VKR_ALIGN_BYTES`). You should be able to grep for every limit, alignment, and capacity in the codebase.

## 2. Functions
- **Small and single-purpose:** Keep functions under ~40–60 lines. A function should do one thing, do it well, and do it at **one level of abstraction**. Don't mix high-level orchestration (`create_graphics_pipeline`) with bit-twiddling or raw struct field packing in the same body.
- **The Stepdown Rule:** Order functions top-to-bottom like a newspaper article — public entry point or dispatcher near the bottom/top, with each helper reading at one level of detail below its caller.
- **Minimize parameters (0–3 ideal):** More than 3–4 parameters is a smell, especially when multiple parameters share the same type (`uint32_t width, uint32_t height, uint32_t depth, uint32_t layers`). Group related parameters into a `const struct foo_desc *` passed by pointer and initialized with designated initializers.
- **Ban flag arguments:** Passing `bool is_v2` or `bool should_clear` means the function does two things. Split it into two named functions (`fill_buffer_reqs` / `fill_buffer_reqs2`) that call one shared internal helper.
- **Command-Query Separation (CQS):** A function should either *mutate state* (command) or *compute/return a value* (query), not secretly both. Prefer pure value-returning functions for struct building/normalization (`struct foo normalize_foo(const struct foo *in)`) over functions that mutate hidden globals.
- **No hidden side effects:** If `check_format_supported()` also zeroes an output buffer or mutates a cache, its name is lying. Make every mutation explicit in the signature (`out_` parameter prefix, non-`const` pointer) and name.
- **Avoid output parameters when a return value works:** In C, returning a small struct by value (or a `{ result, error }` struct) is cleaner than `int compute(int a, int *out_b, int *out_c)`. Use `out_*` pointers only for large structs or arrays.
- **Bury `switch` statements at the boundary:** Don't repeat the same `switch (kind)` across five different files. Centralize dispatch in one table or one small factory/dispatcher function; everything downstream operates on normalized data.
- **DRY without premature abstraction:** Deduplicate copy-pasted v1/v2 handlers, repeated bounds checks, and repeated capability literals via small `_common` helpers or `static const` tables — never via clever nested macros that obscure control flow.

## 3. Control Flow & Simplicity
- **Guard clauses over arrow code:** Validate preconditions and `nullptr` at the top and return early. Keep the happy path at indentation depth 1–2.
- **Bound every loop:** Every `for`/`while` walking a linked list (`pNext`), ring buffer, or external byte stream must have an explicit upper bound (`depth < MAX_DEPTH`) and guaranteed forward progress.
- **No recursion in low-level C:** Replace recursive traversals with an explicit fixed-capacity stack or iterative loop to guarantee bounded stack usage.
- **Explicit switch coverage:** Handle every enum value or cast explicitly (`switch ((uint32_t)stype)`) with a `default:` branch that either fails loudly or intentionally ignores unknown extensions.
- **Positive conditions:** Prefer `if (is_valid)` over `if (!is_invalid)`. Split dense boolean expressions into named `const bool` variables (`const bool has_color = ...;`) before branching.

## 4. Comments
- **Comments do not excuse bad code:** If you feel the urge to write a comment explaining *what* a 30-line block does, extract that block into a well-named 30-line function instead.
- **Explain *why*, never *what*:** Good comments document invariants, non-obvious hardware/spec quirks, ordering constraints, or why a seemingly simpler approach fails (e.g., `// wasm32 -nostdlib lacks __multi3 for 64-bit ckd_mul`).
- **Delete commented-out code and journal comments:** Git owns history. Dead code in `#if 0` or `/* ... */` rots immediately and confuses readers.
- **Keep comments local and accurate:** A stale comment is worse than no comment. Update or delete comments whenever you touch the code beneath them.

## 5. Formatting & File Structure
- **One file, one cohesive responsibility (< 250–300 lines):** When a `.c` or `.h` grows past ~300 lines, it almost always mixes two concerns (e.g., static capability tables + command dispatchers, or pipeline state packing + draw commands). Split along data/responsibility boundaries.
- **Vertical proximity:** Keep tightly coupled concepts together. Declare variables as close to their first use as possible (C99/C23 — not C89 top-of-function dumps). Place a `static` helper right next to its sole caller.
- **Header discipline:** Headers (`.h`) should expose minimal public declarations, types, and `static inline` tiny accessors — avoid stuffing 500 lines of `static` implementation functions into headers unless required by a deliberate single-TU build layout, and even then split by cohesive domain.
- **Consistent mechanical formatting:** Let `clang-format` and `clang-tidy` own indentation, line length (80–100 cols), brace style, and naming checks. Never waste review cycles on manual whitespace.

## 6. Data Structures, Encapsulation & Ownership
- **Data structures vs. modules:** Plain structs expose data cleanly (`struct vkr_blend_state`) and let callers operate on them freely; opaque handles/modules hide internal invariants behind a narrow API (`vkr_arena_alloc`, `vkr_obj_size_get`) so representation can change without breaking callers. Don't create half-opaque hybrids.
- **Single source of truth:** Never store or hardcode the same fact twice (e.g., an extension feature flag in both a KHR struct and a Vulkan 1.2/1.3 struct, or heap size in both `fill_mem_props` and `fill_mem_props2_pnext`). Put the truth in one `static const` struct/table and derive all views from it.
- **Zero-initialized by default:** Design structs so `{ 0 }` is a safe, valid default state. Zero output buffers before populating them to prevent uninitialized-memory leaks ("buffer bleeds").
- **Explicit memory ownership & arenas:** Every pointer must have an obvious owner and lifetime. Prefer caller-owned buffers or explicit static/linear arenas over ad-hoc `malloc`/`free` scattered across functions. Never allocate dynamically inside hot paths or post-init loops.
- **`const` by default, zero `const`-casts:** Mark every non-mutated parameter, pointer target, local variable, and lookup table `const`. Treat a `const`-cast as a design bug.

## 7. Error Handling & Safety
- **Never swallow errors:** Every fallible function must return an explicit status (`VkResult`, `bool`, negative error code) and every caller must check it. Use `[[nodiscard]]` when ignoring a return value is a real bug.
- **Separate error handling from business logic:** Keep error paths short and uniform (early return or single cleanup path) so they don't drown out the main algorithm.
- **Use `assert()` for programmer invariants, runtime checks for untrusted input:** `assert()` catches broken internal contracts (non-null internal pointers, alignment, table bounds). Never rely on `assert()` alone to validate external wire data, shader bytecode, or user-supplied sizes — validate those explicitly and return an error.
- **Overflow-safe arithmetic:** Use C23 `<stdckdint.h>` (`ckd_add`, `ckd_mul`) for size/offset calculations before slicing or allocating buffers. Beware unsigned wraparound and implicit narrowing (`-Wconversion`).
- **Never return `nullptr` unexpectedly without contract:** Document and enforce nullability consistently (`assert(ptr != nullptr)` for non-null contracts; explicit `if (ptr == nullptr) return ...;` for optional API pointers).

## 8. Boundaries & Third-Party Code
- **Isolate external/generated headers:** Wrap upstream protocol headers, OS APIs, and compiler/graphics backends behind thin, stable internal adapter headers (`-isystem` for third-party code so your `-Werror` flags stay strict on first-party code).
- **Don't leak third-party types everywhere:** Convert external wire formats, Vulkan `pNext` chains, or NIR structs into clean internal structs at the boundary so the core logic stays simple and testable.
- **Write learning/characterization tests at boundaries:** Before refactoring a boundary module, pin its observable behavior with fast unit/host tests (under ASan + UBSan) and golden fixtures.

## 9. Cohesion, Coupling & Refactoring Discipline
- **High cohesion, low coupling:** Every function in a module should operate on the same core state or domain concept. If half the functions in a file use struct A and the other half use struct B, you have two modules pretending to be one.
- **Zero mutable globals outside explicit state structs:** Pass explicit context/arena pointers instead of reading hidden file-scope globals. Explicit state makes functions re-entrant, testable, and easy to reason about.
- **The Boy Scout Rule:** Leave every file cleaner than you found it — but keep behavioral changes and structural refactors in separate commits so regressions are trivial to bisect.
- **Automate the rules:** Humans forget style rules under deadline pressure. Enforce them mechanically with `-Wall -Wextra -Werror -Wpedantic -Wconversion -Wshadow -Wcast-qual -Wswitch-enum`, `clang-tidy`, ASan/UBSan, and repo-level lint scripts for naming, function length, and file length.

## 10. Top C Slop Smells (Quick Audit Checklist)
1. **God header/file (> 300 lines)** mixing data tables, validation, and dispatch.
2. **God function (> 60 lines)** mixing multiple abstraction levels or 15+ `else if` branches.
3. **Copy-paste v1/v2 or twin handlers** differing by one struct field.
4. **Duplicated truth** (same flag, limit, or magic number written in 2+ places).
5. **Unbounded pointer-chasing loops** (`while (p != nullptr)`) on external data.
6. **Silent error drops** (ignored return values, unchecked `status < 0`, unchecked arithmetic).
7. **Opaque abbreviations & numbered temps** (`tmp1`, `buf2`, `val`, `p`, `sz`).
8. **Hidden state mutation** via globals or surprise side effects inside query functions.
9. **Dead code / `#if 0` / stale comments** left behind after iterative fixes.
10. **Macro cleverness** replacing simple `static inline` functions or `const` tables.

## Workflow & Verification
1. Audit target files against the 10-point C Slop Smells checklist.
2. Split god files (`> 300` lines) and god functions (`> 60` lines), centralize duplicated truth into `static const` tables, and share `_common` helpers across twin/v1-v2 handlers.
3. Keep behavioral changes separate from pure structural refactors.
4. Verify with strict C23 warnings (`clang -std=c23 -Wall -Wextra -Werror -Wpedantic -Wconversion -Wshadow -Wcast-qual -Wswitch-enum`), `clang-tidy`, and `-fsanitize=address,undefined` tests.
