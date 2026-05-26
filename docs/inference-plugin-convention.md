# Inference-Plugin Convention Spec

## What an inference plugin is

An inference plugin connects the Olympus agent runtime to an LLM provider. It implements a standardized interface contract that enables:

- Model invocation (sync + streaming)
- Tool/function call handling
- Context window management
- Token counting and cost accounting (drachma)

## Required interface contract

Every inference plugin MUST implement:

| Method | Signature | Description |
|---|---|---|
| `invoke` | `(prompt: PromptSequence, opts: InvokeOpts) → Response` | Single-turn invocation |
| `stream` | `(prompt: PromptSequence, opts: InvokeOpts) → Stream<Token>` | Streaming invocation |
| `count_tokens` | `(text: string) → integer` | Token estimation |
| `supports_tools` | `() → boolean` | Tool-use capability flag |

## Required atoms

An inference plugin MUST declare:

- One `capability-declaration` atom for each supported capability
- One `permission-scope` atom for the API credentials it needs
- One `interface-contract` atom describing the above interface

The canonical atoms for inference plugins are:

- `plugin-atoms://atoms/interface-contract/inference-plugin-v1` — this contract
- `plugin-atoms://atoms/capability-declaration/llm-inference` — base LLM generation capability

## Composing a plugin with this contract

To declare a concrete inference plugin, create a composition in `conventions/` that references these atoms:

```json
{
  "schema": "https://plugin-atoms.com/schemas/composition-v1.json",
  "type": "plugin",
  "id": "<provider>-inference-v1",
  "version": "1.0.0",
  "name": "<Provider> Inference Plugin",
  "description": "Inference plugin for <Provider>.",
  "references": {
    "interface_contract": {
      "ref": "plugin-atoms://atoms/interface-contract/inference-plugin-v1",
      "version": "1.0.0"
    },
    "capabilities": [
      {
        "ref": "plugin-atoms://atoms/capability-declaration/llm-inference",
        "version": "1.0.0"
      }
    ],
    "permission_scopes": [
      {
        "ref": "plugin-atoms://atoms/permission-scope/<provider>-api-key",
        "version": "1.0.0"
      }
    ]
  }
}
```

## Seed interface contract atom

`atoms/interface-contract/inference-plugin-v1.json` defines the canonical contract atom. See that file for the full atom definition.

## Drachma cost accounting

Inference plugins MUST report token usage after each `invoke` or `stream` call in the Olympus drachma ledger format so cost attribution flows correctly through the runtime. The `count_tokens` method provides pre-invocation estimation; post-invocation actuals come from the provider response.
