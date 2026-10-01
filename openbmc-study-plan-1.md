# Investigation Plan: BMC-Lite + FBOSS Switch (switch_model_1 / AST2600 / Read-Only)
## Basic Fixed Preconditions (Confirmed)
1. Switch Platform: `switch_model_1` (placeholder)
2. BMC SoC: ASPEED AST2600
3. Permissions: Full root SSH on BMC & Host, password placeholder: `bmc_password`
4. Network Capture: tcpdump available on BMC OOB interface
5. BMC ↔ Host protocol: **unknown, need automatic detection in workflow** (IPMI / Redfish / mixed)
6. Need inventory for **standard IPMI + OEM IPMI commands**
7. PIM definition & presence: unknown, add dedicated auto-detect + TODO tag
8. Skip ASIC/SAI register analysis (out of scope)
9. Operation policy: **100% read-only, no write / no reboot / no config change**

## Core Research Targets (Full Report Coverage)
1. Hardware chip topology: BMC/Host-side I2C/SPI/PCIe/MDIO/CPLD/FPGA inventory
2. All peripheral user-space interfaces: sysfs / /dev / tool utilities
3. All running daemons (BMC OpenBMC + Host FBOSS) for platform/hardware monitoring
4. BMC-Lite exclusive design features (I2C ownership, service stripping)
5. BMC ↔ Host communication mechanism + full IPMI/OEM/Redfish command inventory

---

# Phase 1: Platform Identity & High-Level Topology Discovery (Read-Only)
## 1.1 BMC Side (AST2600 OpenBMC)
```bash
# Basic platform info
cat /etc/os-release
cat /proc/device-tree/model
cat /sys/devices/platform/ast2600-fb/board_id
cat /sys/devices/platform/ast2600-fb/version
eeprom-util

# Bus & hardware baseline
lsmod
lspci
ls /sys/bus/i2c/devices/
ls /sys/bus/spi/devices/
ls /sys/class/gpio/

# Export device tree (core design source)
dtc -I fs /proc/device-tree > bmc_ast2600_dts.dts
```

## 1.2 Host FBOSS Side (switch_model_1)
```bash
# FBOSS official platform info dump
/platform_manager --dump_platform_info
cat /etc/fboss/platform_config.json

# Hardware bus inventory
lspci -nnvv > host_pci_detail.txt
i2cdetect -y 0 && i2cdetect -y 1 && i2cdetect -y 2
ls /dev/i2c-*
ls /sys/bus/mdio/devices/
dtc -I fs /proc/device-tree > host_dts.dts
```

## 1.3 Source Code Static Cross-Check
1. OpenBMC:
   - Locate `meta-facebook` machine config for AST2600 platform
   - Verify **I2C bus status = disabled/okay** (BMC-Lite core flag)
2. FBOSS:
   - Parse `platform_config.json`: `useBmcI2cProxy` / `bmcAccessorType`
   - Record BMC-Lite runtime mode flag

## Phase1 Output
- Fixed platform baseline: AST2600 BMC + switch_model_1 host
- Full bus topology snapshot
- Confirm BMC-Lite enable status via code + live system

---

# Phase 2: Full Peripheral Hardware Inventory (I2C/SPI/PCIe/MDIO/CPLD/FPGA)
## 2.1 BMC Side Read-Only Scan
```bash
# Enumerate all I2C devices + driver binding
for i in $(ls /sys/bus/i2c/devices/i2c-* | grep -v name); do echo $i; cat $i/name; cat $i/driver 2>/dev/null; done

# SPI/Platform devices (CPLD/FPGA)
ls /sys/bus/spi/devices/
ls /sys/devices/platform/

# BMC sensor stack status
sensors
busctl tree org.openbmc
```

## 2.2 Host Side Read-Only Scan
```bash
# FBOSS dedicated peripheral dump
platform_manager --dump_i2c_info

# Kernel driver binding check
ls /sys/bus/i2c/devices/*/driver
ls /sys/bus/mdio/devices/

# CPLD/FPGA regmap/sysfs entry check
ls /sys/devices/platform/ | grep -i fpga\|cpld\|regmap
```

## 2.3 Auto-Detect PIM (Unknown Item Resolution)
### PIM Definition (Pre-Insert Doc Definition)
**PIM (Platform Input/Output Module)**：Meta/OCP switch front-panel pluggable I/O module, carrying QSFP ports, PHY, I2C mux, hotplug logic.
### Auto-Detect Workflow
1. Check FBOSS platform config for `pim` keyword
2. Check host dts for `pim`/`port-module` node
3. Check i2c bus for port-module related devices
### Mark in Report
- If detected: record PIM count / bus / owner
- If no PIM: mark `No PIM hardware on this platform`
- Undetermined: leave **TODO: Further schematic cross-check**

## 2.4 Final Standard Report Table (Fixed Template)
| Peripheral Chip | Bus Type | Address | Owner(BMC/Host) | Kernel Driver | User API(sysfs/dev) | Function |
|---|---|---|---|---|---|---|
| Temp Sensor | I2C |  |  |  |  |  |
| Fan Controller | I2C |  |  |  |  |  |
| QSFP EEPROM/DOM | I2C-MUX |  |  |  |  |  |
| PSU | I2C |  |  |  |  |  |
| CPLD | SPI/Platform |  |  |  |  |  |
| FPGA | PCIe/SPI |  |  |  |  |  |
| ASIC | PCIe |  | Host | SAI | FBOSS Agent | Forwarding |

---

# Phase 3: Daemon & Service Inventory (BMC vs Host BMC-Lite Stack)
## 3.1 BMC OpenBMC Service Scan (ReadOnly)
```bash
systemctl list-units --type=service
busctl list
ps aux

# BMC-Lite core check (key disabled services)
systemctl status phosphor-entity-manager
systemctl status phosphor-fan-control
systemctl status phosphor-qsfp*
```
### BMC-Lite Standard Verification Rule
- Heavy-BMC services (entity-manager/fan-control/qsfp-monitor) = **disabled/inactive**
- Reserved minimal services: redfish/ipmi/power/console/network

## 3.2 Host FBOSS Service Scan (ReadOnly)
```bash
systemctl list-units | grep fboss
ps aux | grep fboss
cat /lib/systemd/system/fboss*.service
```

## 3.3 Service Inventory Table (Final Report)
| Side | Daemon | Core Function | Controlled Hardware |
|---|---|---|---|
| BMC |  |  |  |
| Host |  |  |  |

---

# Phase 4: Auto-Detect BMC ↔ Host Communication Protocol (IPMI / Redfish / Mixed)
## Requirement
Unknown protocol → **fully automated detection workflow (read-only)**

## 4.1 Step1: Discover BMC IP from FBOSS config
```bash
grep -E "bmc|redfish|ipmi" /etc/fboss/platform_config.json
```

## 4.2 Step2: Redfish Detection
```bash
curl -k -m 3 https://${BMC_IP}/redfish/v1/
```

## 4.3 Step3: IPMI (Standard + OEM) Detection
```bash
ipmitool -I lanplus -H ${BMC_IP} -U admin -P bmc_password help
ipmitool -I lanplus -H ${BMC_IP} -U admin -P bmc_password raw help
```

## 4.4 Step4: Live Traffic Capture (tcpdump Read-Only)
Capture OOB traffic during FBOSS runtime, identify **actual used protocols/commands**
```bash
tcpdump -i <oob-if> host ${BMC_IP} -nn -tt
```

## 4.5 Step5: Source Code Confirmation
1. FBOSS `bmc_accessor` code: check if loading `RedfishBmcAccessor` or `IpmiBmcAccessor`
2. OpenBMC phosphor-ipmi-oem: list all OEM command handlers
3. OpenBMC bmcweb: list all Redfish resource paths

## 4.6 Output
- Final conclusion: IPMI-only / Redfish-only / dual-stack
- Full list: Standard IPMI commands + **OEM IPMI commands** supported
- Exact command set used by FBOSS runtime

---

# Phase 5: BMC-Lite Architecture Static Analysis (Code + Live System)
## Key Check Items (Exclusive for This Report)
1. BMC side: All I2C buses for QSFP/FAN/PIM are **disabled in DTS**
2. BMC side: No sensor/fan/qsfp management daemons running
3. Host side: `useBmcI2cProxy=false` (no BMC I2C proxy)
4. Host side: All hardware management migrated to `platform_manager/qsfp_service/fan_service`
5. BMC only provides: power/reset/PSU status/Redfish/IPMI OOB channel

---

# Phase 6: Final Report Standard Outline (Fixed Output)
1. Platform Overview（switch_model_1 + AST2600 + BMC-Lite mode）
2. Full Hardware Bus & Peripheral Inventory Table
3. All User-Space Access Interfaces（sysfs / /dev / tools）
4. BMC & Host Full Daemon/Service Function Inventory
5. BMC-Lite Architecture Design Differences vs Traditional Heavy-BMC
6. BMC ↔ Host Communication Mechanism（Auto-Detected Protocol）
7. Standard + OEM IPMI Command Full Inventory
8. Open Risk & TODO Items（PIM undetermined / bus arbitration risk etc.）

---

# Persistent TODO Tags Reserved in Final Report
1. TODO: PIM hardware structure needs schematic cross-validation for full specification
2. TODO: I2C mux host/bmc exclusive ownership hardware arbitration risk verification
