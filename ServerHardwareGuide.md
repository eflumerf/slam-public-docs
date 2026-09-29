# SLAM Server Hardware Guide

**Owners:** SLAM

**Applies To:** server-class deployments

**Last Reviewed:** 2026-09-29

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

---

## Prohibited

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-PROH-01 | MUST NOT | Use Intel VROC, even if present in firmware | BIOS settings (VROC/RSTe disabled); `mdadm --detail-platform`; no `imsm` container in `/proc/mdstat` |
| SLAM-2026-PROH-02 | MUST NOT | Include NVIDIA GPUs unsupported by the `nvidia-open` kernel modules. Check the [NVIDIA open GPU kernel modules supported list](https://github.com/NVIDIA/open-gpu-kernel-modules) before procurement | Quote; `lspci -nn` |
| SLAM-2026-PROH-03 | MUST NOT | Any component's firmware update requires DOS or Windows | Vendor documentation |
| SLAM-2026-PROH-04 | MUST NOT | Use shared-backplane enclosures (blade chassis, shared midplanes). Each node must occupy its own independent chassis | Quote |
| SLAM-2026-PROH-05 | MUST NOT | Depend on internet access for any component. All components must operate on an air-gapped network without an external route | Quote; vendor documentation |
| SLAM-2026-PROH-06 | MUST NOT | Procure prototypes or products not part of a standard production line without prior approval | Quote |
| SLAM-2026-PROH-07 | MUST NOT | Procure any hardware component that is end of life at purchase time, or with an announced end of life within 18 months of purchase | Quote; vendor lifecycle notice |

### Rationale

- **VROC** is firmware-assisted software RAID tied to Intel platform
  firmware and a hardware license key. It adds a vendor-specific layer on
  top of `mdadm` (IMSM metadata) without adding value over plain `mdadm`,
  and the array becomes dependent on a specific motherboard generation.
- **`nvidia-open` only**: the proprietary kernel modules are out-of-tree
  and lag kernel releases. Requiring GPUs supported by the open modules
  keeps the GPU stack buildable across the 3+ RHEL major versions in
  service at any time.
- **No shared backplanes**: chassis midplanes and chassis managers are a
  shared failure domain and a shared lifecycle. A node's service life
  should not be bounded by an enclosure's.
- **Air gap**: many installations have no external route. Any component
  that "phones home" for licensing, activation, or updates will fail.
- **No prototypes / no near-EOL**: service life requires a spares
  pipeline and a support contract that a prototype or EOL part
  cannot provide.

---

## Platform

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-PLAT-01 | MUST | CPU implements `x86_64-v3` or newer | `lscpu` flags include `avx2`, `bmi2`, `fma`, `f16c`, `lzcnt`, `movbe`; on RHEL 9+ `ld.so --help` lists `x86-64-v3` as supported |
| SLAM-2026-PLAT-02 | MUST | SMT ("Hyperthreading") is available and can be disabled in firmware | BIOS settings; `lscpu` (Thread(s) per core) |
| SLAM-2026-PLAT-03 | MUST | ECC RAM | `dmidecode -t memory`; `ras-mc-ctl --status` |
| SLAM-2026-PLAT-04 | MUST | NUMA topology is reported correctly to the OS. Single-node NUMA is acceptable | `numactl --hardware` |
| SLAM-2026-PLAT-05 | MUST | UEFI boot | `ls /sys/firmware/efi` |
| SLAM-2026-PLAT-06 | MUST | Secure Boot capable | `mokutil --sb-state` |
| SLAM-2026-PLAT-07 | MUST | Hardware virtualization support with IOMMU (VT-x/VT-d, AMD-V/AMD-Vi, or equivalent) | `virt-host-validate` |
| SLAM-2026-PLAT-08 | MUST | Listed on the current [RHEL Hardware Compatibility List](https://catalog.redhat.com/hardware) | Quote; catalog lookup |
| SLAM-2026-PLAT-09 | MUST | CPU and RAM meet RHEL minimums and do not exceed RHEL maximums for the intended configuration. Sizing is otherwise workload-dependent | Quote |
| SLAM-2026-PLAT-10 | MUST | At least 2 external USB ports available for installation/crash/recovery | Physical inspection |
| SLAM-2026-PLAT-11 | MUST | Local video console port (see SLAM-2026-SERV-04) | Physical inspection |
| SLAM-2026-PLAT-12 | MUST | Hardware watchdog timer usable from Linux via an in-tree driver, exposed as `/dev/watchdog` and configurable with `systemd` `RuntimeWatchdogSec=` | `ls /dev/watchdog*`; `wdctl`; `systemd-analyze cat-config systemd/system.conf \| grep Watchdog` |

`aarch64` MAY be approved on a case by case basis as an alternative to
SLAM-2026-PLAT-01.

Hardware TPM requirements are in the Hardware TPM v2 section
(SLAM-2026-TPM-01 through SLAM-2026-TPM-04).

### Rationale

- `x86_64-v3` is the floor for current RHEL builds and gives headroom for
  the service life.
- SMT must be present and switchable because some DAQ workloads need
  it off for latency determinism while others benefit from it.
- ECC and correct NUMA reporting are needed for reliable long-running systems and
  for pinning DAQ processes.
- The RHEL HCL requirement is what makes the vendor support contract and
  Red Hat support usable together.
- A hardware watchdog is the only recovery path for a hung kernel on a
  remote site when nobody is present to power-cycle the node. `systemd`
  drives it natively.

---

## Serviceability

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-SERV-01 | MUST | Any node is removable from the rack without interrupting power distribution, network switching, or operation of any other node | Quote; rack design review |
| SLAM-2026-SERV-02 | MUST | Rack-mount chassis has an easily operable rail system compatible with the 19 inch installation rack | Quote; physical inspection |
| SLAM-2026-SERV-03 | MUST | Internal access requires no special tooling beyond standard hand tools | Physical inspection |
| SLAM-2026-SERV-04 | MUST | VGA port for local console. HDMI/DisplayPort is acceptable only if a compatible monitor and cables are **permanently** co-located with the system | Physical inspection |
| SLAM-2026-SERV-05 | MUST | Each installation maintains a crash cart (monitor supporting all video interfaces, keyboard, mouse, required cables) usable on all deployed nodes. The crash cart is the responsibility of the experiment or installation owner. One crash cart is required for every two physical racks of systems | Installation review |
| SLAM-2026-SERV-06 | MUST | A vendor hardware support contract covering repair/replacement, firmware updates, and support escalation is in place for the node's operational lifetime | Quote; contract |
| SLAM-2026-SERV-07 | MAY | A rack mount KVM replaces a crash cart under SLAM-2026-SERV-05, provided a KVM is in every rack the crash cart would service and its cables reach all servers in neighboring racks | Installation review |
| SLAM-2026-SERV-08 | MUST | Each installation has at least one mobile crash cart regardless of SLAM-2026-SERV-07 | Installation review |

Minimum support tier is an open question (see SLAM-2026-OQ-01).

### Rationale

Installations are often remote with limited on-site staff. A node that
can be pulled and worked on without touching its neighbors, using tools
and a console that are already present, keeps repair time short and
avoids collateral downtime. VGA is preferred because crash carts and rack
KVMs are overwhelmingly VGA.

A rack KVM covers the common case; the mobile cart covers the KVM itself
failing and nodes outside KVM cable reach.

---

## Power

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-PWR-01 | MUST | Redundant (N+1) power supplies | Quote; physical inspection |
| SLAM-2026-PWR-02 | MUST | PSUs are hot-swappable | Quote |

PSU state and consumption monitoring through the BMC is required by
SLAM-2026-BMC-07.

### Rationale

PSU failure is the most common field repair. Redundant, hot-swap,
monitored supplies turn it from an outage into a scheduled swap.

---

## Storage

### General

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-STOR-01 | MUST | All drive bays used for RAID arrays support hot-swap | Quote |
| SLAM-2026-STOR-02 | MUST | Per-drive health and wear is readable from Linux for every drive type, including drives behind RAID controllers (`nvme-cli`, `smartmontools`, `smartctl -d megaraid,N`, etc). Controllers that obscure per-drive SMART data are prohibited | `smartctl --scan-open`; `smartctl -a -d megaraid,N /dev/sdX`; `nvme list` |

### HDD and SATA SSD

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-STOR-03 | MUST | Broadcom-based hardware RAID controller with battery or capacitor-backed write cache | Quote; `storcli /c0 show` |
| SLAM-2026-STOR-04 | MUST | RAID1 or RAID10 for redundant arrays. RAID5 or RAID6 MAY be used only when capacity matters more than performance and rebuild time | `storcli /c0 /vall show` |

### NVMe

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-STOR-05 | MUST | NVMe is configured as one of: `mdadm` software RAID, hardware RAID (controller supported per SLAM-2026-STOR-08), LVM on raw devices, or raw devices | `lsblk` |
| SLAM-2026-STOR-06 | MUST | NVMe drives are installed in pairs. RAID (SLAM-2026-STOR-05, SLAM-2026-STOR-07) is the expected NVMe configuration; raw or LVM-on-raw devices require justification in the procurement request | Quote; `lsblk` |
| SLAM-2026-STOR-07 | SHOULD | `mdadm` is used for NVMe RAID1/RAID10 | `cat /proc/mdstat` |

### Hardware RAID Management Tooling

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-STOR-08 | MUST | Any hardware RAID controller is fully manageable from Linux with one of the tools below | Tool runs and enumerates the controller |

| Tool | Scope |
|---|---|
| `storcli` | Broadcom MegaRAID, legacy generations |
| `storcli2` | Broadcom MegaRAID, Gen 3+ (9600-series and newer) |
| `perccli` / `perccli2` | Dell OEM rebrands of `storcli` / `storcli2`; functionally identical |

### System Volume

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-STOR-09 | MUST | OS volume is at least 256 GB usable (after any RAID) | `lsblk` |
| SLAM-2026-STOR-10 | MUST | OS volume resides on a RAID array | `lsblk`; `cat /proc/mdstat` or `storcli /c0 show` |

Boot volume and EFI System Partition layout is an OS installation
concern and is covered by the installation guide, not this document.

### Rationale

- **Per-drive SMART**: wear and pre-failure indicators are the only
  warning before a rebuild. A controller that hides them removes the
  warning.
- **NVMe in pairs**: RAID1/RAID10 needs an even member count. Buying in
  pairs keeps the RAID path open even when a system is initially
  deployed with raw devices.
- **`mdadm` for NVMe**: matches native NVMe performance for RAID1/RAID10,
  is monitored by the kernel and standard tooling, and has no vendor
  dependency. Hardware RAID is allowed where the controller is fully
  covered by SLAM-2026-STOR-08.
- **Broadcom for Hardware RAID**: standardization. One controller family
  and one toolset across all installations means one set of monitoring
  scripts, one spares pool, and one runbook.
- **RAID1/10 default**: rebuild time on large RAID5/6 arrays is a
  multi-day exposure window on a system with no on-site staff.

---

## Networking

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-NET-01 | SHOULD | Onboard RJ45 Ethernet for the primary facility network interface. Optical or external network cards must justify their use | Quote; `ethtool <if>` (Port: Twisted Pair) |
| SLAM-2026-NET-02 | MUST | NIC has an in-tree driver in the current RHEL kernel that is not on the RHEL deprecated-driver list, and appears on the RHEL HCL. Intel, QLogic, Mellanox, and Broadcom are established examples; other vendors require approval before procurement | Quote; `ethtool -i <if>`; RHEL release notes (deprecated devices) |
| SLAM-2026-NET-03 | MUST | Add-in NICs used for DAQ data paths support SR-IOV and RoCE v2, and have an upstream DPDK poll-mode driver. The onboard facility interface (SLAM-2026-NET-01) is exempt | Quote; `lspci -vv` (SR-IOV capability); `rdma link` |
| SLAM-2026-NET-04 | MUST | NICs expose correct NUMA affinity and interrupt topology to the OS where relevant | `cat /sys/class/net/<if>/device/numa_node`; `/proc/interrupts` |
| SLAM-2026-NET-05 | MUST | LLDP frames reach the OS so that `lldpd`/`lldpad` can see the switch neighbour (chassis ID, port ID, VLAN). NIC firmware LLDP agents (e.g. Intel `fw-lldp-agent`) that consume LLDP frames before the OS MUST be disableable and remain disabled across reboots | `lldpcli show neighbors`; `ethtool --show-priv-flags <if>` |
| SLAM-2026-NET-06 | MUST | Network boot supports both PXE and UEFI HTTP boot over IPv4 **and** IPv6 for all interfaces. Some UEFI HTTP boot implementations have dual-stack quirks and must be validated before deployment | Boot test on facility network |

### Rationale

Onboard copper is the lowest-variance option across a 10-year refresh
cycle of facility switches. In-tree drivers are what allow the same
hardware to move across RHEL major versions without vendor DKMS packages.

SR-IOV/RoCE/DPDK are baseline for DAQ data paths. The facility interface
carries management and provisioning traffic and does not need them;
requiring them there would exclude most onboard 10GBase-T controllers.
LLDP visibility from the OS is how cabling and VLAN errors are caught
during commissioning and audited later.

Dual-stack PXE/HTTP boot is required because the provisioning network
is IPv6-capable and firmware implementations vary.

---

## GPU

GPUs are optional and workload-dependent. There are currently no GPU
requirements beyond the prohibition in SLAM-2026-PROH-02 (NVIDIA GPUs
must be supported by the `nvidia-open` kernel modules).

GPU vendor policy beyond NVIDIA is an open question (see SLAM-2026-OQ-02).

---

## Firmware

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-FW-01 | SHOULD | System firmware is updatable via LVFS/`fwupd` | `fwupdmgr get-devices` |
| SLAM-2026-FW-02 | MUST | Support UEFI Firmware Capsule Updates | Vendor documentation |
| SLAM-2026-FW-03 | MUST | System firmware is updatable from at least one of: Linux, BMC, or on-board firmware UI | Vendor documentation; test |
| SLAM-2026-FW-04 | MUST | Firmware updates are possible via Redfish | Redfish `UpdateService` |
| SLAM-2026-FW-05 | MUST | Firmware updates are applicable via standardized tooling when mandated | Test |
| SLAM-2026-FW-06 | MUST | Secure Boot key rotation is supported.  `db`/`dbx`/`KEK` updates are applicable from Linux via `fwupd` | `fwupdmgr get-devices` (UEFI dbx) |
| SLAM-2026-FW-07 | MUST | Owner enrolment of custom `db` keys and MOK is supported | `mokutil --db`; `mokutil --kek` |

### Rationale

Firmware is the one component that cannot be replaced by reinstalling
the OS.

LVFS uses a registry of hardware to locate a UEFI Firmware Capsule for
hardware in a similar method to software updates.

Over the years of runtime it will need updating for security (dbx, Secure
Boot CA rotation) and for compatibility with newer RHEL releases.

Every update path must work from Linux or the BMC on an air-gapped network.

---

## Hardware TPM v2

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-TPM-01 | MUST | Hardware TPM 2.0 is present. It MAY be integrated into the motherboard, CPU, or another component | `tpm2_getcap properties-fixed` |
| SLAM-2026-TPM-02 | MUST | Driven by an in-tree Linux driver, exposed as `/dev/tpmrm0`, with a SHA-256 PCR bank, and fully usable via `tpm2-tools` / TPM2-TSS | `ls /dev/tpmrm0`; `tpm2_pcrread sha256`; `tpm2_getcap algorithms` |
| SLAM-2026-TPM-03 | MUST | EK certificate retrieval and standard attestation workflows are supported | `tpm2_getekcertificate` |
| SLAM-2026-TPM-04 | MUST | EK certificate provisioning behavior (on-chip vs. retrieved from vendor infrastructure) is documented per vendor. Retrieval MUST NOT require internet access from the node (see SLAM-2026-PROH-05) | Vendor documentation |

### Rationale

Intended uses are encrypted-at-rest storage, secret-zero bootstrapping,
and zero-trust identity anchoring. All three depend on a TPM that Linux
can drive fully and whose EK can be validated without an external route.

---

## BMC

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-BMC-01 | MUST | All BMC features (remote console, virtual media, Redfish, firmware update) are included with perpetual licenses at procurement. No capability tiers | Quote |
| SLAM-2026-BMC-02 | MUST | If an exception to SLAM-2026-BMC-01 is approved for a vendor that only offers time-limited licenses, expiration is tracked in monitoring or the asset management system and renewal is budgeted for the full support window | Exception record; asset record |
| SLAM-2026-BMC-03 | MUST | BMC operates independently of OS state, including during OS installation, kernel panic, and power-cycle recovery | Test |
| SLAM-2026-BMC-04 | MUST | Remote serial console (SOL) | `ipmitool sol activate`; Redfish `SerialConsole` |
| SLAM-2026-BMC-05 | MUST | Remote graphical console (HTML5/VNC or equivalent) | Test |
| SLAM-2026-BMC-06 | MUST | Remote firmware update and remote BIOS/UEFI configuration | Redfish `UpdateService`, `Bios`, Web UI |
| SLAM-2026-BMC-07 | MUST | Power supply state and consumption are readable | Redfish `/Chassis/*/Power`; `ipmitool sdr` |
| SLAM-2026-BMC-08 | MUST | Fan speeds are readable | Redfish `/Chassis/*/Thermal`; `ipmitool sdr type Fan` |
| SLAM-2026-BMC-09 | MUST | System temperatures are readable | Redfish `/Chassis/*/Thermal`; `ipmitool sdr type Temperature` |
| SLAM-2026-BMC-10 | MUST | IPMI over LAN (`lanplus`) | `ipmitool -I lanplus ... chassis status` |
| SLAM-2026-BMC-11 | MUST | Redfish 1.20 or newer, Redfish schema 2023.3 or newer | `GET /redfish/v1` (`RedfishVersion`) |
| SLAM-2026-BMC-12 | MUST | BMC configuration (users, network, BIOS settings) is fully exportable and importable via Redfish | Test export/import |
| SLAM-2026-BMC-13 | MUST | Dedicated physical BMC network port, connected to an isolated out-of-band management network | Physical inspection |

### Rationale

The BMC is the only path to a node whose OS is down, and on remote sites
it substitutes for a person at the console. License-gated features
discovered during an outage are unacceptable. Redfish is the automation
interface; IPMI is retained for legacy tooling. Exportable configuration
is what makes a batch of nodes reproducible and a replacement node
identical to the one it replaces.

Out of band firmware updates provide the ability to update the system without
physical access.

---

## Open Questions

| ID | Question | Section |
|---|---|---|
| SLAM-2026-OQ-01 | Minimum vendor support tier: next-business-day on-site vs. depot return. Likely varies by installation access/location/geography | Serviceability |
| SLAM-2026-OQ-02 | GPU vendor policy beyond NVIDIA `nvidia-open` | GPU |
| SLAM-2026-OQ-03 | NVMe hot-swap is currently required for RAID arrays only. Should it extend to single NVMe devices? | Storage |
| SLAM-2026-OQ-04 | Should data-path NICs require PTP hardware timestamping (`ethtool -T`)? | Networking |
| SLAM-2026-OQ-05 | Should BIOS/UEFI serial console redirection be required so SOL (SLAM-2026-BMC-04) covers firmware setup and early boot? | BMC |
| SLAM-2026-OQ-06 | BMC security and operations baseline: central authentication (LDAP), owner-installed TLS certificates, ability to disable IPMI-over-LAN when unused, SEL/event log via Redfish `LogServices`, NTP, syslog forwarding | BMC |
| SLAM-2026-OQ-07 | Should firmware downgrade/rollback and Redfish boot-order control be required? | Firmware; BMC |
| SLAM-2026-OQ-08 | Should PSU input voltage and connector type be required to match the installation PDU at quote time? | Power |
| SLAM-2026-OQ-09 | Should the chassis serial number be required to match across the physical label, `dmidecode -s system-serial-number`, and Redfish `/Systems/*` for asset tracking? | Serviceability |
| SLAM-2026-OQ-10 | Should SSDs require power-loss protection (SHOULD level)? | Storage |
| SLAM-2026-OQ-11 | Should balanced DIMM population across memory channels be required for DAQ bandwidth? | Platform |

---

## Future Considerations

- Procurement compliance matrix and delivery acceptance checklist derived
  from the requirement tables above

---
