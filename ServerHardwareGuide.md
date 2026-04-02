# SLAM Server Hardware Specification

**Owners:** SLAM
**Applies To:** DAQ/experiment server-class deployments

---

## Prohibited

- **Intel VROC** — must not be used even if present in firmware
- **Pre-Turing NVIDIA GPUs** (Pascal/GTX 10-series, P100, Maxwell, Kepler, and older - 2018) - no GSP support, unsupported by the open-source `nvidia-open` driver stack
- **SFP/SFP+** for the primary facility network interface
- **Shared-backplane enclosures** (blade chassis, shared midplanes) - each node must occupy its own independent chassis.

---

## Platform

- `x86_64-v3` or newer (`aarch64` may be approved on a case by case basis)
- Server class hardware should have ECC RAM
- UEFI
- Secure Boot
- Hardware TPM v2
- Must appear on the current [RHEL Hardware Compatibility List](https://catalog.redhat.com/hardware)
- CPU and RAM must meet RHEL minimums and must not exceed RHEL maximums for the intended configuration; sizing is otherwise workload-dependent
- Must have at least 2 external USB ports for installation/crash/recovery

---

## Serviceability

Each node must be independently serviceable without affecting the rest of the rack:

- Any node must be removable from the rack without interrupting power distribution, network switching, or operation of any other node
- Chassis must have an easily operable rail system compatible with the installation rack
- Internal access must not require special tooling beyond standard hand tools
- VGA port for local console; HDMI/DisplayPort acceptable only if a compatible monitor and cables are **permanently** co-located with the system

Each installation must maintain a crash cart (monitor, keyboard, mouse, required cables) compatible with and usable on all deployed nodes. The crash cart is the responsibility of the experiment or installation owner.

A valid vendor hardware support contract covering repair/replacement, firmware updates, and support escalation is required for the duration of the node's operational lifetime. Contract expiration must be tracked in the asset management system.

> **Open question:** Minimum support tier (next-business-day on-site vs. depot return) - likely varies by installation geography.

---

## Power

- Redundant (N+1) power supplies required
- PSUs must be hot-swappable
- PSU status must be monitorable via BMC/IPMI

---

## Storage

### System Volume

The OS disk must be 255G or larger. The OS disk must be on a RAID array.

All drive bays used for RAID arrays must support hot-swap.

### NVMe

Configure as one of: `mdadm` software RAID (preferred for RAID1/RAID10), hardware RAID (see tooling requirements below), LVM on raw devices, or raw devices.

`mdadm` is preferred - it matches native NVMe performance for RAID1/RAID10, provides straightforward kernel-native monitoring, and avoids vendor lock-in. Hardware RAID is permitted where the controller is fully supported by the tooling in the table below.

Boot disk note: with `mdadm`, the EFI System Partition (ESP) exists on one member device only and must be explicitly replicated to all RAID members (e.g., via `efibootmgr` and a sync hook). Do not rely on UEFI firmware to discover a software-mirrored ESP.

### HDD and SATA SSD

Must use a Broadcom-based hardware RAID controller with battery backup (BBU or capacitor-backed write cache). RAID1 or RAID10 required for redundant arrays.  RAID5 or RAID6 may only be used when storage size is more important than storage/recovery speed.

Boot disk note: the hardware RAID controller presents a single logical volume to the OS. The ESP on that volume is implicitly mirrored - no additional replication is needed.

### RAID Management Tooling

| Tool | Scope |
|---|---|
| `storcli` | Broadcom MegaRAID, legacy generations |
| `storcli2` | Broadcom MegaRAID, Gen 3+ (9600-series and newer) |
| `perccli` / `perccli2` | Dell OEM rebrands of `storcli` / `storcli2`; functionally identical |

---

## Networking

- Onboard RJ45 Ethernet required for the primary facility network interface
- NIC must have an in-tree Linux kernel driver with a good upstream maintenance record and must appear on the RHEL HCL; Intel, QLogic, and Broadcom are established examples - other vendors require approval before procurement
- Network boot must support both PXE and UEFI HTTP boot over IPv4 **and** IPv6; note that some UEFI HTTP boot implementations have dual-stack quirks and must be validated before deployment
- Dedicated physical BMC port required, connected to an isolated out-of-band management network (separate VLAN or physical network)

---

## GPU

NVIDIA GPUs must be Turing generation (2018) or newer - RTX 20-series, Quadro RTX, Tesla T4, and all subsequent generations. GPUs are otherwise optional and workload-dependent.

---

## Firmware

Systems with LVFS/FWUPD support are preferred for simplified firmware lifecycle management. All systems must support firmware updates via standard tooling when mandated.

It must be possible to update the firmware from one of Linux, BMC, or on board firmware. Firmware updates must not require DOS or Windows.

**Current operational note:** Automatic firmware updates are disabled. Updates are applied through a controlled change process.

Secure Boot CA rotation must be supported. The current Microsoft CA expires June 2026. EK certificate provisioning behavior (on-chip vs. retrieved from vendor infrastructure) must be documented per vendor.

---

## Hardware TPM v2

Required. Must support EK certificate retrieval and standard attestation workflows (`tpm2-tools` / TPM2-TSS). Intended uses include encrypted-at-rest storage, secret-zero bootstrapping, and zero-trust identity anchoring.

This can be integrated into the motherboard, CPU, or some other method.

The TPM must have full Linux support.

---

## BMC

- Fully unlocked at procurement - no capability tiers or license-gated features
- License expiration tracked in monitoring or asset management system; renewal budgeted for the full support window
- Remote console (GUI/VNC or equivalent), remote firmware update, remote BIOS/UEFI configuration
- Ability to monitor power supply state/consumption
- IPMI over LAN (`lanplus`)
- Redfish 1.20 or newer, Redfish schema 2023.3 or newer
- Dedicated physical BMC network port (see Networking above)

---

## Open Questions

- Minimum vendor support tier (Serviceability)
- GPU vendor policy beyond NVIDIA GSP (GPU)
- NVMe hot-swap requirement - currently required for RAID arrays only; should it extend to single NVMe devices?

---

## Future Considerations

- Encrypted-at-rest by default for all new deployments
- BMC credential management database
- BMC license database
- Zero-trust infrastructure expansion
