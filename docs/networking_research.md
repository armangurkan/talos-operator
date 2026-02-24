# Talos Linux on Proxmox: Network & Infrastructure Research

> **Date**: 2026-02-24
> **Status**: Implementation proposal — addressing all 4 shortcomings for fleet provisioning
> **Context**: Build a fleet of Kubernetes clusters on Proxmox hypervisors using Talos Linux VMs, managed by a supervisor cluster running kubemox + talos-operator.

---

## Requirements Summary

**Fleet topology**: Control plane nodes can share a hypervisor or be distributed to dedicated hosts. Worker nodes have varying compute shapes coupled with storage.

**Storage**: Multiple iSCSI attachments from Proxmox ZFS pools as pre-determined units (24GB, 32GB, 64GB) with IOPS profiles.

**Networking**: SR-IOV passthrough VNFs from high-speed NICs (Proxmox resource mappings) with virtio bridge fallback. **Static IP only — no DHCP.**

**Management**: Supervisor K8s cluster running kubemox + talos-operator. GitOps-friendly via Crossplane v2 compositions.

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
type NetworkSpec struct {
    // IPAddress is the static IP (e.g., "192.168.1.11")
    // +kubebuilder:validation:Required
    IPAddress string `json:"ipAddress"`
    // CIDR prefix length (e.g., 24)
    // +kubebuilder:default=24
    CIDR int `json:"cidr,omitempty"`
    // Gateway is the default gateway
    // +kubebuilder:validation:Required
    Gateway string `json:"gateway"`
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
  version: "v1.10.3"
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

**Talos** has no built-in mount path concept for data disks — it only knows about the install disk (`/machine/install/disk`). For persistent storage, Talos 1.8+ supports `VolumeConfig` resources for arbitrary disk management, and the `extraMounts` field for raw bind mounts.

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

#### B. talos-operator — Talos VolumeConfig for Data Disks

Talos 1.8+ `VolumeConfig` allows declaring disk volumes. These get appended to the machine config as separate YAML documents (the operator already does this for ImageCache in `talosmachine_controller.go:203`).

**File**: `talos-operator/pkg/talos/bundle.go` — Add volume config templates:

```go
// VolumeConfigTemplate for data disks
// Appended as multi-doc YAML after the machine config
var VolumeConfigTemplate = `
---
apiVersion: v1alpha1
kind: VolumeConfig
name: %s
provisioning:
  diskSelector:
    match: '%s'
  grow: true
  minSize: %s
  maxSize: %s
  fileSystems:
    - type: xfs
      label: %s
mount:
  path: %s
`
```

**File**: `talos-operator/api/v1alpha1/talosmachine_types.go` — Add to MachineSpec:

```go
// PROPOSED addition to MachineSpec
type MachineSpec struct {
    // ... existing fields (InstallDisk, Image, AirGap, etc.) ...

    // DataVolumes defines additional disk volumes to configure in Talos.
    // Each volume maps a disk to a mount path with optional size constraints.
    // +kubebuilder:validation:Optional
    DataVolumes []DataVolume `json:"dataVolumes,omitempty"`
}

type DataVolume struct {
    // Name of the volume (e.g., "data-24g-1")
    Name string `json:"name"`
    // DiskSelector is a CEL expression to match the disk in Talos
    // (e.g., "disk.size > 20u * GB && disk.size < 30u * GB")
    DiskSelector string `json:"diskSelector"`
    // MountPath where this volume is mounted (e.g., "/var/data/shard-1")
    MountPath string `json:"mountPath"`
    // MinSize (e.g., "24GiB")
    // +kubebuilder:validation:Optional
    MinSize string `json:"minSize,omitempty"`
    // MaxSize (e.g., "24GiB")
    // +kubebuilder:validation:Optional
    MaxSize string `json:"maxSize,omitempty"`
    // Filesystem type (default: xfs)
    // +kubebuilder:default="xfs"
    FSType string `json:"fsType,omitempty"`
}
```

In the controller, volume configs are generated and appended to the machine config bytes (same pattern as `ImageCacheVolumeConfig`):

```go
// In handleControlPlaneMachine() or handleWorkerMachine():
if tm.Spec.MachineSpec != nil && len(tm.Spec.MachineSpec.DataVolumes) > 0 {
    for _, vol := range tm.Spec.MachineSpec.DataVolumes {
        volConfig := fmt.Sprintf(talos.VolumeConfigTemplate,
            vol.Name, vol.DiskSelector, vol.MinSize, vol.MaxSize, vol.Name, vol.MountPath)
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
  version: "v1.10.3"
  workerRef:
    name: prod-workers
  machineSpec:
    dataVolumes:
      - name: data-shard-1
        diskSelector: "disk.transport == 'scsi' && disk.size > 20u * GB && disk.size < 30u * GB"
        mountPath: /var/data/shard-1
        minSize: "24GiB"
        maxSize: "24GiB"
      - name: data-shard-2
        diskSelector: "disk.transport == 'scsi' && disk.busPath == '/dev/sdc'"
        mountPath: /var/data/shard-2
        minSize: "24GiB"
        maxSize: "24GiB"
      - name: data-large
        diskSelector: "disk.size > 60u * GB"
        mountPath: /var/data/large
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
| `disk.busPath` | Kernel device path | `"/dev/sdb"` |
| `disk.serial` | Disk serial number | `"lun-24g-001"` (from iSCSI target) |
| `disk.name` | Kernel name | `"sdb"` |

**Serial number** is the most reliable discriminator for iSCSI LUNs because Proxmox propagates the zvol name as the SCSI serial. This means the `diskSelector` can match specific LUNs without relying on enumeration order:

```cel
disk.transport == 'scsi' && disk.serial == 'lun-24g-001'
```

#### Phase 4: VolumeConfig Mounts the Disk

The `VolumeConfig` resource (appended to the machine config as a multi-doc YAML) handles the full lifecycle:

```
1. Boot → Talos machined enumerates disks
2. VolumeConfig.diskSelector → CEL expression evaluated against each disk
3. Match found → Check provisioning constraints (minSize, maxSize)
4. Disk unformatted? → Format with specified filesystem (xfs)
5. Label filesystem → e.g., "data-shard-1"
6. Mount at specified path → e.g., /var/data/shard-1
7. Grow if needed → If `grow: true` and disk is larger than current FS
```

The rendered VolumeConfig for an iSCSI LUN:

```yaml
---
apiVersion: v1alpha1
kind: VolumeConfig
name: data-shard-1
provisioning:
  diskSelector:
    match: 'disk.transport == "scsi" && disk.serial == "lun-24g-001"'
  grow: true
  minSize: 24GiB
  maxSize: 24GiB
  fileSystems:
    - type: xfs
      label: data-shard-1
mount:
  path: /var/data/shard-1
```

#### Phase 5: Failure Modes and Mitigations

| Failure | Impact | Mitigation |
|---|---|---|
| **iSCSI portal unreachable at boot** | VM BIOS/UEFI may stall waiting for SCSI devices. Talos boot delayed but not blocked (boot disk is local). | Proxmox retries the iSCSI initiator session. VolumeConfig mounts are non-blocking — Talos boots with the local disk and mounts data volumes when they become available. |
| **LUN path changes** (different SCSI bus position) | `/dev/sdb` may become `/dev/sdc` after reboot. | Use `disk.serial` in diskSelector, not `disk.busPath`. Serial is stable across reboots; bus position is not. |
| **iSCSI session timeout** (storage network flap) | Mounted filesystem goes read-only or I/O errors. | Proxmox iSCSI initiator handles reconnection. For the application layer, workloads should use retry logic. Consider `noop` scheduler for iSCSI disks. |
| **Multipath** (multiple paths to same LUN) | Duplicate block devices visible to Talos. | Not applicable in single-portal configurations. If multipath is needed, configure it at the Proxmox level (DM-Multipath on the host), not inside the VM. The VM sees a single SCSI device regardless. |
| **Disk serial collision** (two LUNs with same serial) | diskSelector matches multiple disks — VolumeConfig fails. | Enforce unique zvol names in the IQN naming convention. kubemox should validate serial uniqueness across all iSCSI disks attached to the same VM. |
| **VolumeConfig format on wrong disk** | Data loss if diskSelector matches the boot disk. | Always include `disk.size` or `disk.serial` constraints. The boot disk has a different size and serial. The `Talos install disk` is excluded from VolumeConfig by default. |

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
  version: "v1.10.3"
  workerRef:
    name: prod-workers
  machineSpec:
    dataVolumes:
      # Match by serial — stable across reboots, independent of bus enumeration
      - name: data-shard-1
        diskSelector: 'disk.transport == "scsi" && disk.serial == "lun-24g-001"'
        mountPath: /var/data/shard-1
        minSize: "24GiB"
        maxSize: "24GiB"
        fsType: xfs
      - name: data-shard-2
        diskSelector: 'disk.transport == "scsi" && disk.serial == "lun-24g-002"'
        mountPath: /var/data/shard-2
        minSize: "24GiB"
        maxSize: "24GiB"
        fsType: xfs
      - name: data-large
        diskSelector: 'disk.transport == "scsi" && disk.serial == "lun-64g-001"'
        mountPath: /var/data/large
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
│ zvol/lun-001 │────▶│ iSCSI Target │────▶│ /dev/sdb (SCSI) │────▶│ VolumeConfig │
│ (24 GiB)     │ TCP │ LIO/targetcli│ PCI │ serial: lun-001 │ CEL │ match serial │
│              │ 3260│              │pass-│                 │     │ format xfs   │
│              │     │ iscsi-zfs-   │thru │                 │     │ mount /var/  │
│              │     │ pool storage │     │                 │     │ data/shard-1 │
└─────────────┘     └──────────────┘     └─────────────────┘     └──────────────┘

kubemox role:                              talos-operator role:
  - Register iSCSI storage (if needed)       - Generate VolumeConfig YAML
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

The `go-proxmox` library's `VirtualMachineCloneOptions` does NOT have a VMID field. kubemox uses a fork: `github.com/alperencelik/go-proxmox v0.0.0-20260201203053-5a1bc2aed607`.

The Proxmox API itself supports `newid` parameter in `POST /nodes/{node}/qemu/{vmid}/clone`. The `go-proxmox` `CloneOptions` struct needs to be extended (or the fork patched).

### Proposed Changes

#### A. go-proxmox fork — Add VMID to CloneOptions

**File**: `go-proxmox/types.go` (in the fork `alperencelik/go-proxmox`)

```go
// CURRENT
type VirtualMachineCloneOptions struct {
    Full    int    `json:"full"`
    Name    string `json:"name"`
    Target  string `json:"target"`
    // ...
}

// PROPOSED — add NewID
type VirtualMachineCloneOptions struct {
    Full    int    `json:"full"`
    Name    string `json:"name"`
    Target  string `json:"target"`
    NewID   int    `json:"newid,omitempty"`  // If 0, Proxmox auto-assigns
    // ...
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
    name: talos-v1.10-nocloud
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

**File**: `talos-operator/api/v1alpha1/taloscontrolplane_types.go:93`

```go
// CURRENT
type MetalSpec struct {
    Machines    []string     `json:"machines,omitempty"`
    MachineSpec *MachineSpec `json:"machineSpec,omitempty"`
}

// PROPOSED — add NetworkAllocation for deterministic IP + VMID assignment
type MetalSpec struct {
    Machines    []string     `json:"machines,omitempty"`
    MachineSpec *MachineSpec `json:"machineSpec,omitempty"`
    // NetworkAllocation defines how IPs and VMIDs are assigned to machines.
    // +kubebuilder:validation:Optional
    NetworkAllocation *NetworkAllocation `json:"networkAllocation,omitempty"`
}

type NetworkAllocation struct {
    // Subnet in CIDR notation (e.g., "10.0.1.0/24")
    Subnet string `json:"subnet"`
    // Gateway for all machines
    Gateway string `json:"gateway"`
    // StartIP is the first IP to allocate. Subsequent machines get +1.
    StartIP string `json:"startIP"`
    // StartVMID is the first Proxmox VMID. Subsequent machines get +1.
    // +kubebuilder:validation:Optional
    StartVMID int `json:"startVMID,omitempty"`
    // Nameservers
    Nameservers []string `json:"nameservers,omitempty"`
    // VIP for control plane HA
    VIP string `json:"vip,omitempty"`
}
```

The TalosControlPlane controller assigns IPs and VMIDs sequentially:

```go
func (r *TalosControlPlaneReconciler) allocateForMachine(
    alloc *NetworkAllocation, index int,
) (ip string, vmid int) {
    ip = incrementIP(net.ParseIP(alloc.StartIP), index).String()
    vmid = 0
    if alloc.StartVMID > 0 {
        vmid = alloc.StartVMID + index
    }
    return
}
```

---

## Crossplane v2 Composition: GitOps Orchestration

A Crossplane v2 composition provides the user-facing API — a single `TalosKubernetesCluster` Claim that generates all the underlying resources with deterministic IPs, VMIDs, and disk configs.

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
        - size: 24
          iopsLimit: 500
          mountPath: /var/data/shard-1
        - size: 24
          iopsLimit: 500
          mountPath: /var/data/shard-2
        - size: 64
          iopsLimit: 1000
          mountPath: /var/data/large
    placement:
      mode: shared
      nodeNames:
        - pve-worker-1
        - pve-worker-2

  # Networking — single source of truth
  network:
    controlPlane:
      subnet: "10.0.1.0/24"
      gateway: "10.0.1.1"
      startIP: "10.0.1.11"
      vip: "10.0.1.10"
    workers:
      subnet: "10.0.1.0/24"
      gateway: "10.0.1.1"
      startIP: "10.0.1.21"
    nameservers:
      - "10.0.1.1"
      - "8.8.8.8"
    # SR-IOV primary NIC (mapped PCI device)
    sriovResourceMapping: "sriov-vf-pool-1"

  # Proxmox
  proxmox:
    connectionRef:
      name: proxmox-main
    templateName: talos-v1.10-nocloud
    storage:
      bootDisk: local-lvm
      dataDisk: zfs-pool

  # Talos
  talos:
    version: "v1.10.3"
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
├── TalosControlPlane: prod-cluster-cp   (endpoint: https://10.0.1.10:6443, vip: 10.0.1.10)
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
apiVersion: apiextensions.crossplane.io/v2
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
            {{- range $i := until (int $s.controlPlane.replicas) }}
            ---
            apiVersion: proxmox.alperen.cloud/v1alpha1
            kind: VirtualMachine
            metadata:
              annotations:
                gotemplating.fn.crossplane.io/composition-resource-name: cp-vm-{{ $i }}
            spec:
              name: "{{ $s.metadata.name }}-cp-{{ $i }}"
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
            {{- $nodeCount := len $s.workers.placement.nodeNames }}
            {{- range $i := until (int $s.workers.replicas) }}
            ---
            apiVersion: proxmox.alperen.cloud/v1alpha1
            kind: VirtualMachine
            metadata:
              annotations:
                gotemplating.fn.crossplane.io/composition-resource-name: wk-vm-{{ $i }}
            spec:
              name: "{{ $s.metadata.name }}-wk-{{ $i }}"
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
                  {{- range $i := until (int $s.controlPlane.replicas) }}
                  - "{{ incrementIP $s.network.controlPlane.startIP $i }}"
                  {{- end }}
                networkAllocation:
                  subnet: {{ $s.network.controlPlane.subnet }}
                  gateway: {{ $s.network.controlPlane.gateway }}
                  startIP: {{ $s.network.controlPlane.startIP }}
                  startVMID: {{ $s.vmidBase }}
                  vip: {{ $s.network.controlPlane.vip }}
                  nameservers: {{ toYaml $s.network.nameservers | nindent 20 }}

    # Step 4: TalosMachines (generated by talos-operator from TalosControlPlane)
    # Not in composition — talos-operator creates these automatically
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
   ├── 1 TalosControlPlane CR (with networkAllocation)
   └── 1 TalosWorker CR
   |
4. kubemox reconciles VirtualMachine CRs:
   ├── Clones template with specified VMID
   ├── Configures disks with IOPS limits
   ├── Attaches SR-IOV VF via PCI passthrough
   ├── Reports MAC address in status
   └── Starts VMs
   |
5. talos-operator reconciles TalosControlPlane:
   ├── Sees networkAllocation: startIP=10.0.1.11, startVMID=10100
   ├── Creates TalosMachine CRs:
   │   ├── cp-0: endpoint=10.0.1.11, networkSpec={ip, mac, vip}
   │   ├── cp-1: endpoint=10.0.1.12, networkSpec={ip, mac, vip}
   │   └── cp-2: endpoint=10.0.1.13, networkSpec={ip, mac, vip}
   └── (MAC addresses come from kubemox VM status)
   |
6. talos-operator reconciles each TalosMachine:
   ├── Apply META key (pre-install network via metakey_tpl.go)
   ├── Generate machine config with:
   │   ├── Static network patch (deviceSelector.hardwareAddr)
   │   ├── VolumeConfig for data disks
   │   └── VIP for control plane
   ├── Apply config via Talos API
   ├── Talos installs → reboots at SAME IP
   └── Bootstrap etcd on first CP node
   |
7. Cluster ready at VIP 10.0.1.10:6443
```

---

## Summary of Changes by Project

### kubemox (5 changes)

| Change | File | Description |
|---|---|---|
| Add `MACAddress` to `VirtualMachineNetwork` | `api/proxmox/v1alpha1/virtualmachine_types.go` | Allow setting/reading MAC |
| Add `MACAddress` to `QEMUStatus` | Same file | Report MAC in status |
| Add `VMID` to `VirtualMachineSpec` | Same file | Deterministic VM IDs |
| Add `IOPSLimit`, `MBpsLimit`, `ISCSI` to `VirtualMachineDisk` | Same file | iSCSI + IOPS support |
| Pass VMID in `CloneOptions`, read MAC after clone | `pkg/proxmox/virtualmachine.go` | Wire up new fields |

### go-proxmox fork (1 change)

| Change | File | Description |
|---|---|---|
| Add `NewID` to `VirtualMachineCloneOptions` | `types.go` | Support `newid` in clone API |

### talos-operator (5 changes)

| Change | File | Description |
|---|---|---|
| Add `NetworkSpec` to `TalosMachineSpec` | `api/v1alpha1/talosmachine_types.go` | Static IP + SR-IOV config |
| Add `DataVolume` to `MachineSpec` | Same file | Multi-disk mount paths |
| Add `NetworkAllocation` to `MetalSpec` | `api/v1alpha1/taloscontrolplane_types.go` | Deterministic IP + VMID allocation |
| Static network patch generation | `pkg/talos/bundle.go` | New patch templates |
| Apply network + volume patches | `internal/controller/talosmachine_controller.go` | Wire into reconcile loop |

### Crossplane (new)

| Change | Description |
|---|---|
| XRD: `XTalosKubernetesCluster` | Composite resource definition |
| Composition pipeline | go-templating steps for VMs, TalosCP, TalosWorker |
| Claim: `TalosKubernetesCluster` | User-facing API |

---

## Key References

- [Official Talos Proxmox Guide (v1.11)](https://www.talos.dev/v1.11/talos-guides/install/virtualized-platforms/proxmox/)
- [Talos Metal Network Configuration](https://www.talos.dev/v1.10/advanced/metal-network-configuration/)
- [Talos Nocloud Documentation](https://docs.siderolabs.com/talos/v1.8/platform-specific-installations/cloud-platforms/nocloud)
- [Talos Image Factory](https://factory.talos.dev/)
- [Talos Static Addressing Docs](https://docs.siderolabs.com/talos/v1.12/networking/configuration/static)
- [JYSK Tech: 3000+ Clusters with NoCloud](https://jysk.tech/3000-clusters-part-3-how-to-boot-talos-linux-nodes-with-cloud-init-and-nocloud-acdce36f60c0)
- [siderolabs/omni-infra-provider-proxmox](https://github.com/siderolabs/omni-infra-provider-proxmox)
- [pfSense REST API](https://github.com/jaredhendrickson13/pfsense-api)
- [pfSense Go Client](https://github.com/sjafferali/pfsense-api-goclient)
- [GitHub Issue #11651 - Guest Agent in Maintenance Mode](https://github.com/siderolabs/talos/issues/11651)
- [Proxmox VM Cloning and MAC Behavior](https://forum.proxmox.com/threads/when-cloning-a-kvm-vm-the-mac-address-is-renewed.28153/)
- [Proxmox iSCSI Storage](https://pve.proxmox.com/wiki/Storage:_iSCSI)
- [Proxmox SR-IOV Resource Mappings](https://pve.proxmox.com/wiki/PCI_Passthrough)
