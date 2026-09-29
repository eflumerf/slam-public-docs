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
| SLAM-2026-PROH-02 | MUST NOT | Include NVIDIA GPUs unsupported by the `nvidia-open` kernel modules | Quote; `lspci -nn` |
| SLAM-2026-PROH-03 | MUST NOT | Depend on DOS or Windows for any firmware update of any component | Vendor documentation |
| SLAM-2026-PROH-04 | MUST NOT | Use shared-backplane enclosures (blade chassis, shared midplanes). Each node must occupy its own independent chassis | Quote |
| SLAM-2026-PROH-05 | MUST NOT | Depend on internet access for any component function. All components must operate on an air-gapped network without an external route | Quote; vendor documentation |
| SLAM-2026-PROH-06 | MUST NOT | Procure prototypes or products not part of a standard production line without prior approval | Quote |
| SLAM-2026-PROH-07 | MUST NOT | Procure any hardware component that is end of life at purchase time, or with an announced end of life within 18 months of purchase | Quote; vendor lifecycle notice |

### Rationale

- **VROC** is firmware-assisted software RAID tied to Intel platform
  firmware and a hardware license key. It adds a vendor-specific layer on
  top of `mdadm` (IMSM metadata) without adding value over plain `mdadm`,
  and the array becomes dependent on a specific motherboard generation.
- **No DOS or Windows for firmware updates**: all update paths must work
  from Linux or the BMC on an air-gapped network.
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
| SLAM-2026-PLAT-13 | MUST | Local video console port (see SLAM-2026-SERV-04) | Physical inspection |
| SLAM-2026-PLAT-14 | MUST | Hardware watchdog timer usable from Linux via an in-tree driver and exposed as `/dev/watchdog` or `/dev/watchdogN` | `ls /dev/watchdog*`; `wdctl` |
| SLAM-2026-PLAT-15 | SHOULD | DIMMs are populated in a balanced configuration across memory channels per CPU vendor guidelines | `dmidecode -t memory`; vendor population guidelines |

`aarch64` MAY be approved on a case by case basis as an alternative to
SLAM-2026-PLAT-01.

Hardware TPM requirements are in the Hardware TPM v2 section
(SLAM-2026-TPM-01 through SLAM-2026-TPM-06).

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
  remote site when nobody is present to power-cycle the node.
- Balanced DIMM population preserves memory bandwidth and the NUMA/channel
  behavior that DAQ workloads expect.

---

## Serviceability

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-SERV-01 | MUST | Any node is removable from the rack without interrupting power distribution, network switching, or operation of any other node | Quote; rack design review |
| SLAM-2026-SERV-02 | MUST | Rack-mount chassis has an easily operable rail system compatible with the 19 inch installation rack | Quote; physical inspection |
| SLAM-2026-SERV-03 | MUST | Internal access requires no special tooling beyond standard hand tools | Physical inspection |
| SLAM-2026-SERV-04 | MUST | VGA port for local console. HDMI/DisplayPort is acceptable only if a compatible monitor and cables are **permanently** co-located with the system | Physical inspection |
| SLAM-2026-SERV-05 | MUST | Each installation maintains a crash cart (monitor supporting all video interfaces, keyboard, mouse, required cables) usable on all deployed nodes. The crash cart is the responsibility of the experiment or installation owner. One crash cart is required for every two physical racks of systems | Installation review |
| SLAM-2026-SERV-06 | MAY | A rack mount KVM replaces a crash cart under SLAM-2026-SERV-05, provided a KVM is in every rack the crash cart would service and its cables reach all servers in neighboring racks | Installation review |
| SLAM-2026-SERV-07 | MUST | Each installation has at least one mobile crash cart regardless of SLAM-2026-SERV-06 | Installation review |
| SLAM-2026-SERV-08 | MUST | A vendor hardware support contract covering repair/replacement, firmware update availablility, and support escalation is in place for the node's operational lifetime | Quote; contract |
| SLAM-2026-SERV-09 | MUST | The chassis serial number matches across the physical label, `dmidecode -s system-serial-number`, and Redfish `/Systems/*` | Physical inspection; `dmidecode`; Redfish `GET /redfish/v1/Systems/*` |

Minimum hardware support tier is an open question (see SLAM-2026-OQ-01).

### Rationale

Installations are often remote with limited on-site staff. A node that
can be pulled and worked on without touching its neighbors, using tools
and a console that are already present, keeps repair time short and
avoids collateral downtime. VGA is preferred because crash carts and rack
KVMs are overwhelmingly VGA.

A rack KVM covers the common case; the mobile cart covers the KVM itself
failing and nodes outside KVM cable reach.

Consistent serial numbers across the label, DMI, and Redfish are required
for asset tracking, automated inventory, and support contracts .

---

## Power

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-PWR-01 | MUST | Redundant (N+1) power supplies | Quote; physical inspection |
| SLAM-2026-PWR-02 | MUST | PSUs are hot-swappable | Quote |

PSU state and consumption monitoring through the BMC is required by
SLAM-2026-BMC-08.

### Rationale

PSU failure is the most common field repair. Redundant, hot-swap,
monitored supplies turn it from an outage into a scheduled swap.

---

## Storage

### General

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-STOR-01 | MUST | All drive bays used for RAID arrays support hot-swap | Quote |
| SLAM-2026-STOR-02 | MUST | Per-drive health and wear is readable from Linux for every drive type | `smartctl --scan-open`; `nvme list`; `nvme smart-log` |
| SLAM-2026-STOR-03 | MUST | Per-drive health and wear is readable for drives behind RAID controllers | `smartctl -a -d megaraid,N /dev/sdX` |
| SLAM-2026-STOR-04 | MUST NOT | Use RAID controllers that obscure per-drive SMART data | Quote; `smartctl --scan-open`; `storcli /c0 show` |
| SLAM-2026-STOR-05 | SHOULD | SSDs provide power-loss protection for in-flight writes | Vendor datasheet; `smartctl -a` (PLP/Power Loss Protection field) |

### HDD and SATA SSD

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-STOR-06 | MUST | Broadcom-based hardware RAID controller with battery or capacitor-backed write cache | Quote; `storcli /c0 show` |
| SLAM-2026-STOR-07 | MUST | RAID1 or RAID10 for redundant arrays. RAID5 or RAID6 MAY be used only when capacity matters more than performance and rebuild time | `storcli /c0 /vall show` |

### NVMe

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-STOR-08 | MUST | NVMe is configured as one of: `mdadm` software RAID, hardware RAID (controller supported per SLAM-2026-STOR-13), LVM on raw devices, or raw devices | `lsblk` |
| SLAM-2026-STOR-09 | MUST | NVMe drives are installed in pairs | Quote; `lsblk` |
| SLAM-2026-STOR-10 | MUST | RAID is the expected NVMe configuration; raw or LVM-on-raw devices require justification in the procurement request | Quote; `lsblk` |
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

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-STOR-14 | MUST | OS volume is at least 256 GB usable (after any RAID) | `lsblk` |
| SLAM-2026-STOR-15 | MUST | OS volume resides on a RAID array | `lsblk`; `cat /proc/mdstat` or `storcli /c0 show` |

### Data Volume(s)

Data volumes are storage volumes used for experiment or DAQ data. They are
distinct from the system volume covered by SLAM-2026-STOR-14 and
SLAM-2026-STOR-15.

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-STOR-16 | MUST | Data volumes reside on a RAID array | `lsblk`; `cat /proc/mdstat` or `storcli /c0 show` |

### Rationale

- **Per-drive SMART**: wear and pre-failure indicators are the only
  warning before a rebuild. A controller that hides them removes the
  warning.
- **SSD power-loss protection**: prevents data corruption from PSU failure
  or unexpected power events, reducing the need for on-site recovery.
- **NVMe in pairs**: RAID1/RAID10 needs an even member count. Buying in
  pairs keeps the RAID path open even when a system is initially
  deployed with raw devices.
- **`mdadm` for NVMe**: matches native NVMe performance for RAID1/RAID10,
  is monitored by the kernel and standard tooling, and has no vendor
  dependency. Hardware RAID is allowed where the controller is fully
  covered by SLAM-2026-STOR-13.
- **Broadcom for Hardware RAID**: standardization. One controller family
  and one toolset across all installations means one set of monitoring
  scripts, one spares pool, and one runbook.
- **RAID1/10 default**: rebuild time on large RAID5/6 arrays is a
  multi-day exposure window on a system with no on-site staff.
- **Data volumes on RAID**: experiment and DAQ data must survive a single
  drive failure without on-site intervention.
- **NVMe hot-swap**: a failed NVMe drive should be replaceable without
  removing the node from service. Non-hot-swap form factors such as M.2
  require explicit justification because they make replacement a chassis
  service event.

---

## Networking

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-NET-01 | SHOULD | Onboard RJ45 Ethernet for the primary facility network interface. Optical or external network cards must justify their use | Quote; `ethtool <if>` (Port: Twisted Pair) |
| SLAM-2026-NET-02 | MUST | NIC has an in-tree driver in the current RHEL kernel that is not on the RHEL deprecated-driver list | `ethtool -i <if>`; RHEL release notes (deprecated devices) |
| SLAM-2026-NET-03 | MUST | NIC appears on the RHEL HCL | Quote; catalog lookup |
| SLAM-2026-NET-04 | MUST | Intel, QLogic, Mellanox, and Broadcom NICs are pre-approved; other vendors require approval before procurement | Quote |
| SLAM-2026-NET-05 | MUST | NICs expose correct NUMA affinity and interrupt topology to the OS where relevant | `cat /sys/class/net/<if>/device/numa_node`; `/proc/interrupts` |
| SLAM-2026-NET-06 | MUST | LLDP frames reach the OS so that `lldpd`/`lldpad` can see the switch neighbour (chassis ID, port ID, VLAN) | `lldpcli show neighbors` |
| SLAM-2026-NET-07 | MUST | NIC firmware LLDP agents (e.g. Intel `fw-lldp-agent`) that consume LLDP frames before the OS are disableable and remain disabled across reboots | `ethtool --show-priv-flags <if>`; `lldpcli show neighbors` after reboot |
| SLAM-2026-NET-08 | MUST | Network boot supports PXE and UEFI HTTP boot over IPv4 | Boot test on facility network |
| SLAM-2026-NET-09 | MUST | Network boot supports PXE and UEFI HTTP boot over IPv6 | Boot test on facility network |
| SLAM-2026-NET-10 | MUST | UEFI HTTP boot implementations with dual-stack quirks are validated before deployment | Boot test on facility network |
| SLAM-2026-NET-11 | MUST | DAQ data-path add-in NICs support SR-IOV | Quote; `lspci -vv` (SR-IOV capability) |
| SLAM-2026-NET-12 | MUST | DAQ data-path add-in NICs support RoCE v2 | Quote; `rdma link` |
| SLAM-2026-NET-13 | MUST | DAQ data-path add-in NICs have an upstream DPDK poll-mode driver | Quote; `dpdk-devbind.py -s` |
| SLAM-2026-NET-14 | MUST | DAQ data-path NICs support PTP hardware timestamping | `ethtool -T <if>` shows hardware transmit and receive timestamping |

### Rationale

Onboard copper is the lowest-variance option across a 10-year refresh
cycle of facility switches. In-tree drivers are what allow the same
hardware to move across RHEL major versions without vendor DKMS packages.

SR-IOV, RoCE v2, DPDK, and PTP hardware timestamping are baseline for
DAQ data paths. The facility interface carries management and provisioning
traffic and does not need them; requiring them there would exclude most
onboard 10GBase-T controllers.

LLDP visibility from the OS is how cabling and VLAN errors are caught
during commissioning and audited later.

Dual-stack PXE/HTTP boot is required because the provisioning network
is IPv6-capable and firmware implementations vary.

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

The proprietary NVIDIA kernel modules are out-of-tree and lag kernel
releases. Requiring GPUs supported by the open modules keeps the GPU
stack buildable across the 3+ RHEL major versions in service at any time.

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

Firmware is the one component that cannot be replaced by reinstalling
the OS.

LVFS uses a registry of hardware to locate a UEFI Firmware Capsule for
hardware in a similar method to software updates.

Over the years of runtime it will need updating for security (dbx, Secure
Boot CA rotation) and for compatibility with newer RHEL releases.

Every update path must work from Linux or the BMC on an air-gapped network.

Downgrade/rollback is required because an update may break compatibility
with an installed RHEL release or DAQ firmware; remote sites need a way
back without physical intervention.

---

## BMC

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-BMC-01 | MUST | All BMC features (remote console, virtual media, Redfish, firmware update) are included with perpetual licenses at procurement. No capability tiers | Quote |
| SLAM-2026-BMC-02 | MUST | If an exception to SLAM-2026-BMC-01 is approved for a vendor that only offers time-limited licenses, expiration is tracked in monitoring or the asset management system and renewal is budgeted for the full support window | Exception record; asset record |
| SLAM-2026-BMC-03 | MUST | BMC operates independently of OS state during OS installation, kernel panic, and power-cycle recovery | Test |
| SLAM-2026-BMC-04 | MUST | Remote serial console (SOL) | `ipmitool sol activate`; Redfish `SerialConsole` |
| SLAM-2026-BMC-05 | MUST | Remote graphical console (HTML5/VNC or equivalent) | Test |
| SLAM-2026-BMC-06 | MUST | Remote firmware update | Redfish `UpdateService`; Web UI |
| SLAM-2026-BMC-07 | MUST | Remote BIOS/UEFI configuration | Redfish `Bios`; Web UI |
| SLAM-2026-BMC-08 | MUST | Power supply state and consumption are readable | Redfish `/Chassis/*/Power`; `ipmitool sdr` |
| SLAM-2026-BMC-09 | MUST | Fan speeds are readable | Redfish `/Chassis/*/Thermal`; `ipmitool sdr type Fan` |
| SLAM-2026-BMC-10 | MUST | System temperatures are readable | Redfish `/Chassis/*/Thermal`; `ipmitool sdr type Temperature` |
| SLAM-2026-BMC-11 | MUST | IPMI over LAN (`lanplus`) | `ipmitool -I lanplus ... chassis status` |
| SLAM-2026-BMC-12 | MUST | Redfish 1.20 or newer, Redfish schema 2023.3 or newer | `GET /redfish/v1` (`RedfishVersion`) |
| SLAM-2026-BMC-13 | MUST | BMC configuration (users, network, BIOS settings) is fully exportable and importable via Redfish | Test export/import |
| SLAM-2026-BMC-14 | MUST | Dedicated physical BMC port, connected to an isolated out-of-band management network | Physical inspection |
| SLAM-2026-BMC-15 | MUST | BIOS/UEFI serial console redirection is enabled so that SOL covers firmware setup and early boot | `ipmitool sol activate` during firmware POST; BIOS settings |
| SLAM-2026-BMC-16 | MUST | Boot order is configurable via Redfish | Redfish `Boot` property under `/Systems/*`; test |

### Rationale

The BMC is the only path to a node whose OS is down, and on remote sites
it substitutes for a person at the console. License-gated features
discovered during an outage are unacceptable. Redfish is the automation
interface; IPMI is retained for legacy tooling. Exportable configuration
is what makes a batch of nodes reproducible and a replacement node
identical to the one it replaces.

Out-of-band firmware updates provide the ability to update the system
without physical access.

Serial console redirection extends SOL coverage to firmware setup and
early boot, which is essential when no one is on site to use a crash
cart.

Redfish boot-order control is needed for automated provisioning and
recovery without physical presence.

---

## Hardware TPM v2

| ID | Level | Requirement | Verify |
|---|---|---|---|
| SLAM-2026-TPM-01 | MUST | Hardware TPM 2.0 is present. It MAY be integrated into the motherboard, CPU, or another component | `tpm2_getcap properties-fixed` |
| SLAM-2026-TPM-02 | MUST | TPM is driven by an in-tree Linux driver and exposed as `/dev/tpmrm0` | `ls /dev/tpmrm0` |
| SLAM-2026-TPM-03 | MUST | TPM has a SHA-256 PCR bank | `tpm2_pcrread sha256` |
| SLAM-2026-TPM-04 | MUST | TPM is fully usable via `tpm2-tools` / TPM2-TSS | `tpm2_getcap algorithms` |
| SLAM-2026-TPM-05 | MUST | EK certificate retrieval and standard attestation workflows are supported | `tpm2_getekcertificate` |
| SLAM-2026-TPM-06 | MUST | EK certificate provisioning behavior (on-chip vs. retrieved from vendor infrastructure) is documented per vendor. Retrieval MUST NOT require internet access from the node (see SLAM-2026-PROH-05) | Vendor documentation |

### Rationale

Intended uses are encrypted-at-rest storage, secret-zero bootstrapping,
and zero-trust identity anchoring. All three depend on a TPM that Linux
can drive fully and whose EK can be validated without an external route.

---

## Open Questions

| ID | Question | Section |
|---|---|---|
| SLAM-2026-OQ-01 | Minimum vendor support tier: next-business-day on-site vs. depot return. Likely varies by installation access/location/geography | Serviceability |
| SLAM-2026-OQ-02 | GPU vendor policy beyond NVIDIA `nvidia-open` | GPU |
| SLAM-2026-OQ-03 | BMC security and operations baseline: central authentication (LDAP), owner-installed TLS certificates, ability to disable IPMI-over-LAN when unused, SEL/event log via Redfish `LogServices`, NTP, syslog forwarding | BMC |
| SLAM-2026-OQ-04 | Should PSU input voltage and connector type be required to match the installation PDU at quote time? | Power |

---

## Future Considerations

- Procurement compliance matrix and delivery acceptance checklist derived
  from the requirement tables above

---
