# RFC-0017: Directory Naming (Hearth & Grove)

- **Status:** Implemented
- **Created:** 2026-03-16
- **Author:** Pete Pavlovski
- **Amends:** RFC-0008 (Package Manager)

---

## Summary

Rename Sage's build and cache directories to reflect their distinct responsibilities:

- **Grove** (`~/.grove/`) — The global package cache, where dependencies live
- **Hearth** (`hearth/`) — The per-project build output, where your project is forged

---

## Motivation

Sage currently uses `~/.sage/` for global state and `target/` for build output. This naming is functional but lacks character and conflates the language name with its tooling.

The proposed naming gives each component a distinct identity:

| Directory | Metaphor | Responsibility |
|-----------|----------|----------------|
| `~/.grove/` | A grove of trees | Package ecosystem — cached dependencies from the wider world |
| `hearth/` | A forge's hearth | Build output — where your project's code is compiled and shaped |

This mirrors how Rust separates concerns: Cargo manages dependencies, but `target/` is where rustc does its work. Here, Grove manages the package ecosystem, and Hearth is where the compiler does its forging.

---

## Changes

### Directory renames

| Current | Proposed | Purpose |
|---------|----------|---------|
| `~/.sage/packages/` | `~/.grove/packages/` | Global package cache |
| `~/.sage/` (other) | `~/.grove/` | Any other global Grove state |
| `target/` | `hearth/` | Per-project build output |
| `.sage/` (project-local) | Removed | Unnecessary intermediate state |

### Why remove `.sage/`?

The project-local `.sage/` directory was used for intermediate state that properly belongs in the build output. With `hearth/` as the dedicated build directory, there's no need for a separate hidden directory. Any project-local cache or state goes in `hearth/`.

If future features require project-local Grove configuration (e.g., local package overrides), a `.grove/` directory could be introduced, but this is out of scope for now.

### Affected components

1. **sage-package** — Update `CACHE_DIR` from `~/.sage/packages` to `~/.grove/packages`
2. **sage-codegen** — Update output directory from `target` to `hearth`
3. **sage-cli** — Update any references to build output paths
4. **sage-loader** — Remove any `.sage/` project-local handling
5. **.gitignore templates** — Update `sage new` to ignore `hearth/` instead of `target/`

### Backwards compatibility

**Global cache:** On first run after upgrade, Grove will check for `~/.sage/packages/` and emit a one-time migration notice. Users can manually move their cache or let it rebuild.

**Build output:** No migration needed. Old `target/` directories can be deleted manually. The new `hearth/` directory will be created on next build.

---

## Implementation

1. Update `CACHE_DIR` constant in `sage-package/src/lib.rs`
2. Update output path in `sage-codegen/src/generator.rs`
3. Update `.gitignore` template in `sage new`
4. Remove any `.sage/` project-local logic
5. Add migration notice for old global cache location
6. Update documentation and examples

---

## Open Questions

None.
