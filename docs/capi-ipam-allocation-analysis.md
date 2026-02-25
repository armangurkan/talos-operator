# CAPI IPAM Provider In-Cluster: Allocation Logic Deep Analysis

Source: `kubernetes-sigs/cluster-api-ipam-provider-in-cluster` (main branch)

---

## 1. The Allocation Algorithm

### Core Function: `FindFreeAddress`

**File**: `internal/poolutil/pool.go`

```go
func FindFreeAddress(poolIPSet *netipx.IPSet, inUseIPSet *netipx.IPSet) (netip.Addr, error) {
    for _, iprange := range poolIPSet.Ranges() {
        ip := iprange.From()
        for {
            if !inUseIPSet.Contains(ip) {
                return ip, nil
            }
            if ip == iprange.To() {
                break
            }
            ip = ip.Next()
        }
    }
    return netip.Addr{}, errors.New("no address available")
}
```

**Algorithm**: **Sequential scan, lowest-first**. It iterates through every IP range
in the pool in order, starting from the lowest IP (`From()`), and walks forward one
IP at a time (`ip.Next()`). The first IP not present in the `inUseIPSet` is returned.
There is no randomization, no hashing, no bitmap. It is a simple linear scan.

### Data Structure for Tracking Allocated IPs

There is **no persistent bitmap or allocation table**. The "allocated set" is
reconstructed on every reconciliation by querying Kubernetes:

```go
// In EnsureAddress (internal/controllers/ipaddressclaim.go):
addressesInUse, err := poolutil.ListAddressesInUse(ctx, h.Client, h.pool.GetNamespace(), h.claim.Spec.PoolRef)
```

`ListAddressesInUse` queries the Kubernetes API for all `IPAddress` objects that
reference the given pool, using an **indexed field**:

```go
func ListAddressesInUse(ctx context.Context, c client.Reader, namespace string,
    poolRef ipamv1.IPPoolReference) ([]ipamv1.IPAddress, error) {
    addresses := &ipamv1.IPAddressList{}
    err := c.List(ctx, addresses,
        client.MatchingFields{
            index.IPAddressPoolRefCombinedField: index.IPPoolRefValue(poolRef),
        },
        client.InNamespace(namespace),
    )
    // filter to only ipam.cluster.x-k8s.io group
    addr := []ipamv1.IPAddress{}
    for _, a := range addresses.Items {
        gv, _ := schema.ParseGroupVersion(a.APIVersion)
        if gv.Group != "ipam.cluster.x-k8s.io" { continue }
        addr = append(addr, a)
    }
    return addr, err
}
```

The index is registered at startup:

```go
// internal/index/index.go
func IPPoolRefValue(ref ipamv1.IPPoolReference) string {
    return fmt.Sprintf("%s%s", ref.Kind, ref.Name)
}
```

So the "database" of allocated IPs is **the set of IPAddress custom resources in etcd**.
Each reconciliation:
1. Lists all `IPAddress` objects for the pool (via field index)
2. Converts their `.Spec.Address` strings into an `netipx.IPSet`
3. Converts the pool spec into another `netipx.IPSet`
4. Calls `FindFreeAddress(poolIPSet, inUseIPSet)` to find the next gap

### Full Allocation Flow (EnsureAddress)

**File**: `internal/controllers/ipaddressclaim.go`

```go
func (h *IPAddressClaimHandler) EnsureAddress(ctx context.Context, address *ipamv1.IPAddress) (*ctrl.Result, error) {
    addressesInUse, err := poolutil.ListAddressesInUse(ctx, h.Client, h.pool.GetNamespace(), h.claim.Spec.PoolRef)
    // ...
    allocated := slices.ContainsFunc(addressesInUse, func(a ipamv1.IPAddress) bool {
        return a.Name == address.Name && a.Namespace == address.Namespace
    })

    if !allocated {
        poolSpec := h.pool.PoolSpec()
        inUseIPSet, err := poolutil.AddressesToIPSet(buildAddressList(addressesInUse, poolSpec.Gateway))
        poolIPSet, err := poolutil.PoolSpecToIPSet(poolSpec)
        freeIP, err := poolutil.FindFreeAddress(poolIPSet, inUseIPSet)

        address.Spec.Address = freeIP.String()
        address.Spec.Gateway = poolSpec.Gateway
        address.Spec.Prefix = ptr.To(int32(poolSpec.Prefix))
    }
    return nil, nil
}
```

Key detail: the gateway is included in the "in use" set via `buildAddressList`:

```go
func buildAddressList(addressesInUse []ipamv1.IPAddress, gateway string) []string {
    addrStrings := make([]string, len(addressesInUse), len(addressesInUse)+1)
    for i, address := range addressesInUse {
        addrStrings[i] = address.Spec.Address
    }
    if gateway != "" {
        addrStrings = append(addrStrings, gateway)
    }
    return addrStrings
}
```

### Pool Spec to IPSet Conversion

**File**: `internal/poolutil/pool.go`

```go
func PoolSpecToIPSet(poolSpec *v1alpha2.InClusterIPPoolSpec) (*netipx.IPSet, error) {
    addressesIPSet, err := AddressesToIPSet(poolSpec.Addresses)
    builder := &netipx.IPSetBuilder{}
    builder.AddSet(addressesIPSet)

    // Remove excluded addresses
    if len(poolSpec.ExcludedAddresses) > 0 {
        excludedAddressesIPSet, _ := AddressesToIPSet(poolSpec.ExcludedAddresses)
        builder.RemoveSet(excludedAddressesIPSet)
    }

    // Remove reserved addresses (network + broadcast for IPv4)
    if !poolSpec.AllocateReservedIPAddresses {
        subnet := netip.PrefixFrom(addressesIPSet.Ranges()[0].From(), poolSpec.Prefix)
        subnetRange := netipx.RangeOfPrefix(subnet)
        builder.Remove(subnetRange.From())  // network address
        if subnet.Addr().Is4() {
            builder.Remove(subnetRange.To()) // broadcast address
        }
    }

    // Remove gateway
    if poolSpec.Gateway != "" {
        gateway, _ := netip.ParseAddr(poolSpec.Gateway)
        builder.Remove(gateway)
    }

    return builder.IPSet()
}
```

Addresses can be specified as:
- Individual IPs: `"10.0.0.5"`
- CIDR ranges: `"10.0.0.0/24"`
- Hyphenated ranges: `"10.0.0.10-10.0.0.50"`

---

## 2. Concurrency Handling

### Primary mechanism: MaxConcurrentReconciles = 1

**File**: `internal/controllers/ipaddressclaim.go`

```go
func (i *InClusterProviderAdapter) SetupWithManager(_ context.Context, b *ctrl.Builder) error {
    b.
        For(&ipamv1.IPAddressClaim{}, /* predicates... */).
        WithOptions(controller.Options{
            // To avoid race conditions when allocating IP Addresses,
            // we explicitly set this to 1
            MaxConcurrentReconciles: 1,
        }).
        // ...
    return nil
}
```

**This is the entire concurrency strategy.** The claim controller processes exactly
one IPAddressClaim at a time, serialized. There is:
- **No optimistic locking on IP selection**
- **No compare-and-swap**
- **No distributed locks**
- **No resource version checks for allocation**

The single-threaded reconciler guarantees that between "list all allocated IPs" and
"create new IPAddress object", no other allocation can happen.

### Secondary mechanism: Cache consistency wait

After creating/patching an `IPAddress`, the reconciler polls to ensure the
controller-runtime cache has seen the new object before processing the next claim:

**File**: `pkg/ipamutil/reconciler.go`

```go
// We need to ensure the address is properly watched by controller-runtime
// client (as it is cached) before moving to the next reconciliation request.
err = wait.PollUntilContextTimeout(ctx, 5*time.Millisecond, 5*time.Second, true,
    func(ctx context.Context) (bool, error) {
        key := client.ObjectKeyFromObject(&address)
        if err := r.Client.Get(ctx, key, &ipamv1.IPAddress{}); err != nil {
            return false, client.IgnoreNotFound(err)
        }
        return true, nil
    })
```

This prevents a race where the single-threaded reconciler processes the next claim
before the informer cache reflects the newly created IPAddress, which could cause
the `ListAddressesInUse` call to miss it and allocate a duplicate.

### Additional: CreateOrPatch for idempotency

```go
operationResult, err := controllerutil.CreateOrPatch(ctx, r.Client, &address, func() error {
    if res, err = handler.EnsureAddress(ctx, &address); err != nil {
        return err
    }
    // ... owner references, labels, finalizers
    return nil
})
```

`CreateOrPatch` uses the Kubernetes API server's optimistic concurrency (resource
versions) to ensure the create/update is atomic. If the object already exists,
it patches it. This provides idempotency for re-reconciliation.

### Leader Election

The controller uses standard controller-runtime leader election (configured at
the manager level, not visible in the IPAM code itself). Only one replica runs
the reconcile loop at a time.

---

## 3. Minimal Surface Area for "Allocate Next Free IP"

Stripping away ALL CAPI-specific code (Cluster references, pause checks, owner
references, predicates, watches, finalizer lifecycle), the core allocation is:

```go
import (
    "errors"
    "net/netip"
    "strings"

    "go4.org/netipx"
)

// AddressToIPSet converts a single address string to an IPSet.
// Supports: individual IPs, CIDR notation, hyphenated ranges.
func AddressToIPSet(addressStr string) (*netipx.IPSet, error) {
    builder := &netipx.IPSetBuilder{}
    if strings.Contains(addressStr, "-") {
        addrRange, err := netipx.ParseIPRange(addressStr)
        if err != nil { return nil, err }
        builder.AddRange(addrRange)
    } else if strings.Contains(addressStr, "/") {
        prefix, err := netip.ParsePrefix(addressStr)
        if err != nil { return nil, err }
        builder.AddPrefix(prefix)
    } else {
        addr, err := netip.ParseAddr(addressStr)
        if err != nil { return nil, err }
        builder.Add(addr)
    }
    return builder.IPSet()
}

// AddressesToIPSet converts multiple address strings to a merged IPSet.
func AddressesToIPSet(addresses []string) (*netipx.IPSet, error) {
    builder := &netipx.IPSetBuilder{}
    for _, addr := range addresses {
        ipSet, err := AddressToIPSet(addr)
        if err != nil { return nil, err }
        builder.AddSet(ipSet)
    }
    return builder.IPSet()
}

// FindFreeAddress returns the lowest available IP not in the inUseIPSet.
func FindFreeAddress(poolIPSet *netipx.IPSet, inUseIPSet *netipx.IPSet) (netip.Addr, error) {
    for _, iprange := range poolIPSet.Ranges() {
        ip := iprange.From()
        for {
            if !inUseIPSet.Contains(ip) {
                return ip, nil
            }
            if ip == iprange.To() { break }
            ip = ip.Next()
        }
    }
    return netip.Addr{}, errors.New("no address available")
}

// AllocateNextFreeIP is the minimal allocation function.
// poolAddresses: ["10.0.0.0/24"] or ["10.0.0.10-10.0.0.50"]
// excludedAddresses: ["10.0.0.1"] (gateway, etc.)
// allocatedAddresses: ["10.0.0.10", "10.0.0.11"] (already in use)
func AllocateNextFreeIP(
    poolAddresses []string,
    excludedAddresses []string,
    allocatedAddresses []string,
) (netip.Addr, error) {
    poolIPSet, err := AddressesToIPSet(poolAddresses)
    if err != nil { return netip.Addr{}, err }

    // Remove excluded
    if len(excludedAddresses) > 0 {
        excludedIPSet, err := AddressesToIPSet(excludedAddresses)
        if err != nil { return netip.Addr{}, err }
        builder := &netipx.IPSetBuilder{}
        builder.AddSet(poolIPSet)
        builder.RemoveSet(excludedIPSet)
        poolIPSet, _ = builder.IPSet()
    }

    // Build in-use set
    inUseIPSet, err := AddressesToIPSet(allocatedAddresses)
    if err != nil { return netip.Addr{}, err }

    return FindFreeAddress(poolIPSet, inUseIPSet)
}
```

**Single dependency**: `go4.org/netipx` (Brad Fitzpatrick's IP set library, ~2000 LOC).
Plus stdlib `net/netip`.

---

## 4. Pool Status Tracking

### Computed on each reconciliation (not stored incrementally)

**File**: `internal/controllers/inclusterippool.go` -- `genericReconcile` function

```go
func genericReconcile(ctx context.Context, c client.Client,
    pool pooltypes.GenericInClusterPool) (_ ctrl.Result, reterr error) {
    // ... patchHelper setup ...

    // 1. List ALL addresses in use for this pool
    addressesInUse, err := poolutil.ListAddressesInUse(ctx, c, pool.GetNamespace(), poolTypeRef)
    inUseCount := len(addressesInUse)

    // 2. Build the pool's IPSet
    poolIPSet, err := poolutil.PoolSpecToIPSet(pool.PoolSpec())
    poolCount := poolutil.IPSetCount(poolIPSet)

    // 3. Subtract gateway if it's in pool range
    if pool.PoolSpec().Gateway != "" {
        gatewayAddr, _ := netip.ParseAddr(pool.PoolSpec().Gateway)
        if poolIPSet.Contains(gatewayAddr) { poolCount-- }
    }

    // 4. Calculate free = total - used
    free := poolCount - inUseCount

    // 5. Calculate out-of-range (IPs allocated but no longer in pool spec)
    outOfRangeIPSet, _ := poolutil.AddressesOutOfRangeIPSet(addressesInUse, poolIPSet)

    // 6. Write to status
    pool.PoolStatus().Addresses = &v1alpha2.InClusterIPPoolStatusIPAddresses{
        Total:      poolCount,
        Used:       inUseCount,
        Free:       free,
        OutOfRange: poolutil.IPSetCount(outOfRangeIPSet),
    }
    // patchHelper.Patch writes this back
}
```

### IPSetCount implementation

```go
func IPSetCount(ipSet *netipx.IPSet) int {
    if ipSet == nil { return 0 }
    total := big.NewInt(0)
    for _, iprange := range ipSet.Ranges() {
        total.Add(total,
            big.NewInt(0).Sub(
                big.NewInt(0).SetBytes(iprange.To().AsSlice()),
                big.NewInt(0).SetBytes(iprange.From().AsSlice()),
            ),
        )
        total.Add(total, big.NewInt(1)) // inclusive range
    }
    if total.IsInt64() && total.Uint64() <= uint64(math.MaxInt) {
        return int(total.Uint64())
    }
    return math.MaxInt
}
```

The pool reconciler runs whenever:
- The pool object is modified
- An `IPAddress` referencing the pool is created/modified/deleted (via Watch)

So status is **recomputed from scratch** each time, using the same
`ListAddressesInUse` query. There is no incremental counter.

### Status CRD Fields

```go
type InClusterIPPoolStatusIPAddresses struct {
    Total      int `json:"total"`
    Used       int `json:"used"`
    Free       int `json:"free"`
    OutOfRange int `json:"outOfRange"`
}
```

---

## 5. Deallocation

### Mechanism: Finalizers + IPAddress deletion

**Two finalizers are involved:**

1. `ReleaseAddressFinalizer` = `"ipam.cluster.x-k8s.io/ReleaseAddress"` on the **IPAddressClaim**
2. `ProtectAddressFinalizer` = `"ipam.cluster.x-k8s.io/ProtectAddress"` on the **IPAddress**

### Deallocation Flow

When an `IPAddressClaim` is deleted, Kubernetes marks it for deletion but the
finalizer prevents actual removal. The reconciler detects `!claim.ObjectMeta.DeletionTimestamp.IsZero()`
and calls `reconcileDelete`:

**File**: `pkg/ipamutil/reconciler.go`

```go
func (r *ClaimReconciler) reconcileDelete(ctx context.Context,
    claim *ipamv1.IPAddressClaim, handler ClaimHandler) (ctrl.Result, error) {

    // 1. Call provider's ReleaseAddress (no-op for in-cluster provider)
    if res, err := handler.ReleaseAddress(ctx); err != nil {
        return unwrapResult(res), fmt.Errorf("release address: %w", err)
    }

    // 2. Find the IPAddress object
    address := &ipamv1.IPAddress{}
    namespacedName := types.NamespacedName{
        Namespace: claim.Namespace,
        Name:      claim.Name,  // IPAddress has same name as claim
    }
    r.Client.Get(ctx, namespacedName, address)

    // 3. Remove ProtectAddress finalizer from the IPAddress
    if address.Name != "" {
        p := client.MergeFrom(address.DeepCopy())
        if controllerutil.RemoveFinalizer(address, ProtectAddressFinalizer) {
            r.Client.Patch(ctx, address, p)
        }
        // 4. Delete the IPAddress object
        r.Client.Delete(ctx, address)
    }

    // 5. Remove ReleaseAddress finalizer from the claim (allows actual deletion)
    controllerutil.RemoveFinalizer(claim, ReleaseAddressFinalizer)
    return ctrl.Result{}, nil
}
```

### The provider's ReleaseAddress is a no-op

```go
// internal/controllers/ipaddressclaim.go
func (h *IPAddressClaimHandler) ReleaseAddress(_ context.Context) (*ctrl.Result, error) {
    // We don't need to do anything here, since the ip address is released
    // when the IPAddress is deleted
    return nil, nil
}
```

Since the allocation is tracked purely via `IPAddress` Kubernetes objects, deleting
the `IPAddress` object IS the deallocation. The next time `ListAddressesInUse` runs,
that IP will not appear, and `FindFreeAddress` will consider it available again.

### Owner References

The `IPAddress` object has two owner references:
1. **Controller owner**: the `IPAddressClaim` (so it gets GC'd when claim is deleted)
2. **Non-controller owner**: the `IPPool` (with `BlockOwnerDeletion: true`)

```go
func ensureIPAddressOwnerReferences(scheme *runtime.Scheme,
    address *ipamv1.IPAddress, claim *ipamv1.IPAddressClaim, pool client.Object) error {
    // claim is the controller owner
    controllerutil.SetControllerReference(claim, address, scheme)
    // pool is a non-controller owner
    controllerutil.SetOwnerReference(pool, address, scheme)
    // Explicitly set pool ref to NOT be controller, but block deletion
    address.OwnerReferences[poolRefIdx].Controller = ptr.To(false)
    address.OwnerReferences[poolRefIdx].BlockOwnerDeletion = ptr.To(true)
    return nil
}
```

### Pool deletion protection

The pool has its own finalizer (`ProtectPoolFinalizer = "ipam.cluster.x-k8s.io/ProtectPool"`).
The pool reconciler only removes this finalizer when `inUseCount == 0`:

```go
if !pool.GetDeletionTimestamp().IsZero() {
    if inUseCount == 0 {
        controllerutil.RemoveFinalizer(pool, ProtectPoolFinalizer)
    }
    return ctrl.Result{}, nil
}
```

Additionally, the webhook's `ValidateDelete` blocks pool deletion if any IPAddresses
exist (unless annotated with `SkipValidateDeleteWebhookAnnotation`).

---

## 6. CRD Fields: Essential vs CAPI-Specific

### InClusterIPPoolSpec

```go
type InClusterIPPoolSpec struct {
    Addresses                  []string `json:"addresses"`
    Prefix                     int      `json:"prefix"`
    Gateway                    string   `json:"gateway,omitempty"`
    AllocateReservedIPAddresses bool    `json:"allocateReservedIPAddresses,omitempty"`
    ExcludedAddresses          []string `json:"excludedAddresses,omitempty"`
}
```

| Field | Essential? | Notes |
|-------|-----------|-------|
| `Addresses` | **YES** | The IP ranges to allocate from. Supports CIDR, ranges, individual IPs. |
| `Prefix` | **YES** | Network prefix length. Used for IPAddress.Spec.Prefix and reserved addr calculation. |
| `Gateway` | **YES** | Auto-excluded from allocation, written to IPAddress.Spec.Gateway. |
| `AllocateReservedIPAddresses` | Optional | Controls whether network/broadcast addrs are allocatable. Nice UX. |
| `ExcludedAddresses` | Optional | Explicit exclusion list. Very useful for real deployments. |

### InClusterIPPoolStatus

```go
type InClusterIPPoolStatus struct {
    Addresses *InClusterIPPoolStatusIPAddresses `json:"ipAddresses,omitempty"`
}
type InClusterIPPoolStatusIPAddresses struct {
    Total      int `json:"total"`
    Free       int `json:"free"`
    Used       int `json:"used"`
    OutOfRange int `json:"outOfRange"`
}
```

| Field | Essential? | Notes |
|-------|-----------|-------|
| `Total` | **YES** | Total allocatable IPs. |
| `Free` | **YES** | Available IPs. Critical for monitoring. |
| `Used` | **YES** | Allocated IPs. |
| `OutOfRange` | Optional | IPs allocated but no longer in the pool spec (after pool shrink). |

### CAPI-Specific Fields (NOT needed for a TalosIPPool)

These are on the IPAddressClaim/IPAddress CAPI types, not on the pool itself:

- `IPAddressClaim.Spec.ClusterName` -- CAPI cluster reference
- `IPAddressClaim.Spec.PoolRef` -- uses CAPI's `IPPoolReference` type
- `IPAddress.Spec.ClaimRef` -- back-reference to claim
- `IPAddress.Spec.PoolRef` -- uses CAPI's `IPPoolReference` type
- All the Cluster pause/unpause logic
- The `clusterv1.ClusterNameLabel` label propagation

### Proposed Minimal TalosIPPool CRD

```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
kind: TalosIPPool
metadata:
  name: management-network
  namespace: talos-system
spec:
  # Required: address ranges (CIDR, ranges, or individual IPs)
  addresses:
    - "10.0.0.10-10.0.0.50"
  # Required: network prefix
  prefix: 24
  # Optional: gateway (auto-excluded from allocation)
  gateway: "10.0.0.1"
  # Optional: explicit exclusions
  excludedAddresses:
    - "10.0.0.20"
status:
  addresses:
    total: 40
    free: 38
    used: 2
```

---

## 7. Code Size Estimate

### Core allocation logic (what you'd port)

| File / Function | Lines (approx) | Purpose |
|-----------------|----------------|---------|
| `FindFreeAddress` | 15 | Sequential IP scan |
| `AddressToIPSet` | 25 | Parse single address string |
| `AddressesToIPSet` | 12 | Parse multiple address strings |
| `PoolSpecToIPSet` | 35 | Build allocatable IP set with exclusions |
| `IPSetCount` | 20 | Count IPs in a set |
| `AddressesOutOfRangeIPSet` | 15 | Detect out-of-range allocations |
| `ListAddressesInUse` | 20 | Query allocated IPs |
| **Subtotal: Pool utilities** | **~142** | |
| `EnsureAddress` | 30 | Claim handler allocation logic |
| `buildAddressList` | 10 | Helper for in-use set |
| **Subtotal: Claim handler** | **~40** | |
| Pool status computation (`genericReconcile`) | 50 | Status reconciler |
| Index setup | 30 | Field indexes for efficient queries |
| CRD types (spec + status) | 60 | Type definitions |
| **TOTAL core logic** | **~320 lines** | |

### What you would NOT port

| Component | Lines (approx) | Why skip |
|-----------|----------------|----------|
| `pkg/ipamutil/reconciler.go` (ClaimReconciler) | ~250 | CAPI lifecycle, cluster pause, cluster watches |
| `pkg/ipamutil/address.go` (NewIPAddress, owner refs) | ~60 | CAPI IPAddress type construction |
| Webhooks (`internal/webhooks/`) | ~200 | Validation logic (partially reusable) |
| Predicates (`pkg/predicates/`) | ~80 | CAPI-specific event filtering |
| Global pool variant | ~100 | InClusterIPPool vs GlobalInClusterIPPool duplication |

### External dependency

The single non-stdlib dependency for the core logic is:

```
go4.org/netipx  (BSD license)
```

This provides `IPSet`, `IPSetBuilder`, `IPRange`, `ParseIPRange`, `RangeOfPrefix`.
It is a well-maintained library by Brad Fitzpatrick (Go team member). Approximately
2000 LOC. No transitive dependencies beyond stdlib.

---

## Summary: Key Architectural Insights

1. **No bitmap, no database**: Allocation state is the set of `IPAddress` CR objects.
   Every reconciliation reconstructs the "allocated" set from a Kubernetes list query.

2. **Sequential lowest-first**: `FindFreeAddress` always returns the lowest available
   IP. This is deterministic and predictable.

3. **Single-threaded concurrency**: `MaxConcurrentReconciles: 1` is the entire
   concurrency model. Plus a cache-wait poll after each allocation.

4. **~320 lines of core logic** to port, with a single external dependency (`go4.org/netipx`).

5. **Deallocation = delete the IPAddress object**. The provider's `ReleaseAddress`
   is literally a no-op.

6. **Status is fully computed**, never incremented. Safe but potentially expensive
   for very large pools (mitigated by field indexes on the API server cache).

7. **For a TalosIPPool**: you need `Addresses`, `Prefix`, `Gateway`, and optionally
   `ExcludedAddresses`. That is it. Everything else is CAPI ceremony.
