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
