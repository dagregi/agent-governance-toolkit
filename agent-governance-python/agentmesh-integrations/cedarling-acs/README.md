# cedarling-acs

Cedarling policy dispatcher for the Agent Governance Toolkit ACS v5 runtime.

## Installation

```bash
pip install 'agent-governance-toolkit-integrations[cedarling]'
```

The `cedarling` extra pulls in `cedarling-python`. Without it, constructing the
dispatcher raises `ImportError` with install guidance.

## Quick start

```python
from agent_control_specification import AgentControl
from cedarling_acs import CedarlingPolicyDispatcher, CedarlingConfig

dispatcher = CedarlingPolicyDispatcher.from_bootstrap(
    {"CEDARLING_POLICY_STORE_LOCAL_FN": "policy-store.json"},
)
runtime = AgentControl.from_path("manifest.yaml", policy_dispatcher=dispatcher)
```

Bind a `custom` policy to the intervention points Cedarling should govern:

```yaml
agent_control_specification_version: 0.3.0-alpha-agt
metadata:
  name: cedarling_governed
policies:
  cedarling:
    type: custom
    adapter: cedarling
intervention_points:
  pre_tool_call:
    policy_target: $.tool_call.args
    policy_target_kind: tool_args
    tool_name_from: $.tool_call.name
    policy:
      id: cedarling
```

Passing no `policy_dispatcher` keeps the bundled `cedar` dispatcher. Passing this
one replaces it for `custom` policies bound to the `cedarling` adapter.

## Request mapping

The dispatcher receives the ACS final policy input (spec section 7) under
`invocation["input"]` and builds a Cedar request from it. The defaults mirror the
bundled `cedar` dispatcher (spec section 12.4):

| Cedar field | Source |
| --- | --- |
| principal | `Agent::"<snapshot.envelope.agent.id>"` |
| action | `Action::"<intervention_point>"` |
| resource | `Tool::"<tool.name>"` at tool points, else `PolicyTarget::"<policy_target.kind>"` |
| context | snapshot minus `envelope`, plus each annotation keyed `annotations.<name>` |

`CedarlingConfig` overrides the entity types, an optional Cedar `namespace`, the
`auth_type` (`unsigned` or `multi-issuer`), and where multi-issuer tokens are read
from in the snapshot (`token_paths`, default `snapshot.tokens` then
`snapshot.envelope.agent.tokens`).

## Verdicts

| Cedarling | Verdict |
| --- | --- |
| permit | `{"decision": "allow"}` |
| forbid | `{"decision": "deny", "reason": "<first contributing policy id>"}` |
| `AuthorizeError` | `{"decision": "deny", "reason": "cedarling_authorization_error"}` |
| any other error, missing input, missing `cedarling-python` | `{"decision": "deny", ...}` |

Every failure path denies. Reasons never use the reserved `runtime_error:` prefix.

## Migration from the v4 backend

The removed `cedarling-agentmesh` backend registered a `CedarlingBackend` with
`BackendRegistry`. Replace that with constructing this dispatcher and passing it
as `policy_dispatcher=`. Cedarling policy stores and `.cedar` files carry over
unchanged. See `8149ebf9^:agent-governance-python/agentmesh-integrations/cedarling-agentmesh/`
for the old code.
