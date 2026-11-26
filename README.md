# clean-c

Clean Code principles tailored to modern C (C23), TigerStyle, and NASA Power of 10 systems programming.

## Install

```bash
npx skills add Octember/clean-c --skill clean-c
```

## What It Covers

- **P0 — Safety & UB:** Strict aliasing (`memcpy`), explicit-endian wire decoding, C23 `<stdckdint.h>` (`ckd_add`/`ckd_sub`/`ckd_mul`), `nullptr` slice guards (`len > 0` before `memcpy`), `snprintf` truncation checks, `_Atomic` vs. `volatile`, bounded loops with cycle/overflow reporting, and `assert()` vs. untrusted-input validation.
- **P1 — Architecture & Anti-Slop:** `< 300`-line modules, `< 60`-line functions (with lookup-table escape hatches), single-source-of-truth constants/tables, C23 `{}` defaults, `realloc` safety (no `realloc(p, 0)` UB), `goto cleanup;` (pragmatic exception to Power-of-10 Rule 1) / arena ownership, and closed vs. open enum `switch`.
- **P2 — Naming, Layout & Macros:** Units-last naming, leaf-top/API-bottom vertical layout, *why*-only comments, and `do { ... } while (0)` macro hygiene without double evaluation.
- **Before/After C23 Examples & Audit Mode:** Compile-checked C23 snippets plus a `file:line | P0/P1/P2 | smell | fix` review format.

## License

MIT
