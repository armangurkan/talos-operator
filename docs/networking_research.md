# Talos Linux on Proxmox: Network Configuration Research

> **Date**: 2026-02-24
> **Status**: Research findings — informing design of talos-operator + kubemox networking integration
> **Context**: We need deterministic, persistent IPs for Talos VMs on Proxmox so that talos-operator can reliably reach nodes before, during, and after Talos installation.

---

## The Core Problem

When kubemox clones a Talos VM and boots it on Proxmox:

1. **Pre-install**: VM boots from ISO, gets a DHCP IP `X`
2. **Config apply**: talos-operator uses IP `X` to apply machine config via Talos API
3. **Talos installs**: Talos writes to disk and reboots
4. **Post-install**: Talos brings up networking per its *own* config — may get DHCP IP `Y` (different from `X`)
5. **Lost contact**: talos-operator can no longer reach the node at `X`

Additionally, the **QEMU guest agent does not run on Talos by default** (immutable OS, no package manager), so Proxmox cannot report the VM's IP through `AgentGetNetworkIFaces()`. The kubemox status field for IP would remain empty/nil for stock Talos VMs.

**Conclusion**: Pure DHCP without reservations is not viable for this integration. We need a deterministic IP strategy.

---

## Solution Options (Ranked)

### Option 1: Static IP in Talos Machine Config (Recommended)

Configure the IP directly in the Talos machine configuration YAML. This is the most reliable approach because Talos owns its own networking.

```yaml
machine:
  network:
    interfaces:
      - deviceSelector:
          busPath: "0000:00:12.0"  # or use hardwareAddr for MAC-based selection
        dhcp: false
        addresses:
          - 192.168.1.11/24
        routes:
          - network: 0.0.0.0/0
            gateway: 192.168.1.1
        vip:
          ip: 192.168.1.9  # optional: VIP for HA control plane
```

**Key details**:
- Interface name is typically `ens18` in Proxmox VMs, but can vary
- Using `deviceSelector` with `hardwareAddr` (MAC address) is **more reliable** than specifying interface name directly
- Must set `dhcp: false` when using static — cannot mix DHCP and static on the same /24
- The static IP persists across reboots and reinstalls since it's in the machine config

**How it fits our operators**:
- **kubemox**: Creates VM, sets MAC address on the NIC (deterministic), reports MAC in status
- **talos-operator**: Takes the user-specified static IP + MAC, generates machine config with the static network block, applies via `talosctl apply-config`

### Option 2: Nocloud Image + Cloud-Init (For Pre-Install Networking)

Use the Talos **nocloud** image (not the metal image) which supports cloud-init. Proxmox can then inject network config via cloud-init before Talos boots.

**Two delivery methods**:

#### A. Local Attached Storage (CIDATA volume)
- Create a VFAT or ISO9660 filesystem with volume label `cidata` or `CIDATA`
- Place `user-data` (Talos machine config) and `network-config` files on it
- Talos reads these **before any network is established** — no DHCP needed at all
- Network config uses **version 1** format:
  ```yaml
  version: 1
  config:
    - type: physical
      name: eth0
      mac_address: "BC:24:11:xx:xx:xx"
      subnets:
        - type: static
          address: 192.168.1.11/24
          gateway: 192.168.1.1
  ```

#### B. SMBIOS Serial (nocloud-net)
- Set VM's SMBIOS serial to `ds=nocloud-net;s=http://config-server/configs/`
- Talos fetches `user-data` and `network-config` from that URL after initial DHCP
- Requires network to be up first (chicken-and-egg for non-DHCP networks)
- Proxmox supports this via VM Options > SMBIOS Settings

**Critical version caveat**:
- Talos **1.7.x**: `nocloud` images available on GitHub releases, cloud-init works
- Talos **1.8.0+**: Switched to `metal` images that **ignore cloud-init entirely**
- For 1.8+, nocloud images must be obtained from [Talos Image Factory](https://factory.talos.dev/)
- The `metal` image is what the official Proxmox guide uses — it does NOT support cloud-init

**Proxmox integration**:
```bash
# Set custom cloud-init config
qm set 100 --cicustom user=local:snippets/controlplane-1.yml
# Snippet must be placed at /var/lib/vz/snippets/ manually
# Then click "Regenerate Image" in Proxmox UI
```

### Option 3: DHCP Reservations (MAC-Based)

Bind a fixed IP to the VM's MAC address in the DHCP server (router). The VM always gets the same IP without any Talos config changes.

**How to get MAC addresses**:
1. **Proxmox VM config**: `/etc/pve/qemu-server/<VMID>.conf` contains MAC for each NIC
2. **Proxmox UI**: VM > Hardware > Network Device
3. **Proxmox API**: `GET /nodes/{node}/qemu/{vmid}/config` returns `net0` with MAC
4. **ARP table**: Ping the IP, then `arp -a`

**Pros**:
- No Talos config changes needed — Talos defaults to DHCP and always gets the same IP
- Simple for small clusters

**Cons**:
- Requires external DHCP server configuration (not managed by our operators)
- Not GitOps-friendly — the reservation lives outside the cluster manifests
- Breaks if someone changes the DHCP server config

### Option 4: META-Based Network Configuration (Advanced)

Available since Talos 1.4.0 for the `metal` platform. Network config is embedded in the boot image or passed via `INSTALLER_META_BASE64` environment variable.

**How it works**:
- When creating boot assets with `imager`, pass `--meta` flag with network config
- The config is used immediately at boot AND written to the META partition on install
- Survives reboots and reinstalls

**Useful for**:
- Bare-metal / air-gapped environments
- When you need networking before machine config is applied
- When cloud-init (nocloud) is not available

**Limitation**: Requires building custom boot images per-node (each with its own IP), making it less practical for dynamic provisioning.

---

## QEMU Guest Agent: Making Proxmox Report Talos VM IPs

By default, Talos does not include the QEMU guest agent. Without it:
- Proxmox UI shows no IP for the VM
- `AgentGetNetworkIFaces()` API call returns nothing
- kubemox cannot populate `status.status.IPAddress`
- Terraform cannot detect when VMs are ready

**Solution**: Build a custom Talos image with `siderolabs/qemu-guest-agent` extension via [Image Factory](https://factory.talos.dev/).

**Schematic YAML**:
```yaml
customization:
  systemExtensions:
    officialExtensions:
      - siderolabs/qemu-guest-agent
```

**Methods to build**:
1. **Web UI**: Go to https://factory.talos.dev/, select version, tick `qemu-guest-agent`, get ISO URL
2. **API**: POST the schematic to `https://factory.talos.dev/schematics`, get a schematic ID, use it in image URLs
3. **Machine config**: Set install image to `factory.talos.dev/installer/<schematic-id>:v1.x.x`

**Important**: The extension must be in **both** the boot ISO and the install image. If you upgrade Talos with a different OCI image, the extension is lost.

After installation, verify with: `talosctl get extensions`

---

## Recommended Architecture for Our Operators

Given these findings, the recommended approach for talos-operator + kubemox integration:

### Layer 1: kubemox (VM Provisioning)
1. Use the **nocloud** Talos image (from Image Factory, with `qemu-guest-agent` extension)
2. Set a **deterministic MAC address** on the VM NIC
3. Inject **cloud-init** with:
   - `user-data`: Talos machine config (from talos-operator)
   - `network-config`: Static IP assignment (v1 format, matched to MAC)
4. Set `ipconfig0=ip=X.X.X.X/24,gw=Y.Y.Y.Y` via Proxmox cloud-init API
5. Report the assigned IP and MAC in kubemox VM status

### Layer 2: talos-operator (Talos Lifecycle)
1. Generate machine config with **static network interface** block matching the kubemox-assigned IP
2. Use `deviceSelector.hardwareAddr` to bind config to the correct NIC by MAC
3. Apply config via `talosctl apply-config` to the known static IP
4. The IP remains stable through install, reboot, and upgrades

### Layer 3: Control Plane VIP
For HA control planes, configure a floating VIP in the machine config:
```yaml
machine:
  network:
    interfaces:
      - deviceSelector:
          hardwareAddr: "BC:24:11:xx:xx:xx"
        dhcp: false
        addresses:
          - 192.168.1.11/24
        routes:
          - network: 0.0.0.0/0
            gateway: 192.168.1.1
        vip:
          ip: 192.168.1.9  # shared across all control plane nodes
```

### IP Assignment Flow
```
User specifies in TalosCluster CR:
  - controlPlaneIP: 192.168.1.11 (or range: 192.168.1.11-13)
  - gateway: 192.168.1.1
  - vip: 192.168.1.9

talos-operator:
  1. Validates IPs are available
  2. Creates TalosMachine CRs with assigned IPs
  3. Generates machine configs with static network blocks

kubemox (watches TalosMachine):
  1. Creates Proxmox VM with deterministic MAC
  2. Attaches nocloud image + cloud-init with IP config
  3. Reports VM status (IP, MAC, VMID)

talos-operator (watches kubemox status):
  1. Applies machine config to the static IP
  2. Bootstraps etcd, waits for Talos install
  3. Node comes back at the SAME IP after reboot
  4. Proceeds with cluster bootstrap
```

---

## Alternative: Crossplane v2 Composition for Deployment

For the deployment layer, a **Crossplane v2 composition** can orchestrate the full stack:
- Compose kubemox `VirtualMachine` resources + talos-operator `TalosCluster` resources
- Handle IP allocation via a Crossplane function (e.g., IPAM provider)
- GitOps-friendly: entire cluster defined as a single Crossplane Claim
- Enables self-service Kubernetes cluster provisioning

This would sit above both operators and provide the user-facing API.

---

## Key References

- [Official Talos Proxmox Guide (v1.11)](https://www.talos.dev/v1.11/talos-guides/install/virtualized-platforms/proxmox/)
- [Talos Metal Network Configuration](https://www.talos.dev/v1.10/advanced/metal-network-configuration/)
- [Talos Nocloud Documentation](https://docs.siderolabs.com/talos/v1.8/platform-specific-installations/cloud-platforms/nocloud)
- [Talos Image Factory](https://factory.talos.dev/)
- [JYSK Tech: 3000+ Clusters with NoCloud](https://jysk.tech/3000-clusters-part-3-how-to-boot-talos-linux-nodes-with-cloud-init-and-nocloud-acdce36f60c0)
- [kubebn/talos-proxmox-kaas](https://github.com/kubebn/talos-proxmox-kaas)
- [siderolabs/omni-infra-provider-proxmox](https://github.com/siderolabs/omni-infra-provider-proxmox)
- [Cluster API + Talos + Proxmox](https://a-cup-of.coffee/blog/talos-capi-proxmox/)
- [GitHub Discussion #9291 - Configuring Talos in Proxmox](https://github.com/siderolabs/talos/discussions/9291)
- [GitHub Discussion #9446 - Static IP for Control Plane](https://github.com/siderolabs/talos/discussions/9446)
- [GitHub Discussion #8509 - Getting MAC Address](https://github.com/siderolabs/talos/discussions/8509)
- [GitHub Discussion #6970 - Automated Install on Proxmox](https://github.com/siderolabs/talos/discussions/6970)
- [GitHub Discussion #11175 - Talos 1.10 and cloud-init](https://github.com/siderolabs/talos/discussions/11175)
- [DEV Community: Fortress Kubernetes Cluster](https://dev.to/jorisvilardell/building-a-fortress-kubernetes-cluster-talos-linux-proxmox-and-network-isolation-1p4g)
- [Talos on Proxmox with Terraform (Stonegarden)](https://blog.stonegarden.dev/articles/2024/08/talos-proxmox-tofu/)
- [Talos on Proxmox with Terraform (xoid.net)](https://xoid.net/2024/07/27/talos-terraform-proxmox.html)
- [TechDufus: Talos Homelab with Terraform](https://techdufus.com/tech/2025/06/30/building-a-talos-kubernetes-homelab-on-proxmox-with-terraform.html)
- [Secsys: Talos with Kubernetes on Proxmox](https://secsys.pages.dev/posts/talos/)
