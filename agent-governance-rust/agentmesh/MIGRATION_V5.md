# Rust framework policy migration

`FrameworkGovernanceAdapter` now consumes native Agent Control Specification
objects. The removed local framework policy and pattern types are not
translated at runtime.

```rust
use agentmesh::{
    AgentControl, FrameworkGovernanceAdapter, FrameworkKind, Manifest,
};

let manifest = Manifest::from_path("manifest.yaml")?;
let control = AgentControl::from_manifest(manifest)?;
let adapter = FrameworkGovernanceAdapter::new(
    FrameworkKind::Tower,
    MyHook,
    control,
);
```

Move tool catalogs, intervention-point bindings, budgets, approval rules, and
content policies into the manifest. Keep framework-only drift and checkpoint
settings in `FrameworkHostConfig`.

Legacy-shaped YAML is rejected with `runtime_error:manifest_invalid`. The Rust
SDK does not guess at or partially translate removed fields.

## Budgets are host-reported

`FrameworkGovernanceAdapter` counts tool calls itself and tracks elapsed time
from construction, but it mediates requests without ever seeing a model
response, so it cannot observe tokens or cost on its own.

Call `record_usage` after each model call, or manifest budget rules keyed on
`token_count` or `cost_usd` will evaluate against zero and never fire:

```rust
let adapter = FrameworkGovernanceAdapter::new(FrameworkKind::Actix, hook, control);
// ... after a model call returns usage ...
adapter.record_usage(response.total_tokens, response.cost_usd);
```

`tool_call_count` and `elapsed_seconds` need no host involvement.

## Removed: `FrameworkExecutionResult::matched_patterns`

The field carried the regex rule ids matched by the v4 pattern engine. ACS
returns a verdict with a reason rather than a match list, so the field had no
source and would have returned an empty vector on every call. It is removed
rather than left in place, so code that read it fails to compile instead of
silently seeing no matches. Read `decision` and the emitted `events` instead.

## `with_host_config` returns `Result`

It previously panicked on a `FrameworkHostConfig` outside the valid range. It
now returns `RuntimeError::ManifestInvalid`. `new` is unchanged and stays
infallible, because the default config is valid by construction.
