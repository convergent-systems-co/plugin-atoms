# plugin-atoms — Goals

> Plugin interface standards — not implementations — making plugins ecosystem-portable across Convergent Systems runtimes. USB-C for software plugins.

*This document is derived from `aish/ARCHITECTURE.md` (now `xdao/xdao/ARCHITECTURE.md` §The *-Atoms Catalogs). Sections marked **Generated** are pattern-based and are intended as a starting point for revision, not as decided plan.*

---

## What this catalog makes civilization-grade

npm, cargo, Homebrew already catalog plugin implementations. What doesn't exist is a catalog of plugin interface standards. Every runtime (aish, Olympus, future Universal Bus) invents its own plugin contract, so plugins don't port. A 'Claude provider' plugin gets rewritten three times.

By cataloging the primitives, `plugin-atoms` turns this domain from opaque-and-ephemeral to typed, versioned, composable, machine-readable, and open — the civilization-grade properties the ecosystem requires.

## What it catalogs

### Atom types

- **`interface-contract`** — JSON-RPC method signatures, expected behavior, error model.
- **`capability-declaration`** — What the plugin claims to do (inference, transform, source, sink).
- **`permission-scope`** — What the plugin needs access to (network, filesystem, secrets).
- **`lifecycle-hook`** — Start, shutdown, health-check, reload semantics.
- **`trust-primitive`** — Signing, attestation, sandbox-boundary declarations.

### Compositions: `conventions`

A convention composition assembles interface + capabilities + permissions + lifecycle + trust into a complete plugin interface standard. Specialized conventions (inference-plugin, transform-plugin, source-plugin, sink-plugin) layer atop the base.

### Rule types

- **`capability-grant`** — How capabilities translate to runtime permissions.
- **`semver-compatibility`** — Plugin and runtime compatibility windows.
- **`trust-chain-verification`** — Required signature verification for plugin load.

## Runtime consumers

- **aish** — Defines its plugins against this standard. The Olympus plugin, aish-inference-ollama, aish-inference-cloud all conform.
- **olympus** — Pantheon Modules conform (same standard, specialized for agentic context).
- **universal-bus** — Adapters conform.

## Status & priority

**Current status:** `proposed`

**Priority tier:** Tier 3 — Build when supporting runtimes mature

**Trigger / activation condition:** After observation. Build aish and Olympus first, observe the interface patterns, then extract plugin-atoms as the unified standard.

## Roadmap *(Generated — milestone shapes mirror aish's roadmap pattern; revise as actual work begins)*

### v0.1 — Bootstrap & spec acceptance

**Goal:** Observe interface patterns from aish v0.3 plugins and Olympus Pantheon Modules. Draft the catalog from the union.

**Success criterion:** aish v0.3 plugins and Olympus Pantheon Modules both validate against plugin-atoms conventions.

**Kill criterion:** aish and Olympus plugin contracts diverge enough that no unified standard is meaningful — pivot to per-runtime standards.

**Work:**

- [ ] Observe aish v0.3 plugin contract
- [ ] Observe Olympus Pantheon Module interface
- [ ] Draft base convention schema
- [ ] XAIP: plugin convention extraction

### v0.2 — Adoption & expansion

**Goal:** First write-once-run-anywhere plugin (inference provider).

**Work:**

- [ ] Inference-plugin convention spec
- [ ] Claude provider plugin written against the convention
- [ ] Verify across aish + Olympus + Universal Bus mock

### v1.0 — Operational

**Goal:** A plugin written once runs across every Convergent Systems runtime that supports its convention.

## Concrete atom example *(Generated — illustrative, not seed content)*

```yaml
conventions/inference-plugin/definition.yml
---
id: inference-plugin
type: composition
version: 1.0.0
interface: { ref: atoms/interface-contract/jsonrpc-stdio-inference }
capabilities:
  - { ref: atoms/capability-declaration/inference }
  - { ref: atoms/capability-declaration/streaming }
permissions:
  - { ref: atoms/permission-scope/network }
  - { ref: atoms/permission-scope/api-key-read }
lifecycle: { ref: atoms/lifecycle-hook/standard-rpc-server }
trust: { ref: atoms/trust-primitive/sigstore-signed }
```

## Adoption strategy *(Generated)*

Emergent — extracted from observation rather than designed upfront. Adoption follows the first cross-runtime plugin.

## Civilization-grade property checklist

Every catalog must satisfy these before v1.0. Failing any blocks a release.

| Property | Mechanism in this catalog |
|---|---|
| Typed | JSON Schema in `schemas/` validates every atom, composition, rule |
| Versioned | Every atom has a semver `version` field; compositions reference atoms by version-pinned ID |
| Machine-readable | `exports/catalog.json` published on every release |
| Composable | Compositions reference atoms by ID; CI verifies references resolve and no circular dependencies |
| Open | Apache-2.0 licensed; LICENSE file present |
| Durable | No external dependencies for primary content (no remote image URLs, no vendor APIs in the hot path) |

## Related

- **Spec:** [atoms-spec](https://github.com/convergent-systems-co/atoms-spec) — the canonical structure every catalog conforms to
- **Tools:** [atoms-tools](https://github.com/convergent-systems-co/atoms-tools) — CLI for validate / export / bootstrap / resolve
- **Federation:** [xdao](https://github.com/convergent-systems-co/xdao) — ecosystem directory and discovery
- **Umbrella:** [atoms](https://github.com/convergent-systems-co/atoms) — every catalog as a git submodule
- **Manifest:** [`ATOMS.yml`](./ATOMS.yml) — this catalog's machine-readable manifest
- **Standard:** [`README.md`](./README.md) — catalog overview and contribution flow
