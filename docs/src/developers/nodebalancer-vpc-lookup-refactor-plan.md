# Plan: Consolidate NodeBalancer VPC Subnet Lookup Helpers

## Problem Statement
`cloud/services/loadbalancers.go` currently contains overlapping VPC subnet lookup logic in two helpers:
- `getSubnetID(...)`
- `getSubnetIDAndCIDR(...)`

Both resolve VPC by `VPCID` or `VPCRef`, apply `subnetName` selection, and validate subnet IDs. This duplication increases maintenance cost and creates drift risk when one path is updated without the other.

## Why Defer This Refactor
The recent behavioral fix (derive backend primary CIDR from VPC subnet instead of `NodeBalancerBackendIPv4Range`) is already working and tested. A follow-up refactor should be done deliberately to avoid destabilizing load balancer creation and backend registration paths.

## Goals
- Remove duplicated VPC/subnet lookup logic in load balancer services.
- Keep behavior unchanged for:
  - VPCID flows
  - VPCRef flows
  - subnetName selection
  - error messaging semantics where practical
- Preserve vpcless behavior and private-IP fallback logic.

## Non-Goals
- Changing the semantics of `ShouldUseVPC(...)`.
- Changing NodeBalancer API payload behavior beyond lookup consolidation.
- Refactoring controller-layer VPC lookup helpers in this same change.

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

### Legacy Helper Handling
- Keep `getSubnetID(...)` and `getSubnetIDAndCIDR(...)` temporarily as thin wrappers (optional short transition), then delete in same PR once tests are green.
- Do not export new helper.

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

## Rollout Steps
1. Add `resolvedSubnet` + `resolvePrimarySubnet(...)` in `cloud/services/loadbalancers.go`.
2. Migrate `EnsureNodeBalancer(...)` and `AddNodesToNB(...)` to new helper.
3. Remove duplicated logic (`getSubnetID(...)`, `getSubnetIDAndCIDR(...)`) or reduce to temporary wrappers.
4. Update/add tests as listed above.
5. Run `go test ./cloud/services -count=1`.
6. Run broader package tests if needed by CI policy.

## Risk Areas
- Subtle mismatch in error text that existing tests assert with exact strings.
- Different treatment of empty/invalid `IPv4` values between VPCID and VPCRef sources.
- Accidentally invoking VPC lookup in vpcless scenarios.

## Success Criteria
- Single source of truth for subnet resolution in load balancer service code.
- No functional behavior changes in existing VPC-enabled or vpcless flows.
- `go test ./cloud/services -count=1` remains green.
