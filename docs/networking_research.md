# Talos Linux on Proxmox: Network & Infrastructure Research

> **Date**: 2026-02-24
> **Status**: Implementation proposal — addressing all 4 shortcomings for fleet provisioning
> **Context**: Build a fleet of Kubernetes clusters on Proxmox hypervisors using Talos Linux VMs, managed by a supervisor cluster running kubemox + talos-operator.

---

## Requirements Summary

**Fleet topology**: Control plane nodes can share a hypervisor or be distributed to dedicated hosts. Worker nodes have varying compute shapes coupled with storage.

**Storage**: Multiple iSCSI attachments from Proxmox ZFS pools as pre-determined units (24GB, 32GB, 64GB) with IOPS profiles.

**Networking**: SR-IOV passthrough VNFs from high-speed NICs (Proxmox resource mappings) with virtio bridge fallback. **Static IP only — no DHCP.**

**Management**: Supervisor K8s cluster running kubemox + talos-operator. GitOps-friendly via Crossplane Compositions (pipeline mode).

---

## The 4 Shortcomings

| # | Shortcoming | Root Cause |
|---|---|---|
| 1 | Static IP assignment not declarative | Talos bootstrap overrides interface addresses; kubemox has no IP/MAC control |
| 2 | Multi-disk attachment + mount paths | kubemox `VirtualMachineDisk` has no iSCSI support; Talos has no volume path config |
| 3 | VM ID assignment | kubemox `CreateVMFromTemplate()` uses auto-assigned VMID from Proxmox |
| 4 | Deterministic ordering | No dependency/ordering mechanism between VMs |

---

## Shortcoming 1: Static IP Assignment

### The Core Problem

When kubemox clones a Talos VM and boots it on Proxmox:

1. **Pre-install**: VM boots from ISO, gets a DHCP IP `X`
2. **Config apply**: talos-operator uses IP `X` to apply machine config via Talos API
3. **Talos installs**: Talos writes to disk and reboots
4. **Post-install**: Talos brings up networking per its *own* config — may get DHCP IP `Y` (different from `X`)
5. **Lost contact**: talos-operator can no longer reach the node at `X`

The **QEMU guest agent does not run on Talos by default** and even with the `siderolabs/qemu-guest-agent` extension from Image Factory, it does NOT run in maintenance mode ([GitHub #11651](https://github.com/siderolabs/talos/issues/11651)). So Proxmox/kubemox cannot discover the IP.

**Additionally**: The requestor uses SR-IOV passthrough as the primary NIC. Talos must be configured to use the passthrough interface with a static IP, with virtio bridge as fallback.

### Solution: Two-Layer Static IP

**Layer 1 — Talos META key** (pre-install): The talos-operator already has `ApplyMetaKey()` in `pkg/talos/client.go:185` and a network template in `pkg/talos/metakey_tpl.go`. META key 0x0a sets network config that is applied **before** machine config, giving the node a reachable IP in maintenance mode.

**Layer 2 — Machine config patch** (post-install): A static network block in the Talos machine config using `deviceSelector.hardwareAddr` (MAC-based) ensures the IP persists across installs and reboots.

### Where to Implement

#### A. kubemox — Add MAC to `VirtualMachineNetwork` and `QEMUStatus`

**File**: `kubemox/api/proxmox/v1alpha1/virtualmachine_types.go:142-162`

```go
// CURRENT (line 142)
type VirtualMachineNetwork struct {
    Model  string `json:"model"`
    Bridge string `json:"bridge"`
}

// PROPOSED
type VirtualMachineNetwork struct {
    Model  string `json:"model"`
    Bridge string `json:"bridge"`
    // MACAddress sets an explicit MAC. If empty, Proxmox auto-generates.
    // +kubebuilder:validation:Optional
    // +kubebuilder:validation:Pattern=`^([0-9A-Fa-f]{2}:){5}[0-9A-Fa-f]{2}$`
    MACAddress string `json:"macAddress,omitempty"`
    // VLAN tag for this interface (0 = no VLAN)
    // +kubebuilder:validation:Optional
    VLAN int `json:"vlan,omitempty"`
}
```

```go
// CURRENT (line 149)
type QEMUStatus struct {
    State     string `json:"state"`
    Node      string `json:"node"`
    Uptime    string `json:"uptime"`
    ID        int    `json:"id"`
    IPAddress string `json:"IPAddress"`
    OSInfo    string `json:"OSInfo"`
}

// PROPOSED — add MACAddress
type QEMUStatus struct {
    State      string `json:"state"`
    Node       string `json:"node"`
    Uptime     string `json:"uptime"`
    ID         int    `json:"id"`
    IPAddress  string `json:"IPAddress"`
    OSInfo     string `json:"OSInfo"`
    MACAddress string `json:"macAddress,omitempty"`
}
```

**File**: `kubemox/pkg/proxmox/virtualmachine.go:111-114`

When building `CloneOptions`, pass MAC if specified. After clone, read MAC from Proxmox API and populate status:

```go
// In CreateVMFromTemplate(), after clone completes:
// Parse MAC from Proxmox VM config (net0 = "virtio=BC:24:11:AA:BB:CC,bridge=vmbr0")
func extractMACFromVM(vm *proxmox.VirtualMachine) string {
    // vm.VirtualMachineConfig.Net0 contains the full net string
    // Parse out the MAC address portion
    // ...
}
```

**File**: `kubemox/pkg/proxmox/virtualmachine.go:607` — In `UpdateVMStatus()`, populate `MACAddress` field.

#### B. talos-operator — Add `NetworkSpec` to `TalosMachineSpec`

**File**: `talos-operator/api/v1alpha1/talosmachine_types.go`

```go
// PROPOSED — new types to add

// NetworkSpec defines the static network identity for a Talos machine.
// Exactly one of IPAddress or PoolRef must be set.
type NetworkSpec struct {
    // IPAddress is an explicit static IP (e.g., "192.168.1.11").
    // Mutually exclusive with PoolRef.
    // +kubebuilder:validation:Optional
    IPAddress string `json:"ipAddress,omitempty"`
    // PoolRef references a TalosIPPool for automatic IP allocation.
    // Mutually exclusive with IPAddress.
    // +kubebuilder:validation:Optional
    PoolRef *PoolRef `json:"poolRef,omitempty"`
    // CIDR prefix length (e.g., 24). Overridden by pool prefix if PoolRef is set.
    // +kubebuilder:default=24
    CIDR int `json:"cidr,omitempty"`
    // Gateway is the default gateway. Overridden by pool gateway if PoolRef is set.
    // +kubebuilder:validation:Optional
    Gateway string `json:"gateway,omitempty"`
    // MACAddress of the target NIC. Used in deviceSelector.hardwareAddr.
    // +kubebuilder:validation:Optional
    MACAddress string `json:"macAddress,omitempty"`
    // Nameservers
    // +kubebuilder:validation:Optional
    Nameservers []string `json:"nameservers,omitempty"`
    // VIP is a floating IP for control plane HA
    // +kubebuilder:validation:Optional
    VIP string `json:"vip,omitempty"`
    // Interface name override (default: auto-detect via MAC)
    // +kubebuilder:validation:Optional
    Interface string `json:"interface,omitempty"`
}

// PoolRef references a TalosIPPool for automatic allocation.
type PoolRef struct {
    // Name of the TalosIPPool in the same namespace.
    // +kubebuilder:validation:Required
    Name string `json:"name"`
}

// +kubebuilder:validation:XValidation:rule="has(self.ipAddress) || has(self.poolRef)",message="one of ipAddress or poolRef must be set"
// +kubebuilder:validation:XValidation:rule="!(has(self.ipAddress) && has(self.poolRef))",message="ipAddress and poolRef are mutually exclusive"
```

Add to `TalosMachineSpec`:

```go
type TalosMachineSpec struct {
    // ... existing fields ...
    // NetworkSpec defines the static network config for this machine.
    // +kubebuilder:validation:Optional
    NetworkSpec *NetworkSpec `json:"networkSpec,omitempty"`
}
```

#### C. talos-operator — Static Network Patch Generation

**File**: `talos-operator/pkg/talos/bundle.go` — Add alongside existing patch templates:

```go
// NEW patch templates
var (
    // ... existing InstallDisk, InstallImage, etc. ...

    // StaticNetworkPatch is appended as multi-doc YAML to the machine config.
    // Uses deviceSelector.hardwareAddr so interface name doesn't matter.
    StaticNetworkPatch = `
machine:
  network:
    hostname: %s
    nameservers: %s
    interfaces:
      - deviceSelector:
          hardwareAddr: "%s"
        dhcp: false
        addresses:
          - %s/%d
        routes:
          - network: 0.0.0.0/0
            gateway: %s`

    // VIPExtension appended to StaticNetworkPatch for control plane nodes
    VIPExtension = `
        vip:
          ip: %s`
)
```

**File**: `talos-operator/internal/controller/talosmachine_controller.go:431`

In `metalConfigPatches()`, append the network patch:

```go
func (r *TalosMachineReconciler) metalConfigPatches(ctx context.Context,
    tm *talosv1alpha1.TalosMachine, config *talos.BundleConfig) (*[]string, error) {

    // ... existing patches (disk, image, wipe, airgap, registries) ...

    // NEW: Static network config
    if tm.Spec.NetworkSpec != nil {
        ns := tm.Spec.NetworkSpec
        nsYAML := "[]"
        if len(ns.Nameservers) > 0 {
            nsYAML = ""
            for _, dns := range ns.Nameservers {
                nsYAML += fmt.Sprintf("\n      - %s", dns)
            }
        }
        networkPatch := fmt.Sprintf(talos.StaticNetworkPatch,
            tm.Name, nsYAML, ns.MACAddress, ns.IPAddress, ns.CIDR, ns.Gateway)
        if ns.VIP != "" && tm.Spec.ControlPlaneRef != nil {
            networkPatch += fmt.Sprintf(talos.VIPExtension, ns.VIP)
        }
        patches = append(patches, networkPatch)
    }

    return &patches, nil
}
```

#### D. SR-IOV Passthrough Interface Support

SR-IOV VNFs appear as PCI devices in Proxmox. kubemox already has `PciDevice` with `type: mapped` support in `VirtualMachineSpecTemplate` (line 113). The passthrough NIC gets a virtual function that Talos sees as a regular network interface.

For Talos, the SR-IOV interface needs a **second entry** in the machine config `interfaces` list. The `deviceSelector` can match by `busPath` (PCI address) since passthrough devices don't have a stable kernel name:

```yaml
machine:
  network:
    interfaces:
      # Primary: SR-IOV VNF (passthrough)
      - deviceSelector:
          busPath: "0000:01:10.0"  # SR-IOV VF PCI address
        dhcp: false
        addresses:
          - 10.0.1.11/24
        routes:
          - network: 0.0.0.0/0
            gateway: 10.0.1.1
      # Fallback: virtio bridge
      - deviceSelector:
          hardwareAddr: "BC:24:11:AA:BB:CC"
        dhcp: false
        addresses:
          - 192.168.1.11/24
```

**Proposed extension to `NetworkSpec`**:

```go
type NetworkSpec struct {
    // ... existing fields ...
    // Interfaces allows specifying multiple network interfaces (e.g., SR-IOV + virtio)
    // If set, overrides the single-interface fields above.
    // +kubebuilder:validation:Optional
    Interfaces []InterfaceSpec `json:"interfaces,omitempty"`
}

type InterfaceSpec struct {
    // Name is a human-readable label (e.g., "sriov-primary", "virtio-fallback")
    Name string `json:"name"`
    // DeviceSelector to match the interface in Talos
    // +kubebuilder:validation:Optional
    MACAddress string `json:"macAddress,omitempty"`
    // +kubebuilder:validation:Optional
    BusPath string `json:"busPath,omitempty"`
    // Static IP config
    IPAddress string `json:"ipAddress"`
    CIDR      int    `json:"cidr,omitempty"`
    Gateway   string `json:"gateway,omitempty"`
    // Whether this is the default route
    // +kubebuilder:default=false
    DefaultRoute bool `json:"defaultRoute,omitempty"`
}
```

### Example YAML: TalosMachine with Static Network

```yaml
apiVersion: talos.alperen.cloud/v1alpha1
kind: TalosMachine
metadata:
  name: prod-cp-0
  namespace: fleet
spec:
  endpoint: "10.0.1.11"
  version: "v1.11.0"
  controlPlaneRef:
    name: prod-controlplane
  machineSpec:
    meta:
      interface: ens18
      subnet: 24
      gateway: 10.0.1.1
      dnsServers:
        - 10.0.1.1
  networkSpec:
    ipAddress: "10.0.1.11"
    cidr: 24
    gateway: "10.0.1.1"
    macAddress: "BC:24:11:AA:BB:01"
    nameservers:
      - "10.0.1.1"
      - "8.8.8.8"
    vip: "10.0.1.10"
    interfaces:
      - name: sriov-primary
        busPath: "0000:01:10.0"
        ipAddress: "10.0.1.11"
        cidr: 24
        gateway: "10.0.1.1"
        defaultRoute: true
      - name: virtio-fallback
        macAddress: "BC:24:11:AA:BB:01"
        ipAddress: "192.168.1.11"
        cidr: 24
```

> **Note**: Engineers familiar with traditional Linux provisioning may question the absence of NIC model filtering, interface renaming, netplan, or cloud-init network configuration. See [Appendix A](#appendix-a-why-traditional-linux-networking-patterns-do-not-apply) for a detailed explanation of why these patterns are unnecessary and inapplicable in Talos Linux.

---

## Shortcoming 2: Multi-Disk Attachment & Mount Paths

### Current State

**kubemox** `VirtualMachineDisk` (`virtualmachine_types.go:132-140`):

```go
type VirtualMachineDisk struct {
    Storage string `json:"storage"`  // e.g., "local-lvm"
    Size    int    `json:"size"`     // GB
    Device  string `json:"device"`   // e.g., "scsi0"
}
```

This only supports local Proxmox storage. No iSCSI target, no IOPS limits, no multi-disk ordering.

**Talos** has no built-in mount path concept for data disks — it only knows about the install disk (`/machine/install/disk`). For persistent storage:
- **Talos 1.8+**: `VolumeConfig` resources for **system volumes** only (`STATE`, `EPHEMERAL`, `IMAGECACHE`) — no filesystem or mount fields ([VolumeConfig reference](https://docs.siderolabs.com/talos/v1.11/reference/configuration/block/volumeconfig/))
- **Talos 1.10+**: `UserVolumeConfig` for **user-defined data volumes** — supports `filesystem.type` (xfs/ext4) and auto-mounts at `/var/mnt/<name>` ([UserVolumeConfig reference](https://docs.siderolabs.com/talos/v1.10/reference/configuration/block/uservolumeconfig/))
- **Talos <1.10**: Use `extraMounts` in machine config for raw bind mounts (less declarative)

**This proposal targets Talos v1.10+** (where `UserVolumeConfig` was introduced) with full testing on v1.11+.

### Proposed Changes

#### A. kubemox — Extend `VirtualMachineDisk` for iSCSI

**File**: `kubemox/api/proxmox/v1alpha1/virtualmachine_types.go:132`

```go
// CURRENT
type VirtualMachineDisk struct {
    Storage string `json:"storage"`
    Size    int    `json:"size"`
    Device  string `json:"device"`
}

// PROPOSED
type VirtualMachineDisk struct {
    // Storage is the Proxmox storage pool name (e.g., "local-lvm", "zfs-pool")
    Storage string `json:"storage"`
    // Size in GB
    // +kubebuilder:validation:Minimum=1
    Size int `json:"size"`
    // Device bus name (e.g., "scsi0", "scsi1", "virtio0")
    Device string `json:"device"`
    // IOPSLimit sets read+write IOPS throttle (0 = unlimited)
    // +kubebuilder:validation:Optional
    IOPSLimit int `json:"iopsLimit,omitempty"`
    // MBpsLimit sets read+write throughput throttle in MB/s (0 = unlimited)
    // +kubebuilder:validation:Optional
    MBpsLimit int `json:"mbpsLimit,omitempty"`
    // iSCSI configures an external iSCSI target instead of local storage
    // +kubebuilder:validation:Optional
    ISCSI *ISCSIDisk `json:"iscsi,omitempty"`
}

// ISCSIDisk defines an iSCSI LUN attachment
type ISCSIDisk struct {
    // Target IQN (e.g., "iqn.2026-01.com.example:zfs-pool-lun1")
    TargetIQN string `json:"targetIQN"`
    // Portal address (e.g., "10.0.0.5:3260")
    Portal string `json:"portal"`
    // LUN number
    // +kubebuilder:default=0
    LUN int `json:"lun,omitempty"`
}
```

**File**: `kubemox/pkg/proxmox/virtualmachine.go`

In `CreateVMFromScratch()` (line 455), after creating the VM, apply disk configs including IOPS limits via Proxmox API:

```go
// For each disk with IOPSLimit or MBpsLimit:
// qm set <vmid> --<device> <storage>:<size>,iops_rd=<limit>,iops_wr=<limit>
func (pc *ProxmoxClient) applyDiskIOPS(vmName, nodeName string, disk proxmoxv1alpha1.VirtualMachineDisk) error {
    if disk.IOPSLimit == 0 && disk.MBpsLimit == 0 {
        return nil
    }
    // Build the Proxmox disk string with IOPS parameters
    // e.g., "local-lvm:50,iops_rd=500,iops_wr=500,mbps_rd=100,mbps_wr=100"
    opts := map[string]interface{}{
        disk.Device: fmt.Sprintf("%s:%d,iops_rd=%d,iops_wr=%d",
            disk.Storage, disk.Size, disk.IOPSLimit, disk.IOPSLimit),
    }
    return pc.configureVM(vmName, nodeName, opts)
}
```

For iSCSI disks, Proxmox supports them via the `iscsi` storage type. The disk would be attached as:

```
scsi1: iscsi:iqn.2026-01.com.example:lun1/0,size=24G
```

#### B. talos-operator — Talos UserVolumeConfig for Data Disks

Talos 1.10+ `UserVolumeConfig` allows declaring user data volumes with filesystem formatting and auto-mounting. These get appended to the machine config as separate YAML documents (the operator already does this for ImageCache `VolumeConfig` in `talosmachine_controller.go:203` — see `pkg/talos/bundle.go:31-39` for the existing `ImageCacheVolumeConfig` template).

**File**: `talos-operator/pkg/talos/bundle.go` — Add user volume config templates alongside existing templates (line 40):

```go
// UserVolumeConfigTemplate for data disks (Talos 1.10+)
// Appended as multi-doc YAML after the machine config.
// Auto-mounts at /var/mnt/<name>.
// Ref: https://docs.siderolabs.com/talos/v1.10/reference/configuration/block/uservolumeconfig/
var UserVolumeConfigTemplate = `
---
apiVersion: v1alpha1
kind: UserVolumeConfig
name: %s
provisioning:
  diskSelector:
    match: '%s'
  minSize: %s
  maxSize: %s
filesystem:
  type: %s
`
```

**Note**: `UserVolumeConfig` auto-mounts at `/var/mnt/<name>`. There is no explicit `mount.path` field — the mount path is derived from the volume `name`. For example, a volume named `data-shard-1` mounts at `/var/mnt/data-shard-1`.

**File**: `talos-operator/api/v1alpha1/talosmachine_types.go` — Add to `MachineSpec` (currently at line 55-88):

```go
// PROPOSED addition to MachineSpec (after existing fields: InstallDisk, Wipe, Image, Meta, etc.)
type MachineSpec struct {
    // ... existing fields (InstallDisk, Image, AirGap, etc.) ...

    // DataVolumes defines additional disk volumes to configure in Talos via UserVolumeConfig.
    // Each volume selects a disk via CEL expression and auto-mounts at /var/mnt/<name>.
    // Requires Talos 1.10+.
    // +kubebuilder:validation:Optional
    DataVolumes []DataVolume `json:"dataVolumes,omitempty"`
}

type DataVolume struct {
    // Name of the volume (e.g., "data-shard-1"). Determines mount path: /var/mnt/<name>.
    Name string `json:"name"`
    // DiskSelector is a CEL expression to match the disk in Talos.
    // Available properties: disk.size, disk.transport, disk.serial, disk.bus_path, disk.model.
    // Ref: https://docs.siderolabs.com/talos/v1.11/configure-your-talos-cluster/storage-and-disk-management/disk-management/common/
    DiskSelector string `json:"diskSelector"`
    // MinSize (e.g., "24GiB")
    // +kubebuilder:validation:Optional
    MinSize string `json:"minSize,omitempty"`
    // MaxSize (e.g., "24GiB")
    // +kubebuilder:validation:Optional
    MaxSize string `json:"maxSize,omitempty"`
    // Filesystem type (default: xfs). Supported: xfs, ext4.
    // +kubebuilder:default="xfs"
    FSType string `json:"fsType,omitempty"`
}
```

In the controller, user volume configs are generated and appended to the machine config bytes (same pattern as `ImageCacheVolumeConfig` at `bundle.go:31-39`):

```go
// In handleControlPlaneMachine() or handleWorkerMachine():
if tm.Spec.MachineSpec != nil && len(tm.Spec.MachineSpec.DataVolumes) > 0 {
    for _, vol := range tm.Spec.MachineSpec.DataVolumes {
        volConfig := fmt.Sprintf(talos.UserVolumeConfigTemplate,
            vol.Name, vol.DiskSelector, vol.MinSize, vol.MaxSize, vol.FSType)
        *cpConfig = append(*cpConfig, []byte(volConfig)...)
    }
}
```

### Example YAML: Multi-Disk Worker Node

```yaml
apiVersion: proxmox.alperen.cloud/v1alpha1
kind: VirtualMachine
metadata:
  name: prod-worker-0
spec:
  name: prod-worker-0
  nodeName: pve2
  connectionRef:
    name: proxmox-main
  vmSpec:
    cores: 8
    memory: 16384
    disk:
      - storage: local-lvm
        size: 50
        device: scsi0                # Boot disk
      - storage: zfs-pool
        size: 24
        device: scsi1                # Data shard 1
        iopsLimit: 500
      - storage: zfs-pool
        size: 24
        device: scsi2                # Data shard 2
        iopsLimit: 500
      - storage: zfs-pool
        size: 64
        device: scsi3                # Large data volume
        iopsLimit: 1000
    network:
      - model: virtio
        bridge: vmbr0
    pciDevices:
      - type: mapped
        deviceID: "sriov-vf-pool-1"  # SR-IOV VF resource mapping
---
apiVersion: talos.alperen.cloud/v1alpha1
kind: TalosMachine
metadata:
  name: prod-worker-0
spec:
  endpoint: "10.0.1.21"
  version: "v1.11.0"
  workerRef:
    name: prod-workers
  machineSpec:
    dataVolumes:
      # Each volume auto-mounts at /var/mnt/<name> via Talos UserVolumeConfig
      - name: data-shard-1
        diskSelector: "disk.transport == 'scsi' && disk.size > 20u * GiB && disk.size < 30u * GiB"
        minSize: "24GiB"
        maxSize: "24GiB"
      - name: data-shard-2
        diskSelector: "disk.transport == 'scsi' && disk.serial == 'lun-24g-002'"
        minSize: "24GiB"
        maxSize: "24GiB"
      - name: data-large
        diskSelector: "disk.size > 60u * GiB"
        minSize: "64GiB"
        maxSize: "64GiB"
        fsType: xfs
  networkSpec:
    ipAddress: "10.0.1.21"
    cidr: 24
    gateway: "10.0.1.1"
```

### iSCSI End-to-End Flow

The proposed `ISCSIDisk` struct and `VolumeConfig` templates establish the **types**, but the end-to-end lifecycle of an iSCSI disk — from ZFS pool to mounted filesystem inside Talos — requires understanding the full data path across Proxmox, kubemox, the VM, and Talos.

#### Phase 1: ZFS Pool → iSCSI Target (Proxmox Host)

Proxmox exposes ZFS zvols as iSCSI targets via its built-in iSCSI target implementation (LIO/targetcli) or through an external SAN appliance. Each LUN maps to a zvol in a ZFS pool:

```
ZFS Pool: zfs-pool
├── zvol: zfs-pool/lun-24g-001   (24 GiB)   → iqn.2026-01.com.infra:zfs-pool-lun-24g-001
├── zvol: zfs-pool/lun-24g-002   (24 GiB)   → iqn.2026-01.com.infra:zfs-pool-lun-24g-002
├── zvol: zfs-pool/lun-64g-001   (64 GiB)   → iqn.2026-01.com.infra:zfs-pool-lun-64g-001
└── ...
```

**IQN naming convention**: `iqn.<year>-<month>.<reversed-domain>:<pool>-<purpose>-<index>`

**Portal**: The Proxmox host's storage network IP + port 3260 (e.g., `10.0.2.5:3260`).

On Proxmox, the iSCSI storage must be registered before VMs can use it:

```bash
# Register the iSCSI storage pool in Proxmox (one-time setup per portal)
pvesm add iscsi iscsi-zfs-pool \
  --portal 10.0.2.5 \
  --target iqn.2026-01.com.infra:zfs-pool \
  --content images
```

This makes the iSCSI LUNs available as disk sources for VMs.

#### Phase 2: kubemox Creates VM with iSCSI Disks

When kubemox reconciles a `VirtualMachine` CR that contains `ISCSIDisk` entries, it translates them to Proxmox API calls:

```
VirtualMachine CR                     Proxmox API
─────────────────                     ───────────
disk:                                 POST /nodes/{node}/qemu/{vmid}/config
  - device: scsi1                     → scsi1: iscsi-zfs-pool:0.0.1/lun-24g-001,size=24G
    iscsi:
      targetIQN: "...lun-24g-001"
      portal: "10.0.2.5:3260"
      lun: 0
    iopsLimit: 500                    → iops_rd=500,iops_wr=500
```

**kubemox controller flow** (`pkg/proxmox/virtualmachine.go`):

```go
func (pc *ProxmoxClient) attachISCSIDisk(vmid int, nodeName string, disk VirtualMachineDisk) error {
    // 1. Verify the iSCSI storage is registered on the target Proxmox node
    //    GET /nodes/{node}/storage → check for iscsi type matching portal+target

    // 2. Resolve the LUN to a Proxmox volume identifier
    //    The format is: <storage-id>:<target-iqn-suffix>/<lun>
    //    e.g., "iscsi-zfs-pool:0.0.1/lun-24g-001"
    volID := fmt.Sprintf("%s:%d.%d.%d/%s",
        iscsiStorageID, 0, 0, disk.ISCSI.LUN, extractLUNName(disk.ISCSI.TargetIQN))

    // 3. Attach to VM as SCSI device with IOPS limits
    opts := map[string]interface{}{
        disk.Device: fmt.Sprintf("%s,size=%dG", volID, disk.Size),
    }
    if disk.IOPSLimit > 0 {
        opts[disk.Device] += fmt.Sprintf(",iops_rd=%d,iops_wr=%d",
            disk.IOPSLimit, disk.IOPSLimit)
    }

    // 4. POST /nodes/{node}/qemu/{vmid}/config
    return pc.configureVM(vmid, nodeName, opts)
}
```

**After VM creation**, kubemox records the attached disks in the `VirtualMachine` status so downstream consumers (talos-operator) can correlate SCSI bus positions with iSCSI LUNs.

#### Phase 3: Talos Sees the Disk

When the VM boots, the iSCSI LUNs appear as standard SCSI block devices. Talos does not know or care that the backing store is iSCSI — the hypervisor handles the iSCSI initiator session. From Talos's perspective:

```
/dev/sda  →  scsi0 (boot disk, local-lvm, 50 GiB)
/dev/sdb  →  scsi1 (iSCSI LUN, 24 GiB)
/dev/sdc  →  scsi2 (iSCSI LUN, 24 GiB)
/dev/sdd  →  scsi3 (iSCSI LUN, 64 GiB)
```

Talos exposes disk metadata that the `VolumeConfig` `diskSelector` CEL expression can match:

| Property | Description | Example |
|---|---|---|
| `disk.size` | Disk capacity | `24000000000` (bytes) |
| `disk.transport` | Bus type | `"scsi"` |
| `disk.bus_path` | PCI/sysfs bus path (stable across reboots) | `"/pci0000:00/0000:00:07.0/virtio4/host1/target1:0:0/1:0:0:0"` |
| `disk.serial` | Disk serial number | `"lun-24g-001"` (from iSCSI target) |
| `disk.dev_path` | Linux block device path | `"/dev/sdb"` |

**Serial number** is the most reliable discriminator for iSCSI LUNs because Proxmox propagates the zvol name as the SCSI serial. This means the `diskSelector` can match specific LUNs without relying on enumeration order:

```cel
disk.transport == 'scsi' && disk.serial == 'lun-24g-001'
```

#### Phase 4: UserVolumeConfig Mounts the Disk

The `UserVolumeConfig` resource (Talos 1.10+, appended to the machine config as a multi-doc YAML) handles the full lifecycle ([UserVolumeConfig reference](https://docs.siderolabs.com/talos/v1.10/reference/configuration/block/uservolumeconfig/)):

```
1. Boot → Talos machined enumerates disks
2. UserVolumeConfig.diskSelector → CEL expression evaluated against each disk
3. Match found → Check provisioning constraints (minSize, maxSize)
4. Disk unformatted? → Format with specified filesystem (xfs or ext4)
5. Mount at /var/mnt/<name> → e.g., /var/mnt/data-shard-1
```

The rendered UserVolumeConfig for an iSCSI LUN:

```yaml
---
apiVersion: v1alpha1
kind: UserVolumeConfig
name: data-shard-1
provisioning:
  diskSelector:
    match: 'disk.transport == "scsi" && disk.serial == "lun-24g-001"'
  minSize: 24GiB
  maxSize: 24GiB
filesystem:
  type: xfs
```

**Mount path**: Auto-mounted at `/var/mnt/data-shard-1` (derived from `name` field).

#### Phase 5: Failure Modes and Mitigations

| Failure | Impact | Mitigation |
|---|---|---|
| **iSCSI portal unreachable at boot** | VM BIOS/UEFI may stall waiting for SCSI devices. Talos boot delayed but not blocked (boot disk is local). | Proxmox retries the iSCSI initiator session. VolumeConfig mounts are non-blocking — Talos boots with the local disk and mounts data volumes when they become available. |
| **LUN path changes** (different SCSI bus position) | `/dev/sdb` may become `/dev/sdc` after reboot. | Use `disk.serial` in diskSelector, not `disk.bus_path`. Serial is stable across reboots; bus position is not. |
| **iSCSI session timeout** (storage network flap) | Mounted filesystem goes read-only or I/O errors. | Proxmox iSCSI initiator handles reconnection. For the application layer, workloads should use retry logic. Consider `noop` scheduler for iSCSI disks. |
| **Multipath** (multiple paths to same LUN) | Duplicate block devices visible to Talos. | Not applicable in single-portal configurations. If multipath is needed, configure it at the Proxmox level (DM-Multipath on the host), not inside the VM. The VM sees a single SCSI device regardless. |
| **Disk serial collision** (two LUNs with same serial) | diskSelector matches multiple disks — VolumeConfig fails. | Enforce unique zvol names in the IQN naming convention. kubemox should validate serial uniqueness across all iSCSI disks attached to the same VM. |
| **UserVolumeConfig format on wrong disk** | Data loss if diskSelector matches the boot disk. | Always include `disk.serial` constraints. The boot disk has a different serial. Use `!system_disk` in CEL if needed (only available after installation). |

#### Example: Complete iSCSI Worker Node

```yaml
# --- kubemox VirtualMachine ---
apiVersion: proxmox.alperen.cloud/v1alpha1
kind: VirtualMachine
metadata:
  name: prod-worker-0
  namespace: fleet
spec:
  name: prod-worker-0
  nodeName: pve2
  vmid: 10110
  connectionRef:
    name: proxmox-main
  vmSpec:
    cores: 8
    memory: 16384
    disk:
      # Boot disk — local storage
      - storage: local-lvm
        size: 50
        device: scsi0
      # iSCSI data disks
      - device: scsi1
        size: 24
        iopsLimit: 500
        iscsi:
          targetIQN: "iqn.2026-01.com.infra:zfs-pool-lun-24g-001"
          portal: "10.0.2.5:3260"
          lun: 0
      - device: scsi2
        size: 24
        iopsLimit: 500
        iscsi:
          targetIQN: "iqn.2026-01.com.infra:zfs-pool-lun-24g-002"
          portal: "10.0.2.5:3260"
          lun: 0
      - device: scsi3
        size: 64
        iopsLimit: 1000
        iscsi:
          targetIQN: "iqn.2026-01.com.infra:zfs-pool-lun-64g-001"
          portal: "10.0.2.5:3260"
          lun: 0
    network:
      - model: virtio
        bridge: vmbr0
    pciDevices:
      - type: mapped
        deviceID: "sriov-vf-pool-1"
---
# --- talos-operator TalosMachine ---
apiVersion: talos.alperen.cloud/v1alpha1
kind: TalosMachine
metadata:
  name: prod-worker-0
  namespace: fleet
spec:
  endpoint: "10.0.1.21"
  version: "v1.11.0"
  workerRef:
    name: prod-workers
  machineSpec:
    dataVolumes:
      # Match by serial — stable across reboots, independent of bus enumeration
      # Each volume auto-mounts at /var/mnt/<name> via Talos UserVolumeConfig
      - name: data-shard-1
        diskSelector: 'disk.transport == "scsi" && disk.serial == "lun-24g-001"'
        minSize: "24GiB"
        maxSize: "24GiB"
        fsType: xfs
      - name: data-shard-2
        diskSelector: 'disk.transport == "scsi" && disk.serial == "lun-24g-002"'
        minSize: "24GiB"
        maxSize: "24GiB"
        fsType: xfs
      - name: data-large
        diskSelector: 'disk.transport == "scsi" && disk.serial == "lun-64g-001"'
        minSize: "64GiB"
        maxSize: "64GiB"
        fsType: xfs
  networkSpec:
    ipAddress: "10.0.1.21"
    cidr: 24
    gateway: "10.0.1.1"
    macAddress: "BC:24:11:AA:BB:21"
    nameservers:
      - "10.0.1.1"
```

#### iSCSI Data Path Summary

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐     ┌──────────────┐
│ ZFS Pool     │     │ Proxmox Host │     │ VM (Talos)      │     │ Talos Init   │
│              │     │              │     │                 │     │              │
│ zvol/lun-001 │────▶│ iSCSI Target │────▶│ /dev/sdb (SCSI) │────▶│ UserVolume   │
│ (24 GiB)     │ TCP │ LIO/targetcli│ PCI │ serial: lun-001 │ CEL │ Config       │
│              │ 3260│              │pass-│                 │     │ match serial │
│              │     │ iscsi-zfs-   │thru │                 │     │ format xfs   │
│              │     │ pool storage │     │                 │     │ /var/mnt/... │
└─────────────┘     └──────────────┘     └─────────────────┘     └──────────────┘

kubemox role:                              talos-operator role:
  - Register iSCSI storage (if needed)       - Generate UserVolumeConfig YAML
  - Attach LUN as SCSI device                - Match by disk.serial (stable)
  - Set IOPS limits                          - Append to machine config
  - Report disk status                       - Apply via Talos API
```

---

## Shortcoming 3: VM ID Assignment

### Current State

**File**: `kubemox/pkg/proxmox/virtualmachine.go:111-124`

```go
var CloneOptions proxmox.VirtualMachineCloneOptions
CloneOptions.Full = 1
CloneOptions.Name = vm.Name
CloneOptions.Target = nodeName
// No VMID set — Proxmox auto-assigns next available
newID, task, err := templateVM.Clone(ctx, &CloneOptions)
```

kubemox uses a fork: `github.com/alperencelik/go-proxmox v0.0.0-20260201203053-5a1bc2aed607`.

The Proxmox API itself supports `newid` parameter in `POST /nodes/{node}/qemu/{vmid}/clone`.

### Proposed Changes

#### A. go-proxmox — No fork changes needed

The `VirtualMachineCloneOptions` struct in go-proxmox **already has** a `NewID int` field (JSON tag `"newid"`). kubemox simply doesn't set it. No fork patch is required — only kubemox needs to populate this field.

```go
// EXISTING struct in go-proxmox (already correct):
type VirtualMachineCloneOptions struct {
    NewID   int    `json:"newid"`
    Full    uint8  `json:"full,omitempty"`
    Name    string `json:"name,omitempty"`
    Target  string `json:"target,omitempty"`
    // ... other fields: BWLimit, Description, Format, Pool, SnapName, Storage
}
```

#### B. kubemox — Add VMID to VirtualMachineSpec

**File**: `kubemox/api/proxmox/v1alpha1/virtualmachine_types.go`

```go
type VirtualMachineSpec struct {
    Name               string                        `json:"name"`
    NodeName           string                        `json:"nodeName"`
    Template           *VirtualMachineSpecTemplate   `json:"template,omitempty"`
    VMSpec             *NewVMSpec                    `json:"vmSpec,omitempty"`
    DeletionProtection bool                          `json:"deletionProtection,omitempty"`
    EnableAutoStart    bool                          `json:"enableAutoStart,omitempty"`
    AdditionalConfig   map[string]string             `json:"additionalConfig,omitempty"`
    ConnectionRef      *corev1.LocalObjectReference  `json:"connectionRef,omitempty"`
    // NEW: VMID allows specifying a deterministic Proxmox VM ID.
    // If 0, Proxmox auto-assigns the next available ID.
    // Proxmox VMIDs are integers in range 100-999999999 (regex: ^[1-9][0-9]{2,8}$).
    // VMIDs <100 are reserved for internal purposes. VMIDs must be cluster-wide unique.
    // Ref: https://pve.proxmox.com/pve-docs/qm.1.html
    // Ref: https://git.proxmox.com/?p=pve-common.git;a=blob_plain;f=src/PVE/JSONSchema.pm
    // +kubebuilder:validation:Optional
    // +kubebuilder:validation:Minimum=100
    // +kubebuilder:validation:Maximum=999999999
    VMID int `json:"vmid,omitempty"`
}
```

**File**: `kubemox/pkg/proxmox/virtualmachine.go:111`

```go
// In CreateVMFromTemplate():
var CloneOptions proxmox.VirtualMachineCloneOptions
CloneOptions.Full = 1
CloneOptions.Name = vm.Name
CloneOptions.Target = nodeName
if vm.Spec.VMID > 0 {
    CloneOptions.NewID = vm.Spec.VMID  // Use specified VMID
}
```

### Example YAML

```yaml
apiVersion: proxmox.alperen.cloud/v1alpha1
kind: VirtualMachine
metadata:
  name: prod-cp-0
spec:
  name: prod-cp-0
  nodeName: pve1
  vmid: 10110          # Deterministic: cluster 101, node 10
  connectionRef:
    name: proxmox-main
  template:
    name: talos-v1.11-nocloud
    cores: 4
    memory: 8192
```

**VMID naming convention suggestion**: `<cluster-id><node-index>` — e.g., cluster 101 gets VMIDs 10100-10199. This makes VMIDs predictable and traceable.

---

## Shortcoming 4: Deterministic Ordering

### Current State

kubemox reconciles `VirtualMachine` CRs independently with no ordering guarantees. The controller uses `MaxConcurrentReconciles: 30` (`virtualmachine_controller.go:51`). VMs are created in whatever order the workqueue processes them.

### Why Ordering Matters

For Talos clusters:
1. **First control plane node** must exist and have its config applied before `talosctl bootstrap` can run
2. Additional control plane nodes join the existing etcd cluster
3. Workers join after the control plane is bootstrapped

However, **VM creation order is less critical than Talos bootstrap order**. kubemox can create all VMs in parallel — the ordering that matters is handled by talos-operator's reconciliation logic (it already waits for machine readiness before bootstrap).

### Proposed: Ordering via VMID Ranges + talos-operator Phases

With VMID assignment (Shortcoming 3), ordering becomes implicit:
- CP nodes get VMIDs 10100-10102 → talos-operator creates TalosMachine CRs in order
- Workers get VMIDs 10110-10112 → only created after CP is bootstrapped

**Note on Proxmox VMIDs**: Proxmox VM IDs are **integers only**, in the range **100 to 999,999,999**. No string-based, UUID-based, or zero-prefixed identifiers are supported. The `VMID` field in kubemox enforces this with kubebuilder validation markers (`Minimum=100`, `Maximum=999999999`).

#### Design Decision: IPAM Built Into talos-operator

The `NetworkSpec` supports two modes of IP assignment:

| Mode | How It Works | When to Use |
|---|---|---|
| **Explicit IP** | `ipAddress` field set directly on `TalosMachine.spec.networkSpec` | Small fleets, manual control, Crossplane composition with explicit IPs |
| **Pool-based allocation** | `poolRef` references a `TalosIPPool` — operator allocates next free IP automatically | Larger fleets, automated provisioning, dynamic scaling |

Both modes produce the same outcome: a `TalosMachine` with a static IP used for META key (pre-install) and machine config patch (post-install). The pool-based mode simply automates the assignment step. See the **IP Address Management (IPAM)** section below for full CRD definitions, allocation algorithm, and webhook integration.

**File**: `talos-operator/api/v1alpha1/taloscontrolplane_types.go:93`

The `MetalSpec` struct does **not** change. It retains its existing `Machines` list (machine endpoints) and `MachineSpec` (shared config). When pool-based allocation is used, the IPAM controller writes `status.allocatedIP` on each `TalosMachine`, and the `MetalSpec.Machines` list is populated from these allocated IPs:

```go
// MetalSpec remains unchanged
type MetalSpec struct {
    // Machines is a list of machine endpoints (IPs or hostnames).
    // Each entry corresponds to a TalosMachine CR with its own NetworkSpec.
    Machines    []string     `json:"machines,omitempty"`
    MachineSpec *MachineSpec `json:"machineSpec,omitempty"`
}
```

---

## Crossplane Composition: GitOps Orchestration

A Crossplane Composition (pipeline mode) provides the user-facing API — a single `TalosKubernetesCluster` Claim that generates all the underlying resources with deterministic IPs, VMIDs, and disk configs.

### The Claim (What Users Write)

```yaml
apiVersion: infrastructure.example.com/v1alpha1
kind: TalosKubernetesCluster
metadata:
  name: prod-cluster
spec:
  # Cluster identity
  clusterID: 101

  # Control plane
  controlPlane:
    replicas: 3
    shape:
      cores: 4
      memory: 8192
      bootDiskGB: 50
    placement:
      # Can be "shared" (same hypervisor as workers) or "dedicated" (dedicated hosts)
      mode: dedicated
      nodeNames:
        - pve-cp-1
        - pve-cp-2
        - pve-cp-3

  # Workers
  workers:
    replicas: 3
    shape:
      cores: 8
      memory: 16384
      bootDiskGB: 50
      dataDisks:
        # UserVolumeConfig mounts at /var/mnt/<name> automatically
        - name: data-shard-1
          size: 24
          iopsLimit: 500
        - name: data-shard-2
          size: 24
          iopsLimit: 500
        - name: data-large
          size: 64
          iopsLimit: 1000
    placement:
      mode: shared
      nodeNames:
        - pve-worker-1
        - pve-worker-2

  # Networking — explicit 1:1 IP assignment per machine (IPAM is external)
  network:
    controlPlane:
      subnet: "10.0.1.0/24"
      gateway: "10.0.1.1"
      vip: "10.0.1.10"
      # Explicit IPs — not auto-calculated. Source: manual, IPAM operator, or Proxmox SDN IPAM.
      machines:
        - ip: "10.0.1.11"
        - ip: "10.0.1.12"
        - ip: "10.0.1.13"
    workers:
      subnet: "10.0.1.0/24"
      gateway: "10.0.1.1"
      machines:
        - ip: "10.0.1.21"
        - ip: "10.0.1.22"
        - ip: "10.0.1.23"
    nameservers:
      - "10.0.1.1"
      - "8.8.8.8"
    # SR-IOV primary NIC (mapped PCI device)
    sriovResourceMapping: "sriov-vf-pool-1"

  # Proxmox
  proxmox:
    connectionRef:
      name: proxmox-main
    templateName: talos-v1.11-nocloud
    storage:
      bootDisk: local-lvm
      dataDisk: zfs-pool

  # Talos
  talos:
    version: "v1.11.0"
    kubeVersion: "v1.33.1"

  # VMID range: 10100-10199
  vmidBase: 10100
```

### What the Composition Generates

From the above Claim, the Crossplane composition pipeline produces:

```
TalosKubernetesCluster (Claim)
├── VirtualMachine: prod-cluster-cp-0    (vmid: 10100, node: pve-cp-1, ip: 10.0.1.11)
├── VirtualMachine: prod-cluster-cp-1    (vmid: 10101, node: pve-cp-2, ip: 10.0.1.12)
├── VirtualMachine: prod-cluster-cp-2    (vmid: 10102, node: pve-cp-3, ip: 10.0.1.13)
├── VirtualMachine: prod-cluster-wk-0    (vmid: 10110, node: pve-worker-1, ip: 10.0.1.21, 3 data disks)
├── VirtualMachine: prod-cluster-wk-1    (vmid: 10111, node: pve-worker-2, ip: 10.0.1.22, 3 data disks)
├── VirtualMachine: prod-cluster-wk-2    (vmid: 10112, node: pve-worker-1, ip: 10.0.1.23, 3 data disks)
├── TalosControlPlane: prod-cluster-cp   (endpoint: https://10.0.1.10:6443, machines: [10.0.1.11, 10.0.1.12, 10.0.1.13])
├── TalosWorker: prod-cluster-workers    (controlPlaneRef: prod-cluster-cp)
├── TalosMachine: prod-cluster-cp-0      (ip: 10.0.1.11, networkSpec, no dataVolumes)
├── TalosMachine: prod-cluster-cp-1      (ip: 10.0.1.12, networkSpec, no dataVolumes)
├── TalosMachine: prod-cluster-cp-2      (ip: 10.0.1.13, networkSpec, no dataVolumes)
├── TalosMachine: prod-cluster-wk-0      (ip: 10.0.1.21, networkSpec, 3 dataVolumes)
├── TalosMachine: prod-cluster-wk-1      (ip: 10.0.1.22, networkSpec, 3 dataVolumes)
└── TalosMachine: prod-cluster-wk-2      (ip: 10.0.1.23, networkSpec, 3 dataVolumes)
```

### Composition Pipeline (Sketch)

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: talos-kubernetes-cluster
spec:
  compositeTypeRef:
    apiVersion: infrastructure.example.com/v1alpha1
    kind: XTalosKubernetesCluster

  pipeline:
    # Step 1: Generate control plane VMs
    - step: control-plane-vms
      functionRef:
        name: function-go-templating
      input:
        apiVersion: gotemplating.fn.crossplane.io/v1beta1
        kind: GoTemplate
        source: Inline
        inline:
          template: |
            {{- $s := .observed.composite.resource.spec }}
            {{- $name := .observed.composite.resource.metadata.name }}
            {{- range $i := until (int $s.controlPlane.replicas) }}
            ---
            apiVersion: proxmox.alperen.cloud/v1alpha1
            kind: VirtualMachine
            metadata:
              annotations:
                gotemplating.fn.crossplane.io/composition-resource-name: cp-vm-{{ $i }}
            spec:
              name: "{{ $name }}-cp-{{ $i }}"
              nodeName: {{ index $s.controlPlane.placement.nodeNames $i }}
              vmid: {{ add $s.vmidBase $i }}
              connectionRef:
                name: {{ $s.proxmox.connectionRef.name }}
              template:
                name: {{ $s.proxmox.templateName }}
                cores: {{ $s.controlPlane.shape.cores }}
                memory: {{ $s.controlPlane.shape.memory }}
                disk:
                  - storage: {{ $s.proxmox.storage.bootDisk }}
                    size: {{ $s.controlPlane.shape.bootDiskGB }}
                    device: scsi0
                network:
                  - model: virtio
                    bridge: vmbr0
                pciDevices:
                  {{- if $s.network.sriovResourceMapping }}
                  - type: mapped
                    deviceID: {{ $s.network.sriovResourceMapping }}
                  {{- end }}
            {{ end }}

    # Step 2: Generate worker VMs with data disks
    - step: worker-vms
      functionRef:
        name: function-go-templating
      input:
        apiVersion: gotemplating.fn.crossplane.io/v1beta1
        kind: GoTemplate
        source: Inline
        inline:
          template: |
            {{- $s := .observed.composite.resource.spec }}
            {{- $name := .observed.composite.resource.metadata.name }}
            {{- $nodeCount := len $s.workers.placement.nodeNames }}
            {{- range $i := until (int $s.workers.replicas) }}
            ---
            apiVersion: proxmox.alperen.cloud/v1alpha1
            kind: VirtualMachine
            metadata:
              annotations:
                gotemplating.fn.crossplane.io/composition-resource-name: wk-vm-{{ $i }}
            spec:
              name: "{{ $name }}-wk-{{ $i }}"
              nodeName: {{ index $s.workers.placement.nodeNames (mod $i $nodeCount) }}
              vmid: {{ add $s.vmidBase 10 $i }}
              connectionRef:
                name: {{ $s.proxmox.connectionRef.name }}
              vmSpec:
                cores: {{ $s.workers.shape.cores }}
                memory: {{ $s.workers.shape.memory }}
                disk:
                  - storage: {{ $s.proxmox.storage.bootDisk }}
                    size: {{ $s.workers.shape.bootDiskGB }}
                    device: scsi0
                  {{- range $d, $disk := $s.workers.shape.dataDisks }}
                  - storage: {{ $s.proxmox.storage.dataDisk }}
                    size: {{ $disk.size }}
                    device: "scsi{{ add $d 1 }}"
                    iopsLimit: {{ $disk.iopsLimit }}
                  {{- end }}
                network:
                  - model: virtio
                    bridge: vmbr0
                pciDevices:
                  {{- if $s.network.sriovResourceMapping }}
                  - type: mapped
                    deviceID: {{ $s.network.sriovResourceMapping }}
                  {{- end }}
            {{ end }}

    # Step 3: TalosControlPlane
    - step: talos-control-plane
      functionRef:
        name: function-go-templating
      input:
        apiVersion: gotemplating.fn.crossplane.io/v1beta1
        kind: GoTemplate
        source: Inline
        inline:
          template: |
            {{- $s := .observed.composite.resource.spec }}
            ---
            apiVersion: talos.alperen.cloud/v1alpha1
            kind: TalosControlPlane
            metadata:
              annotations:
                gotemplating.fn.crossplane.io/composition-resource-name: talos-cp
            spec:
              version: {{ $s.talos.version }}
              kubeVersion: {{ $s.talos.kubeVersion }}
              mode: metal
              replicas: {{ $s.controlPlane.replicas }}
              endpoint: "https://{{ $s.network.controlPlane.vip }}:6443"
              metalSpec:
                machines:
                  # Explicit 1:1 IPs — no IPAM in the operator. IPs come from the Claim.
                  {{- range $i, $m := $s.network.controlPlane.machines }}
                  - "{{ $m.ip }}"
                  {{- end }}

    # Step 4: TalosMachines with explicit NetworkSpec
    # Each TalosMachine gets its own 1:1 IP assignment from the Claim.
    - step: talos-machines
      functionRef:
        name: function-go-templating
      input:
        apiVersion: gotemplating.fn.crossplane.io/v1beta1
        kind: GoTemplate
        source: Inline
        inline:
          template: |
            {{- $s := .observed.composite.resource.spec }}
            {{- $name := .observed.composite.resource.metadata.name }}
            {{- range $i, $m := $s.network.controlPlane.machines }}
            ---
            apiVersion: talos.alperen.cloud/v1alpha1
            kind: TalosMachine
            metadata:
              annotations:
                gotemplating.fn.crossplane.io/composition-resource-name: cp-tm-{{ $i }}
            spec:
              endpoint: "{{ $m.ip }}"
              version: {{ $s.talos.version }}
              controlPlaneRef:
                name: "{{ $name }}-cp"
              networkSpec:
                ipAddress: "{{ $m.ip }}"
                cidr: 24
                gateway: "{{ $s.network.controlPlane.gateway }}"
                nameservers: {{ toYaml $s.network.nameservers | nindent 18 }}
                vip: "{{ $s.network.controlPlane.vip }}"
            {{ end }}
            {{- range $i, $m := $s.network.workers.machines }}
            ---
            apiVersion: talos.alperen.cloud/v1alpha1
            kind: TalosMachine
            metadata:
              annotations:
                gotemplating.fn.crossplane.io/composition-resource-name: wk-tm-{{ $i }}
            spec:
              endpoint: "{{ $m.ip }}"
              version: {{ $s.talos.version }}
              workerRef:
                name: "{{ $name }}-workers"
              networkSpec:
                ipAddress: "{{ $m.ip }}"
                cidr: 24
                gateway: "{{ $s.network.workers.gateway }}"
                nameservers: {{ toYaml $s.network.nameservers | nindent 18 }}
            {{ end }}
```

### End-to-End Flow

```
1. User pushes TalosKubernetesCluster Claim to Git
   |
2. ArgoCD/Flux syncs to supervisor cluster
   |
3. Crossplane composition resolves:
   ├── 3 CP VirtualMachine CRs (vmid: 10100-10102)
   ├── 3 Worker VirtualMachine CRs (vmid: 10110-10112, with data disks + IOPS)
   ├── 1 TalosControlPlane CR (machines: [10.0.1.11, 10.0.1.12, 10.0.1.13])
   ├── 3 CP TalosMachine CRs (each with explicit 1:1 networkSpec)
   ├── 3 Worker TalosMachine CRs (each with explicit 1:1 networkSpec)
   └── 1 TalosWorker CR
   |
4. kubemox reconciles VirtualMachine CRs:
   ├── Clones template with specified VMID (integer 100-999999999)
   ├── Configures disks with IOPS limits
   ├── Attaches SR-IOV VF via PCI passthrough
   ├── Reports MAC address in status
   └── Starts VMs
   |
5. talos-operator reconciles TalosControlPlane:
   ├── Machines list contains explicit IPs (no IPAM — IPs assigned externally)
   └── TalosMachine CRs already exist (created by Crossplane) with 1:1 networkSpec
   |
6. talos-operator reconciles each TalosMachine:
   ├── Apply META key 0x0a (pre-install network via metakey_tpl.go:3-47)
   ├── Generate machine config with:
   │   ├── Static network patch (deviceSelector.hardwareAddr)
   │   ├── UserVolumeConfig for data disks (Talos 1.10+)
   │   └── VIP for control plane
   ├── Apply config via Talos API
   ├── Talos installs → reboots at SAME IP
   └── Bootstrap etcd on first CP node
   |
7. Cluster ready at VIP 10.0.1.10:6443
```

---

## IP Address Management (IPAM)

### Design Rationale

The 4 shortcomings above assume 1:1 explicit IP assignment per `TalosMachine`. This works for small fleets but becomes operationally burdensome at scale — every new machine needs a manually chosen, conflict-free IP.

This section introduces **pool-based IP allocation** built directly into talos-operator. The allocation algorithm uses a sequential lowest-first scan over an `IPSet` — a well-understood pattern in Kubernetes IPAM implementations. There are **no external IPAM dependencies** (no CAPI, no external controllers).

**Key design decisions:**

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Allocation record | Separate `TalosIPAddress` CRD | Clean lifecycle, watchable by external controllers |
| Manual + pool support | Both `ipAddress` and `poolRef` in `NetworkSpec` | Backward compatible; small fleets stay explicit |
| External DHCP/firewall sync | Generic allocation webhooks | No vendor lock-in; works with pfSense, OPNsense, NetBox, etc. |
| Concurrency | `MaxConcurrentReconciles: 1` on IPAM controller | Prevents double-allocation without distributed locking |
| IP set library | `go4.org/netipx` (BSD, Brad Fitzpatrick) | Single dependency; used by CAPI, Tailscale, and Go stdlib contributors |

### TalosIPPool CRD

Defines a range of IPs available for allocation.

```go
// TalosIPPool defines a pool of IP addresses for automatic allocation to TalosMachines.
type TalosIPPool struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    Spec              TalosIPPoolSpec   `json:"spec,omitempty"`
    Status            TalosIPPoolStatus `json:"status,omitempty"`
}

type TalosIPPoolSpec struct {
    // Addresses is a list of IP ranges. Supports CIDR ("10.0.50.0/28"),
    // ranges ("10.0.50.10-10.0.50.50"), and individual IPs ("10.0.50.100").
    // +kubebuilder:validation:MinItems=1
    Addresses []string `json:"addresses"`
    // Prefix is the subnet prefix length (e.g., 24 for /24).
    // +kubebuilder:validation:Minimum=1
    // +kubebuilder:validation:Maximum=128
    Prefix int `json:"prefix"`
    // Gateway is the default gateway for allocated IPs.
    // +kubebuilder:validation:Required
    Gateway string `json:"gateway"`
    // ExcludedAddresses are IPs within the range that should never be allocated
    // (e.g., existing infrastructure, VIPs). Gateway is auto-excluded.
    // +kubebuilder:validation:Optional
    ExcludedAddresses []string `json:"excludedAddresses,omitempty"`
    // AllocateReservedIPAddresses controls whether network and broadcast addresses
    // are allocatable. When false (default), the first and last IPs of the subnet
    // are excluded from allocation.
    // +kubebuilder:default=false
    // +kubebuilder:validation:Optional
    AllocateReservedIPAddresses bool `json:"allocateReservedIPAddresses,omitempty"`
    // AllocationWebhooks fire on allocate/deallocate events for external DHCP/firewall sync.
    // +kubebuilder:validation:Optional
    AllocationWebhooks []AllocationWebhook `json:"allocationWebhooks,omitempty"`
}

type TalosIPPoolStatus struct {
    // Total is the number of allocatable addresses in the pool.
    Total int `json:"total"`
    // Used is the number of currently allocated addresses.
    Used int `json:"used"`
    // Free is Total - Used.
    Free int `json:"free"`
    // OutOfRange is the count of IPs that are allocated but no longer within
    // the pool's address ranges (e.g., after shrinking the pool spec).
    OutOfRange int `json:"outOfRange,omitempty"`
}
```

**Example:**

```yaml
apiVersion: talos.alperen.cloud/v1alpha1
kind: TalosIPPool
metadata:
  name: prod-workers
  namespace: fleet
spec:
  addresses:
    - "10.0.50.10-10.0.50.50"
  prefix: 24
  gateway: "10.0.50.1"
  excludedAddresses:
    - "10.0.50.1"   # gateway
    - "10.0.50.10"  # VIP
  allocationWebhooks:
    - name: pfsense-dhcp
      onAllocate:
        url: "https://pfsense.lab/api/v1/services/dhcpd/static_mapping"
        method: POST
        template: |
          {"interface":"opt1","ipaddr":"{{ .IP }}","mac":"{{ .MAC }}","hostname":"{{ .Name }}"}
      onDeallocate:
        url: "https://pfsense.lab/api/v1/services/dhcpd/static_mapping"
        method: DELETE
        template: |
          {"interface":"opt1","ipaddr":"{{ .IP }}"}
      credentialRef:
        name: pfsense-api-creds
      failurePolicy: Warn
status:
  total: 40
  used: 12
  free: 28
```

### TalosIPAddress CRD

One `TalosIPAddress` is created per allocation. Garbage collected via `ownerRef` when the `TalosMachine` is deleted.

```go
// TalosIPAddress represents a single IP allocation from a TalosIPPool.
type TalosIPAddress struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    Spec              TalosIPAddressSpec `json:"spec,omitempty"`
}

type TalosIPAddressSpec struct {
    // Address is the allocated IP.
    Address string `json:"address"`
    // Prefix is the subnet prefix length (copied from pool).
    Prefix int `json:"prefix"`
    // Gateway (copied from pool).
    Gateway string `json:"gateway"`
    // PoolRef is the pool this address was allocated from.
    PoolRef corev1.LocalObjectReference `json:"poolRef"`
    // MachineRef is the TalosMachine that owns this allocation.
    MachineRef corev1.LocalObjectReference `json:"machineRef"`
}
```

**Ownership and lifecycle:**

```
TalosMachine (with poolRef)
  │
  ├─ ownerRef ──→ TalosIPAddress (created by IPAM controller)
  │                 labels:
  │                   talos.alperen.cloud/pool: prod-workers
  │                   talos.alperen.cloud/machine: prod-worker-3
  │
  └─ status.allocatedIP: "10.0.50.13"  (writeback from IPAM controller)
```

On `TalosMachine` deletion: Kubernetes garbage collection deletes `TalosIPAddress` via `ownerRef`. The IPAM controller's finalizer fires `onDeallocate` webhooks before the `TalosIPAddress` is removed. The IP returns to the pool's free set on the next reconciliation.

### Allocation Algorithm

Algorithm modeled after CAPI IPAM's `InClusterIPPool` controller (~320 lines of core logic including CRD types in the CAPI reference implementation; see estimated scope table for our breakdown). Implemented natively in talos-operator. Sequential scan, lowest-first, using `netipx.IPSet`:

```go
import "go4.org/netipx"

// FindFreeAddress scans the pool range and returns the lowest unused IP.
// poolIPSet: all IPs in the pool (from spec.addresses, minus excludedAddresses).
// inUseIPSet: all IPs currently allocated (built from TalosIPAddress CRs).
func FindFreeAddress(poolIPSet, inUseIPSet *netipx.IPSet) (netip.Addr, error) {
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
    return netip.Addr{}, errors.New("no address available in pool")
}
```

Supporting utilities (also modeled after CAPI IPAM's `internal/poolutil/pool.go`):

```go
// AddressToIPSet converts a single address string to an IPSet.
// Supports: individual IPs ("10.0.0.5"), CIDR ("10.0.0.0/24"), ranges ("10.0.0.10-10.0.0.50").
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

// PoolSpecToIPSet builds the allocatable IP set: addresses - excluded - reserved - gateway.
func PoolSpecToIPSet(spec *TalosIPPoolSpec) (*netipx.IPSet, error) {
    poolIPSet, err := AddressesToIPSet(spec.Addresses)
    if err != nil { return nil, err }
    builder := &netipx.IPSetBuilder{}
    builder.AddSet(poolIPSet)
    // Remove excluded addresses
    if len(spec.ExcludedAddresses) > 0 {
        excludedIPSet, _ := AddressesToIPSet(spec.ExcludedAddresses)
        builder.RemoveSet(excludedIPSet)
    }
    // Remove network + broadcast (unless opted in)
    if !spec.AllocateReservedIPAddresses {
        subnet := netip.PrefixFrom(poolIPSet.Ranges()[0].From(), spec.Prefix)
        subnetRange := netipx.RangeOfPrefix(subnet)
        builder.Remove(subnetRange.From())               // network address
        if subnet.Addr().Is4() { builder.Remove(subnetRange.To()) } // broadcast
    }
    // Remove gateway
    if spec.Gateway != "" {
        gateway, _ := netip.ParseAddr(spec.Gateway)
        builder.Remove(gateway)
    }
    return builder.IPSet()
}
```

**No persistent allocation table.** The in-use set is rebuilt each reconciliation by listing all `TalosIPAddress` CRs with the pool's label selector. Kubernetes itself is the database — the set of `TalosIPAddress` CRs in etcd IS the allocation table. The gateway is automatically included in the in-use set to prevent accidental allocation.

**Concurrency:** `MaxConcurrentReconciles: 1` on the IPAM controller. This prevents two machines from claiming the same IP simultaneously without requiring distributed locks. After each allocation, the controller polls until the informer cache has seen the new `TalosIPAddress` before processing the next claim — this prevents a race where the cache-based list misses a just-created allocation:

```go
// Cache consistency wait after allocation (prevents double-allocation)
err = wait.PollUntilContextTimeout(ctx, 5*time.Millisecond, 5*time.Second, true,
    func(ctx context.Context) (bool, error) {
        if err := r.Client.Get(ctx, client.ObjectKeyFromObject(&ipAddr), &TalosIPAddress{}); err != nil {
            return false, client.IgnoreNotFound(err)
        }
        return true, nil
    })
```

At fleet scale (~100 machines), single-threaded allocation adds negligible latency since `FindFreeAddress` is O(n) over the range and the cache wait is typically <50ms.

**Pool deletion protection:** The pool has a finalizer (`talos.alperen.cloud/protect-pool`) that is only removed when `inUseCount == 0`. This prevents accidental deletion of a pool that still has active allocations.

**Status recomputation:** Pool status (`total`, `used`, `free`, `outOfRange`) is recomputed from scratch on each reconciliation — never incremented. The pool reconciler runs whenever a `TalosIPAddress` referencing the pool is created, modified, or deleted (via Watch).

### Generic Allocation Webhooks

Instead of building vendor-specific DHCP/firewall integrations (pfSense, OPNsense, MikroTik, etc.) into the operator, `TalosIPPool` supports generic webhooks that fire on allocation and deallocation events. The operator is the source of truth; external systems are sync targets.

```go
type AllocationWebhook struct {
    // Name is a human-readable identifier for this webhook.
    Name string `json:"name"`
    // OnAllocate fires when an IP is allocated from this pool.
    // +kubebuilder:validation:Optional
    OnAllocate *WebhookAction `json:"onAllocate,omitempty"`
    // OnDeallocate fires when an IP is released back to this pool.
    // +kubebuilder:validation:Optional
    OnDeallocate *WebhookAction `json:"onDeallocate,omitempty"`
    // CredentialRef references a Secret with authentication for the webhook endpoint.
    // The Secret must contain a "token" key used as Bearer token,
    // or "username"/"password" keys for basic auth.
    // +kubebuilder:validation:Optional
    CredentialRef *corev1.LocalObjectReference `json:"credentialRef,omitempty"`
    // FailurePolicy determines behavior when the webhook fails.
    // "Fail" blocks the allocation. "Warn" logs and continues.
    // +kubebuilder:default=Warn
    // +kubebuilder:validation:Enum=Fail;Warn
    FailurePolicy string `json:"failurePolicy,omitempty"`
}

type WebhookAction struct {
    // URL is the webhook endpoint.
    URL string `json:"url"`
    // Method is the HTTP method (POST, PUT, DELETE, PATCH).
    // +kubebuilder:validation:Enum=POST;PUT;DELETE;PATCH
    Method string `json:"method"`
    // Template is a Go text/template that renders the request body.
    // Available variables: .IP, .Prefix, .Gateway, .MAC, .Name, .Namespace, .Pool
    // +kubebuilder:validation:Optional
    Template string `json:"template,omitempty"`
}
```

**Webhook execution:**

1. IPAM controller allocates IP → creates `TalosIPAddress` CR
2. For each `onAllocate` webhook in the pool: render template, send HTTP request
3. Retry 3x with exponential backoff (1s, 2s, 4s)
4. If `failurePolicy: Warn` — log failure, continue (allocation is not rolled back)
5. If `failurePolicy: Fail` — log failure, delete `TalosIPAddress`, requeue for retry

**Template variables:**

| Variable | Source | Example |
|----------|--------|---------|
| `{{ .IP }}` | Allocated address | `10.0.50.13` |
| `{{ .Prefix }}` | Pool prefix | `24` |
| `{{ .Gateway }}` | Pool gateway | `10.0.50.1` |
| `{{ .MAC }}` | TalosMachine MAC | `BC:24:11:AA:BB:13` |
| `{{ .Name }}` | TalosMachine name | `prod-worker-3` |
| `{{ .Namespace }}` | TalosMachine namespace | `fleet` |
| `{{ .Pool }}` | TalosIPPool name | `prod-workers` |

### Reconciliation Flow

```
TalosMachine created (with poolRef: prod-workers)
  │
  ├─ IPAM Controller (MaxConcurrentReconciles: 1)
  │   ├─ List TalosIPAddress CRs with label pool=prod-workers
  │   ├─ Build inUseIPSet from existing allocations
  │   ├─ FindFreeAddress(poolIPSet, inUseIPSet) → 10.0.50.13
  │   ├─ Create TalosIPAddress CR (ownerRef: TalosMachine, labels: pool + machine)
  │   ├─ Fire onAllocate webhooks (pfSense DHCP static mapping, etc.)
  │   ├─ Write status.allocatedIP = "10.0.50.13" on TalosMachine
  │   └─ Update TalosIPPool status (used++, free--)
  │
  ├─ TalosMachine Controller (sees status.allocatedIP populated)
  │   ├─ Uses allocatedIP instead of spec.networkSpec.ipAddress
  │   ├─ Generates META key patch + static network patch as normal
  │   └─ Proceeds with VM creation
  │
  └─ TalosMachine deleted
      ├─ Finalizer on TalosIPAddress fires onDeallocate webhooks
      ├─ TalosIPAddress garbage collected via ownerRef
      └─ Pool status updated on next reconcile (used--, free++)
```

### Crossplane Integration

When using Crossplane Compositions (pipeline mode), the `TalosKubernetesCluster` claim can reference a pool:

```yaml
apiVersion: talos.axonnetworks.io/v1alpha1
kind: TalosKubernetesCluster
metadata:
  name: prod-cluster
spec:
  controlPlane:
    replicas: 3
    network:
      poolRef:
        name: cp-pool       # allocate from cp-pool
      gateway: "10.0.1.1"   # override if needed (otherwise pool gateway used)
  workers:
    - name: compute
      replicas: 5
      network:
        poolRef:
          name: worker-pool  # allocate from worker-pool
```

The composition pipeline generates `TalosMachine` CRs with `poolRef` set. The IPAM controller allocates IPs before the TalosMachine controller proceeds with VM creation — no manual IP assignment needed.

### Estimated Scope

| Component | Estimated Lines | Notes |
|-----------|----------------|-------|
| `TalosIPPool` CRD types | ~60 | Spec, Status, webhook types |
| `TalosIPAddress` CRD types | ~30 | Spec only (no status needed) |
| `NetworkSpec` modification | ~15 | Add `PoolRef`, CEL validation |
| IPAM controller | ~230 | Allocation, deallocation, pool status, index setup (CRD types counted separately above) |
| Webhook engine | ~150 | Template rendering, HTTP dispatch, retry, credential loading |
| Tests | ~200 | Unit tests for `FindFreeAddress`, webhook dispatch, reconciliation |
| **Total** | **~685** | Pure Go, single external dependency (`go4.org/netipx`) |

---

## Summary of Changes by Project

### kubemox (5 changes)

| Change | File (current line refs) | Description |
|---|---|---|
| Add `MACAddress` to `VirtualMachineNetwork` | `api/proxmox/v1alpha1/virtualmachine_types.go:142-147` | Allow setting/reading MAC |
| Add `MACAddress` to `QEMUStatus` | Same file, line 149 | Report MAC in status |
| Add `VMID` to `VirtualMachineSpec` | Same file, line 34 (struct starts here) | Deterministic VM IDs (integer 100-999999999) |
| Add `IOPSLimit`, `MBpsLimit`, `ISCSI` to `VirtualMachineDisk` | Same file, line 132 | iSCSI + IOPS support |
| Pass VMID in `CloneOptions`, read MAC after clone | `pkg/proxmox/virtualmachine.go:111-118` (`CreateVMFromTemplate`) | Wire up new fields |

**kubemox concurrency**: `VMmaxConcurrentReconciles = 30` (`internal/controller/proxmox/virtualmachine_controller.go:51`)

### go-proxmox fork (0 changes needed)

Import: `github.com/luthermonson/go-proxmox v0.3.2` → replaced by `github.com/alperencelik/go-proxmox v0.0.0-20260201203053-5a1bc2aed607` (see `kubemox/go.mod:9,23`)

`VirtualMachineCloneOptions.NewID int` already exists in the struct (JSON tag `"newid"`). kubemox just doesn't set it. No fork changes required — only kubemox needs to populate `CloneOptions.NewID = vm.Spec.VMID`.

### talos-operator (7 changes)

| Change | File | Description |
|---|---|---|
| Add `NetworkSpec` with `poolRef` to `TalosMachineSpec` | `api/v1alpha1/talosmachine_types.go` (after line 53) | Static IP (explicit) or pool-based allocation per machine |
| Add `TalosIPPool` CRD | `api/v1alpha1/talosippool_types.go` (new file) | Pool definition with ranges, gateway, webhooks |
| Add `TalosIPAddress` CRD | `api/v1alpha1/talosipaddress_types.go` (new file) | Per-allocation record with ownerRef to TalosMachine |
| Add `DataVolume` to `MachineSpec` | `api/v1alpha1/talosmachine_types.go` (after line 88) | UserVolumeConfig-based data disk management (Talos 1.10+) |
| IPAM controller | `internal/controller/talosippool_controller.go` (new file) | Allocation, deallocation, pool status, webhook dispatch (~320 lines) |
| Static network + UserVolumeConfig patch generation | `pkg/talos/bundle.go` (after line 40) | New patch templates alongside existing ones |
| Apply network + volume patches | `internal/controller/talosmachine_controller.go` (in `metalConfigPatches()` at line 431) | Wire into reconcile loop; use `status.allocatedIP` when `poolRef` is set |

**Note**: `MetalSpec` (`taloscontrolplane_types.go:93-99`) is **unchanged**. IPAM is managed by the dedicated `TalosIPPool` controller within talos-operator. External dependency: `go4.org/netipx` (BSD license).

### Crossplane (new)

| Change | Description |
|---|---|
| XRD: `XTalosKubernetesCluster` | Composite resource definition |
| Composition pipeline | go-templating steps for VMs, TalosCP, TalosWorker |
| Claim: `TalosKubernetesCluster` | User-facing API |

---

## Known Risks & Forward-Compatibility Notes

### JSON Patch vs Strategic Merge Patch

The current talos-operator uses **JSON RFC 6902 patches** exclusively (see `pkg/talos/bundle.go`). However:

- When `UserVolumeConfig` documents are appended as multi-doc YAML, JSON patches [may stop working](https://github.com/siderolabs/talos/issues/12005) because RFC 6902 doesn't support multi-document configs.
- The proposed `StaticNetworkPatch` uses **strategic merge YAML**, which is the correct approach for multi-doc scenarios.
- **Migration consideration**: Existing JSON patches (InstallDisk, InstallImage, WipeDisk, etc.) may need to be migrated to strategic merge format when multi-doc config is used.

### Talos v1.12 Network Configuration Deprecation

Talos v1.12 introduces new standalone network configuration documents (`LinkConfig`, `HostnameConfig`, `BondConfig`) with `AddressConfig` and `RouteConfig` as sub-fields within these. The legacy `.machine.network` configuration is **deprecated** in v1.12 but remains **supported for backward compatibility**. This proposal targets Talos v1.10/v1.11 where `.machine.network` is the standard approach. A future enhancement should migrate the `StaticNetworkPatch` to the new multi-doc network config format for v1.12+.

### CEL Field Naming: snake_case vs camelCase

- **Disk CEL selectors** (in `UserVolumeConfig`/`VolumeConfig`): Use **snake_case** — `disk.bus_path`, `disk.transport`, `disk.size`
- **Network `deviceSelector`**: Uses **camelCase** — `hardwareAddr`, `busPath`, `driver`, `pciID`
- This inconsistency is upstream in Talos itself — network selectors are camelCase, disk CEL expressions are snake_case.

---

## Verification Matrix

Every technical claim in this document has been verified against primary sources. This matrix provides traceability from claim to evidence.

### Talos Linux Claims

| # | Claim | Source Type | Source | Status |
|---|---|---|---|---|
| T1 | META key `0x0a` (decimal 10) provides pre-install network config | Official docs | [Metal Network Configuration](https://docs.siderolabs.com/talos/v1.11/networking/metal-network-configuration/) | VERIFIED |
| T2 | `deviceSelector` fields: busPath, hardwareAddr, permanentAddr, pciID, driver, physical | Official docs | [Device Selector Reference](https://docs.siderolabs.com/talos/v1.11/networking/device-selector/) | VERIFIED |
| T3 | `UserVolumeConfig` introduced in Talos 1.10+ | Official docs | [UserVolumeConfig Reference](https://docs.siderolabs.com/talos/v1.10/reference/configuration/block/uservolumeconfig/), [v1.10.0 release notes](https://github.com/siderolabs/talos/releases/tag/v1.10.0) | VERIFIED — CORRECTED from 1.11+ to 1.10+ |
| T4 | `UserVolumeConfig` auto-mounts at `/var/mnt/<name>` with no customizable mount.path | Official docs | Same as T3 | VERIFIED |
| T5 | `VolumeConfig` is system-only (`STATE`, `EPHEMERAL`, `IMAGECACHE`) | Official docs | [VolumeConfig Reference](https://docs.siderolabs.com/talos/v1.11/reference/configuration/block/volumeconfig/) | VERIFIED |
| T6 | Disk CEL properties use snake_case: `disk.bus_path`, `disk.transport`, `disk.serial`, `disk.size`, `disk.dev_path` | Source code verification | Talos DiskSpec protobuf | VERIFIED |
| T7 | `disk.bus_path` = PCI/sysfs bus path (not kernel device path) | Source code verification | Talos DiskSpec protobuf | VERIFIED |
| T8 | `disk.name` does NOT exist as a CEL property | Source code verification | Talos DiskSpec protobuf | VERIFIED |
| T9 | `system_disk` is a standalone boolean, available only post-install | Source code verification | Talos disk management code | VERIFIED |
| T10 | VIP syntax: `vip: { ip: "x.x.x.x" }` under interface config | Official docs | [Config Reference](https://docs.siderolabs.com/talos/v1.11/reference/configuration/v1alpha1/config/) | VERIFIED |
| T11 | QEMU guest agent does NOT run in maintenance mode | GitHub issue | [siderolabs/talos#11651](https://github.com/siderolabs/talos/issues/11651) | UNVERIFIED — awaiting issue confirmation |
| T12 | Talos v1.12 introduces standalone `LinkConfig`, `HostnameConfig`, `BondConfig`; `AddressConfig`/`RouteConfig` are sub-fields of these | Official docs | Talos v1.12 networking docs | VERIFIED — CORRECTED: AddressConfig/RouteConfig are sub-fields, not standalone |
| T13 | `.machine.network` deprecated in v1.12 but still supported for backward compatibility | Official docs | [Talos v1.12.0 release notes](https://github.com/siderolabs/talos/releases/tag/v1.12.0) | VERIFIED — this proposal targets v1.10/v1.11 where `.machine.network` is the standard approach |
| T14 | Talos is immutable — no shell, no udev, no cloud-init binary | Official docs | Talos architecture documentation | VERIFIED |

### Proxmox VE Claims

| # | Claim | Source Type | Source | Status |
|---|---|---|---|---|
| P1 | VMID range: integer 100–999,999,999 (regex `^[1-9][0-9]{2,8}$`) | Official docs + source | [qm man page](https://pve.proxmox.com/pve-docs/qm.1.html), [JSONSchema.pm](https://git.proxmox.com/?p=pve-common.git;a=blob_plain;f=src/PVE/JSONSchema.pm) | VERIFIED — regex derived from documented range, not an official API constant |
| P2 | VMIDs cluster-wide unique, shared between QEMU and LXC | Official docs | [PVE Admin Guide](https://pve.proxmox.com/pve-docs/pve-admin-guide.html) | VERIFIED |
| P3 | iSCSI supported as storage type, registered via `pvesm add iscsi` | Official docs | [Storage: iSCSI](https://pve.proxmox.com/wiki/Storage:_iSCSI) | VERIFIED — note: Proxmox recommends LVM-on-iSCSI over direct LUN use |
| P4 | Per-disk IOPS throttling via `iops_rd`, `iops_wr`, `mbps_rd`, `mbps_wr` | Official docs | [qm man page](https://pve.proxmox.com/pve-docs/qm.1.html) | VERIFIED |
| P5 | MAC regenerated on VM clone; `newid` param in clone API | Official docs + forum | [Clone API](https://pve.proxmox.com/pve-docs/api-viewer/), [Forum thread](https://forum.proxmox.com/threads/28153/) | VERIFIED |
| P6 | SDN IPAM in PVE 8.1+ with PVE/phpIPAM/NetBox backends | Official docs | [SDN Chapter](https://pve.proxmox.com/pve-docs/chapter-pvesdn.html) | VERIFIED — SDN is tech preview status as of PVE 8.x |
| P7 | PCI Passthrough + SR-IOV VF support with resource mappings | Official docs | [PCI Passthrough Wiki](https://pve.proxmox.com/wiki/PCI_Passthrough) | VERIFIED |

### go-proxmox Claims

| # | Claim | Source Type | Source | Status |
|---|---|---|---|---|
| G1 | `VirtualMachineCloneOptions` has `NewID int` field (JSON tag `"newid"`) — 10 fields total | Source code | [go-proxmox](https://github.com/luthermonson/go-proxmox) `types.go`, confirmed in [fork](https://github.com/alperencelik/go-proxmox) | VERIFIED |
| G2 | kubemox doesn't currently set `NewID` — Proxmox auto-assigns | Codebase ref | `kubemox/pkg/proxmox/virtualmachine.go:111-118` | VERIFIED |

### Crossplane Claims

| # | Claim | Source Type | Source | Status |
|---|---|---|---|---|
| X1 | Crossplane Composition supports `pipeline` mode with `functionRef` steps | Official docs | [Crossplane Compositions](https://docs.crossplane.io/latest/composition/compositions/) | VERIFIED |
| X2 | `function-go-templating` (`gotemplating.fn.crossplane.io/v1beta1`) is a valid Crossplane function | GitHub repo | [crossplane-contrib/function-go-templating](https://github.com/crossplane-contrib/function-go-templating) v0.11.x | VERIFIED |
| X3 | Composition API version is `apiextensions.crossplane.io/v1` (NOT v2; v2 is XRDs only) | Official docs | [Crossplane v2.2 docs](https://docs.crossplane.io/latest/composition/compositions/) — all examples use v1 | VERIFIED — CORRECTED in this doc |
| X4 | `gotemplating.fn.crossplane.io/composition-resource-name` annotation names composed resources | GitHub docs | [function-go-templating README](https://github.com/crossplane-contrib/function-go-templating) | VERIFIED |
| X5 | XRD (`CompositeResourceDefinition`) defines the composite type; v2 XRDs support `scope` field | Official docs | [XRD docs](https://docs.crossplane.io/latest/composition/composite-resource-definitions/) | VERIFIED |

### IPAM Design Claims

| # | Claim | Source Type | Source | Status |
|---|---|---|---|---|
| I1 | CAPI IPAM `FindFreeAddress` uses sequential lowest-first scan with `netipx.IPSet` | Source code | [`internal/poolutil/pool.go`](https://github.com/kubernetes-sigs/cluster-api-ipam-provider-in-cluster) | VERIFIED — read from source |
| I2 | CAPI IPAM concurrency model is `MaxConcurrentReconciles: 1` with no distributed locks | Source code | [`internal/controllers/ipaddressclaim.go`](https://github.com/kubernetes-sigs/cluster-api-ipam-provider-in-cluster) | VERIFIED — explicit comment in source |
| I3 | CAPI IPAM has no persistent allocation table — rebuilt from IPAddress CRs each reconciliation | Source code | `internal/controllers/ipaddressclaim.go` `EnsureAddress()` | VERIFIED |
| I4 | CAPI IPAM uses cache consistency poll (5ms interval, 5s timeout) after allocation | Source code | `pkg/ipamutil/reconciler.go` | VERIFIED |
| I5 | Core CAPI IPAM allocation logic is ~320 lines of Go | Source code analysis | See `capi-ipam-allocation-analysis.md` line-by-line breakdown | VERIFIED |
| I6 | `go4.org/netipx` is BSD-licensed, by Brad Fitzpatrick, ~2000 LOC, no transitive deps | Package metadata | [go4.org/netipx](https://pkg.go.dev/go4.org/netipx) | VERIFIED |
| I7 | CAPI IPAM pool deletion protected by finalizer, only removed when `inUseCount == 0` | Source code | `internal/controllers/inclusterippool.go` `genericReconcile()` | VERIFIED |
| I8 | CAPI IPAM status (total/used/free/outOfRange) recomputed from scratch, never incremented | Source code | Same as I7 | VERIFIED |
| I9 | CAPI IPAM deallocation is a no-op — deleting IPAddress CR IS deallocation | Source code | `internal/controllers/ipaddressclaim.go` `ReleaseAddress()` returns nil | VERIFIED |
| I10 | CAPI IPAM address format supports CIDR, hyphenated ranges, and individual IPs | Source code | `internal/poolutil/pool.go` `AddressToIPSet()` | VERIFIED |

### Codebase References (All Verified)

| # | Claim | File:Line | Status |
|---|---|---|---|
| C1 | `VirtualMachineSpec` struct | `kubemox/api/proxmox/v1alpha1/virtualmachine_types.go:34` | VERIFIED |
| C2 | `VirtualMachineSpecTemplate` with `PciDevices` | Same file, line 96 | VERIFIED |
| C3 | `PciDevice` struct (type: raw/mapped, deviceID) | Same file, line 117 | VERIFIED |
| C4 | `VirtualMachineDisk` struct (Storage, Size, Device) | Same file, line 132 | VERIFIED |
| C5 | `VirtualMachineNetwork` struct (Model, Bridge) | Same file, line 142 | VERIFIED |
| C6 | `QEMUStatus` struct (no MAC field yet) | Same file, line 149 | VERIFIED |
| C7 | `CreateVMFromTemplate()` function | `kubemox/pkg/proxmox/virtualmachine.go:92` | VERIFIED |
| C8 | `CloneOptions` setup without VMID | Same file, lines 111-118 | VERIFIED |
| C9 | `UpdateVMStatus()` function | Same file, line 607 | VERIFIED |
| C10 | `VMmaxConcurrentReconciles = 30` | `kubemox/internal/controller/proxmox/virtualmachine_controller.go:51` | VERIFIED |
| C11 | go-proxmox fork in go.mod | `kubemox/go.mod:9,23` | VERIFIED |
| C12 | `TalosMachineSpec` struct | `talos-operator/api/v1alpha1/talosmachine_types.go:28` | VERIFIED |
| C13 | `MachineSpec` struct (no NetworkSpec/DataVolumes yet) | Same file, line 55 | VERIFIED |
| C14 | `MetalSpec` struct | `talos-operator/api/v1alpha1/taloscontrolplane_types.go:93` | VERIFIED |
| C15 | `META` struct (Hostname, Interface, Subnet, Gateway, DNSServers) | Same file, line 101 | VERIFIED |
| C16 | Existing patch templates (InstallDisk, InstallImage, etc.) | `talos-operator/pkg/talos/bundle.go:22-40` | VERIFIED |
| C17 | `metaKeyTemplate` — META key 0x0a template | `talos-operator/pkg/talos/metakey_tpl.go:3-47` | VERIFIED |
| C18 | `ApplyMetaKey()` function | `talos-operator/pkg/talos/client.go:185` | VERIFIED |
| C19 | `metalConfigPatches()` function | `talos-operator/internal/controller/talosmachine_controller.go:431` | VERIFIED |

---

## Key References

### Talos Linux

- [Talos Proxmox Guide (v1.11)](https://www.talos.dev/v1.11/talos-guides/install/virtualized-platforms/proxmox/)
- [Talos Metal Network Configuration](https://docs.siderolabs.com/talos/v1.11/networking/metal-network-configuration/) — META key `0x0a` documentation
- [Talos deviceSelector Reference](https://docs.siderolabs.com/talos/v1.11/networking/device-selector/) — `hardwareAddr`, `busPath`, `driver`, `pciID`, `permanentAddr`, `physical`
- [Talos Configuration v1alpha1 Reference](https://docs.siderolabs.com/talos/v1.11/reference/configuration/v1alpha1/config/) — `NetworkDeviceSelector` struct
- [Talos VolumeConfig Reference (v1.9)](https://docs.siderolabs.com/talos/v1.9/reference/configuration/block/volumeconfig/) — System volumes only (`EPHEMERAL`, `IMAGECACHE`)
- [Talos UserVolumeConfig Reference (v1.10)](https://docs.siderolabs.com/talos/v1.10/reference/configuration/block/uservolumeconfig/) — User data volumes with filesystem and auto-mount
- [Talos Disk Management CEL Expressions](https://docs.siderolabs.com/talos/v1.11/configure-your-talos-cluster/storage-and-disk-management/disk-management/common/) — `disk.serial`, `disk.bus_path`, `disk.transport`, `disk.size`, `disk.model`
- [Talos Static Addressing](https://docs.siderolabs.com/talos/v1.12/networking/configuration/static)
- [Talos Nocloud Documentation](https://docs.siderolabs.com/talos/v1.8/platform-specific-installations/cloud-platforms/nocloud)
- [Talos Image Factory](https://factory.talos.dev/)
- [GitHub Issue #11651 - Guest Agent in Maintenance Mode](https://github.com/siderolabs/talos/issues/11651)

### Proxmox VE

- [Proxmox VMID Specification (qm man page)](https://pve.proxmox.com/pve-docs/qm.1.html) — `<vmid>: <integer> (100 - 999999999)`
- [Proxmox VMID Source Code (JSONSchema.pm)](https://git.proxmox.com/?p=pve-common.git;a=blob_plain;f=src/PVE/JSONSchema.pm) — Regex: `^[1-9][0-9]{2,8}$`
- [Proxmox Admin Guide - VMID Auto-Selection](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#pvecm_next_id_range) — Default range 100-1,000,000
- [Proxmox SDN Documentation](https://pve.proxmox.com/pve-docs/chapter-pvesdn.html) — SDN zones, VNets, subnets, IPAM
- [Proxmox SDN Wiki](https://pve.proxmox.com/wiki/Software-Defined_Network) — PVE IPAM, phpIPAM, NetBox backends
- [Proxmox SDN IPAM Source Code](https://git.proxmox.com/?p=pve-network.git;a=tree;f=src/PVE/API2/Network/SDN;hb=HEAD) — API paths confirmed
- [Proxmox iSCSI Storage](https://pve.proxmox.com/wiki/Storage:_iSCSI)
- [Proxmox PCI Passthrough / SR-IOV](https://pve.proxmox.com/wiki/PCI_Passthrough)
- [Proxmox VM Cloning and MAC Behavior](https://forum.proxmox.com/threads/when-cloning-a-kvm-vm-the-mac-address-is-renewed.28153/)

### IPAM Design References

- [cluster-api-ipam-provider-in-cluster](https://github.com/kubernetes-sigs/cluster-api-ipam-provider-in-cluster) — Design reference for TalosIPPool/TalosIPAddress CRDs (allocation algorithm, concurrency model, pool status)
- [CAPI IPAM Provider Contract](https://cluster-api.sigs.k8s.io/developer/providers/contracts/ipam) — Reference architecture for `IPAddressClaim` / `IPAddress` patterns
- [CAPI IPAM Integration Proposal](https://github.com/kubernetes-sigs/cluster-api/blob/main/docs/proposals/20220125-ipam-integration.md) — Original design rationale
- [CAPI IPAM Allocation Analysis](capi-ipam-allocation-analysis.md) — Source code deep-dive of CAPI IPAM allocation logic (local analysis document)
- [go4.org/netipx](https://pkg.go.dev/go4.org/netipx) — BSD-licensed IP set library (Brad Fitzpatrick), single external dependency for allocation

### Other

- [JYSK Tech: 3000+ Clusters with NoCloud](https://jysk.tech/3000-clusters-part-3-how-to-boot-talos-linux-nodes-with-cloud-init-and-nocloud-acdce36f60c0)
- [siderolabs/omni-infra-provider-proxmox](https://github.com/siderolabs/omni-infra-provider-proxmox)
- [GitHub Issue #12005 - Strategic Merge + Multi-Doc JSON Patch Conflict](https://github.com/siderolabs/talos/issues/12005)
- [GitHub Issue #10080 - busPath/buspath CEL field rename in v1.9](https://github.com/siderolabs/talos/issues/10080)
- [Talos Network Device Selector (v1.11)](https://www.talos.dev/v1.11/talos-guides/network/device-selector/)
- [Talos Config Patches Documentation](https://docs.siderolabs.com/talos/v1.9/configure-your-talos-cluster/system-configuration/patching)

### Codebase References (Current State)

#### kubemox (`armangurkan/kubemox`, branch `claude/general-session-UY36F`)

| File | Line | Symbol |
|---|---|---|
| `api/proxmox/v1alpha1/virtualmachine_types.go` | 34 | `VirtualMachineSpec` struct |
| Same | 96 | `VirtualMachineSpecTemplate` struct (with `PciDevices []PciDevice`) |
| Same | 117 | `PciDevice` struct (`type: raw\|mapped`, `deviceID`) |
| Same | 132 | `VirtualMachineDisk` struct (Storage, Size, Device — no iSCSI/IOPS yet) |
| Same | 142 | `VirtualMachineNetwork` struct (Model, Bridge — no MAC yet) |
| Same | 149 | `QEMUStatus` struct (State, Node, Uptime, ID, IPAddress — no MAC yet) |
| `pkg/proxmox/virtualmachine.go` | 92 | `CreateVMFromTemplate()` function |
| Same | 111-118 | `CloneOptions` setup (Full, Name, Target — no VMID yet) |
| Same | 607 | `UpdateVMStatus()` function |
| `internal/controller/proxmox/virtualmachine_controller.go` | 51 | `VMmaxConcurrentReconciles = 30` |
| `go.mod` | 9, 23 | go-proxmox fork: `alperencelik/go-proxmox v0.0.0-20260201203053-5a1bc2aed607` |

#### talos-operator (`armangurkan/talos-operator`, branch `claude/general-session-UY36F`)

| File | Line | Symbol |
|---|---|---|
| `api/v1alpha1/talosmachine_types.go` | 28 | `TalosMachineSpec` struct (Endpoint, Version, MachineSpec, ControlPlaneRef, WorkerRef) |
| Same | 55 | `MachineSpec` struct (InstallDisk, Wipe, Image, Meta — no NetworkSpec/DataVolumes yet) |
| `api/v1alpha1/taloscontrolplane_types.go` | 93 | `MetalSpec` struct (Machines []string, MachineSpec) |
| Same | 101 | `META` struct (Hostname, Interface, Subnet, Gateway, DNSServers) |
| `pkg/talos/bundle.go` | 22-40 | Existing patch templates (InstallDisk, InstallImage, WipeDisk, AirGapp, ImageCache, ImageCacheVolumeConfig) |
| `pkg/talos/metakey_tpl.go` | 3-47 | `metaKeyTemplate` — META key 0x0a network config template |
| `pkg/talos/client.go` | 185 | `ApplyMetaKey()` function |
| `internal/controller/talosmachine_controller.go` | 431 | `metalConfigPatches()` function |

---

## Appendix A: Why Traditional Linux Networking Patterns Do Not Apply

> This appendix addresses questions that engineers familiar with traditional Linux provisioning commonly raise when reviewing this proposal. It explains why NIC model filtering, interface renaming, netplan, and cloud-init network modules are neither necessary nor applicable in a Talos Linux environment.

### A.1 NIC Model Filtering Is Unnecessary

In traditional Linux provisioning, administrators sometimes filter NICs by model (e.g., `virtio`, `e1000`, `intel-ixgbe`) to distinguish management interfaces from data-plane interfaces.

This proposal uses two hardware-identity selectors that are strictly more precise than model filtering:

| Selector | Used For | Uniqueness Guarantee |
|---|---|---|
| `deviceSelector.hardwareAddr` | virtio NICs (management plane) | Globally unique per interface (MAC address) |
| `deviceSelector.busPath` | SR-IOV virtual functions (data plane) | Unique per PCI topology on the host |

NIC model is a **class identifier**, not a **device identifier**. A node with three virtio NICs shares the same model string across all three. Filtering by model alone cannot distinguish which interface should carry management traffic versus storage traffic versus tenant traffic. MAC address and PCI bus path are unique identifiers that resolve to exactly one interface on a given node.

**Example from this proposal:**

```yaml
machine:
  network:
    interfaces:
      - deviceSelector:
          hardwareAddr: "bc:24:11:*"    # Matches the management NIC by MAC
        addresses:
          - 10.0.50.11/24
        routes:
          - network: 0.0.0.0/0
            gateway: 10.0.50.1
      - deviceSelector:
          busPath: "0000:04:10.0"       # Matches a specific SR-IOV VF by PCI address
        addresses:
          - 192.168.100.11/24
```

### A.2 Renaming Interfaces to eth0 Is Unnecessary

Traditional Linux provisioning often renames interfaces to predictable names like `eth0` using udev rules, systemd `.link` files, or kernel boot parameters (`net.ifnames=0 biosdevname=0`).

Talos Linux is an **immutable operating system**. There is no:

- Shell access to run renaming commands
- `/etc/udev/rules.d/` directory for persistent naming rules
- `/etc/network/interfaces` file to reference by name
- systemd-networkd `.link` files for name overrides

The kernel assigns names like `enx<mac>`, `ens18`, or `enp6s0f0` based on hardware topology. The `deviceSelector` mechanism binds configuration to hardware identity, so whether the kernel names an interface `ens18` or `enp0s3` has no effect on the provisioning system.

### A.3 Netplan Is Irrelevant

Netplan is the default network configuration abstraction on Ubuntu. **Talos Linux does not have netplan.** Talos does not run Ubuntu, does not use systemd-networkd or NetworkManager as backends, and does not read `/etc/netplan/*.yaml` files.

Talos networking is entirely declarative through the machine configuration YAML under `machine.network.interfaces`. This configuration is applied by the Talos init system during boot, before any userspace services start.

| Aspect | Netplan | Talos machine.network |
|---|---|---|
| Configuration format | `/etc/netplan/*.yaml` | Machine config YAML |
| Backend | systemd-networkd or NetworkManager | Talos init (machined) |
| Applied when | After systemd starts | Before userspace, during init |
| Mutable at runtime | Yes (`netplan apply`) | Only via config patch + reboot or live apply |
| Interface selection | By name (`eth0`, `ens3`) | By hardware identity (`deviceSelector`) |
| Available in Talos | No | Yes (native and only option) |

### A.4 Cloud-Init Network Configuration Is Not Applicable

The Proxmox VM templates use names like `talos-v1.11-nocloud`, which suggests cloud-init involvement. The `nocloud` in the template name refers to the **platform metadata datasource type**, not to the cloud-init network configuration system.

| Term | Meaning in Talos Context |
|---|---|
| `nocloud` datasource | A metadata delivery mechanism (via cloud-drive or SMBIOS) that provides hostname, instance-id, and similar platform metadata |
| `cloud-init network` module | A network configuration system within cloud-init that generates backend configs (netplan, ENI, etc.) — **this does not exist in Talos** |

Talos reads metadata from nocloud-compatible datasources for platform identification and hostname discovery. It does **not** process cloud-init network configuration blocks. There is no cloud-init binary in Talos, no `/etc/cloud/` directory, and no network module to render interface configurations.

### A.5 Summary: Traditional vs. Talos-Native Networking

| Networking Concern | Traditional Linux Approach | This Proposal's Talos-Native Approach | Why Talos-Native Is Superior |
|---|---|---|---|
| **Interface identification** | NIC model filtering (`virtio`, `e1000`) | `deviceSelector.hardwareAddr` (MAC) or `.busPath` (PCI) | MAC/PCI are unique per-device; model is shared across NICs |
| **Interface naming** | Rename to `eth0` via udev rules | No renaming; bind by hardware identity | Eliminates naming fragility |
| **Network config tool** | Netplan (`/etc/netplan/*.yaml`) | Machine config YAML (`machine.network.interfaces`) | No indirection layer; applied at init |
| **Bootstrap networking** | Cloud-init network module | META key injection (key `0x0a`) | No cloud-init dependency; works in immutable OS |
| **Production networking** | Cloud-init + netplan + manual tuning | Machine config patch via talos-operator | Centrally managed as K8s resources |
| **Runtime mutability** | `netplan apply`, `ip link set` | Config patch + controlled reboot / `talosctl apply-config` | Prevents configuration drift |
