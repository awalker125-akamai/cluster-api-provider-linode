# Plan: Support `firewallRef` on `linodeInterfaces`

## Problem Statement
When a `LinodeMachineTemplate` uses `linodeInterfaces`, users cannot currently set a firewall reference per interface. The interface schema supports `firewallID` only, which forces users to know numeric firewall IDs ahead of time.

This creates friction in declarative workflows, and it conflicts with common CAPL usage where users prefer object references (`firewallRef`) over hard-coded IDs.

## Current Temporary Behavior (Implemented)
To unblock current workflows before `firewallRef` support is added, CAPL now inherits interface-level firewall settings from existing `linodeInterfaces` entries when it auto-generates VPC interfaces.

What this means in practice:
- If a user sets `firewallID: -1` on their declared `linodeInterfaces` entry, generated VPC interfaces inherit `-1` too.
- This avoids API failures that require an explicit default/interface firewall decision.

This is an interim compatibility behavior and may be removed or simplified once native `firewallRef` support exists at interface level.

## Goals
- Allow interface-level firewall attachment through Kubernetes references.
- Keep existing `firewallID` behavior unchanged.
- Make precedence deterministic when both ID and ref are provided.
- Avoid instance-level firewall attachment when interfaces are used.

## Non-Goals
- Changing Linode API behavior.
- Reworking `LinodeFirewall` reconciliation lifecycle.
- Requiring migration for existing manifests that use `firewallID`.

## Proposed API Changes
Update `LinodeInterfaceCreateOptions` (v1alpha2) to add:
- `firewallRef *corev1.ObjectReference 'json:"firewallRef,omitempty"'`
- Optional legacy alias if needed for compatibility conventions: `firewall_ref`.

### Precedence Rules
At interface level:
1. `firewallID` takes precedence over `firewallRef`.
2. If `firewallID` is unset and `firewallRef` is set, resolve ref to ID at reconcile time.
3. If neither is set, leave interface firewall unset.

At machine level (existing fields):
1. If `linodeInterfaces` are in use, do not set top-level `InstanceCreateOptions.FirewallID`.
2. If machine-level firewall fields are present while using `linodeInterfaces`, either:
   - treat as default interface firewall (apply to all interfaces lacking explicit firewall), or
   - reject via webhook to avoid ambiguity.

Recommendation: reject mixed machine-level firewall with `linodeInterfaces` in initial implementation for clear semantics.

## Controller Changes
Files primarily impacted:
- `internal/controller/linodemachine_controller_helpers.go`
- `api/v1alpha2/linodemachine_types.go`
- CRD manifests under `config/crd/bases/...`

### 1. Build Path for Interfaces
In `constructLinodeInterfaceCreateOpts(...)`:
- Resolve interface-level `firewallRef` to an ID and set `LinodeInterfaceCreateOptions.FirewallID`.
- Keep current ID-first behavior.

Because this function currently lacks scope/client context, ref resolution should happen in a later stage where `machineScope` is available, or the function signature should be refactored to accept a resolver.

### 2. Firewall Configuration Path
In `configureFirewall(...)`:
- Current behavior sets both top-level `createConfig.FirewallID` and interface firewall IDs.
- New behavior for `linodeInterfaces`:
  - do not set `createConfig.FirewallID`.
  - set `FirewallID` only on each interface.

For legacy interfaces path, keep existing behavior unless API constraints require change.

### 3. Shared Resolver
Introduce helper:
- `resolveFirewallRef(ctx, machineScope, ref) (int, error)`

Use this for:
- machine-level firewall resolution
- interface-level firewallRef resolution

## Validation and Webhook Changes
Add webhook validation for `LinodeMachine`/`LinodeMachineTemplate`:
- If interface entry has both `firewallID` and `firewallRef`, allow with documented precedence.
- If `linodeInterfaces` are used and machine-level `firewallRef` or `firewallID` is set, enforce chosen policy (recommended: reject initially).
- Ensure namespace defaulting behavior for refs is consistent with existing CAPL ref handling.

## Backward Compatibility
- Existing manifests with interface `firewallID` continue to work.
- Existing manifests with machine-level firewall + legacy interfaces remain unchanged.
- New `firewallRef` field is additive.

## Test Plan
### Unit Tests
Add/extend tests in:
- `internal/controller/linodemachine_controller_helpers_test.go`
- webhook validation tests

Scenarios:
- interface `firewallID` only
- interface `firewallRef` only
- both interface ID and ref (ID wins)
- `linodeInterfaces` + machine-level firewall (reject or mapped default per chosen policy)
- legacy interfaces unaffected

### Integration/E2E
- Create cluster with `linodeInterfaces` and interface `firewallRef`.
- Verify create payload has interface `FirewallID` populated.
- Verify no top-level instance firewall attach is attempted.
- Validate reconciliation errors are clear for bad refs.

## Rollout Plan
1. Add API field and regenerate CRDs.
2. Implement controller resolution + behavior split for interface generation.
3. Add webhook validation + tests.
4. Add docs and example template updates.
5. Ship behind release notes callout.

## Cleanup Considerations After `firewallRef`
When interface-level `firewallRef` is implemented, revisit the temporary inheritance behavior and decide whether to:
- keep inheritance as a convenience default, or
- remove inheritance and require explicit per-interface intent (`firewallID` or `firewallRef`).

Document the final decision in the release notes to avoid ambiguous firewall behavior for generated interfaces.

## Documentation Updates
- `docs/src/topics/firewalling.md`
- template comments in `templates/infra/linodeMachineTemplate.yaml`
- generated reference docs for new field

## Open Questions
1. Should machine-level firewall act as a default for all `linodeInterfaces` (inheritance), or should it be disallowed to reduce ambiguity?
2. Do we need a legacy alias `firewall_ref` for consistency with prior API compatibility patterns?
3. Should firewall ref be allowed on all interface types (`public`, `vpc`, `vlan`) equally, or constrained by provider/API behavior?
