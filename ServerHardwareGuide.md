# SLAM Server Hardware Guide

**Owners:** SLAM

**Applies To:** server-class deployments

**Last Reviewed:** 2026-09-30

---

## Purpose and Scope

This document defines the hardware requirements for server-class systems
procured for DAQ and experiment use and supported by SLAM. Systems are
expected to run for 10+ years across multiple RHEL/AlmaLinux major
versions, often on air-gapped networks, with limited on-site staff. The
requirements below prioritize long-term Linux support, remote
serviceability, and uniform tooling over peak performance or lowest
purchase price. Installation and lifecycle items that directly affect
hardware serviceability (crash carts, support contracts, rack layout) are
in scope.

### Requirement Levels

The key words MUST, MUST NOT, SHOULD, SHOULD NOT, and MAY are used as
defined in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119). In short:

- **MUST** / **MUST NOT**: absolute requirement. Non-compliance requires a
  written exception.
- **SHOULD** / **SHOULD NOT**: recommended. Deviations must be justified
  in the procurement request.
- **MAY**: optional.

### Exceptions and Approvals

Any item in this document marked as requiring approval, and any deviation
from a MUST, is approved by **SLAM**. Approval must be obtained before
procurement and recorded with the purchase request.

### Requirement IDs

Every requirement has a stable ID (e.g. `SLAM-2026-PLAT-03`). Reference IDs in
quotes, tickets, and exception requests rather than quoting the text, so
that references survive rewording. IDs denote when they were added.

### Verify Column

The Verify column names how compliance is confirmed. `Quote` means the
item is checked against the vendor quote or datasheet before purchase.
Commands are run on the delivered system from a RHEL/AlmaLinux install.
Detailed acceptance procedures will be provided in a separate checklist.

### Fleet Standardization

When procuring multiple systems, care SHOULD be taken to minimize the
number of different parts (for example hard disks, RAID controllers,
power supplies, and network interfaces). Fewer distinct components
reduce spares inventory, monitoring complexity, firmware variance, and
operational risk across the service life.

---

## Prohibited

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-PROH-01 | MUST NOT | Use Intel VROC, even if present in firmware | BIOS settings (VROC/RSTe disabled); `mdadm --detail-platform`; no `imsm` container in `/proc/mdstat` |
| SLAM-2026-PROH-02 | MUST NOT | Include NVIDIA GPUs unsupported by the `nvidia-open` kernel modules | Quote; `lspci -nn` |
| SLAM-2026-PROH-03 | MUST NOT | Depend on DOS or Windows for any firmware update of any component | Vendor documentation |
| SLAM-2026-PROH-04 | MUST NOT | Use shared-backplane enclosures (blade chassis, shared midplanes). Each node must occupy its own independent chassis | Quote |
| SLAM-2026-PROH-05 | MUST NOT | Depend on internet access for any component function. All components must operate on an air-gapped network without an external route | Quote; vendor documentation |
| SLAM-2026-PROH-06 | MUST NOT | Procure prototypes or products not part of a standard production line | Quote |
| SLAM-2026-PROH-07 | MUST NOT | Procure any hardware component that is end of life at purchase time, or with an announced end of life within 18 months of purchase | Quote; vendor lifecycle notice |

### Rationale

- **No VROC (PROH-01)**: VROC is `mdadm` software RAID with Intel IMSM
  metadata, gated by platform firmware and a hardware license key. It
  offers nothing over plain `mdadm` and ties the array to a specific
  motherboard generation, so a board replacement can strand the data.
- **No unsupported NVIDIA GPUs (PROH-02)**: see the GPU rationale
  (GPU-01).
- **No DOS or Windows for firmware updates (PROH-03)**: nodes are
  Linux-only and often air-gapped. An update path that needs another OS
  is an update path that will not be exercised, leaving known firmware
  defects in place for the service life.
- **No shared backplanes (PROH-04)**: a chassis midplane or chassis
  manager is a shared failure domain and a shared lifecycle. One fault
  can take down every node in the enclosure, and every node is retired
  when the enclosure is.
- **No internet dependency (PROH-05)**: many installations have no
  external route. Any component that requires licensing, activation, or
  update servers to function will fail there, possibly long after
  acceptance.
- **No prototypes, no near-EOL parts (PROH-06, PROH-07)**: a 10-year
  service life needs a spares pipeline, firmware updates, and a support
  contract. Prototypes have none of these; parts nearing end of life
  lose them early. The 18-month window gives time to reach steady-state
  operation before the vendor stops shipping replacements.

---

## Platform

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-PLAT-01 | MUST | CPU implements `x86_64-v3` or newer | on RHEL 9+ `ld.so --help` lists `x86-64-v3` as supported |
| SLAM-2026-PLAT-02 | MUST | SMT ("Hyperthreading") if available, can be disabled in firmware | BIOS settings; `lscpu` (Thread(s) per core) |
| SLAM-2026-PLAT-03 | MUST | ECC RAM is present | `dmidecode -t memory` (Error Correction Type) |
| SLAM-2026-PLAT-04 | MUST | ECC error reporting is functional | `ras-mc-ctl --status`; EDAC driver loaded (`lsmod \| grep edac`) |
| SLAM-2026-PLAT-05 | MUST | NUMA topology is reported correctly to the OS. Single-node NUMA is acceptable | `numactl --hardware` |
| SLAM-2026-PLAT-06 | MUST | UEFI boot | `ls /sys/firmware/efi` |
| SLAM-2026-PLAT-07 | MUST | Secure Boot capable | `mokutil --sb-state` |
| SLAM-2026-PLAT-08 | MUST | Hardware virtualization support (VT-x or AMD-V) | `grep -E 'vmx|svm' /proc/cpuinfo`; `virt-host-validate` |
| SLAM-2026-PLAT-09 | MUST | IOMMU support (VT-d, AMD-Vi, or equivalent) | `virt-host-validate`; `dmesg \| grep -i iommu` |
| SLAM-2026-PLAT-10 | MUST | Listed on the current [RHEL Hardware Compatibility List](https://catalog.redhat.com/hardware) | Quote; catalog lookup |
| SLAM-2026-PLAT-11 | MUST | CPU and RAM meet RHEL minimums and do not exceed RHEL maximums for the intended configuration | Quote |
| SLAM-2026-PLAT-12 | MUST | At least 2 unused external USB ports available for installation/crash/recovery | Physical inspection |
| SLAM-2026-PLAT-13 | MUST | Local video console port (see SLAM-2026-SERV-06) | Physical inspection |
| SLAM-2026-PLAT-14 | MUST | Hardware watchdog timer usable from Linux via an in-tree driver and exposed as `/dev/watchdog` or `/dev/watchdogN` | `ls /dev/watchdog*`; `wdctl` |
| SLAM-2026-PLAT-15 | SHOULD | DIMMs are populated in a balanced configuration across memory channels per CPU vendor guidelines | `dmidecode -t memory`; vendor population guidelines |

`aarch64` MAY be approved on a case by case basis as an alternative to
SLAM-2026-PLAT-01.

Hardware TPM requirements are in the Hardware TPM v2 section.

### Rationale

- **`x86_64-v3` (PLAT-01)**: RHEL 10 is built for `x86-64-v3` and will
  not boot on older CPUs. A system that only meets the RHEL 9 baseline
  (`x86-64-v2`) is capped at RHEL 9 and cannot follow the fleet through
  the next major version. `aarch64` is the approved alternative when a
  workload requires it.
- **SMT switchable (PLAT-02)**: some DAQ workloads need SMT off for
  latency determinism; others gain throughput from it. The choice must
  be made per system in firmware, not fixed by the hardware.
- **ECC present and reporting (PLAT-03, PLAT-04)**: ECC corrects
  single-bit errors, but without functional EDAC reporting a DIMM that
  is degrading is invisible until it produces an uncorrectable error.
  Reporting turns a crash into a scheduled DIMM swap.
- **Correct NUMA topology (PLAT-05)**: DAQ processes and NIC interrupts
  are pinned to the node that owns their memory and PCIe root. If the
  firmware misreports topology, pinning is silently wrong and
  performance is unpredictable.
- **UEFI and Secure Boot (PLAT-06, PLAT-07)**: UEFI is the prerequisite
  for capsule firmware updates (FW-02), Secure Boot key management
  (FW-06, FW-07), and TPM measured boot (TPM section). Legacy BIOS boot
  offers none of these.
- **Virtualization and IOMMU (PLAT-08, PLAT-09)**: VMs and containers
  isolate DAQ components from each other and from the host. The IOMMU
  is required for safe PCIe device passthrough and for SR-IOV (NET-11),
  and protects the host from DMA by misbehaving devices.
- **RHEL HCL and configuration limits (PLAT-10, PLAT-11)**: Red Hat
  support is conditional on certified hardware within documented
  limits. Meeting both keeps the vendor contract and the Red Hat
  subscription usable together when a fault has to be escalated.
- **USB ports and local console (PLAT-12, PLAT-13)**: when network,
  BMC, or boot storage has failed, installation from USB media with a
  crash cart (SERV-06, SERV-07) is the last recovery path. Two ports
  allow a keyboard and install media at the same time.
- **Hardware watchdog (PLAT-14)**: a hung kernel on a remote site with
  no one present cannot be power-cycled by hand. An in-tree watchdog
  driven by `systemd` is the only automatic recovery path.
- **Balanced DIMM population (PLAT-15)**: unbalanced channels reduce
  memory bandwidth and can change interleaving behavior between
  otherwise identical nodes, producing performance differences that
  are hard to diagnose.

---

## Serviceability

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-SERV-01 | MUST | Any node is removable from the rack without interrupting power distribution, network switching, or operation of any other node | Quote; rack design review |
| SLAM-2026-SERV-02 | MUST | Rack-mount chassis has an easily operable rail system compatible with the 19 inch installation rack | Quote; physical inspection |
| SLAM-2026-SERV-03 | MUST | Internal access requires no special tooling beyond standard hand tools | Physical inspection |
| SLAM-2026-SERV-04 | MUST | A vendor hardware support contract covering repair/replacement, firmware update availability, and support escalation is in place for the node's operational lifetime | Quote; contract |
| SLAM-2026-SERV-05 | MUST | The chassis serial number matches across the physical label, `dmidecode -s system-serial-number`, and Redfish `/Systems/*` | Physical inspection; `dmidecode`; Redfish `GET /redfish/v1/Systems/*` |
| SLAM-2026-SERV-06 | MUST | VGA port for local console. DVI/HDMI/DisplayPort is acceptable only if a compatible monitor and cables are **permanently** co-located with the system | Physical inspection |
| SLAM-2026-SERV-07 | MUST | Each installation maintains a crash cart (monitor supporting all video interfaces, keyboard, mouse, required cables) usable on all deployed nodes. The crash cart is the responsibility of the experiment or installation owner. One crash cart is recommended for every two physical racks of systems. The minimum required number will be determined by SLAM taking into account data center layout and available space | Installation review |
| SLAM-2026-SERV-08 | MAY | A rack mount KVM replaces a crash cart under SLAM-2026-SERV-07, provided a KVM is in every rack the crash cart would service and its cables reach all servers in neighboring racks | Installation review |
| SLAM-2026-SERV-09 | MUST | Each installation has at least one mobile crash cart regardless of SLAM-2026-SERV-08 | Installation review |

Minimum hardware support tier is an open question (see SLAM-2026-OQ-01).

### Rationale

- **Independent removal (SERV-01)**: repairing one node must not take
  down its neighbors. Shared power strips, shared uplinks, or cabling
  routed across other nodes turn a single repair into a multi-node
  outage.
- **Rails and standard tools (SERV-02, SERV-03)**: installations are
  remote with limited on-site staff. A node that slides out on rails and
  opens with a screwdriver can be serviced by whoever is present; one
  needing a lift or a vendor-specific tool waits for a site visit.
- **Support contract (SERV-04)**: the contract is what provides
  replacement parts, firmware updates, and an escalation path for the
  full service life. Without it a hardware fault becomes a procurement
  exercise.
- **Consistent serial numbers (SERV-05)**: the physical label, DMI, and
  Redfish are the three places a serial number is read: by a technician,
  by the OS inventory, and by out-of-band automation. If they disagree,
  asset records and support-contract claims cannot be matched to the
  node in the rack.
- **VGA console (SERV-06)**: crash carts and rack KVMs are overwhelmingly
  VGA. HDMI or DisplayPort is only acceptable when a matching monitor
  and cables are guaranteed to be at the rack, because a console with no
  matching display is no console.
- **Crash cart coverage (SERV-07, SERV-08, SERV-09)**: one cart per two
  racks keeps a console within reach of every node. A rack KVM may
  substitute for the common case, but the mobile cart is still required
  because the KVM itself can fail and cables do not reach nodes in
  neighboring racks.

---

## Power

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-PWR-01 | MUST | Redundant (N+1) power supplies | Quote; physical inspection |
| SLAM-2026-PWR-02 | MUST | PSUs are hot-swappable | Quote |

PSU state and consumption monitoring through the BMC is required by
SLAM-2026-BMC-14.

### Rationale

- **Redundant, hot-swap PSUs (PWR-01, PWR-02)**: PSU failure is a
  common field repair. With N+1 redundancy the node keeps running
  on the remaining supply; with hot-swap the replacement is a scheduled
  visit rather than a shutdown. Monitoring through the BMC (BMC-14) is
  what makes the failure visible before the second supply fails.

---

## Storage

### General

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-STOR-01 | MUST | All drive bays used for RAID arrays support hot-swap | Quote |
| SLAM-2026-STOR-02 | MUST NOT | Use RAID controllers that obscure per-drive SMART data | Quote; `smartctl --scan-open`; `storcli /c0 show` |
| SLAM-2026-STOR-03 | MUST | Per-drive health and wear is readable from Linux for every drive type | `smartctl --scan-open`; `nvme list`; `nvme smart-log` |
| SLAM-2026-STOR-04 | MUST | Per-drive health and wear is readable for drives behind RAID controllers | `smartctl -a -d megaraid,N /dev/sdX` |
| SLAM-2026-STOR-05 | MUST | SSDs provide power-loss protection for in-flight writes | Vendor datasheet; `smartctl -a` (PLP/Power Loss Protection field) |

### HDD and SATA SSD

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-STOR-06 | MUST | Broadcom-based hardware RAID controller with a write cache that survives power loss and requires no scheduled replacement over the node's service life | Quote; `storcli /c0 show` |
| SLAM-2026-STOR-07 | MUST | RAID1 or RAID10 for redundant arrays. RAID5 or RAID6 MAY be used only when capacity matters more than performance and rebuild time | `storcli /c0 /vall show` |

### NVMe

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-STOR-08 | MUST | NVMe is configured as one of: `mdadm` software RAID, hardware RAID (controller supported per SLAM-2026-STOR-13), LVM on raw devices, or raw devices | `lsblk` |
| SLAM-2026-STOR-09 | MUST | NVMe drives are installed in pairs | Quote; `lsblk` |
| SLAM-2026-STOR-10 | MUST | RAID is the expected NVMe configuration | Quote; `lsblk` |
| SLAM-2026-STOR-11 | SHOULD | `mdadm` is used for NVMe RAID1/RAID10 | `cat /proc/mdstat` |
| SLAM-2026-STOR-12 | SHOULD | NVMe drives are installed in hot-swappable bays (e.g. U.2, U.3). Non-hot-swap form factors such as M.2 require justification in the procurement request | Quote; physical inspection |

### Hardware RAID Management Tooling

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-STOR-13 | MUST | Any hardware RAID controller is fully manageable from Linux with one of the tools below | Tool runs and enumerates the controller |

| Tool | Scope |
|---|---|
| `storcli` | Broadcom MegaRAID, legacy generations |
| `storcli2` | Broadcom MegaRAID, Gen 3+ (9600-series and newer) |
| `perccli` / `perccli2` | Dell OEM rebrands of `storcli` / `storcli2`; functionally identical |

### System Volume

The system volume holds the operating system, installed software, and
system state. It is not required to include `/home` or other
user-writable directories; those MAY reside on a separate filesystem or
on a data volume when workload needs justify it.

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-STOR-14 | MUST | OS volume is at least 256 GB usable (after any RAID) | `lsblk` |
| SLAM-2026-STOR-15 | MUST | OS volume resides on a RAID array | `lsblk`; `cat /proc/mdstat` or `storcli /c0 show` |

### Data Volume(s)

Data volumes are storage volumes used for experiment or DAQ data. They are
distinct from the system volume covered by SLAM-2026-STOR-14 and
SLAM-2026-STOR-15. A data volume MAY reside on the same physical RAID
array as the system volume when the combined capacity and performance
meet the workload requirements.

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-STOR-16 | MUST | Data volumes reside on a RAID array | `lsblk`; `cat /proc/mdstat` or `storcli /c0 show` |

### Rationale

- **Hot-swap RAID bays (STOR-01)**: the point of RAID is to survive a
  drive failure without downtime. If replacing the failed member
  requires powering off the node, the downtime has only been deferred.
- **Per-drive SMART, including behind RAID (STOR-02, STOR-03,
  STOR-04)**: STOR-02 prohibits controllers that obscure per-drive
  SMART data. STOR-03 and STOR-04 require that wear and pre-failure
  indicators are readable from Linux for every drive type and for drives
  behind RAID controllers. These indicators are the only warning before
  a drive drops out and a rebuild starts; hiding them turns every
  failure into a surprise.
- **Power-loss protection (STOR-05, STOR-06)**: there are two volatile
  write caches between the OS and the media, and each needs its own
  protection.
  - SSDs buffer writes in on-drive DRAM. Only on-drive PLP (capacitors
    that flush DRAM to NAND on power loss) protects that data. A RAID
    controller's protected cache does not cover it, and under `mdadm`
    NVMe there is no controller at all, so PLP is required on every SSD.
  - HDDs have no PLP equivalent. Behind hardware RAID, the protection is
    the controller's protected write cache, with the drive's own
    volatile cache left disabled so that the controller cache is the
    only volatile stage. Broadcom controllers default to this when their
    cache is protected; `storcli /c0 /vall show` reports it as
    `Disk Cache Policy`.
  - The "no scheduled replacement" clause excludes battery-only cache
    protection. A battery is a consumable with a 2-3 year life, and when
    it fails the controller silently falls back to write-through: a
    performance drop nobody sees and a maintenance visit nobody makes on
    a remote site. Capacitor-backed flash cache (Broadcom CacheVault and
    equivalents) meets the requirement without a maintenance item.
- **RAID1/RAID10 default (STOR-07)**: a RAID5/6 rebuild on large drives
  is a multi-day window in which a second failure loses the array, on a
  system with no on-site staff. Mirrors rebuild in hours and are the
  default; parity RAID is reserved for cases where capacity outweighs
  that exposure.
- **NVMe configuration (STOR-08, STOR-10)**: the allowed configurations
  are the ones standard RHEL tooling manages and monitors. RAID is the
  expected choice; raw or LVM-on-raw is permitted for workloads that
  provide redundancy elsewhere, but the exception must be recorded
  so it is not mistaken for an oversight later.
- **NVMe in pairs (STOR-09)**: RAID1/RAID10 needs an even member count.
  Buying in pairs keeps the RAID path open even when a system is
  initially deployed with raw devices.
- **`mdadm` for NVMe (STOR-11)**: `mdadm` RAID1/RAID10 matches native
  NVMe performance, is monitored by the kernel and standard tooling, and
  has no vendor dependency. Hardware RAID is allowed only where the
  controller is fully covered by STOR-13.
- **NVMe hot-swap bays (STOR-12)**: a failed U.2/U.3 drive is replaced
  from the front of the chassis. M.2 requires opening the node, which
  makes the replacement a chassis service event and usually a shutdown.
- **Broadcom, one toolset (STOR-06, STOR-13)**: one controller family
  and one management tool across all installations means one set of
  monitoring scripts, one spares pool, and one runbook. A controller
  that cannot be managed from Linux cannot be monitored or repaired
  remotely.
- **System volume (STOR-14, STOR-15)**: 256 GB leaves room for several
  installed kernels, crash dumps, logs, and an in-place major-version
  upgrade. Placing the OS on RAID means a single boot-drive failure does
  not take the node offline.
- **Data volumes on RAID (STOR-16)**: experiment and DAQ data must
  survive a single drive failure without on-site intervention.

---

## Networking

### Facility and Management Networking

These requirements apply to the primary facility/management network
interface and to any network port used for provisioning, remote
administration, or facility traffic.

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-NET-01 | SHOULD | Onboard RJ45 Ethernet for the primary facility network interface. Optical or external network cards must justify their use | Quote; `ethtool <if>` (Port: Twisted Pair) |
| SLAM-2026-NET-02 | MUST | NIC has an in-tree driver in the current RHEL kernel that is not on the RHEL deprecated-driver list | `ethtool -i <if>`; RHEL release notes (deprecated devices) |
| SLAM-2026-NET-03 | MUST | NIC appears on the RHEL HCL | Quote; catalog lookup |
| SLAM-2026-NET-04 | MUST | Intel, QLogic, Mellanox, and Broadcom NICs are pre-approved vendors | Quote |
| SLAM-2026-NET-05 | MUST | NICs expose correct NUMA affinity and interrupt topology to the OS where relevant | `cat /sys/class/net/<if>/device/numa_node`; `/proc/interrupts` |
| SLAM-2026-NET-06 | MUST | LLDP frames reach the OS so that `lldpd`/`lldpad` can see the switch neighbour (chassis ID, port ID, VLAN) | `lldpcli show neighbors` |
| SLAM-2026-NET-07 | MUST | NIC firmware LLDP agents (e.g. Intel `fw-lldp-agent`) that consume LLDP frames before the OS are disableable and remain disabled across reboots | `ethtool --show-priv-flags <if>`; `lldpcli show neighbors` after reboot |
| SLAM-2026-NET-08 | MUST | Network boot supports PXE and UEFI HTTP boot over IPv4 | Boot test on facility network |
| SLAM-2026-NET-09 | MUST | Network boot supports PXE and UEFI HTTP boot over IPv6 | Boot test on facility network |
| SLAM-2026-NET-10 | MUST | UEFI HTTP boot implementations with dual-stack quirks are validated before deployment | Boot test on facility network |

### DAQ Data-Path Networking

These requirements apply only to add-in NICs used for experiment or DAQ
data traffic. They are not required for the facility/management
interface.

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-NET-11 | MUST | DAQ data-path add-in NICs support SR-IOV | Quote; `lspci -vv` (SR-IOV capability) |
| SLAM-2026-NET-12 | MUST | DAQ data-path add-in NICs support RoCE v2 | Quote; `rdma link` |
| SLAM-2026-NET-13 | MUST | DAQ data-path add-in NICs have an upstream DPDK poll-mode driver | Quote; `dpdk-devbind.py -s` |
| SLAM-2026-NET-14 | MUST | DAQ data-path NICs support PTP hardware timestamping | `ethtool -T <if>` shows hardware transmit and receive timestamping |

### Rationale

**Facility and Management Networking**

- **Onboard copper for the facility interface (NET-01)**: RJ45 is the
  lowest-variance option across a 10-year refresh cycle of facility
  switches. Optics introduce transceiver compatibility, cleaning, and
  spares issues that copper does not have.
- **In-tree, non-deprecated driver on the HCL (NET-02, NET-03)**: an
  in-tree driver is what lets the same NIC move across RHEL major
  versions without vendor DKMS packages. A driver on the deprecated list
  will be absent from the next major version, stranding the hardware.
- **Pre-approved NIC vendors (NET-04)**: Intel, QLogic, Mellanox, and
  Broadcom have a long history of upstream, in-tree drivers and
  multi-generation firmware support. Other vendors may be equally good
  but must be evaluated against that standard before purchase.
- **NUMA affinity and interrupt topology (NET-05)**: DAQ data paths pin
  receive queues and consumers to the NUMA node that owns the NIC's PCIe
  root. If the NIC misreports its node, that pinning is wrong and packet
  processing crosses the interconnect on every packet.
- **LLDP visible to the OS (NET-06, NET-07)**: `lldpd` output is how
  cabling and VLAN errors are caught at commissioning and audited later.
  Some NIC firmware runs its own LLDP agent that consumes the frames
  before the OS sees them; it must be disableable and stay disabled, or
  the OS view is silently empty.
- **Dual-stack PXE and UEFI HTTP boot (NET-08, NET-09, NET-10)**: the
  provisioning network is IPv6-capable and unattended reinstall is the
  primary recovery method. UEFI HTTP boot implementations vary in how
  they handle dual-stack, so the specific firmware is tested rather than
  assumed.

**DAQ Data-Path Networking**

- **DAQ data-path features (NET-11, NET-12, NET-13, NET-14)**: SR-IOV,
  RoCE v2, an upstream DPDK driver, and PTP hardware timestamping are
  the baseline DAQ data-path toolkit. They are required only on add-in
  data-path NICs; the facility interface carries management and
  provisioning traffic and does not need them. Requiring them there
  would exclude most onboard 10GBase-T controllers.

---

## GPU

GPUs are optional and workload-dependent. The only supported NVIDIA GPUs
are those covered by the `nvidia-open` kernel modules (see
SLAM-2026-PROH-02).

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-GPU-01 | MUST | Before procuring any NVIDIA GPU, verify it appears on the current [NVIDIA open GPU kernel modules supported list](https://github.com/NVIDIA/open-gpu-kernel-modules) | Quote; `lspci -nn` |

GPU vendor policy beyond NVIDIA is an open question (see SLAM-2026-OQ-02).

### Rationale

- **`nvidia-open` supported GPUs only (GPU-01, PROH-02)**: the
  proprietary NVIDIA kernel modules are closed source, out of tree, and
  lag kernel releases. The open modules can be rebuilt against any
  kernel and patched locally. Restricting purchases to GPUs the open
  modules support keeps the GPU stack buildable across the 3+ RHEL major
  versions in service at any time and removes dependence on NVIDIA
  continuing to ship a proprietary build for an aging GPU.

---

## Firmware

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-FW-01 | SHOULD | System firmware is updatable via LVFS/`fwupd` | `fwupdmgr get-devices` |
| SLAM-2026-FW-02 | MUST | Support UEFI Firmware Capsule Updates | Vendor documentation |
| SLAM-2026-FW-03 | MUST | System firmware is updatable from at least one of: Linux, BMC, or on-board firmware UI | Vendor documentation; test |
| SLAM-2026-FW-04 | MUST | Firmware updates are possible via Redfish | Redfish `UpdateService` |
| SLAM-2026-FW-05 | MUST | Firmware updates are applicable via standardized tooling when mandated | Test |
| SLAM-2026-FW-06 | MUST | Secure Boot key rotation is supported. `db`/`dbx`/`KEK` updates are applicable from Linux via `fwupd` | `fwupdmgr get-devices` (UEFI dbx) |
| SLAM-2026-FW-07 | MUST | Owner enrolment of custom `db` keys and MOK is supported | `mokutil --db`; `mokutil --kek` |
| SLAM-2026-FW-08 | MUST | System firmware supports downgrade/rollback to a previously installed version | Vendor documentation; test |

### Rationale

Firmware is the one component that cannot be fixed by reinstalling the
OS. Over a 10-year service life it will need updates for security (`dbx`
revocations, Secure Boot CA rotation), for hardware defects, and for
compatibility with newer RHEL releases. Every requirement below exists so
that those updates can be applied, and reversed, from Linux or the BMC
on an air-gapped network with nobody on site.

- **LVFS / `fwupd` (FW-01)**: `fwupd` is distribution-bundled and treats
  firmware like any other package: the LVFS catalog maps hardware IDs to
  capsule files, and updates are applied with the same tooling and
  automation as OS updates. Vendors that publish to LVFS remove the need
  for a vendor-specific update utility. LVFS content can be mirrored for
  air-gapped sites.
- **UEFI capsule updates (FW-02)**: the capsule is the standardized
  delivery format that both `fwupd` and Redfish `UpdateService` consume.
  Without capsule support, each component needs its own update
  mechanism.
- **Update paths (FW-03, FW-04, FW-05)**: at least one path must work
  from Linux or the BMC so that updates never require a site visit or
  another operating system (PROH-03, PROH-05). Redfish is required
  because it is the path that works when the OS is down and is the one
  that fleet automation drives. Standardized tooling is required so that
  a mandated update can be rolled out uniformly rather than one vendor
  utility at a time.
- **Secure Boot key management (FW-06, FW-07)**: `dbx` revocations and
  KEK/CA rotations are issued over the service life and must be applied
  from Linux via `fwupd`. Owner-enrolled `db` keys and MOK allow locally
  built kernels and out-of-tree modules to boot with Secure Boot on,
  which is the only way to keep Secure Boot enabled across the full
  fleet without depending on a third-party signing service.
- **Downgrade / rollback (FW-08)**: a firmware update can break
  compatibility with an installed RHEL release or with DAQ hardware
  firmware. Remote sites need a way back that does not require a
  technician with a programmer.

---

## BMC

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-BMC-01 | MUST | All BMC features (remote console, virtual media, Redfish, firmware update) are included with perpetual licenses at procurement. No capability tiers | Quote |
| SLAM-2026-BMC-02 | MUST | If an exception to SLAM-2026-BMC-01 is approved for a vendor that only offers time-limited licenses, expiration is tracked in monitoring or the asset management system and renewal is budgeted for the full support window of the hardware | Exception record; asset record |
| SLAM-2026-BMC-03 | MUST | BMC operates independently of OS state during OS installation, kernel panic, and power-cycle recovery | Test |
| SLAM-2026-BMC-04 | MUST | Dedicated physical BMC port, connected to an isolated out-of-band management network | Physical inspection |
| SLAM-2026-BMC-05 | MUST | IPMI over LAN (`lanplus`) | `ipmitool -I lanplus ... chassis status` |
| SLAM-2026-BMC-06 | MUST | Redfish 1.20 or newer, Redfish schema 2023.3 or newer | `GET /redfish/v1` (`RedfishVersion`) |
| SLAM-2026-BMC-07 | MUST | Remote serial console (SOL) | `ipmitool sol activate`; Redfish `SerialConsole` |
| SLAM-2026-BMC-08 | MUST | BIOS/UEFI serial console redirection is enabled so that SOL covers firmware setup and early boot | `ipmitool sol activate` during firmware POST; BIOS settings |
| SLAM-2026-BMC-09 | MUST | Remote graphical console (HTML5/VNC or equivalent) | Test |
| SLAM-2026-BMC-10 | MUST | Remote BIOS/UEFI configuration | Redfish `Bios`; Web UI |
| SLAM-2026-BMC-11 | MUST | Boot order is configurable via Redfish | Redfish `Boot` property under `/Systems/*`; test |
| SLAM-2026-BMC-12 | MUST | Remote firmware update | Redfish `UpdateService`; Web UI |
| SLAM-2026-BMC-13 | MUST | BMC configuration (users, network, BIOS settings) is fully exportable and importable via Redfish | Test export/import |
| SLAM-2026-BMC-14 | MUST | Power supply state and consumption are readable | Redfish `/Chassis/*/Power`; `ipmitool sdr` |
| SLAM-2026-BMC-15 | MUST | Fan speeds are readable | Redfish `/Chassis/*/Thermal`; `ipmitool sdr type Fan` |
| SLAM-2026-BMC-16 | MUST | System temperatures are readable | Redfish `/Chassis/*/Thermal`; `ipmitool sdr type Temperature` |
| SLAM-2026-BMC-17 | MUST | SEL/event log is readable via Redfish `LogServices` | Redfish `/Systems/*/LogServices`; entries retrievable |

### Rationale

The BMC is the only path to a node whose OS is down. On a remote site it
substitutes for a person at the console.

- **Perpetual licenses, no tiers (BMC-01, BMC-02)**: a feature that is
  license-gated is a feature that may be discovered missing during an
  outage, years after purchase, when the vendor portal is unreachable
  from the air-gapped site. Where a time-limited license is unavoidable,
  its expiry must be tracked and budgeted like any other support item so
  it does not lapse silently.
- **Independent of OS state (BMC-03)**: the BMC exists for the moments
  the OS is unavailable: installation, kernel panic, power-cycle
  recovery. A BMC that depends on a host agent or host power state fails
  exactly when it is needed.
- **Dedicated, isolated management port (BMC-04)**: the BMC is the most
  privileged interface on the node and has its own firmware, network
  stack, and vulnerability history. A dedicated port on an isolated
  network keeps it off the data and facility networks and leaves it
  reachable when those networks are misconfigured.
- **IPMI and Redfish (BMC-05, BMC-06)**: Redfish is the automation
  interface and the one with a versioned schema; the minimum version
  pins the feature set that the fleet tooling depends on. IPMI over LAN
  is retained because existing tooling and SOL workflows use it.
  Disabling IPMI when unused is an open question (OQ-03).
- **Serial, graphical, and firmware-stage console (BMC-07, BMC-08,
  BMC-09)**: SOL covers the kernel console and text-mode recovery;
  the graphical console covers installers and firmware UIs; BIOS serial
  redirection extends SOL back into POST and firmware setup. Together
  they replace the crash cart for everything except a physical swap.
- **Remote configuration and control (BMC-10, BMC-11, BMC-12,
  BMC-13)**: BIOS configuration, boot order, firmware update, and BMC
  configuration export/import are the operations that unattended
  provisioning and recovery consist of. Any of them requiring physical
  presence turns a remote reinstall or replacement into a site visit.
- **Power, fan, temperature, and event-log telemetry (BMC-14, BMC-15,
  BMC-16, BMC-17)**: PSU, fan, cooling, and platform event-log failures
  are the common hardware faults, and all four give warning before they
  become outages if the readings are collected. Reading them out of band
  means they are still available when the OS is down or when the OS has
  no vendor sensor driver. Redfish `LogServices` provides the SEL/event
  log in the same API as the rest of the telemetry.

---

## Hardware TPM v2

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-TPM-01 | MUST | Hardware TPM 2.0+ is present. It MAY be integrated into the motherboard, CPU, or another component | `tpm2_getcap properties-fixed` |
| SLAM-2026-TPM-02 | MUST | TPM is driven by an in-tree Linux driver and exposed as `/dev/tpmrm0` | `ls /dev/tpmrm0` |
| SLAM-2026-TPM-03 | MUST | TPM has a SHA-256 PCR bank | `tpm2_pcrread sha256` |
| SLAM-2026-TPM-04 | MUST | TPM is fully usable via `tpm2-tools` / TPM2-TSS | `tpm2_getcap algorithms` |
| SLAM-2026-TPM-05 | MUST | EK certificate retrieval and standard attestation workflows are supported | `tpm2_getekcertificate` |
| SLAM-2026-TPM-06 | MUST | EK certificate provisioning behavior (on-chip vs. retrieved from vendor infrastructure) is documented per vendor. Retrieval MUST NOT require internet access from the node (see SLAM-2026-PROH-05) | Vendor documentation |

### Rationale

Intended uses are encrypted-at-rest storage, secret-zero bootstrapping,
and zero-trust identity anchoring. All three depend on a TPM that Linux
can drive fully and whose identity can be validated without an external
route.

- **TPM 2.0+ present (TPM-01)**: TPM 1.2 lacks the algorithm agility and
  the tooling support required below. The physical form (discrete,
  firmware, or integrated) does not matter as long as the remaining
  requirements are met.
- **In-tree driver, `/dev/tpmrm0` (TPM-02)**: the resource-manager
  device lets `systemd`, `clevis`, and application software share the
  TPM without coordinating sessions. An in-tree driver keeps it working
  across RHEL major versions without vendor packages.
- **SHA-256 PCR bank (TPM-03)**: SHA-256 is the PCR bank that RHEL
  measured boot, `systemd-pcrlock`, and attestation tooling use. A TPM
  with only a SHA-1 bank cannot participate in current attestation
  policies.
- **Full `tpm2-tools` / TPM2-TSS support (TPM-04)**: the
  distribution-bundled stack is the only one that will be maintained
  over the service life. A TPM that needs a vendor library for any
  operation cannot be relied on after that library is abandoned.
- **EK certificate and attestation (TPM-05, TPM-06)**: the endorsement
  key certificate is how a remote party confirms it is talking to a
  genuine TPM in a specific node. Some vendors ship the certificate on
  the chip; others require retrieval from vendor infrastructure. The
  behavior must be documented per vendor so that enrollment can be
  planned, and retrieval must never require the node itself to reach the
  internet (PROH-05).

---

## Open Questions

| ID | Question | Section |
|---|---|---|
| SLAM-2026-OQ-01 | Minimum vendor support tier: next-business-day on-site vs. depot return. Likely varies by installation access/location/geography | Serviceability |
| SLAM-2026-OQ-02 | GPU vendor policy beyond NVIDIA `nvidia-open` | GPU |
| SLAM-2026-OQ-03 | BMC security and operations baseline: central authentication (LDAP), owner-installed TLS certificates, ability to disable IPMI-over-LAN when unused, NTP, syslog forwarding | BMC |
| SLAM-2026-OQ-04 | Should PSU input voltage and connector type be required to match the installation PDU at quote time? | Power |

---

## Future Considerations

- Procurement compliance matrix and delivery acceptance checklist derived
  from the requirement tables above

---
