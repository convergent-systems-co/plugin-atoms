# XAIP: Plugin Convention Extraction

**Atom type:** `plugin-convention`
**Version:** 0.1
**Audience:** Plugin authors, platform integrators, convention maintainers

---

## 1. Purpose

Plugin convention extraction is the process of deriving a `plugin-convention` atom from an existing plugin implementation. The convention captures the interface contract, permission scopes, lifecycle hooks, and trust requirements in a machine-readable form that other plugins can reference, implement, or extend. A convention is not a plugin; it is the schema that governs a family of plugins.

---

## 2. What Is a Plugin Convention

A plugin convention atom describes:

- **Interface contract** — the methods a plugin implementing this convention must expose, their signatures, and their semantics.
- **Permission scopes** — the capabilities the plugin is permitted to exercise on the host system.
- **Lifecycle hooks** — the lifecycle events the host will invoke on the plugin and the expected response contract for each.
- **Trust primitive** — the trust level the plugin requests from the host and the attestation it must provide.

A convention atom serves as both documentation and a machine-checkable contract. A plugin that claims to implement a convention can be validated against it at load time.

---

## 3. Extraction Procedure

Given an existing plugin implementation, derive its convention atom by following these steps in order.

### Step 1: Extract the interface contract

Read the plugin's public interface (exported class, protocol, trait, or Go interface) and identify every method that a caller invokes on the plugin.

For each method, record:

| Field | Description |
|---|---|
| `method_name` | Stable identifier (no renaming without a new convention version) |
| `input_schema` | JSON Schema for the input payload |
| `output_schema` | JSON Schema for the return value |
| `required` | `true` if every implementing plugin must provide this method |
| `idempotent` | `true` if calling the method twice with the same input produces the same result |
| `description` | One sentence: what the method does, not how |

Map these to an `interface-contract` atom:

```json
{
  "atom_type": "interface-contract",
  "contract_id": "olympus/inference-plugin",
  "version": "1",
  "methods": [
    {
      "method_name": "infer",
      "input_schema": { "$ref": "schemas/infer-request.json" },
      "output_schema": { "$ref": "schemas/infer-response.json" },
      "required": true,
      "idempotent": false,
      "description": "Runs inference on the provided input and returns a structured result."
    },
    {
      "method_name": "health",
      "input_schema": {},
      "output_schema": { "$ref": "schemas/health-response.json" },
      "required": true,
      "idempotent": true,
      "description": "Returns the plugin's current operational health status."
    }
  ]
}
```

### Step 2: Extract permission scopes

Read the plugin's permission declarations (manifest, capability list, requested OS permissions, network access declarations). For each distinct capability, map it to a `permission-scope` atom.

```json
{
  "atom_type": "permission-scope",
  "scope_id": "olympus/inference-plugin/permissions",
  "version": "1",
  "scopes": [
    {
      "scope_name": "model.read",
      "description": "Read access to model weights and configuration files.",
      "resource_pattern": "models/**",
      "access": "read"
    },
    {
      "scope_name": "network.outbound",
      "description": "Outbound HTTP/S connections to model provider APIs.",
      "resource_pattern": "https://*.openai.com/**",
      "access": "network-outbound"
    }
  ]
}
```

Extraction rule: if the original plugin requests a broad permission (e.g., `filesystem.all`), decompose it into the narrowest scopes that the plugin's actual code exercises. Do not carry forward over-broad grants.

### Step 3: Extract lifecycle hooks

Read the plugin's initialization, shutdown, and event-handling code. Identify every lifecycle event the host invokes on the plugin.

```json
{
  "atom_type": "lifecycle-hook",
  "hook_set_id": "olympus/inference-plugin/lifecycle",
  "version": "1",
  "hooks": [
    {
      "event": "on_load",
      "required": true,
      "timeout_ms": 5000,
      "description": "Called once when the host loads the plugin. Plugin performs initialization here.",
      "failure_behavior": "abort-host-startup"
    },
    {
      "event": "on_unload",
      "required": true,
      "timeout_ms": 2000,
      "description": "Called when the host is shutting down. Plugin must release all resources.",
      "failure_behavior": "log-and-continue"
    },
    {
      "event": "on_config_change",
      "required": false,
      "timeout_ms": 1000,
      "description": "Called when the host configuration changes. Plugin may reload affected settings.",
      "failure_behavior": "log-and-continue"
    }
  ]
}
```

`failure_behavior` values: `abort-host-startup`, `abort-request`, `log-and-continue`, `quarantine-plugin`.

### Step 4: Declare the trust primitive

Read the plugin's trust requirements — does it require access to secrets, elevated permissions, or cross-plugin IPC? Map these to a `trust-primitive` atom:

```json
{
  "atom_type": "trust-primitive",
  "primitive_id": "olympus/inference-plugin/trust",
  "version": "1",
  "required_trust_level": "signed",
  "attestation": {
    "method": "code-signature",
    "authority": "https://key-atoms.convergent-systems.co/atoms/olympus-plugin-ca/v1/atom.json"
  },
  "sandbox_profile": "network-restricted",
  "secret_access": false,
  "cross_plugin_ipc": false
}
```

`required_trust_level` maps to the same ladder as identity-atoms (`anonymous` → `authenticated` → `signed` → `verified`).

`sandbox_profile` values: `none`, `filesystem-readonly`, `network-restricted`, `full-isolation`.

### Step 5: Assemble the convention atom

Combine the four extracted atoms into a `plugin-convention` composition:

```json
{
  "atom_type": "plugin-convention",
  "convention_id": "olympus/inference-plugin",
  "version": "1",
  "display_name": "Olympus Inference Plugin Convention",
  "interface_contract_ref": "interface-contract:olympus/inference-plugin/v1",
  "permission_scope_ref": "permission-scope:olympus/inference-plugin/permissions/v1",
  "lifecycle_hook_ref": "lifecycle-hook:olympus/inference-plugin/lifecycle/v1",
  "trust_primitive_ref": "trust-primitive:olympus/inference-plugin/trust/v1",
  "metadata": {
    "catalog_ref": "plugin-atoms.convergent-systems.co",
    "source_implementation": "https://github.com/convergent-systems-co/olympus/tree/main/plugins/inference"
  }
}
```

---

## 4. The Trust Primitive Atom

The `trust-primitive` atom is the single most important output of convention extraction. It declares the trust level a plugin requires from the host before it is allowed to execute.

### 4.1 Fields

| Field | Type | Description |
|---|---|---|
| `required_trust_level` | enum | Minimum trust level the plugin's code signature must reach before the host loads it |
| `attestation.method` | enum | How the host verifies the plugin's identity: `code-signature`, `hash-pin`, `none` |
| `attestation.authority` | URI | The key-atoms CA or key-cert ref used to verify the signature |
| `sandbox_profile` | enum | The isolation sandbox the host applies to this plugin's process |
| `secret_access` | boolean | Whether the plugin is permitted to read secrets from the host's vault |
| `cross_plugin_ipc` | boolean | Whether the plugin is permitted to call other loaded plugins directly |

### 4.2 Trust-level enforcement

The host plugin loader enforces the `required_trust_level` before `on_load` fires:

```
1. Compute the plugin binary's code-signature.
2. Verify the signature against attestation.authority.
3. Determine the trust_level the signature supports.
4. If trust_level < required_trust_level: refuse to load. Emit a LoadRejected event.
5. If trust_level >= required_trust_level: apply sandbox_profile, then call on_load.
```

---

## 5. Worked Example: Olympus Inference Plugin

The Olympus inference plugin provides a gRPC service that accepts an `InferRequest` proto and returns an `InferResponse`. It reads model files from disk, makes outbound HTTP calls to a provider API, and performs no IPC with other plugins.

### 5.1 Extracted interface contract

Two required methods: `infer` (non-idempotent, streaming output supported), `health` (idempotent, returns `{status: "ok"|"degraded"|"unavailable", latency_p99_ms: float}`).

### 5.2 Extracted permission scopes

- `model.read` — reads from `./models/` directory.
- `network.outbound` — HTTPS only, to `api.openai.com`.

No filesystem write. No inbound network. No secret access declared.

### 5.3 Extracted lifecycle hooks

- `on_load` — required, 5s timeout, failure aborts host startup (model loading failure is unrecoverable).
- `on_unload` — required, 2s timeout, failure is logged-and-continued (resources released best-effort).
- No `on_config_change` (model path is fixed at load time).

### 5.4 Extracted trust primitive

- `required_trust_level: signed` — plugin binary must carry a valid code signature from the Olympus plugin CA.
- `sandbox_profile: network-restricted` — allows only declared outbound endpoints.
- `secret_access: false` — no access to host vault.
- `cross_plugin_ipc: false` — inference is self-contained.

### 5.5 Assembled convention atom (abbreviated)

```json
{
  "atom_type": "plugin-convention",
  "convention_id": "olympus/inference-plugin",
  "version": "1",
  "display_name": "Olympus Inference Plugin Convention",
  "interface_contract_ref": "interface-contract:olympus/inference-plugin/v1",
  "permission_scope_ref": "permission-scope:olympus/inference-plugin/permissions/v1",
  "lifecycle_hook_ref": "lifecycle-hook:olympus/inference-plugin/lifecycle/v1",
  "trust_primitive_ref": "trust-primitive:olympus/inference-plugin/trust/v1"
}
```

---

## 6. Catalog Conventions

| Convention | Value |
|---|---|
| Convention atom path | `atoms/conventions/<slug>/v<version>/atom.json` |
| Interface-contract atom path | `atoms/interface-contracts/<slug>/v<version>/atom.json` |
| Permission-scope atom path | `atoms/permission-scopes/<slug>/v<version>/atom.json` |
| Lifecycle-hook atom path | `atoms/lifecycle-hooks/<slug>/v<version>/atom.json` |
| Trust-primitive atom path | `atoms/trust-primitives/<slug>/v<version>/atom.json` |

---

## 7. Related Atoms and Docs

- `interface-contract` atom — method signatures and semantics
- `permission-scope` atom — capability grants at narrowest-sufficient granularity
- `lifecycle-hook` atom — host-to-plugin event contracts and failure behaviors
- `trust-primitive` atom — trust level, attestation authority, and sandbox profile
- key-atoms: key-cert and CA atoms — the attestation authorities referenced in trust primitives
- identity-atoms: `xaip-identity-composition.md` — trust level ladder shared with plugin trust requirements
