# Sources and access limits

Reviewed on **2026-11-05**. These references support the [runbook](../README.md); they do not establish compliance or replace an asset-specific service procedure.

## Sanitization

| Source | Used for | Access and qualification |
| --- | --- | --- |
| [NIST SP 800-88 Rev. 2 publication page](https://csrc.nist.gov/pubs/sp/800/88/r2/final) and [final publication](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-88r2.pdf) | Program guidance, method selection, cryptographic erase prerequisites, assurance, custody, and records. | Final September 2025 publication; relevant sections reviewed. |
| [NIST SP 800-88r2 FAQ](https://csrc.nist.gov/files/pubs/sp/800/88/r2/final/docs/sp800-88r2-faq.pdf) | Clarifies the program focus, media-specific references, and verification versus validation. | July 16, 2026 FAQ linked by NIST's publication page. |
| [IEEE 2883-2022](https://standards.ieee.org/ieee/2883/10277/) | Identifies the standard for sanitizing storage. | Public IEEE scope and active-status listing reviewed; paid full text not accessed. |
| [IEEE 2883.1-2025](https://standards.ieee.org/ieee/2883.1/11015/) | Identifies the related recommended practice for using storage sanitization methods. | Public IEEE scope and active-status listing reviewed; paid full text not accessed. This is a related practice, not a claim that it replaces IEEE 2883-2022. |

No clause-level IEEE requirements, universal destruction particle size, blanket on-site destruction requirement, or universal qualification of an erase command is asserted. Use the applicable full standard, organizational policy, and device documentation to approve a real technique.

## Management interfaces

| Source | Used for |
| --- | --- |
| [Upstream ipmitool manual](https://github.com/ipmitool/ipmitool/blob/master/doc/ipmitool.1.in) | Password prompting, lanplus, evidence queries, SOL, and chassis power operations. |
| [DMTF Redfish specification, DSP0266 1.25.0](https://www.dmtf.org/sites/default/files/standards/documents/DSP0266_1.25.0.pdf) | Service root, linked resources, collections, and pagination. |
| [DMTF Redfish resource and schema guide, DSP2046 2026.2](https://www.dmtf.org/sites/default/files/standards/documents/DSP2046_2026.2.pdf) | Computer-system resources and schema-dependent properties. |
| [curl manual](https://curl.se/docs/manpage.html) | Username-only password prompting, CA trust, TLS validation, and failure handling. |

A platform can implement an older Redfish version or a subset of capabilities. The example requires service-relative paths and a trusted CA; other URI forms require deliberate same-origin handling.

## Linux and physical links

| Source | Used for |
| --- | --- |
| [Upstream smartctl manual](https://github.com/smartmontools/smartmontools/blob/main/src/smartctl.8.in) | Extended drive health and log queries; controller and device limitations. |
| [Upstream nvme-cli migration notes](https://github.com/linux-nvme/nvme-cli/blob/master/NEWS.md) and [smart-log compatibility manual](https://github.com/linux-nvme/nvme-cli/blob/master/Documentation/nvme-smart-log.txt) | The older smart-log spelling and the 3.x log smart naming change. |
| [mdadm manual](https://man7.org/linux/man-pages/man8/mdadm.8.html) | Read-only array detail queries and array identity. |
| [edac-util manual](https://manpages.debian.org/unstable/edac-utils/edac-util.1.en.html) | Verbose EDAC reporting and dependence on kernel-exported information. |
| [e2fsck manual](https://man7.org/linux/man-pages/man8/e2fsck.8.html) | ext2/ext3/ext4 assessment, non-modifying mode, and mounted-filesystem restrictions. |
| [xfs_repair manual](https://man7.org/linux/man-pages/man8/xfs_repair.8.html) | XFS non-modifying assessment, dirty-log handling, and log-zeroing risks. |
| [ethtool manual](https://man7.org/linux/man-pages/man8/ethtool.8.html) | Link, driver counters, module information, and cable-test capabilities. |
| [Linux kernel ethtool interfaces](https://docs.kernel.org/networking/ethtool-netlink.html) | Module/driver capability dependence and electrical cable-test behavior. |
| [Cisco fiber inspection and cleaning procedure](https://www.cisco.com/c/en/us/support/docs/optical/synchronous-digital-hierarchy-sdh/51834-cleanfiber2.html) | Laser precautions, video inspection, cleaning when needed, and re-inspection. |
| [Brendan Gregg's USE method](https://www.brendangregg.com/usemethod.html) | Resource utilization, saturation, and errors as diagnostic categories. |
| [Upstream sysstat iostat manual](https://github.com/sysstat/sysstat/blob/master/man/iostat.in) | Interval reports and the limits of device utilization on parallel storage. |

The man7 pages reproduce the respective projects' manuals; they are used for the tool authors' documentation rather than third-party tutorials. Manufacturer guidance is an example of an equipment procedure, not an endorsement or a universal prescription for different hardware.

All examples still require checking the installed tool's help/manual and the actual server, controller, filesystem, transceiver, and cable documentation. No real hardware or service recovery was validated for this documentation update.

See [corrections and validation](validation.md).
