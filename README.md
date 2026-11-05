# Linux Server & Hyperscale Hardware Troubleshooting Runbook

**Author:** Syed (Wamiq Ali) Jafri  
**Credentials:** CompTIA Security+ (SY0-701) | Google IT Support Professional | Cisco CCNA 1 & IT Essentials | AAS in Cloud Computing  
**Focus:** Bare-Metal Server Diagnostics, Out-of-Band (IPMI/BMC/Redfish) Management, Layer 1/2 Optical Networking, Linux CLI Triage, and NIST SP 800-88 Rev. 2 Media Sanitization.

---

## 1. Core Troubleshooting Philosophy (The 6-Step MOP Loop)

In mission-critical data center operations, hardware and OS remediation must follow strict **Method of Procedures (MOPs)** to protect uptime, redundancy, and technician safety:

1. **Safety & Asset Verification:** Verify PPE, Lockout/Tagout (LOTO) boundaries, Electrostatic Discharge (ESD) grounding, rack/U coordinates, Asset ID/Barcode, and dual-feed PDU redundancy (A+B power rails) before touching hardware.
2. **Reproduce & Isolate (One Variable at a Time):** Never swap multiple Field Replaceable Units (FRUs) simultaneously. Isolate Layer 1 (physical) before Layer 2/3 (logical) or OS/kernel layers.
3. **Harvest Out-of-Band & In-Band Evidence:** Capture System Event Logs (SEL), kernel ring buffers (`dmesg`), and interface counters *before* rebooting or clearing logs.
4. **Remediate per MOP:** Execute the component swap, fiber cleaning, or firmware/OS recovery step.
5. **Verify Stability:** Confirm POST, sensor health, link negotiation, and zero error increments across a defined observation window.
6. **Document & Audit:** Log UTC timestamps, pre/post evidence outputs, and old/new FRU serial numbers for RMA and CMDB asset accuracy.

---

## 2. Out-of-Band (OOB) Management: IPMI, BMC & Redfish

When a host is hung, unresponsive over SSH, or failing POST, the Baseboard Management Controller (BMC) is the primary diagnostic interface.

```bash
# 1. Check chassis power status remotely
ipmitool -I lanplus -H <bmc_ip> -U <user> -P <password> power status

# 2. Gracefully cycle or hard power-cycle a hung host (per ticket MOP)
ipmitool -I lanplus -H <bmc_ip> -U <user> -P <password> power cycle

# 3. Pull System Event Log (SEL) to identify hardware faults (ECC, PSU, Thermal)
ipmitool -I lanplus -H <bmc_ip> -U <user> -P <password> sel elist | tail -n 25

# 4. Inspect real-time hardware sensors (Fan RPM, Inlet/Exhaust Temps, Voltage)
ipmitool -I lanplus -H <bmc_ip> -U <user> -P <password> sdr elist | grep -iE 'temp|fan|psu|volt'

# 5. Print Field Replaceable Unit (FRU) serials and part numbers for RMA tracking
ipmitool -I lanplus -H <bmc_ip> -U <user> -P <password> fru print

# 6. Open Serial-over-LAN (SOL) console to watch live BIOS POST codes and kernel boot
ipmitool -I lanplus -H <bmc_ip> -U <user> -P <password> sol activate
# (Exit SOL session safely using: ~. or Ctrl+\)

# 7. Modern Redfish API query (JSON hardware health check via HTTPS)
curl -sk -u <user>:<password> https://<bmc_ip>/redfish/v1/Systems/1 | jq '.Status, .PowerState'
```

---

## 3. Bare-Metal Component Triage & FRU Replacement Protocols

### A. Memory (DIMM) Faults — Correctable vs. Uncorrectable ECC
* **Symptom:** Unexpected host reboot, kernel panic, or degraded memory capacity; BMC SEL logs `Uncorrectable ECC` or `Memory Check Exception (MCE)`.
* **In-Band Linux Verification:**

```bash
# Check kernel ring buffer for EDAC (Error Detection and Correction) & MCE events
dmesg -T | grep -iE 'edac|mce|memory|hardware error'

# Query EDAC utility for specific memory controller, channel, and slot errors
edac-util -v

# Map physical DIMM slot designators, part numbers, and serial numbers via SMBIOS
sudo dmidecode -t memory | grep -A16 "Memory Device$"
```

* **Remediation Workflow:**
  1. Correlate the BMC SEL slot identifier with the motherboard silkscreen map.
  2. Verify host is powered down safely (unless hot-plug memory architecture is explicitly supported) and attach ESD wrist strap.
  3. Inspect slot for debris or bent pins; reseat or replace the faulty DIMM with a qualified spare.
  4. Boot host, verify full memory capacity via `free -h` / `lsmem`, and confirm 0 new ECC errors in `ipmitool sel list` and `dmesg -T`.

---

### B. Storage (NVMe / SATA / SAS HDD & SSD) Faults
* **Symptom:** Degraded RAID array, read-only filesystem mount, high I/O wait latency, or drive fault LED asserted.
* **In-Band Linux Verification:**

```bash
# List block devices, mount points, and partition hierarchy
lsblk -o NAME,MODEL,SERIAL,SIZE,TYPE,MOUNTPOINT,STATE

# Inspect SMART health attributes and error logs on SATA/SAS drives
sudo smartctl -a /dev/sdX

# Inspect NVMe SMART health log, percentage used, and media errors
sudo nvme list
sudo nvme smart-log /dev/nvmeXn1

# Check software RAID status (if applicable)
cat /proc/mdstat
```

* **Remediation Workflow:**
  1. Illuminate the physical drive bay locator LED (`ledctl locate=/dev/sdX` or vendor storage CLI) to prevent pulling a healthy member drive.
  2. Verify the failed drive's serial number against `smartctl`/`nvme list` output before unlatching the hot-swap carrier.
  3. Insert replacement drive, verify link negotiation and RAID/volume rebuild initiation, and route the removed drive directly to the **Secure Media Sanitization Workflow (Section 6)**.

---

### C. CPU, PCIe / NIC, and Power Supply (PSU) Triage

```bash
# Verify CPU topology, socket count, core count, and thermal throttling flags
lscpu
dmesg -T | grep -iE 'thermal|throttle|mce'

# Enumerate PCIe devices (NICs, GPUs/Accelerators, HBAs) and kernel driver bindings
lspci -nnk | grep -iA3 'net\|vga\|3d\|non-volatile'

# Check PSU redundancy and fan tachometer readings via IPMI
ipmitool sensor | grep -iE 'ps|pwr|fan'
```

* **PSU Hot-Swap Rule:** Before pulling a faulted PSU (amber LED / off), verify on the Rack PDU and BMC sensor output that the secondary PSU (Rail A or Rail B) is online and carrying the load. Swap within vendor thermal specification window (typically under 60 seconds) to maintain chassis static pressure and airflow.

---

## 4. Layer 1 & Layer 2 Data Center Networking (Clos Fabric & Optics)

90% of link-down and flapping interface tickets in Spine-Leaf / Top-of-Rack (ToR) architectures stem from physical Layer 1 issues (contaminated fiber end-faces, damaged patch cords, or failing transceivers).

### A. Host-Side Link & Counter Diagnostics

```bash
# Check interface administrative and operational state (LOWER_UP vs NO-CARRIER)
ip -br link show

# Inspect negotiated link speed, duplex, and transceiver signal presence
sudo ethtool eth0

# Query transceiver DOM (Digital Optical Monitoring): TX/RX optical power (dBm) & temp
sudo ethtool -m eth0

# Check hardware ring counters for CRC errors, symbol errors, and packet drops
sudo ethtool -S eth0 | grep -iE 'crc|err|drop|fifo|miss'

# Verify LLDP neighbor discovery (confirm exact ToR switch name and port ID)
sudo lldpctl eth0
```

### B. Optical & Copper Physical Layer Protocol

| Form Factor | Typical Speed | Media Type | Diagnostic Priority |
| :--- | :--- | :--- | :--- |
| **SFP+ / SFP28** | 10G / 25G | DAC / MMF / SMF | Check bend radius, latch seating, and DOM RX light levels |
| **QSFP+ / QSFP28** | 40G / 100G | MMF (MPO) / SMF (LC) | Scope all 4 lanes (MPO) or duplex LC ferrules before swap |
| **QSFP-DD / OSFP** | 400G / 800G | High-Density AI/Fabric | Strict thermal seating + zero-dust end-face inspection |

1. **Inspect Before You Connect:** Use a 200x/400x fiber inspection scope on both the transceiver port and patch cable ferrule.
2. **Clean Both Ends:** Use a dry-cloth click cleaner (LC/MPO) on both the cable connector and optic bulkhead. Never touch the ferrule or use canned air.
3. **Measure Light Levels:** Use an Optical Power Meter (OPM) or `ethtool -m` to verify RX power is within transceiver spec (e.g., `-1.0 dBm` to `-9.5 dBm`; a reading of `-40 dBm` indicates total loss of light/dark fiber).
4. **Isolate Component by Component:** Reseat cleaned cable -> Swap known-good transceiver optic -> Swap patch cable -> Request NOC port bounce / TDR test.

---

## 5. Linux Boot Failures & OS Performance Triage

### A. Host Stuck in POST, GRUB, or Emergency Shell
1. Connect via **IPMI SOL** (`ipmitool sol activate`) or crash-cart KVM to observe POST code or kernel panic stack trace.
2. Confirm BIOS/UEFI boot order and verify boot drive visibility.
3. If dropped to `initramfs` / emergency maintenance shell due to filesystem corruption:

```bash
# Identify root/boot block devices
blkid

# Run filesystem check on unmounted partition
fsck -y /dev/nvme0n1p2

# Inspect previous boot failure logs once mounted
journalctl -b -1 -p err
```

### B. The 60-Second Linux Performance Triage (USE Method: Utilization, Saturation, Errors)

```bash
uptime                          # 1, 5, 15-min load averages
dmesg -T | tail -n 30           # Recent kernel/hardware errors (OOM killer, I/O errors)
vmstat 1 5                      # Run queue, memory swap-in/out, CPU wait
mpstat -P ALL 1 3               # Per-core CPU utilization (check for single-core IRQ saturation)
pidstat 1 3                     # Per-process resource consumption
iostat -xz 1 3                  # Block device %util, await (latency in ms), and queue depth
free -h                         # Available physical RAM and swap usage
df -h && df -i                  # Disk capacity AND inode exhaustion check
ss -s                           # Socket summary (TCP established, time-wait, retransmits)
```

---

## 6. Data Security & Media Sanitization (NIST SP 800-88 Rev. 2 & IEEE 2883)

Any Information Storage Media (ISM)—including HDDs, SATA/SAS SSDs, NVMe drives, and NVDIMMs—removed during break-fix or server decommissioning must follow strict **Chain of Custody** and **NIST SP 800-88 Rev. 2 / IEEE 2883-2022** sanitization standards:

1. **Barcode & Asset Custody:** Scan the drive serial number and asset barcode immediately upon removal; log the state transition in the asset database before leaving the rack.
2. **Three Sanitization Tiers:**
   * **Clear:** Logical overwriting of user-addressable storage locations via standard read/write commands (used only for low-risk internal redeployment).
   * **Purge:** Logical or physical techniques (such as **Cryptographic Erase (CE)**, `nvme sanitize` Block Erase, or ATA Secure Erase per IEEE 2883) that render target data recovery infeasible even against laboratory-grade attacks.
   * **Destroy:** Physical destruction (industrial shredding, disintegration, or pulverization to <= 2mm particle size for solid-state flash chips) rendering the media permanently unusable.
3. **Failed Wipe / Dead Drive Rule:** If a failed SSD/HDD cannot power on to complete a verified logical **Purge** or fails post-wipe verification scanning, it must never leave the secure perimeter intact. Lock the drive in the designated secure destruction bin for on-site physical **Destruction** (shredding/pulverization) with a signed Certificate of Destruction.

---

## 7. Ticket Documentation Template (Audit-Ready Standard)

```text
[ASSET / HOST ID]: srv-dal-rack14-u18 (S/N: 9X8821A)
[SEVERITY / SLA]: SEV-2 (Redundancy Degraded)
[SYMPTOM]: 2026-11-04 14:10 UTC — Host alerted for Uncorrectable ECC memory fault; auto-rebooted.

[DIAGNOSIS & EVIDENCE]:
- 14:15 UTC: Connected via OOB BMC. `ipmitool sel elist` confirmed Uncorrectable ECC on CPU0_DIMM_A2.
- 14:18 UTC: Cross-checked OS logs (`dmesg -T | grep -i edac` & `edac-util -v`); confirmed UE count = 1 on Channel 0, Slot 2.
- 14:20 UTC: Pulled SMBIOS serial via `dmidecode -t memory` (Old DIMM S/N: MEM-449102).

[ACTION TAKEN]:
- 14:28 UTC: Verified host maintenance drain complete. Powered down chassis via IPMI.
- 14:34 UTC: Applied ESD grounding. Removed faulty DIMM A2 (S/N: MEM-449102) and installed spare DDR5 RDIMM (New S/N: MEM-992811).
- 14:40 UTC: Powered on chassis via BMC; monitored POST via SOL console.

[VERIFICATION & CLOSURE]:
- 14:46 UTC: Host booted cleanly to OS. `lsmem` and `free -h` confirm all 512GB RAM online.
- 15:46 UTC: 60-minute post-swap burn-in complete; 0 new MCE/EDAC events in `dmesg` or BMC SEL.
- Asset CMDB updated with new FRU serial. Old DIMM tagged for vendor RMA. Ticket Resolved.
```
