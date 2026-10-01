# Read-only data collection script for switch_model_1 (AST2600 BMC-Lite FBOSS switch)
> Rules: ALL commands are READ-ONLY, NO write / NO reboot / NO modification.
> Two separate scripts:
> 1. `collect_bmc.sh` → Run on BMC (root login, password placeholder: bmc_password)
> 2. `collect_host.sh` → Run on FBOSS host (root login)
> Output: All logs dumped into local text files for later report generation.
> Note: All file outputs are appended / saved in current working directory.

## collect_bmc.sh (Run on BMC, AST2600 OpenBMC)
```bash
#!/bin/bash
set -euo pipefail
OUTDIR="./bmc_collect"
mkdir -p ${OUTDIR}
echo "===== Starting BMC Read-only Collection ====="

# 1. Platform identity & OS info
cat /etc/os-release > ${OUTDIR}/os-release.txt
cat /proc/device-tree/model > ${OUTDIR}/model.txt
cat /sys/devices/platform/ast2600-fb/board_id > ${OUTDIR}/board_id.txt 2>/dev/null
cat /sys/devices/platform/ast2600-fb/version > ${OUTDIR}/board_version.txt 2>/dev/null
eeprom-util > ${OUTDIR}/eeprom_chassis.txt 2>/dev/null

# 2. Kernel & PCI / modules
lsmod > ${OUTDIR}/lsmod.txt
lspci > ${OUTDIR}/lspci.txt
lspci -nnvv > ${OUTDIR}/lspci_detail.txt

# 3. Bus enumeration: I2C / SPI / GPIO
ls /sys/bus/i2c/devices/ > ${OUTDIR}/i2c_dev_list.txt
ls /sys/bus/spi/devices/ > ${OUTDIR}/spi_dev_list.txt
ls /sys/class/gpio/ > ${OUTDIR}/gpio_list.txt

# Dump I2C device name + driver binding
> ${OUTDIR}/i2c_device_details.txt
for i in $(ls /sys/bus/i2c/devices/i2c-* | grep -v name); do
  echo "=== $i ===" >> ${OUTDIR}/i2c_device_details.txt
  cat $i/name >> ${OUTDIR}/i2c_device_details.txt 2>/dev/null
  cat $i/driver 2>/dev/null >> ${OUTDIR}/i2c_device_details.txt
done

# Platform devices (CPLD / FPGA candidates)
ls /sys/devices/platform/ > ${OUTDIR}/platform_devices.txt

# 4. Sensors & D-Bus
sensors > ${OUTDIR}/sensors.txt 2>/dev/null
busctl tree org.openbmc > ${OUTDIR}/dbus_tree.txt

# 5. Device Tree blob dump (critical for BMC hardware topology)
dtc -I fs /proc/device-tree > ${OUTDIR}/bmc_dts.dts

# 6. Systemd services & process list
systemctl list-units --type=service > ${OUTDIR}/systemd_services.txt
ps aux > ${OUTDIR}/ps_bmc.txt

# BMC-Lite key service status check
systemctl status phosphor-entity-manager > ${OUTDIR}/svc_entity_manager.txt 2>&1
systemctl status phosphor-fan-control > ${OUTDIR}/svc_fan_control.txt 2>&1
systemctl status phosphor-qsfp* > ${OUTDIR}/svc_qsfp.txt 2>&1

echo "===== BMC collection complete, artifacts stored in ${OUTDIR} ====="
```

## collect_host.sh (Run on FBOSS host, switch_model_1)
```bash
#!/bin/bash
set -euo pipefail
OUTDIR="./host_collect"
mkdir -p ${OUTDIR}
echo "===== Starting Host FBOSS Read-only Collection ====="

# 1. FBOSS platform config & identity
fboss/platform/platform_manager/platform_manager --dump_platform_info > ${OUTDIR}/platform_manager_dump.txt
cp /etc/fboss/platform_config.json ${OUTDIR}/platform_config.json

# 2. PCI, I2C, MDIO
lspci -nnvv > ${OUTDIR}/host_lspci_detail.txt
i2cdetect -y 0 > ${OUTDIR}/i2c_bus0.txt 2>/dev/null
i2cdetect -y 1 > ${OUTDIR}/i2c_bus1.txt 2>/dev/null
i2cdetect -y 2 > ${OUTDIR}/i2c_bus2.txt 2>/dev/null
ls /dev/i2c-* > ${OUTDIR}/i2c_char_devs.txt
ls /sys/bus/mdio/devices/ > ${OUTDIR}/mdio_devices.txt

# I2C driver binding
> ${OUTDIR}/host_i2c_device_details.txt
for i in $(ls /sys/bus/i2c/devices/i2c-* | grep -v name); do
  echo "=== $i ===" >> ${OUTDIR}/host_i2c_device_details.txt
  cat $i/name >> ${OUTDIR}/host_i2c_device_details.txt 2>/dev/null
  cat $i/driver 2>/dev/null >> ${OUTDIR}/host_i2c_device_details.txt
done

# CPLD / FPGA platform nodes
ls /sys/devices/platform/ | grep -i "fpga\|cpld\|regmap" > ${OUTDIR}/fpga_cpld_candidates.txt

# Platform manager I2C detailed dump
fboss/platform/platform_manager/platform_manager --dump_i2c_info > ${OUTDIR}/pm_i2c_info.txt 2>/dev/null

# 3. Host device tree
dtc -I fs /proc/device-tree > ${OUTDIR}/host_dts.dts

# 4. FBOSS services & processes
systemctl list-units | grep fboss > ${OUTDIR}/fboss_systemd_units.txt
ps aux | grep fboss > ${OUTDIR}/fboss_processes.txt
cat /lib/systemd/system/fboss*.service > ${OUTDIR}/fboss_service_defs.txt 2>/dev/null

echo "===== Host collection complete, artifacts stored in ${OUTDIR} ====="
```

## Protocol & Traffic Capture Script (Run on Host, separate step)
> Purpose: Auto discover BMC IP, test Redfish / IPMI, run tcpdump capture for BMC<->Host OOB traffic
> Read-only, no modification.
```bash
#!/bin/bash
set -euo pipefail
OUTDIR="./comm_capture"
mkdir -p ${OUTDIR}
echo "===== BMC <-> Host communication discovery ====="

# Step 1: Extract BMC IP from FBOSS platform config
BMC_IP=$(grep -Eo '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' /etc/fboss/platform_config.json | head -n1)
echo "Detected BMC_IP=${BMC_IP}" > ${OUTDIR}/bmc_ip.txt

# Step2: Redfish test
curl -k -m 3 https://${BMC_IP}/redfish/v1/ > ${OUTDIR}/redfish_root_response.txt 2>&1

# Step3: IPMI test (standard + OEM command list, placeholder password bmc_password)
ipmitool -I lanplus -H ${BMC_IP} -U admin -P bmc_password help > ${OUTDIR}/ipmitool_help.txt 2>&1
ipmitool -I lanplus -H ${BMC_IP} -U admin -P bmc_password raw help > ${OUTDIR}/ipmi_raw_oem_help.txt 2>&1

# Step4: tcpdump capture (OOB interface, adjust interface name manually after checking `ip a`)
# This captures traffic between host and BMC for 30 seconds, read-only
OOB_IF="eth0"
echo "Starting 30s tcpdump capture on interface ${OOB_IF} for host<->BMC traffic..."
timeout 30 tcpdump -i ${OOB_IF} host ${BMC_IP} -nn -tt -w ${OUTDIR}/bmc_host_traffic.pcap 2>/dev/null

echo "===== Communication discovery finished. Check ${OUTDIR} ====="
echo "NOTE: You may need to modify OOB_IF variable to match real OOB ethernet interface (run ip a to find it)."
```

# How to use
1. On BMC:
   - `vi collect_bmc.sh` → paste script
   - `chmod +x collect_bmc.sh`
   - `./collect_bmc.sh`
2. On Host:
   - `vi collect_host.sh` → paste script
   - `chmod +x collect_host.sh`
   - `./collect_host.sh`
3. On Host (communication discovery):
   - `vi comm_capture.sh` → paste script
   - `chmod +x comm_capture.sh`
   - Modify variable `OOB_IF` to match actual OOB NIC name
   - `./comm_capture.sh`

# Post collection
Zip all 3 folders `bmc_collect`, `host_collect`, `comm_capture` and then feed into the report generation phase.
All collected artifacts are static logs, no runtime changes to switch.
