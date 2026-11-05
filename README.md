# Linux Server & Hardware Troubleshooting Runbook

**Maintainer:** [syedtech12](https://github.com/syedtech12)  
**Purpose:** An educational, vendor-neutral reference for Linux and server hardware troubleshooting.

This portfolio documents diagnostic reasoning, example commands, and maintenance planning. Examples are educational; the incident record is fictional. Hardware-specific procedures and local policy govern actual maintenance.

## Navigation

- [Prerequisites and command boundaries](#prerequisites-and-command-boundaries)
- [Six-step method of procedure](#six-step-method-of-procedure)
- [Out-of-band management](#out-of-band-management)
- [Component triage](#component-triage)
- [Host networking and optical hygiene](#host-networking-and-optical-hygiene)
- [Linux boot and performance](#linux-boot-and-performance)
- [Media sanitization and custody](#media-sanitization-and-custody)
- [Fictional incident record](#fictional-incident-record)
- [Sources](docs/sources.md) and [corrections and validation](docs/validation.md)

## Prerequisites and command boundaries

Use an authorized Linux host or disposable lab, Bash, an approved maintenance procedure, and the service manual for the exact hardware. Replace example hostnames, interfaces, and device paths after identifying the intended asset. `example.invalid` names are deliberately nonfunctional.

| Tools | Dependency or limitation |
| --- | --- |
| `ip`, `ss`, `lsblk`, `findmnt`, `journalctl` | Usually supplied by iproute2, util-linux, and systemd; availability and output vary by distribution. |
| `ipmitool`, `curl`, `jq` | Require a reachable management network, authorized account, and supported BMC interfaces. |
| `dmidecode`, `edac-util`, `lspci` | Firmware tables, kernel EDAC support, and hardware visibility vary. |
| `smartctl`, `nvme`, `mdadm` | Supplied by smartmontools, nvme-cli, and mdadm; controllers may hide drives or require documented pass-through options. |
| `ethtool`, `lldpctl` | Driver/module support varies; `lldpctl` requires a configured lldpd service. |
| `vmstat`, `mpstat`, `pidstat`, `iostat` | Usually procps/procps-ng and sysstat. |
| `e2fsck`, `xfs_repair` | Filesystem-specific rescue tools; offline assessment has separate prerequisites below. |

Install missing tools through the distribution's approved process. Commands marked `sudo` need appropriate privileges; other queries may also require elevated access on a hardened system. Missing tools, sensors, EDAC reports, digital optical monitoring (DOM), or locator support are visibility limitations, not proof that a component has failed or is healthy.

**Diagnostic queries** below inspect state; saving evidence writes local files. Console sessions also change session state. **Interventions** such as power operations, cable removal, component swaps, filesystem repair, firmware changes, and sanitization require an approved, asset-specific MOP. A diagnostic result alone does not authorize an intervention.

## Six-step method of procedure

A method of procedure (MOP) describes the scope, dependencies, approvals, exact asset, maintenance window, expected impact, stop conditions, rollback or escalation path, and closure criteria.

| Step | Work and decision gate |
| --- | --- |
| 1. Verify asset and safety | Match ticket, hostname, rack/U location, serial, and service impact. Check electrostatic discharge (ESD) controls and applicable lockout/tagout boundaries. Confirm backups, workload drain, spare compatibility, redundancy, and recovery access. |
| 2. Isolate the fault | Establish the symptom and timeline. Choose the next diagnostic based on evidence; investigate physical links when link symptoms warrant it. Change one planned variable at a time. |
| 3. Preserve evidence | Save full relevant BMC and OS logs, inventories, sensor readings, and counters before rebooting, clearing logs, or replacing parts. Record UTC timestamps and time-source differences. |
| 4. Perform the approved intervention | Use the exact hardware procedure. Stop on an asset mismatch, unexpected loss of redundancy, new damage, uncertain device mapping, or an unexpected result. |
| 5. Verify recovery | Check boot, inventory, sensors, application health, and new error increments against the recorded baseline during a defined observation window. |
| 6. Document and close | Record actions, evidence, old/new field-replaceable unit (FRU) serials, inventory updates, and custody. Close only after the agreed recovery criteria pass. Otherwise follow rollback or escalation. |

Rollback must describe a feasible recovery action and its owner. If a swap or sanitization cannot be reversed, define the recovery or escalation path before starting.

Example evidence capture on a systemd host. First choose an existing, protected persistent directory with enough free space. Run each line separately and stop on failure. Ensure the evidence is retained in the approved store before reboot or other intervention:

```bash
umask 077
EVIDENCE_BASE=/path/to/approved/persistent-evidence
EVIDENCE_DIR=$(mktemp -d "$EVIDENCE_BASE/runbook-evidence.XXXXXX")
date -u +'%Y-%m-%dT%H:%M:%SZ' > "$EVIDENCE_DIR/captured-at.txt"
sudo journalctl -k -b --utc --no-pager > "$EVIDENCE_DIR/kernel-current-boot.log"
sudo journalctl -b -1 --utc --no-pager > "$EVIDENCE_DIR/previous-boot.log"
```

Check each command's exit status and output. A missing previous boot can mean persistent journal storage is unavailable. Preserve stderr and note inaccessible evidence rather than treating an empty file as a clean bill of health. Logs and inventory can contain sensitive details; redact copies before publication.

## Out-of-band management

A baseboard management controller (BMC) can provide out-of-band (OOB) access when the OS is unavailable. Reach it through the approved management network. Intelligent Platform Management Interface (IPMI), Redfish, serial-over-LAN (SOL), and sensor availability depend on the platform.

### IPMI evidence and console access

Set these non-secret values to the verified asset. Each remote command prompts for a password because no password option is supplied:

```bash
BMC_HOST=bmc.example.invalid
BMC_USER=operator

ipmitool -I lanplus -H "$BMC_HOST" -U "$BMC_USER" chassis power status
ipmitool -I lanplus -H "$BMC_HOST" -U "$BMC_USER" sel elist
ipmitool -I lanplus -H "$BMC_HOST" -U "$BMC_USER" sdr elist
ipmitool -I lanplus -H "$BMC_HOST" -U "$BMC_USER" fru print

# Use the evidence directory created above; preserve the complete SEL.
ipmitool -I lanplus -H "$BMC_HOST" -U "$BMC_USER" sel elist > "$EVIDENCE_DIR/bmc-sel.txt"

# Authorized interactive console session; do not send input blindly.
ipmitool -I lanplus -H "$BMC_HOST" -U "$BMC_USER" sol activate
```

The system event log (SEL) can identify reported memory, power, and thermal events. Correlate its timestamps and component labels with OS evidence and the hardware slot map. A report can implicate a channel, socket, firmware, or power path rather than uniquely identify a failed FRU. FRU inventory can be incomplete.

Use the SOL escape sequence reported by the client, normally `~.` at the start of a new line. A surrounding SSH session can intercept escape characters.

**Power operations are interventions:**

| IPMI operation | Meaning and recovery condition |
| --- | --- |
| `chassis power soft` | Requests an ACPI soft shutdown where supported; OS cooperation is required. Confirm shutdown rather than assuming success. |
| `chassis power cycle` | Removes and restores chassis main power. It is a disruptive hard cycle, not a graceful reboot, and can lose pending writes. Use only as an approved fallback after evidence capture and recovery planning. |

Prefer a coordinated OS shutdown when available. Do not confuse a chassis power operation with a BMC reset or removal of AC input.

References: [upstream ipmitool manual](https://github.com/ipmitool/ipmitool/blob/master/doc/ipmitool.1.in).

### Redfish with TLS validation and resource discovery

Use a BMC hostname that matches its certificate and a CA certificate obtained through a trusted channel. The example uses `curl --user "$BMC_USER"` to prompt for a password. Resolve certificate failures through the approved trust process.

Run each step separately and stop if it fails. This Bash helper deliberately accepts only service-relative Redfish paths on the configured BMC:

```bash
set -o pipefail
BMC_ORIGIN=https://bmc.example.invalid
BMC_USER=operator
BMC_CA=/path/to/approved-management-ca.pem

rfget() {
    case "$1" in
        /redfish/v1/*) ;;
        *) printf '%s\n' 'Expected a verified /redfish/v1/ path.' >&2; return 2 ;;
    esac
    curl --fail --silent --show-error --proto '=https' \
        --connect-timeout 10 --max-time 30 \
        --cacert "$BMC_CA" --user "$BMC_USER" "$BMC_ORIGIN$1"
}

SYSTEMS_URI=$(rfget /redfish/v1/ | jq -er '.Systems["@odata.id"]')
rfget "$SYSTEMS_URI" | jq '{members: [.Members[]?["@odata.id"]], next: .["Members@odata.nextLink"]}'

# Choose a member from the returned collection; do not assume /Systems/1.
read -r -p 'Verified system member path: ' SYSTEM_URI
rfget "$SYSTEM_URI" | jq '{Id, SerialNumber, Status, PowerState}'
```

If the collection returns `Members@odata.nextLink`, inspect the remaining pages before choosing the asset. Confirm the selected member's serial against the ticket. Absolute links or other URI forms need a client that validates the same origin before sending credentials; do not blindly concatenate them. Some implementations omit properties or require different account privileges.

References: [DMTF Redfish specification](https://www.dmtf.org/sites/default/files/standards/documents/DSP0266_1.25.0.pdf), [resource and schema guide](https://www.dmtf.org/sites/default/files/standards/documents/DSP2046_2026.2.pdf), and [curl manual](https://curl.se/docs/manpage.html).

## Component triage

For each component, preserve the symptom, timestamp, full evidence, device/slot mapping, and the decision behind the next action.

### Memory and ECC

**Symptoms:** reduced memory capacity, unexpected reboot, or error-correcting code (ECC) events.

```bash
sudo journalctl -k -b --utc --no-pager
sudo dmesg | grep -iE 'edac|mce|memory|hardware error'
sudo edac-util -v
sudo dmidecode --type memory
free -h
lsmem
```

**Interpretation and action:** Correlate correctable/uncorrectable errors, recurrence, and physical location across the SEL, kernel, and service manual. A machine-check exception (MCE) is not automatically a DIMM fault; memory controllers, CPUs, and other hardware can be involved. Missing EDAC output is inconclusive. Confirm population rules and replacement compatibility. Drain the host and power it down as the manual requires before a reseat or swap.

**Recovery and escalation:** Compare expected versus detected capacity and new ECC/MCE events over the approved observation window. Escalate uncertain slot mapping, repeated errors after replacement, multi-component faults, or an unexpected capacity change. Absence of new errors during a short window does not prove long-term reliability.

### Storage and RAID

**Symptoms:** I/O errors, a degraded array, read-only filesystem, latency, or a drive alert.

```bash
lsblk -o NAME,PATH,MODEL,SERIAL,SIZE,TYPE,FSTYPE,MOUNTPOINTS
sudo smartctl -x /dev/sdX
sudo nvme list
sudo nvme smart-log /dev/nvme0
cat /proc/mdstat
sudo mdadm --detail /dev/mdX
```

Replace `/dev/sdX`, `/dev/nvme0`, and `/dev/mdX` only after mapping the intended drive, controller, or array. The paths are examples, not a matching set. Use the appropriate query for the actual hardware. Remove unsupported optional `lsblk` columns if the installed version lacks them. The upstream nvme-cli 3.x documentation prefers `nvme log smart`; `nvme smart-log` is the older compatibility spelling shown here. Check the installed version's help.

**Interpretation and action:** Correlate health logs, error history, controller/array membership, serial, and bay. A SMART overall-health pass does not exclude a fault. Hardware RAID may need the controller's supported diagnostic tool; a Linux device name need not identify an individual physical drive. Confirm backup/recovery status, remaining members, replacement compatibility, and documented hot-swap support before removal. A locator is a state-changing identification aid, not sufficient proof of identity.

**Recovery and escalation:** Follow the array-specific replacement procedure; rebuilding can be automatic or require an explicit approved action. Verify membership, rebuild completion, application I/O, and error trends. Stop if multiple members are suspect, mapping is uncertain, or recovery reads fail. A failing device may need preservation or imaging by a recovery specialist before further scans. Keep removed media under the [custody process](#media-sanitization-and-custody).

References: [upstream smartctl manual](https://github.com/smartmontools/smartmontools/blob/main/src/smartctl.8.in), [nvme-cli migration notes](https://github.com/linux-nvme/nvme-cli/blob/master/NEWS.md), and [mdadm manual](https://man7.org/linux/man-pages/man8/mdadm.8.html).

### CPU, PCIe/NIC, thermal, and PSU

**Symptoms:** hardware errors, a missing device, throttling, fan/temperature alerts, or degraded power redundancy.

```bash
lscpu
lspci -nnk
sudo journalctl -k -b --utc --no-pager
ipmitool -I lanplus -H "$BMC_HOST" -U "$BMC_USER" sensor
```

**Interpretation and action:** Correlate kernel errors with sensors, firmware inventory, driver binding, load, and the exact platform's thresholds. Peripheral Component Interconnect Express (PCIe) errors can involve an endpoint, slot, riser, upstream link, or firmware. Inspect airflow obstructions and fan state before attributing a thermal alert to the CPU.

For a power supply unit (PSU) swap, verify actual hot-swap support, compatible spares, the surviving PSU's available capacity at the current load/input conditions, feed health, and thermal restrictions. Verify A/B feed independence at the upstream power path; two installed PSUs do not establish it. Use the service manual's vacancy/airflow limits, not a universal replacement time. Do not open a PSU or work inside an energized enclosure outside the approved procedure.

**Recovery and escalation:** Recheck inventory, sensor trends, new hardware errors, power redundancy, and workload health. Stop on insufficient surviving capacity, loss of the remaining feed, unsafe temperatures, or unclear replacement instructions. No generic sensor value or LED color alone authorizes removal.

## Host networking and optical hygiene

A leaf-spine Clos fabric commonly connects each leaf to each spine. Those uplinks can provide alternate fabric paths, but a host attached to only one leaf still depends on that access link and leaf. Host redundancy requires a supported host/access design; it is not implied by the fabric.

### Host-to-leaf diagnosis

**Symptoms:** link down, flapping, errors, loss, or poor throughput. Record the affected host port and expected top-of-rack (ToR)/leaf switch port.

```bash
ip -br link show
IFACE=enp1s0  # Replace with the verified interface from the listing.
ip -s link show dev "$IFACE"
sudo ethtool "$IFACE"
sudo ethtool -m "$IFACE"
sudo ethtool -S "$IFACE"
sudo lldpctl "$IFACE"
```

**Interpretation and action:** Separate administrative state from carrier state. Compare host and switch settings, supported speed, autonegotiation, forward error correction (FEC), transceiver/cable compatibility, and interface-counter increments across matching observation windows. Link Layer Discovery Protocol (LLDP) can help map a neighbor when the agent and peer support it; an absent neighbor does not prove a disconnected cable.

DOM values and per-lane alarms depend on the module, driver, and interface. Compare reported transmit/receive power to the specific optic's wavelength, lane, and vendor limits. There is no universal acceptable dBm range or universal number meaning "no light." Unsupported DOM is inconclusive. Counter names and meanings are driver-specific; inspect the full output before filtering.

### Inspect, clean if needed, re-inspect, connect

1. **Make the link safe.** Coordinate the outage and disable optical sources according to the equipment procedure. Verify the path is not live before inspection. Never look into a connector or use a direct-view microscope on a potentially live fiber; use a suitable video inspection probe and the required laser controls.
2. **Inspect both mating end faces.** Use the correct adapter and inspection criteria for the actual connector, polish, and lane arrangement. A form factor alone does not establish LC/MPO connector type, fiber count, or lane mapping.
3. **Clean only when needed.** Use the connector manufacturer's approved tools and technique. Avoid touching ferrules. Cleaning a clean end face can introduce contamination.
4. **Re-inspect.** Connect only after both mating surfaces meet the applicable criteria. Stop for damage or persistent contamination; follow the approved replacement/escalation path.
5. **Verify recovery.** Reconnect the verified path, restore the approved settings, and check carrier, speed/FEC, optical readings, new errors, and end-to-end traffic over the observation window.

An optical power meter must suit the wavelength and test setup. Copper time-domain reflectometry (TDR) tests electrical cable pairs; optical time-domain reflectometry (OTDR) tests fiber and needs its own approved setup. Port bounces, reseats, swaps, and active cable tests can interrupt service and belong in the intervention plan.

Escalate persistent errors, incompatible optics, damaged connectors, readings outside the exact module limits, or a path that cannot be verified safely.

References: [ethtool manual](https://man7.org/linux/man-pages/man8/ethtool.8.html), [kernel ethtool interfaces](https://docs.kernel.org/networking/ethtool-netlink.html), and [manufacturer inspection guidance](https://www.cisco.com/c/en/us/support/docs/optical/synchronous-digital-hierarchy-sdh/51834-cleanfiber2.html). Manufacturer guidance is illustrative; use the actual equipment's instructions.

## Linux boot and performance

### Boot-stage and filesystem assessment

**Symptoms:** power-on self-test (POST) failure, boot-loader failure, kernel panic, initramfs prompt, or emergency mode.

1. Observe the console and identify the failing stage. A POST failure is not yet a Linux filesystem diagnosis.
2. Preserve the error, BMC events, and accessible journal. Check boot device visibility, firmware boot mode/order, storage health, and recent changes.
3. For an emergency shell, distinguish a missing device, an incorrect mount configuration, an unavailable encrypted/LVM/RAID layer, and actual filesystem damage.
4. Before offline scans, confirm the exact filesystem-bearing device, filesystem type, mount state in all relevant namespaces, backup/recovery plan, and stable storage. Use rescue media for a root filesystem that cannot be unmounted. Stop and escalate failing hardware before intensive scanning.

**Offline, non-modifying assessment examples:** use only after those prerequisites pass; replace the synthetic path and run only the command matching the detected filesystem.

```bash
FS_DEVICE=/dev/mapper/example-root
lsblk -f
findmnt --source "$FS_DEVICE"

# ext2/ext3/ext4 only; force a scan and decline modifications.
sudo e2fsck -f -n "$FS_DEVICE"

# XFS only; scan without modifying the filesystem.
sudo xfs_repair -n "$FS_DEVICE"
```

A `findmnt` miss alone is insufficient: trace aliases and stacked storage, and confirm no active mounts elsewhere. Even a non-modifying scan reads the device and can stress failing media. Its result is not a complete recovery guarantee. An unreplayed journal or XFS dirty log can limit assessment; escalate for a planned recovery decision. Mounting for journal replay can write metadata.

Actual repair is a separate intervention after evidence and backups are protected. Do not automatically approve all repairs or zero an XFS log as a routine fix; log zeroing can lose data. After approved recovery, verify mounts, boot, application data, and new storage/kernel errors.

References: [e2fsck manual](https://man7.org/linux/man-pages/man8/e2fsck.8.html) and [xfs_repair manual](https://man7.org/linux/man-pages/man8/xfs_repair.8.html).

### Performance triage with the USE method

The utilization, saturation, and errors (USE) method checks each resource for busy time/capacity use, queued work, and errors. Start with the workload symptom and collect comparable time windows:

```bash
uptime
vmstat 1 5
mpstat -P ALL 1 3
pidstat 1 3
iostat -xz -y 1 3
free -h
df -h
df -i
ss -s
sudo journalctl -k -b --utc --no-pager
```

Load average is not CPU utilization; it can include tasks in uninterruptible sleep. Ignore initial since-boot summaries when comparing interval samples; `iostat -y` omits its first since-boot report. Check CPU queues, memory pressure/swap, latency, I/O queues, filesystem space/inodes, and contemporaneous errors. A device's `%util` alone does not establish saturation on parallel SSDs or arrays. `ss -s` is a socket summary, not a retransmission-counter report.

Change configuration only after forming a supported hypothesis. Verify application latency/error rates and the affected resource after intervention. Escalate sustained pressure, repeated I/O or hardware errors, or a performance regression without a clear cause.

References: [Brendan Gregg's USE method](https://www.brendangregg.com/usemethod.html) and [upstream iostat manual](https://github.com/sysstat/sysstat/blob/master/man/iostat.in).

## Media sanitization and custody

Use [NIST SP 800-88 Rev. 2](https://csrc.nist.gov/pubs/sp/800/88/r2/final) and its [July 2026 FAQ](https://csrc.nist.gov/files/pubs/sp/800/88/r2/final/docs/sp800-88r2-faq.pdf) for program and assurance guidance. Technology-specific techniques require applicable standards, device documentation, and policy. [IEEE 2883-2022](https://standards.ieee.org/ieee/2883/10277/) covers sanitizing storage; [IEEE 2883.1-2025](https://standards.ieee.org/ieee/2883.1/11015/) provides related recommended practice. The public IEEE descriptions were reviewed; the paid full texts were not available for clause-level verification.

### Method selection

| Method | Intended result and limitation |
| --- | --- |
| **Clear** | Addresses user-accessible data against recovery through the ordinary interface. Its suitability depends on media, data sensitivity, and disposition; it is not restricted to one universal reuse scenario. |
| **Purge** | Makes target-data recovery infeasible even with advanced laboratory techniques, using a suitable logical or physical technique. Qualification depends on the device, implementation, scope, and assurance. |
| **Destroy** | Makes recovery infeasible and leaves the media unusable for storage. The destruction process and acceptance criteria must suit the actual media and data sensitivity. |

Determine the media technology, identity, stored-data sensitivity, intended reuse/transfer/disposal, device capabilities, and organizational policy before selecting a method. Formatting, deleting files, or an unspecified factory reset does not establish sanitization. A command called "secure erase" is not automatically a qualifying Purge.

Flash overwrite can miss prior data in remapped or overprovisioned cells. Cryptographic erase requires that the target data was protected by suitable encryption before storage, that the implementation and key management are trustworthy, and that all relevant keys and recoverable copies are addressed. Prior plaintext or keys retained in escrow/backups can defeat its intended assurance.

### Controlled workflow

1. **Identify and retain custody.** Record the media serial, source asset, condition, classification, approved disposition, and authorization. Protect evidence/retention obligations before any erase. Log every handoff with UTC time, custodian, secure location/container, and receiving acknowledgment.
2. **Apply the approved technique.** Match it to the exact device and required method. This runbook supplies no blanket erase command.
3. **Verify the outcome.** Inspect completion/status records, errors, anomalies, media condition, and technique-specific evidence. A successful exit status alone is insufficient; destructive techniques require appropriate inspection of the outcome.
4. **Validate acceptance.** An authorized reviewer evaluates the verification evidence and remaining confidentiality risk against policy. Release only after acceptance; document rejection and the next action.
5. **Record disposition.** Retain identity, method/technique, tool/version, execution evidence, verification, validation decision, responsible people, timestamps, custody, and final reuse/transfer/destruction records.

On failure or an inoperable drive, maintain controlled custody and escalate for an effective alternative, potentially destruction. Off-site handling can be permitted through an approved provider and custody controls; on-site destruction is not a universal rule. There is no universal 2 mm particle-size requirement in this runbook. Follow the applicable media-specific standard and approved acceptance criteria. A destruction certificate supports the record but does not replace assurance.

## Fictional incident record

**Training example only.** The identities, times, capacity, and observations below are invented to demonstrate documentation; they are not evidence of completed work.

```text
Asset: lab-host-01.example.invalid
Ticket: TRAINING-001
Symptom: Reported uncorrectable memory event and restart.
Scope: One verified host; approved memory diagnosis and maintenance.

Evidence:
- Record UTC capture times, SEL and kernel-log references.
- Correlate reported location with the platform's physical slot map.
- Record detected capacity and baseline counters; note visibility limits.

Approved action:
- Drain workload; confirm recovery access and compatible spare.
- Shut down and follow the hardware service procedure.
- Record fictional old/new FRU identifiers: DEMO-OLD / DEMO-NEW.

Recovery criteria:
- POST and expected memory inventory pass.
- Application checks pass.
- Compare new hardware errors over the agreed observation window.
- If criteria fail, keep the ticket open and escalate.

Closure:
- Record actual results and observation duration when used in a lab.
- Update inventory and follow media/custody policy as applicable.
- Link evidence and obtain the required closure acceptance.
```

See [sources and access limits](docs/sources.md) and [corrections and validation](docs/validation.md) for the review record.

