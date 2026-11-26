# clean-c

Clean Code principles tailored to modern C (C23), TigerStyle, and NASA Power of 10 systems programming.

## Install

```bash
npx skills add Octember/clean-c --skill clean-c
```

## What It Covers

- **P0 — Safety & UB:** Strict aliasing (`memcpy`), C23 `<stdckdint.h>` checked arithmetic, explicit byte slices, bounded loops, and `assert()` vs. untrusted-input validation.
- **P1 — Architecture & Anti-Slop:** `< 300`-line modules, `< 60`-line functions (with lookup-table escape hatches), single-source-of-truth tables, `goto cleanup;` / arena ownership, and closed vs. open enum `switch`.
- **P2 — Naming, Layout & Macros:** Units-last naming, leaf-top/API-bottom vertical layout, *why*-only comments, and `do { ... } while (0)` macro hygiene.
- **Before/After C23 Examples & Audit Mode:** Concrete C23 snippets plus a `file:line | P0/P1/P2 | smell | fix` review format.

## License

MIT
