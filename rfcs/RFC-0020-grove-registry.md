# RFC-0020: Grove Registry and Semantic Versioning

- **Status:** Draft
- **Created:** 2026-03-16
- **Author:** Pete Pavlovski
- **Amends:** RFC-0008 (Package Manager)

---

## Table of Contents

1. [Summary](#1-summary)
2. [Motivation](#2-motivation)
3. [Design Goals](#3-design-goals)
4. [Registry Architecture](#4-registry-architecture)
5. [Semantic Versioning](#5-semantic-versioning)
6. [Package Manifest Extensions](#6-package-manifest-extensions)
7. [Version Resolution](#7-version-resolution)
8. [Publishing Workflow](#8-publishing-workflow)
9. [Security Model](#9-security-model)
10. [Private Registries](#10-private-registries)
11. [CLI Commands](#11-cli-commands)
12. [Migration Path](#12-migration-path)
13. [Implementation Plan](#13-implementation-plan)
14. [New Error Codes](#14-new-error-codes)
15. [Open Questions](#15-open-questions)
16. [Alternatives Considered](#16-alternatives-considered)
17. [Out of Scope](#17-out-of-scope)

---

## 1. Summary

This RFC evolves Grove from a Git-first package manager into a **full-fledged package ecosystem** with:

- A **central registry** (`grove.io`) for package discovery and distribution
- **Semantic versioning** with version range constraints (`^1.2.0`, `~1.2.3`, `>=1.0,<2.0`)
- **Intelligent version resolution** that finds compatible dependency graphs
- A **publishing workflow** (`sage publish`) with validation and authentication
- **Security features**: SHA-256 checksums, optional package signing, security advisories
- **Private registry support** for enterprise deployments

Git dependencies remain fully supported as an escape hatch for unpublished packages, forks, and development workflows.

---

## 2. Motivation

RFC-0008 established Grove as a functional package manager. It's done its job admirably — you can share code between projects. But several friction points have emerged:

### 2.1 Discovery Problem

There's no way to find packages. Developers must somehow learn about a package (word of mouth, documentation, searching GitHub) and then manually construct the Git URL and tag. Compare:

```toml
# Today - you need to know the URL exists
http_client = { git = "https://github.com/sage-packages/http-client", tag = "v1.2.0" }

# What we want
http_client = "^1.2"
```

### 2.2 Version Management Burden

Every dependency requires an exact Git ref. When `http_client` releases v1.2.1 with a security fix, you must:

1. Discover that v1.2.1 exists (how?)
2. Edit `grove.toml` to change the tag
3. Run `sage update`

With semantic versioning, `^1.2` automatically resolves to v1.2.1 on the next `sage update` — and you'd get notified about the advisory.

### 2.3 Transitive Version Conflicts

Today, if two dependencies want different versions of the same transitive dependency, Grove fails. A proper resolver can often find a version that satisfies both constraints.

### 2.4 No Quality Signals

There's no way to know if a package is maintained, widely used, or abandoned. A registry provides download counts, last-updated timestamps, and community metrics.

### 2.5 Security Gaps

Git provides commit-level integrity (the SHA matches what you locked), but there's no:
- Independent checksum verification
- Vulnerability database integration
- Package signature verification
- Yanking of compromised versions

---

## 3. Design Goals

1. **Backwards compatible.** Existing `grove.toml` files with Git dependencies continue to work unchanged.
2. **Registry-first, Git-always.** Registry is the default for published packages; Git remains available for everything else.
3. **Semantic versioning.** Version constraints follow Cargo/npm conventions — familiar to most developers.
4. **Reproducible by default.** `grove.lock` continues to pin exact versions. The registry just makes resolution smarter.
5. **Offline capable.** Once dependencies are cached, builds work without network access.
6. **Enterprise ready.** Private registries with authentication for corporate use.
7. **Security conscious.** Checksums, signatures, and advisory integration from day one.

---

## 4. Registry Architecture

### 4.1 The Central Registry: grove.io

The official Sage package registry lives at `grove.io` (or `registry.sagelang.dev` — TBD). It consists of:

**Index** — A Git repository containing package metadata:
```
index/
├── config.json           # Registry configuration
├── 1/
│   └── a/                # Single-char packages (rare)
├── 2/
│   ├── ab/
│   └── ...
├── 3/
│   ├── a/
│   │   └── abc           # 3-char package "abc"
│   └── ...
├── ht/
│   └── tp/
│       └── http_client   # Package metadata file
└── sa/
    └── ge/
        └── sage_stdlib
```

This structure (borrowed from crates.io) enables efficient Git sparse-checkout — you only fetch the metadata files you need.

**Package metadata file** (`ht/tp/http_client`):
```json
{"name":"http_client","vers":"1.0.0","deps":[{"name":"json","req":"^1.0","features":[],"optional":false}],"cksum":"abc123...","features":{},"yanked":false}
{"name":"http_client","vers":"1.1.0","deps":[{"name":"json","req":"^1.0","features":[],"optional":false}],"cksum":"def456...","features":{},"yanked":false}
{"name":"http_client","vers":"1.2.0","deps":[{"name":"json","req":"^1.1","features":[],"optional":false}],"cksum":"789abc...","features":{},"yanked":false}
```

Each line is a JSON object for one version. Append-only — new versions add lines.

**API** — HTTPS endpoints for:
- Package upload (`POST /api/v1/packages/new`)
- Package download (`GET /api/v1/packages/{name}/{version}/download`)
- Search (`GET /api/v1/packages?q={query}`)
- User management (login, token generation)
- Yank/unyank (`DELETE /api/v1/packages/{name}/{version}/yank`)

**CDN** — Package tarballs served from a CDN for fast downloads worldwide.

### 4.2 Package Distribution Format

Published packages are distributed as **gzipped tarballs** (`.tar.gz`), not Git repositories:

```
http_client-1.2.0.tar.gz
├── grove.toml
├── src/
│   ├── lib.sg
│   ├── client.sg
│   └── ...
├── README.md
└── Cargo.toml.orig  # (if the package is a mixed Sage/Rust project)
```

Benefits over Git:
- Faster downloads (no Git overhead)
- No `.git` directory in cache
- Content-addressed storage on CDN
- Works identically for private registries

### 4.3 Index Synchronisation

The registry index is cloned to `~/.sage/registry/index/`:

```bash
# First sync
git clone --bare https://github.com/sagelang/grove-index ~/.sage/registry/index

# Subsequent syncs (fast - incremental fetch)
git fetch origin master
```

The resolver reads package metadata directly from the local index clone. Network calls only happen when:
- Syncing the index (`sage update --index`)
- Downloading a package tarball not in cache
- Publishing a package

### 4.4 Package Storage

Downloaded tarballs are cached at:
```
~/.sage/registry/cache/
├── http_client-1.2.0.tar.gz
├── json-1.1.0.tar.gz
└── ...
```

Extracted packages at:
```
~/.sage/packages/
├── http_client/
│   └── 1.2.0/
│       ├── .grove-meta.toml
│       ├── grove.toml
│       └── src/
└── json/
    └── 1.1.0/
        └── ...
```

This maintains compatibility with the existing cache structure from RFC-0008.

---

## 5. Semantic Versioning

### 5.1 Version Format

Packages use [Semantic Versioning 2.0.0](https://semver.org/):

```
MAJOR.MINOR.PATCH[-PRERELEASE][+BUILD]

Examples:
1.0.0
1.2.3
2.0.0-alpha.1
2.0.0-beta.2+build.456
0.1.0  (pre-1.0: any change may break)
```

### 5.2 Version Requirements

Dependencies specify **version requirements**, not exact versions:

| Syntax | Meaning | Matches |
|--------|---------|---------|
| `"1.2.3"` | Exactly 1.2.3 | 1.2.3 |
| `"^1.2.3"` | Compatible with 1.2.3 | >=1.2.3, <2.0.0 |
| `"^0.2.3"` | Compatible with 0.2.3 | >=0.2.3, <0.3.0 |
| `"~1.2.3"` | Approximately 1.2.3 | >=1.2.3, <1.3.0 |
| `">=1.2,<1.5"` | Range | >=1.2.0, <1.5.0 |
| `"*"` | Any version | (not recommended) |

**Caret (`^`)** is the default and recommended form. It allows updates that don't break the API according to SemVer rules.

### 5.3 Pre-1.0 Semantics

For versions `0.x.y`, the rules tighten:
- `^0.2.3` matches `>=0.2.3, <0.3.0` (MINOR is breaking)
- `^0.0.3` matches `=0.0.3` (PATCH is breaking in 0.0.x)

This follows Cargo's conventions and reflects the reality that pre-1.0 packages haven't committed to API stability.

### 5.4 Dependency Declaration

```toml
[dependencies]
# Registry dependencies (new)
sage_stdlib = "^0.2"
http_client = "^1.2"
json = "~1.0"

# Git dependencies (still supported)
experimental = { git = "https://github.com/someone/experimental", rev = "a3f8c21" }

# Path dependencies (still supported)
my_lib = { path = "../my-lib" }

# Registry with explicit source (for private registries)
internal_lib = { version = "^1.0", registry = "https://grove.internal.corp" }
```

The short form `package = "version"` is syntactic sugar for `package = { version = "version" }`.

---

## 6. Package Manifest Extensions

### 6.1 New Fields in `grove.toml`

```toml
[project]
name = "http_client"
version = "1.2.0"
entry = "src/lib.sg"

# New fields for registry packages
description = "HTTP client library for Sage"
license = "MIT"
repository = "https://github.com/sage-packages/http-client"
documentation = "https://docs.grove.io/http_client"
homepage = "https://grove.io/packages/http_client"
readme = "README.md"
keywords = ["http", "client", "networking", "web"]
categories = ["networking", "web"]

# Authors with optional email
authors = [
    "Jane Developer <jane@example.com>",
    "Bob Contributor"
]

# Minimum Sage version required
sage = ">=1.0"

[dependencies]
json = "^1.1"
url_parser = "^0.5"

# Optional dependencies (enabled by features)
[dependencies.tls_support]
version = "^1.0"
optional = true

# Features for conditional compilation
[features]
default = []
tls = ["tls_support"]
```

### 6.2 Required Fields for Publishing

To run `sage publish`, a package must have:
- `name` — Package name (validated: lowercase, alphanumeric, underscores, hyphens)
- `version` — SemVer version string
- `description` — Brief description (max 500 chars)
- `license` — SPDX license identifier or `license-file` path

### 6.3 Package Naming Rules

- 1-64 characters
- Lowercase ASCII letters, digits, underscores, hyphens
- Must start with a letter
- Cannot be a Sage keyword
- No consecutive underscores or hyphens
- Reserved: `sage`, `grove`, `std`, `core`, `test`

---

## 7. Version Resolution

### 7.1 The Resolution Problem

Given a set of direct dependencies with version requirements, find a set of concrete versions for all transitive dependencies such that:

1. Every version requirement is satisfied
2. Each package appears at most once (no duplicate versions)
3. If multiple solutions exist, prefer newer versions

This is NP-complete in the general case (it's a variant of SAT), but practical dependency graphs are tractable.

### 7.2 Resolution Algorithm

Grove uses a **backtracking resolver** with conflict-driven clause learning (inspired by PubGrub, used by Dart/Flutter):

```
1. Start with direct dependencies from grove.toml
2. For each unresolved package:
   a. Query the index for available versions matching the requirement
   b. Try versions from newest to oldest
   c. For each candidate version:
      - Add its dependencies to the resolution queue
      - Check for conflicts with already-resolved packages
      - If conflict: backtrack and try next candidate
      - If no conflict: continue to next unresolved package
3. If all packages resolved: success → write grove.lock
4. If no valid resolution exists: report the conflict chain
```

### 7.3 Conflict Reporting

When resolution fails, Grove explains **why**:

```
error[E036]: version conflict for 'json'

  http_client 1.2.0 requires json ^1.1
  legacy_parser 2.0.0 requires json ^0.9

  These requirements are incompatible.

help: upgrade legacy_parser to 2.1.0+ which requires json ^1.0
  or: downgrade http_client to 1.0.0 which requires json ^0.9
```

### 7.4 Minimal Version Selection (Optional)

For maximum reproducibility, Grove can use **minimal version selection** (MVS) instead of maximal:

```bash
sage install --minimal
```

This picks the *lowest* version satisfying each requirement, ensuring builds don't accidentally depend on newer APIs. Useful for library authors testing compatibility.

### 7.5 Lock File Changes

The lock file format gains a `source` field to distinguish registry from Git packages:

```toml
# grove.lock
version = 2

[[packages]]
name = "http_client"
version = "1.2.0"
source = "registry+https://grove.io"
checksum = "sha256:abc123def456..."
dependencies = ["json"]

[[packages]]
name = "json"
version = "1.1.0"
source = "registry+https://grove.io"
checksum = "sha256:789xyz..."
dependencies = []

[[packages]]
name = "experimental"
version = "0.1.0"
source = "git+https://github.com/someone/experimental?rev=a3f8c21"
dependencies = []
```

---

## 8. Publishing Workflow

### 8.1 Authentication

Publishers authenticate via API tokens:

```bash
# Generate token at grove.io/settings/tokens
sage login
# Enter your grove.io API token: ****

# Token stored at ~/.sage/credentials.toml
```

### 8.2 Pre-Publish Validation

`sage publish --dry-run` validates the package without uploading:

```
$ sage publish --dry-run
  Packaging http_client v1.2.1...
  Verifying grove.toml...
    ✓ name: http_client
    ✓ version: 1.2.1 (not yet published)
    ✓ description: HTTP client library for Sage
    ✓ license: MIT
    ✓ entry: src/lib.sg (exists, is library)
  Checking dependencies...
    ✓ json ^1.1 → 1.1.0 (published)
    ✓ url_parser ^0.5 → 0.5.2 (published)
  Building package...
    ✓ sage check passed
    ✓ sage test passed (12 tests)
  Packaging...
    ✓ http_client-1.2.1.tar.gz (42 KB)
    ✓ checksum: sha256:abc123...

  Ready to publish! Run `sage publish` to upload.
```

### 8.3 Publishing

```bash
sage publish
```

This:
1. Runs all validations from `--dry-run`
2. Creates the tarball
3. Computes SHA-256 checksum
4. Signs the package (if signing key configured)
5. Uploads to the registry API
6. Updates the index (registry commits a new line to the package metadata file)

### 8.4 Yanking

Yanking marks a version as "do not use" without deleting it (existing lock files still work):

```bash
sage yank http_client@1.2.0 --reason "Security vulnerability CVE-2026-1234"
```

Yanked versions:
- Cannot be newly resolved (not returned in version queries)
- Can still be fetched if pinned in `grove.lock` (with a warning)
- Show warnings in `sage install` output

### 8.5 Version Restrictions

- Cannot publish the same version twice
- Cannot publish a version lower than the highest existing version (no backfilling)
- Pre-release versions (`1.0.0-alpha.1`) can be published at any time

---

## 9. Security Model

### 9.1 Checksum Verification

Every package in the registry has a SHA-256 checksum stored in the index:

```json
{"name":"http_client","vers":"1.2.0",...,"cksum":"abc123def456789..."}
```

The resolver verifies checksums before extracting packages:

```
$ sage install
  Fetching http_client v1.2.0...
  error: checksum mismatch for http_client-1.2.0.tar.gz
    expected: abc123def456789...
    actual:   zzz000111222333...
  This may indicate a corrupted download or a compromised registry.
```

### 9.2 Package Signing (Optional)

Publishers can sign packages with an Ed25519 key:

```bash
# Generate signing key
sage keys generate
# Public key: grove_pub_1a2b3c4d...
# Secret key stored at ~/.sage/signing-key.pem

# Register public key with grove.io
sage keys register

# Packages are now signed automatically on publish
sage publish
  Signing http_client-1.2.1.tar.gz...
  Signature: grove_sig_xyz789...
```

Consumers can require signatures:

```toml
# ~/.sage/config.toml
[security]
require_signatures = true
trusted_keys = [
    "grove_pub_1a2b3c4d...",  # Jane Developer
    "grove_pub_5e6f7a8b...",  # Sage Team
]
```

### 9.3 Security Advisories

Grove integrates with a security advisory database:

```bash
sage audit
  Checking for known vulnerabilities...

  ⚠ http_client 1.2.0 has a known vulnerability
    Advisory: GROVE-2026-0042
    Severity: High
    Title: HTTP header injection vulnerability
    Patched in: 1.2.1

    Run `sage update http_client` to upgrade.

  Found 1 vulnerability in 12 packages.
```

The advisory database is a separate Git repository synced alongside the index:

```
~/.sage/registry/advisory-db/
├── advisories/
│   ├── GROVE-2026-0042.toml
│   └── ...
└── index.toml
```

### 9.4 Supply Chain Protections

- **Typosquatting detection**: Registry rejects packages with names too similar to popular packages
- **Name reservation**: Owners can reserve related names (`http-client` reserves `http_client`, `httpclient`)
- **Transfer verification**: Package ownership transfers require email confirmation
- **Two-factor authentication**: Required for packages with >1000 downloads

---

## 10. Private Registries

### 10.1 Configuration

Private registries are configured in `~/.sage/config.toml`:

```toml
[registries.corp]
url = "https://grove.internal.corp"
token = "corp_token_abc123"

[registries.team]
url = "https://grove.myteam.dev"
token-command = "vault read -field=token secret/grove"
```

### 10.2 Dependency Declaration

```toml
[dependencies]
# Default registry (grove.io)
http_client = "^1.2"

# Corporate private registry
internal_auth = { version = "^2.0", registry = "corp" }

# Team private registry
shared_utils = { version = "^1.0", registry = "team" }
```

### 10.3 Registry Fallback

By default, packages are looked up in grove.io. The `registry` key specifies an alternative. There's no automatic fallback between registries — each package explicitly declares its source.

### 10.4 Private Registry Implementation

A private registry needs:
1. **Index Git repository** — same format as grove.io
2. **Package storage** — HTTP server serving tarballs
3. **API endpoint** — for `sage publish` (can be minimal)

A reference implementation (`grove-registry-server`) will be provided as a Docker image.

---

## 11. CLI Commands

### 11.1 Updated Commands

**`sage add`** — Now defaults to registry:

```bash
# Registry dependency (new default)
sage add http_client
sage add http_client@^1.2
sage add http_client@1.2.3  # Exact version

# Git dependency (explicit)
sage add http_client --git https://github.com/sage-packages/http-client --tag v1.2.0

# Private registry
sage add internal_auth --registry corp
```

**`sage install`** — Unchanged behaviour, now also syncs registry index:

```bash
sage install
  Syncing registry index...
  Resolving dependencies...
  Fetching http_client v1.2.0 (registry)
  Fetching json v1.1.0 (registry)
  Fetching experimental v0.1.0 (git)
  Installed 3 packages
```

**`sage update`** — Now respects version constraints:

```bash
# Update all within constraints
sage update

# Update specific package
sage update http_client

# Update and sync index
sage update --index

# Ignore constraints, get absolute latest (use with caution)
sage update --breaking
```

### 11.2 New Commands

**`sage search`** — Search the registry:

```bash
$ sage search http
  http_client    1.2.0    HTTP client library for Sage
  http_server    0.5.0    Simple HTTP server framework
  http_mock      0.3.1    HTTP mocking for tests

  3 packages found. See grove.io for more.
```

**`sage info`** — Show package details:

```bash
$ sage info http_client
  http_client 1.2.0
  HTTP client library for Sage

  License: MIT
  Repository: https://github.com/sage-packages/http-client
  Documentation: https://docs.grove.io/http_client
  Downloads: 12,432 (last 90 days)

  Dependencies:
    json ^1.1
    url_parser ^0.5

  Versions:
    1.2.0  (latest)  2026-03-10
    1.1.0            2026-02-15
    1.0.0            2026-01-20
```

**`sage publish`** — Publish to registry:

```bash
sage publish
sage publish --dry-run
sage publish --registry corp
```

**`sage yank`** — Yank a version:

```bash
sage yank http_client@1.2.0
sage yank http_client@1.2.0 --undo
```

**`sage login` / `sage logout`** — Manage authentication:

```bash
sage login
sage login --registry corp
sage logout
```

**`sage audit`** — Check for vulnerabilities:

```bash
sage audit
sage audit --fix  # Automatically update vulnerable deps
```

**`sage owner`** — Manage package ownership:

```bash
sage owner add http_client jane@example.com
sage owner remove http_client bob@example.com
sage owner list http_client
```

---

## 12. Migration Path

### 12.1 Compatibility

Existing projects continue to work unchanged:
- Git dependencies remain valid
- `grove.lock` version 1 is still readable
- No required changes to existing `grove.toml` files

### 12.2 Gradual Adoption

Projects can migrate dependencies one at a time:

```toml
[dependencies]
# Already migrated to registry
http_client = "^1.2"
json = "^1.1"

# Still using Git (unpublished package)
internal_tool = { git = "https://github.com/corp/internal-tool", tag = "v2.0.0" }
```

### 12.3 Lock File Upgrade

Running `sage install` on an existing project upgrades the lock file to version 2:
- Adds `source` field to each package
- Adds `checksum` field for registry packages
- Git packages retain their existing format

---

## 13. Implementation Plan

### Phase 1 — Semantic Versioning (1-2 weeks)
- [ ] Implement `Version` type with parsing and comparison
- [ ] Implement `VersionReq` type with constraint parsing (`^`, `~`, ranges)
- [ ] Update `DependencySpec` to support version requirements
- [ ] Write comprehensive version matching tests

### Phase 2 — Version Resolver (2-3 weeks)
- [ ] Implement backtracking resolver with conflict detection
- [ ] Implement conflict explanation generation
- [ ] Support mixed Git/registry/path dependencies
- [ ] Write resolver tests: simple cases, conflicts, complex graphs

### Phase 3 — Registry Index (1-2 weeks)
- [ ] Define index format (JSON-lines metadata files)
- [ ] Implement index parsing and caching
- [ ] Implement incremental index sync (Git fetch)
- [ ] Set up grove.io index repository structure

### Phase 4 — Package Distribution (1-2 weeks)
- [ ] Implement tarball creation for `sage publish`
- [ ] Implement tarball download and extraction
- [ ] Implement checksum generation and verification
- [ ] Update cache structure for registry packages

### Phase 5 — Registry API (2-3 weeks)
- [ ] Design REST API specification
- [ ] Implement API server (Rust, `axum` or similar)
- [ ] Implement authentication (API tokens)
- [ ] Implement package upload endpoint
- [ ] Implement package download endpoint
- [ ] Set up CDN for package storage

### Phase 6 — CLI Commands (1-2 weeks)
- [ ] Update `sage add` for registry dependencies
- [ ] Implement `sage search`
- [ ] Implement `sage info`
- [ ] Implement `sage publish`
- [ ] Implement `sage login` / `sage logout`
- [ ] Implement `sage yank`

### Phase 7 — Security Features (1-2 weeks)
- [ ] Implement package signing (Ed25519)
- [ ] Implement signature verification
- [ ] Set up advisory database
- [ ] Implement `sage audit`

### Phase 8 — Private Registries (1 week)
- [ ] Implement registry configuration
- [ ] Implement multi-registry resolution
- [ ] Create reference registry server Docker image

### Phase 9 — Web Interface (2-3 weeks)
- [ ] grove.io web frontend
- [ ] Package pages with documentation
- [ ] Search interface
- [ ] User dashboard
- [ ] API documentation

**Total estimated effort:** 12-18 weeks

---

## 14. New Error Codes

| Code | Name | Description |
|------|------|-------------|
| E036 | `VersionConflict` | No version satisfies all requirements for a package |
| E037 | `InvalidVersionReq` | Malformed version requirement string |
| E038 | `PackageNotInRegistry` | Package name not found in any configured registry |
| E039 | `VersionNotFound` | Requested version doesn't exist for package |
| E040 | `ChecksumMismatch` | Downloaded package checksum doesn't match index |
| E041 | `SignatureInvalid` | Package signature verification failed |
| E042 | `NotAuthenticated` | Operation requires `sage login` |
| E043 | `PublishFailed` | Registry rejected package upload |
| E044 | `YankedVersion` | Attempting to resolve a yanked version |
| E045 | `VulnerablePackage` | Package has known security vulnerability (with `--deny-vulnerable`) |

---

## 15. Open Questions

**Q1: Should we allow multiple major versions of the same package?**

Cargo does this (via name mangling). It solves version conflicts at the cost of complexity and potential runtime issues. Initial answer: no — require a single version, provide good conflict messages.

**Q2: What's the minimum package name length?**

Single-character names are valuable but prone to squatting. Options:
- Allow single-char, first-come-first-served
- Reserve single-char for official packages
- Require minimum 2-3 characters

**Q3: How do we handle the `sage_stdlib` standard library?**

Options:
- Distribute as a normal registry package (versioned independently)
- Bundle with the compiler (current approach)
- Auto-inject as an implicit dependency

**Q4: Should pre-release versions require explicit opt-in?**

Cargo requires `package = "1.0.0-alpha.1"` exactly — `^1.0` doesn't match pre-releases. This is probably correct behaviour.

**Q5: What's the governance model for grove.io?**

- Who can publish packages?
- How are disputes resolved?
- Who pays for infrastructure?

---

## 16. Alternatives Considered

### 16.1 Decentralised Registry (IPFS)

Store packages on IPFS, use content addressing for integrity.

**Rejected:** Adds significant complexity (IPFS node, pinning services) without clear benefit. Content addressing is achieved via checksums. Can revisit if centralisation becomes problematic.

### 16.2 Git-Only with Tags as Versions

Keep Git-only but add SemVer parsing for tags (`v1.2.0` → version 1.2.0).

**Rejected:** Doesn't solve discovery, requires cloning to read metadata, no checksums beyond Git's integrity.

### 16.3 Namespace/Scoped Packages (@user/package)

Like npm's scoped packages: `@sage-team/http_client`.

**Deferred:** Adds complexity to naming. Can be added later if namespace collisions become a problem. For now, first-come-first-served with squatting prevention.

### 16.4 Monorepo / Workspace Support

Cargo-style workspaces with shared dependencies.

**Deferred:** Valuable feature but orthogonal to registry support. Deserves its own RFC.

---

## 17. Out of Scope

- **Workspaces** — multiple packages in one repository (future RFC)
- **Build scripts** — pre/post install hooks (security implications need careful design)
- **Binary distribution** — pre-compiled packages (Sage compiles fast enough)
- **Mirroring** — automatic grove.io mirrors (CDN handles availability)
- **Package deprecation** — soft alternative to yanking (can add later)
- **Dependency overrides** — Cargo's `[patch]` section (path deps cover most use cases)
- **Platform-specific dependencies** — Sage is platform-agnostic
- **Optional features** — compile-time feature flags (would need Sage language support)

---

## Appendix A: Version Requirement Grammar

```ebnf
version_req  = simple_req | compound_req
simple_req   = [comparator] version
compound_req = simple_req ("," simple_req)*

comparator   = "^" | "~" | "=" | ">" | ">=" | "<" | "<="
version      = major ["." minor ["." patch ["-" prerelease] ["+" build]]]

major        = number
minor        = number
patch        = number
prerelease   = ident ("." ident)*
build        = ident ("." ident)*

number       = "0" | non_zero_digit digit*
ident        = letter (letter | digit | "-")*
```

---

## Appendix B: Registry API Specification

### Authentication

All authenticated endpoints require:
```
Authorization: Bearer <api_token>
```

### Endpoints

**GET /api/v1/packages?q={query}&page={n}**
Search packages.

**GET /api/v1/packages/{name}**
Get package metadata (all versions).

**GET /api/v1/packages/{name}/{version}**
Get specific version metadata.

**GET /api/v1/packages/{name}/{version}/download**
Download package tarball.

**PUT /api/v1/packages/new**
Publish a new package version.
```json
{
  "name": "http_client",
  "vers": "1.2.1",
  "deps": [...],
  "features": {},
  "readme": "..."
}
```
Body: multipart form with JSON metadata and tarball.

**DELETE /api/v1/packages/{name}/{version}/yank**
Yank a version.

**PUT /api/v1/packages/{name}/{version}/unyank**
Unyank a version.

**GET /api/v1/me**
Get current user info.

**GET /api/v1/packages/{name}/owners**
List package owners.

**PUT /api/v1/packages/{name}/owners**
Add owner.

**DELETE /api/v1/packages/{name}/owners/{user_id}**
Remove owner.

---

*A proper package ecosystem transforms a language from a curiosity into a platform. Ward anticipates many good packages.*
