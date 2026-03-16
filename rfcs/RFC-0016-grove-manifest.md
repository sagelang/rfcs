# RFC-0016: Grove Manifest Files

- **Status:** Implemented
- **Created:** 2026-03-16
- **Author:** Pete Pavlovski
- **Amends:** RFC-0008 (Package Manager)

---

## Summary

Rename `sage.toml` to `grove.toml` and `sage.lock` to `grove.lock` to align manifest file naming with **Grove**, Sage's package manager.

---

## Motivation

RFC-0008 introduced the package manager but named the manifest files after the language (`sage.toml`, `sage.lock`) rather than the tool. This creates a naming inconsistency: the package manager is called Grove, but its files don't reflect that.

Other language ecosystems follow the pattern of naming manifest files after the package manager:

| Language | Package Manager | Manifest | Lock File |
|----------|-----------------|----------|-----------|
| Rust | Cargo | `Cargo.toml` | `Cargo.lock` |
| JavaScript | npm | `package.json` | `package-lock.json` |
| Python | Poetry | `pyproject.toml` | `poetry.lock` |

Sage should follow suit: **Grove** manages dependencies, so the files should be `grove.toml` and `grove.lock`.

---

## Changes

### File renames

| Before | After |
|--------|-------|
| `sage.toml` | `grove.toml` |
| `sage.lock` | `grove.lock` |

### Affected components

1. **sage-loader** — Update `MANIFEST_FILE` and `LOCK_FILE` constants
2. **sage-package** — Update manifest/lock file references in resolver
3. **sage-cli** — Update `sage new` template and any help text
4. **Documentation** — Update README, book, and any examples

### Backwards compatibility

For a transitional period, the loader will check for `sage.toml` if `grove.toml` is not found and emit a deprecation warning:

```
warning: sage.toml is deprecated, rename to grove.toml
```

This fallback will be removed in the next minor version.

---

## Implementation

1. Update constants in `sage-loader/src/tree.rs`
2. Update manifest handling in `sage-package/src/lib.rs`
3. Update `sage new` scaffolding in `sage-cli`
4. Add deprecation warning for old filenames
5. Update all example projects and tests

---

## Open Questions

None.
