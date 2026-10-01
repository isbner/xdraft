# Hardware & Software Design Discovery Plan
## FBOSS + OpenBMC / BMC-lite Switch — `switch_model`

## 0. Assumptions & Placeholders

| Item | Value | Status |
|---|---|---|
| Platform codename | `switch_model` | TBD — identify during P0/P5 |
| Architecture | Two chips: **Host CPU** + **BMC** (separate silicon) | Confirmed |
| BMC stack | OpenBMC-derived, **bmc-lite = thinner/customized layer** (do NOT assume stock Phosphor layout) | Confirmed, details TBD |
| BMC source tree | `/home/isbner/workspace/openbmc` (Yocto meta layers) | Available now |
| FBOSS source tree | `~/workspace/fboss` (assumed) | TBD |
| Live access | SSH to CPU and BMC (root/sudo ideally), serial/SoL TBD | In ~2 weeks |
| Deliverable | One design report (hardware inventory, userspace API map, daemons, CPU↔BMC protocols incl. IPMI command table) | — |

Guiding principle: **source shows intent, the running box shows truth.** Every static finding gets a runtime verification step. Mark unconfirmed items explicitly; never infer.

---

## Phase 0 — Prerequisites, Safety & Identification Gate (before touching the live box)

### 0.1 Access & recovery prerequisites (resolve during the 2-week lead)
- [ ] SSH accounts (with sudo/root) on **both** CPU and BMC; jump-host/bastion path documented
- [ ] Serial / SoL console access and OOB power control (PDU) — required before any risky probe
- [ ] Lab/non-production unit; maintenance window approved
- [ ] Known-good firmware image + confirmed dual-image/fallback + reflash/unbrick procedure
- [ ] Exact platform identifiers to collect: board codename, **board revision**, BMC/CPU FRU data
- [ ] Source revisions pinned: `repo manifest`/git SHA of OpenBMC build, FBOSS branch+SHA, Yocto image manifest
- [ ] Named platform engineer on standby for questions

### 0.2 Command discipline (read-only allowlist)
- **Allowed without review:** `ls`, `cat`, `find`, `lspci`, `lsusb`, `dmidecode`, `uname -a`,
  `sensors`, `ethtool`, `mii-tool`, `gpioinfo`, `busctl/gdbus introspect`, `systemctl status/list-units`,
  `ip addr`, `dmesg`, reading `/sys`, `/proc`, `/dev` metadata
- **Forbidden without explicit approval:** `i2cset`, `i2cdetect` on live/muxed buses (can disturb
  mux state, SFPs/PSUs), `ipmitool raw` / OEM NetFn writes (can cut power/reset devices),
  MCTP/PLDM `Set*`, PCIe VDM active tests, any `devmem` write, ASIC SDK write commands
- Every session logged with timestamps (`script`); outputs hashed and archived as evidence

### 0.3 Platform identification (first 5 minutes of live access)
- BMC: `/etc/os-release`, `/etc/*release`, `cat /proc/device-tree/model`, FRU (`busctl` inventory or fru dump)
- CPU: `dmidecode -t baseboard/BIOS`, `/sys/class/dmi/id/board_*`, FBOSS platform config file
- Match runtime identity back to `meta-facebook/meta-<platform>/conf/machine/<platform>.conf`
  (`OBMC_COMPATIBLE_NAMES`) and FBOSS platform mapping

---

## Phase 1 — Source Tree Mapping (can start now)

**Goal:** know which layers/recipes actually compose the BMC image and which FBOSS platform modules apply.

| # | Action | Artifact |
|---|---|---|
| 1.1 | Resolve active machine: `conf/machine/<platform>.conf` → SoC (AST2500/2600/…), KERNEL_DEVICETREE, flash size, included `.inc` features | machine conf |
| 1.2 | Read `bblayers.conf` + layer priorities; isolate **bmc-lite customizations** (removed/overridden recipes vs stock Phosphor) | layer delta list |
| 1.3 | Parse image recipe + `packagegroup-*.bb` / `.bbappend` (e.g. `packagegroup-fb-apps.bb`) | predicted package set |
| 1.4 | Build/obtain the **image package manifest**; later diff vs runtime package DB | manifest |
| 1.5 | In FBOSS tree: locate platform code (`platform/`, `BcmChip`/vendor init, `DeviceProductInfo`, platform config JSONs) | CPU platform module list |

---

## Phase 2 — BMC-Hardware Inventory from Source

**Goal:** complete BMC-view topology: I2C, SPI, CPLD/FPGA, MDIO, GPIO, PCIe, sensors.

| # | Action | Source of truth |
|---|---|---|
| 2.1 | Parse **kernel DTS/DTSI** (`KERNEL_DEVICETREE`, kernel bbappends, u-boot DTS): every `i2c@*` bus, child devices (`@addr` + `compatible` + label), muxes (`pca954x`), SPI controllers + flashes + partitions, CPLD/FPGA nodes, MDIO, GPIO, watchdogs, FSI/LPC/eSPI | `.dts/.dtsi` |
| 2.2 | Parse hwmon configs: `recipes-phosphor/sensors/**/*.conf` (device path → sensor mapping) | hwmon `.conf` |
| 2.3 | Inventory definitions: **entity-manager JSON if present** (probe rules + FRU tree). On bmc-lite EM may be absent → find the equivalent (static config, custom daemon, VPD) | EM JSON / alternative |
| 2.4 | CPLD/FPGA deep dive: access path (MMIO/regmap/I2C/SPI), register maps, firmware-version mechanism, **and note a 3rd MCU is possible** | driver + platform code |
| 2.5 | Build bus-ownership table: per I2C segment/mux/SPI/CPLD — owned by **BMC, CPU, or shared/arbitrated** (classic corruption source) | ownership matrix |
| 2.6 | Power/reset sequencing: power-good GPIOs, power-control daemons, reset causes | gpio-monitor + state-mgr configs |

---

## Phase 3 — CPU / FBOSS Hardware Inventory from Source

**Goal:** CPU-view hardware, especially the switch ASIC complex.

| # | Action | Output |
|---|---|---|
| 3.1 | ASIC identification: vendor/family (e.g. Tomahawk/Trident-class), rev, PCIe BDF, SDK version, attached PHYs | ASIC profile |
| 3.2 | Optics: QSFP/SFP access path (I2C via BMC bridge vs direct CPU I2C), `qsfp_service` config | optics path |
| 3.3 | CPU-attached peripherals: I2C/SPI/GPIO/CPLD devices opened by FBOSS platform code; note x86 CPU likely uses **ACPI not DTS** | CPU device list |
| 3.4 | MDIO/PHY topology, fans/PSUs/temp sensors visible from CPU, LED control | CPU sensor map |
| 3.5 | Storage/boot: BIOS/UEFI version, boot flash, NIC firmware | firmware matrix (static side) |

---

## Phase 4 — CPU↔BMC Protocols & IPMI Command Enumeration (static)

**Goal:** enumerate every channel and every supported IPMI command from handler source.

### 4.1 Channels to investigate (assume none — confirm presence)
IPMB (I2C), KCS/BT/eSPI/LPC, **SSIF**, MCTP (over I2C/PCIe-VDM) and **PLDM** on top,
NCSI/sideband NIC (note VLAN config e.g. `eth0.4088`), PCIe VDM, UART/SoL, USB-net, Redfish.

### 4.2 For each channel record
Physical wiring (bus/addr/port) · endpoint · transport + protocol version · initiator direction ·
source recipe/daemon · read-only verification command.

### 4.3 IPMI command table (core deliverable)
- Standard: `phosphor-ipmi-host` command registration (NetFn, cmd, privilege, handler)
- OEM: `fb-ipmi-oem` (or equivalent) handler files → enumerate every OEM NetFn/cmd
- Bridges: `phosphor-ipmi-ipmb`/SSIF config — which transports are enabled
- Whitelist/permission config; map each handler → D-Bus service it calls
- CPU side: FBOSS code that **issues** IPMI requests (client call sites)
- Output table: `NetFn | Cmd | Name | Direction | Privilege | Source ref | Runtime check (read-only)`

---

## Phase 5 — Live Recon: BMC (SSH, read-only)

Capture everything into timestamped logs; compare with P1/P2 predictions and record drift.

```bash
# Identity / versions
cat /etc/os-release; uname -a; cat /proc/cmdline
cat /proc/device-tree/model; cat /proc/device-tree/compatible | tr '\0' ' '
# Inventory of buses
i2cdetect -l                       # buses only; NO per-bus scans without approval
find /sys/bus/i2c/devices -maxdepth 2 -name name -o -name modalias | sort
ls -l /sys/bus/spi/devices/; cat /proc/mtd
ls /sys/bus/pci/devices/ 2>/dev/null
ls /sys/bus/mdio_bus/devices/ 2>/dev/null
gpioinfo 2>/dev/null || ls /sys/class/gpio
# Sensors / hwmon
for d in /sys/class/hwmon/hwmon*; do echo "== $d"; cat $d/name; ls $d/device/; done
sensors 2>/dev/null
# Services (systemd may be replaced on bmc-lite — detect init first)
ps -ef; systemctl list-units --type=service --all 2>/dev/null
# D-Bus inventory & sensor objects (use gdbus if busctl missing)
busctl list 2>/dev/null; busctl tree xyz.openbmc_project.EntityManager 2>/dev/null
# IPMI endpoints (passive evidence)
ls -l /dev/ipmi* /dev/ipmb* 2>/dev/null; ipmitool mc info
```

Also: failed/restart-looping units, open fds of suspicious daemons, package DB
(`rpm -qa`/`opkg list-installed`) vs P1 manifest, FRU inventory, network config for NCSI.

---

## Phase 6 — Live Recon: CPU / FBOSS (SSH, read-only)

```bash
dmidecode -t baseboard,bios,system; uname -a
lspci -nnvvv                        # ASIC BDF, VDM/ACS caps, kernel drivers
lsusb; lsblk; cat /proc/mtd 2>/dev/null
ip -d link; ethtool -i <ports>; mii-tool 2>/dev/null
sensors; ls /sys/class/hwmon; ls /sys/bus/i2c/devices /sys/bus/spi/devices
ps -ef | grep -Ei 'fboss|qsfp|sensor|wedge|mib|routing'
systemctl list-units --type=service | grep -Ei 'fboss|wedge|bmc|ipmi'
ipmitool mc info 2>/dev/null        # does CPU run an IPMI client to BMC?
# FBOSS utilities (read-only state queries only)
<vendor>_util --version; show platform / optics status commands as documented
```

Capture: ASIC SDK/firmware version, BIOS/NIC/CPLD versions (feeds firmware matrix),
which devices the CPU actually opens (`lsof`/`/proc/*/fd` for fboss daemons),
optics presence read via the sanctioned FBOSS tool only.

---

## Phase 7 — Userspace-API Mapping (cross-correlation P2/P3 ↔ P5/P6)

Produce one master table, one row per chip/device, columns:

| Device (chip, addr) | Bus (DTS name + live bus#) | Owner CPU/BMC | Kernel driver | sysfs path | CLI utility | D-Bus service/interface | Notes |
|---|---|---|---|---|---|---|---|

Cover: I2C devices/muxes/EEPROMs/hot-swap/VRMs/optics · SPI flashes (`/dev/mtd*`, spidev) ·
CPLD/FPGA (misc device / MMIO) · MDIO/PHY · GPIO expanders · hwmon sensors · PCIe devices.
Note numbering drift across reboot (bus numbers are not ABI); snapshot twice if a reboot occurs.

---

## Phase 8 — Runtime Protocol Verification & Report Assembly

### 8.1 Channel verification (read-only first; writes need approval)
- IPMB/IPMI: observe devices + `mc info`, channel info; only then enumerate OEM commands
  via **documented read-only** raw calls, one at a time, with console+PDU ready
- NCSI: confirm sideband MAC (CPU NIC vs ASIC), read Get Version ID/Parameters passively
- MCTP/PLDM: enumerate endpoints + supported PLDM types (platform/fw-update), passive discovery
- PCIe VDM: confirm capability from `lspci`; no active traffic without approval
- Map message sequences: power-on/off, watchdog, SEL, firmware update, graceful reset

### 8.2 Final report structure
1. Executive summary
2. Architecture overview (block/topology diagram; boot & reset sequence)
3. BMC hardware inventory (bus topology, CPLD/FPGA, ownership matrix)
4. CPU hardware inventory (ASIC, optics, PCIe, ACPI/platform devices)
5. Userspace API map (master table from P7)
6. Daemon inventory — BMC **and** CPU (purpose, deps, supervision, failure loops)
7. CPU↔BMC communication (per-channel: wiring, protocol, message sequence diagrams)
8. **IPMI command reference** (standard + OEM full table)
9. Firmware version matrix (BMC, CPLD/FPGA, BIOS, NIC, ASIC SDK/fw) with source-vs-running drift
10. Open-questions register + evidence appendix (raw command outputs, hashed archives)

---

## Work breakdown vs. access window

| Now (static, no device) | At access day (runbook execution) |
|---|---|
| P0.1 prerequisites chase; P1–P4 full source analysis | P0.3 identification → P5 BMC → P6 CPU |
| Pre-build exact command bundle + expected-result hypotheses | P7 cross-correlation on the fly |
| Safety review of every command; get write-probes pre-approved | P8 passive verification; approved probes only; report |

---

## Key risks & assumptions

1. **Biggest risk** is live probing — stray `i2cdetect`/`ipmitool raw` writes have wedged real switches (mux lockups, PSU/SFP faults, fan failsafe, host power-off). Every non-read-only command is gated behind approval and requires console + PDU + a fallback image first.
2. **Biggest assumption to kill early** is "bmc-lite = stock OpenBMC minus a few packages" — it may not use systemd, entity-manager, or even host-ipmid. P5 starts by detecting the init/supervisor model before relying on any Phosphor-specific command, and the CPU side may be x86/ACPI rather than device-tree based.

## Inputs needed when access is ready

- Platform codename / board revision
- FBOSS source path
- SSH details for both chips
- Confirmation: lab unit with console + PDU
