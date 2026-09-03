# Plan: Consolidate NodeBalancer VPC Subnet Lookup Helpers

## Problem Statement
`cloud/services/loadbalancers.go` currently contains overlapping VPC subnet lookup logic in two helpers:
- `getSubnetID(...)`
- `getSubnetIDAndCIDR(...)`

Both resolve VPC by `VPCID` or `VPCRef`, apply `subnetName` selection, and validate subnet IDs. This duplication increases maintenance cost and creates drift risk when one path is updated without the other.

The recent controller-side fix also clarified the key contract: when VPC mode is active, the canonical backend membership check must use the actual primary subnet CIDR for the VPC, not `NodeBalancerBackendIPv4Range`. The workaround in `getIPPortCombo(...)` prefers the resolved VPC subnet and only falls back to `NodeBalancerBackendIPv4Range` when VPC lookup cannot resolve a value.

## Why Defer This Refactor
The recent behavioral fix is already in place and intentionally minimal: it aligns the controller cleanup logic with the real VPC primary subnet selection while preserving legacy compatibility when VPC lookup is unavailable. A follow-up refactor should be done deliberately to avoid destabilizing load balancer creation and backend registration paths.

## Goals
- Remove duplicated VPC/subnet lookup logic in load balancer services.
- Preserve the controller-side contract that desired backend membership is derived from the actual VPC primary subnet CIDR when VPC is in use.
- Keep behavior unchanged for:
  - VPCID flows
  - VPCRef flows
  - subnetName selection
  - error messaging semantics where practical
- Preserve vpcless behavior and private-IP fallback logic.

## Non-Goals
- Changing the semantics of `ShouldUseVPC(...)`.
- Changing NodeBalancer API payload behavior beyond lookup consolidation.
- Replacing the controller helper fallback pattern in the same refactor; the helper is intentionally conservative and should retain its compatibility fallback until broader validation is complete.

## Proposed Design
Create one internal resolver for `cloud/services` that returns a normalized subnet selection result used by both call sites.

### New Internal Type
```go
type resolvedSubnet struct {
    ID   int
    IPv4 string
}
```

### New Internal Helper
```go
func resolvePrimarySubnet(ctx context.Context, clusterScope *scope.ClusterScope, logger logr.Logger) (resolvedSubnet, error)
```

Behavior:
- If `Spec.VPCID` is set:
  - fetch VPC from Linode API
  - choose subnet by `Spec.Network.SubnetName` (or first)
  - return `ID` + `IPv4`
- Else if `Spec.VPCRef` is set:
  - fetch `LinodeVPC` object
  - choose subnet by `Spec.Network.SubnetName` (or first)
  - return `SubnetID` + `IPv4`
- Else:
  - return explicit error (same intent as today)
- Validate selected subnet ID is non-zero.

### Call Site Migration
- `EnsureNodeBalancer(...)`
  - replace `getSubnetID(...)` with `resolvePrimarySubnet(...).ID`
- `AddNodesToNB(...)`
  - replace `getSubnetIDAndCIDR(...)` with `resolvePrimarySubnet(...).ID` and `.IPv4`

The controller-layer helper should remain intentionally separate: `getIPPortCombo(...)` already resolves the VPC primary CIDR via `resolvePrimaryVPCSubnetCIDR(...)`, and that behavior is the source of truth for desired backend membership. The service-layer refactor should not override that decision; it should simply consolidate duplicate subnet lookup logic into one helper.

### Legacy Helper Handling
- Keep `getSubnetID(...)` and `getSubnetIDAndCIDR(...)` temporarily as thin wrappers (optional short transition), then delete in same PR once tests are green.
- Do not export new helper.
- Do not fold controller helper logic into the service resolver; controller fallback semantics are intentionally different and should remain explicit.

## Compatibility and vpcless Safety
This refactor must keep vpcless behavior intact:
- `ShouldUseVPC(...)` remains the gate for VPC-specific paths.
- If no VPC is configured (vpcless flavor), code must continue using private-IP fallback in `AddNodesToNB(...)`.
- No additional VPC lookups should occur when `ShouldUseVPC(...)` is false.

## Test Plan
### Existing Tests To Keep Passing
- `cloud/services/loadbalancers_test.go`:
  - `TestAddNodeToNBConditions`
  - `TestAddNodeToNBFullWorkflow`
  - `TestAddNodeToNBWithSecondaryVPC`
  - `TestGetSubnetID`
  - `TestGetSubnetIDWithVPCID`
  - any `EnsureNodeBalancer(...)` tests covering VPC creation paths
- `internal/controller/linodecluster_controller_helpers_test.go`:
  - `TestGetIPPortCombo`
  - `TestGetIPPortComboWithSecondaryVPC`
  - stale-node cleanup regression coverage for refreshed NodeBalancer state after adds

### New/Adjusted Unit Tests
- Add direct tests for `resolvePrimarySubnet(...)`:
  - VPCID + first subnet
  - VPCID + subnetName match
  - VPCID + no subnets
  - VPCRef + first subnet
  - VPCRef + subnetName match
  - VPCRef fetch failure
  - neither VPCID nor VPCRef
  - selected subnet ID = 0
- Add one explicit vpcless assertion test:
  - with `EnableVPCBackends=true` and no VPC refs/IDs, ensure `ShouldUseVPC(...)` is false and `AddNodesToNB(...)` uses private fallback behavior.
- Add or retain a regression test that confirms `getIPPortCombo(...)` prefers the actual VPC primary subnet CIDR and does not use `NodeBalancerBackendIPv4Range` when VPC resolution succeeds.

## Rollout Steps
1. Add `resolvedSubnet` + `resolvePrimarySubnet(...)` in `cloud/services/loadbalancers.go`.
2. Migrate `EnsureNodeBalancer(...)` and `AddNodesToNB(...)` to the new helper.
3. Remove duplicated logic (`getSubnetID(...)`, `getSubnetIDAndCIDR(...)`) or reduce to temporary wrappers.
4. Keep controller helper compatibility fallback explicit and covered by regression tests.
5. Update/add tests as listed above.
6. Run `go test ./cloud/services -count=1`.
7. Run `go test ./internal/controller -run 'TestGetIPPortCombo|TestGetIPPortComboWithSecondaryVPC' -count=1` and broader package tests if needed by CI policy.

## Risk Areas
- Subtle mismatch in error text that existing tests assert with exact strings.
- Different treatment of empty/invalid `IPv4` values between VPCID and VPCRef sources.
- Accidentally invoking VPC lookup in vpcless scenarios.

## Success Criteria
- Single source of truth for subnet resolution in load balancer service code.
- No functional behavior changes in existing VPC-enabled or vpcless flows.
- Controller cleanup logic continues to derive desired backend membership from the actual VPC primary subnet CIDR when VPC is configured.
- `go test ./cloud/services -count=1` remains green, and the controller regression tests covering VPC membership remain green.
