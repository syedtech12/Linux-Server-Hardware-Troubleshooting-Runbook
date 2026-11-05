# Corrections and validation

Review date: **2026-11-05**

## Scope

This update corrects the existing README and adds its source and validation records. The repository contained only README.md at the reviewed main-branch commit. No existing license or unrelated files were changed.

## Important corrections

| Draft issue | Correction |
| --- | --- |
| Unconfirmed credentials and completed-degree wording | Uses the GitHub maintainer identity and an educational project description. |
| Passwords in IPMI/curl arguments | Uses password prompts and non-secret configuration values. |
| Disabled TLS validation and fixed Redfish resource ID | Requires trusted CA/hostname validation, discovers resources, and checks asset identity. |
| Hard power cycle described as graceful | Separates OS-coordinated shutdown, ACPI soft request, and disruptive hard cycle. |
| Intervention before evidence preservation | Preserves full relevant logs, records limitations, and defines recovery gates. |
| Blanket filesystem repair | Uses offline, filesystem-specific non-modifying assessment with device, mount, backup, and failing-media prerequisites. |
| Universal optical claims and connector assumptions | Uses actual module/connector specifications, per-lane readings, and capability limits. |
| Unconditional cleaning and ambiguous scope safety | Uses laser controls and inspect, clean if needed, re-inspect, connect. |
| Copper TDR mixed with fiber tests | Distinguishes electrical TDR from optical OTDR. |
| Fixed PSU replacement time | Requires service-manual limits, surviving capacity, feed verification, and hot-swap support. |
| Overgeneralized sanitization/destruction rules | Explains method selection, cryptographic prerequisites, verification, validation, custody, and controlled failure handling. |
| Realistic ticket presented without a label | Replaces it with an explicitly fictional training record. |
| Tool-version differences in storage examples | Uses extended smartctl output and explains the nvme-cli 3.x command-name change. |
| Performance summaries treated as definitive diagnosis | Explains interval sampling, load-average limits, parallel-storage utilization, and socket-summary limits. |

## Checks

The following documentation checks passed locally:

- Parsed all **3 Markdown files** using a GitHub-flavored Markdown parser: **30 headings** and **8 tables**.
- Checked **56 link occurrences**, including **16 relative path or heading-anchor references**; all local targets resolved. External source pages were consulted individually, not checked by an exhaustive link crawler.
- Checked balanced code fences and consistent table columns.
- Parsed all **9 Bash blocks** with `bash -n`. The documented diagnostic and intervention examples were not executed.
- Scanned Bash examples for password arguments, disabled TLS verification, blanket automatic filesystem repairs, log zeroing, and erase commands; none were present.
- Generated local HTML for all 3 documents.

A browser visual preview was attempted but unavailable because the environment has no Chromium executable. GitHub light/dark appearance and visual layout were therefore not inspected. This update adds no images or image references.

## Limits

No server, BMC, switch, optical link, array rebuild, filesystem recovery, or sanitization was exercised. Shell parsing cannot prove that a command will work on a specific platform. Examples contain synthetic identifiers and require the installed tool's manual and hardware service procedure.

NIST's final publication and available FAQ were reviewed for relevant guidance. IEEE's public descriptions were reviewed; the paid full texts were unavailable for clause-level review. Source access details are in [sources.md](sources.md).

The wider project plan's separate workflow guides, reusable template files, and five visual assets are outside this README correction update.
