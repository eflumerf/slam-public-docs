# SLAM Server Hardware Specification

**Owners:** SLAM

**Applies To:** DAQ/experiment server-class deployments

---

## Prohibited

- **Intel VROC** It must not be used even if present in firmware
- **NVIDIA GPUs not supported by the `nvidia-open` kernel modules** - the open modules require a GSP-equipped GPU (Turing, 2018, or newer); this excludes Volta/V100, Pascal/GTX 10-series/P100, Maxwell, Kepler, and older. Check the [NVIDIA open GPU kernel modules supported list](https://github.com/NVIDIA/open-gpu-kernel-modules) before procurement
- **SFP/SFP+** for the primary facility network interface
- **Shared-backplane enclosures** (blade chassis, shared midplanes) - each node must occupy its own independent chassis.
- **Internet dependencies** All components must operate in an air gapped network without an external route
- **Prototypes** Products not part of a standard production line must be approved in advance
- **End of Life** Any hardware component that is end of life at purchase time, or with an announced end of life within 18 months of purchase

---

## Platform

- `x86_64-v3` or newer (`aarch64` may be approved on a case by case basis)
  - "Hyperthreading"/"smt" must be possible
  - "Hyperthreading"/"smt" must not be mandatory
- Server class hardware should have ECC RAM
  - Systems must report NUMA topology correctly to the OS (verify via `numactl --hardware` post-installation)
  - Single-node NUMA is acceptable
- UEFI
- Secure Boot
- Hardware TPM v2 (see Hardware TPM v2 below)
- Hardware Virtualization support (VT-d/VT-x/AMD-V/AMD-Vi etc)
- Must appear on the current [RHEL Hardware Compatibility List](https://catalog.redhat.com/hardware)
- CPU and RAM must meet RHEL minimums and must not exceed RHEL maximums for the intended configuration; sizing is otherwise workload-dependent
- Must have at least 2 external USB ports dedicated for installation/crash/recovery
- Local video console port (see Serviceability below)

---

## Serviceability

Each node must be independently serviceable without affecting the rest of the rack:

- Any node must be removable from the rack without interrupting power distribution, network switching, or operation of any other node
- Rack mount chassis must have an easily operable rail system compatible with the 19 inch installation rack
- Internal access must not require special tooling beyond standard hand tools
- VGA port for local console
  -  HDMI/DisplayPort acceptable only if a compatible monitor and cables are **permanently** co-located with the system

Each installation must maintain a crash cart (monitor, keyboard, mouse, required cables) compatible with and usable on all deployed nodes. A rack KVM counts as a crash cart provided its cables can reach every node it serves (one per rack is sufficient if neighboring racks are within reach). The crash cart is the responsibility of the experiment or installation owner.

A valid vendor hardware support contract covering repair/replacement, firmware updates, and support escalation is required for the duration of the node's operational lifetime. Contract expiration must be tracked in the asset management system. Minimum support tier is an open question (see Open Questions).

---

## Power

- Redundant (N+1) power supplies required
- PSUs must be hot-swappable
- PSU status must be monitorable via BMC/IPMI

---

## Storage

All drive bays used for RAID arrays must support hot-swap.

Per-drive health and wear must be readable from Linux for every drive type (`nvme-cli` or `smartmontools`, including passthrough behind RAID controllers, e.g. `smartctl -d megaraid,N`). Controllers that obscure per-drive SMART data are prohibited.

### System Volume

The OS volume must be at least 256 GB usable (after any RAID). The OS volume must be on a RAID array.

### NVMe

Configure as one of: `mdadm` software RAID (preferred for RAID1/RAID10), hardware RAID (see tooling requirements below), LVM on raw devices, or raw devices.

`mdadm` is preferred - it matches native NVMe performance for RAID1/RAID10, provides straightforward kernel-native monitoring, and avoids vendor lock-in. Hardware RAID is permitted where the controller is fully supported by the tooling in the table below.

Boot disk note: with `mdadm`, create a separate EFI System Partition (ESP) on every RAID member. ESP contents must be synchronised by a hook (e.g. triggered on `grub2-mkconfig`, `fwupd`, or bootloader package updates), and each ESP must be registered as a boot entry with `efibootmgr`. Do not place the ESP itself on an `mdadm` device - UEFI firmware cannot read it, and RHEL does not support `/boot/efi` on software RAID.

### HDD and SATA SSD

Must use a Broadcom-based hardware RAID controller with battery backup (BBU or capacitor-backed write cache). Broadcom is required for standardization: one controller family and one toolset (`storcli`/`storcli2`) across all installations. RAID1 or RAID10 required for redundant arrays. RAID5 or RAID6 may only be used when storage size is more important than storage/recovery speed.

Boot disk note: the hardware RAID controller presents a single logical volume to the OS. The ESP on that volume is implicitly mirrored - no additional replication is needed.

### Hardware RAID Management Tooling

| Tool | Scope |
|---|---|
| `storcli` | Broadcom MegaRAID, legacy generations |
| `storcli2` | Broadcom MegaRAID, Gen 3+ (9600-series and newer) |
| `perccli` / `perccli2` | Dell OEM rebrands of `storcli` / `storcli2`; functionally identical |

---

## Networking

- Onboard RJ45 Ethernet required for the primary facility network interface
- NIC must have an in-tree Linux kernel driver with a good upstream maintenance record and must appear on the RHEL HCL; Intel, QLogic, Mellanox, and Broadcom are established examples - other vendors require approval before procurement
- Network interfaces of 10 Gbit/s or faster must support SR-IOV and RoCE v2, and must have an upstream DPDK poll-mode driver
- NICs must expose correct NUMA affinity and interrupt topology to the OS if relevant
- LLDP frames must reach the OS so that `lldpd`/`lldpad` can see the switch neighbour (chassis ID, port ID, VLAN) for topology auditing
  -  NOTE: NIC firmware LLDP agents (e.g. Intel `fw-lldp-agent`) consume LLDP frames before the OS sees them; they must be disableable and must remain disabled across reboots. Verify with `lldpcli show neighbors`
- Network boot must support both PXE and UEFI HTTP boot over IPv4 **and** IPv6
  -  NOTE: some UEFI HTTP boot implementations have dual-stack quirks and must be validated before deployment
- BMC traffic must not share the primary facility interface (see BMC below)

---

## GPU

GPUs are optional and workload-dependent.

NVIDIA GPUs must be supported by the `nvidia-open` kernel modules (see Prohibited). In practice this means Turing (2018) or newer - RTX 20-series, Quadro RTX, Tesla T4, and later.

---

## Firmware

Systems with LVFS/FWUPD support are preferred for simplified firmware lifecycle management. All systems must support firmware updates via standard tooling when mandated.

It must be possible to update the system firmware from at least one of: Linux, BMC, or "on board" firmware.

Firmware updates must be possible via Redfish.

Firmware updates for any component must not **require** DOS or Windows.

**Current operational note:** Automatic firmware updates are disabled. Updates are applied through a controlled change process.

Secure Boot key rotation must be supported. Systems must ship with the current Microsoft UEFI CAs and KEK enrolled (not the expired 2011 set). `db`/`dbx`/`KEK` updates must be applicable from Linux via `fwupd`. Owner enrolment of custom `db` keys and MOK must be supported.

---

## Hardware TPM v2

Required!

Must support EK certificate retrieval and standard attestation workflows (`tpm2-tools` / TPM2-TSS). Intended uses include encrypted-at-rest storage, secret-zero bootstrapping, and zero-trust identity anchoring.

EK certificate provisioning behavior (on-chip vs. retrieved from vendor infrastructure) must be documented per vendor. Retrieval must not require internet access from the node (see Prohibited: Internet dependencies).

This can be integrated into the motherboard, CPU, or some other method.

The TPM must have full Linux support.

---

## BMC

- All BMC features (remote console, virtual media, Redfish, firmware update) must be included with perpetual licenses at procurement - no capability tiers
- Where a vendor only offers time-limited licenses, expiration must be tracked in monitoring or the asset management system and renewal budgeted for the full support window
- BMC must operate independently of OS state, including during OS installation, kernel panic, and power-cycle recovery
- Remote Serial console (SOL), remote GUI console (GUI/VNC or equivalent), remote firmware update, remote BIOS/UEFI configuration
- Ability to monitor power supply state/consumption
- Ability to view fan speed
- Ability to view system temperatures
- IPMI over LAN (`lanplus`)
- Redfish 1.20 or newer, Redfish schema 2023.3 or newer
- BMC configuration (users, network, BIOS settings) must be fully exportable and importable via Redfish
- Dedicated physical BMC network port, connected to an isolated out-of-band management network (separate VLAN or physical network)

---

## Open Questions

- Minimum vendor support tier (Serviceability): next-business-day on-site vs. depot return - likely varies by installation geography
- GPU vendor policy beyond NVIDIA `nvidia-open` (GPU)
- NVMe hot-swap requirement - currently required for RAID arrays only; should it extend to single NVMe devices?

---

## Future Considerations

- Encrypted-at-rest by default for all new deployments
- BMC credential management database
- BMC license database
- Zero-trust infrastructure expansion
