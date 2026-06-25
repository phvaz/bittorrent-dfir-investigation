# Chain of Custody — BitTorrent DFIR Investigation

## Case Information

| Field | Value |
|---|---|
| Case name | BitTorrent DFIR Investigation |
| Examiner | Paulo Vaz |
| Purpose | Academic portfolio — DFIR investigation simulation |

---

## Evidence Description

| Field | Value |
|---|---|
| Description | Windows 10 Enterprise LTSC 21H2 virtual machine disk |
| Source file | `windows-10-dfir-target.vdi` (VirtualBox Dynamic Disk Image) |
| Source location | `C:\Users\Paulo Vaz\VirtualBox VMs\windows-10-dfir-target\` |
| Logical size | 60 GiB (64,424,509,440 bytes) |
| Operating system | Windows 10 Enterprise LTSC 21H2 (10.0.19044.1288) |

---

## Acquisition 01 — Initial Attempt (Incorrect Source)

| Step | Timestamp | Action | Tool | Operator |
|---|---|---|---|---|
| 1 | Jun 22, 2026 ~14:57 | VDI converted to raw image | qemu-img | Paulo Vaz |
| 2 | Jun 22, 2026 ~15:10 | SHA-256 calculated on raw image | sha256sum | Paulo Vaz |
| 3 | Jun 22, 2026 17:49:17 | E01 acquisition started | ewfacquire 20140816 | Paulo Vaz |
| 4 | Jun 22, 2026 17:52:39 | E01 acquisition completed | ewfacquire 20140816 | Paulo Vaz |
| 5 | Jun 22, 2026 17:58:59 | E01 integrity verification started | ewfverify 20140816 | Paulo Vaz |
| 6 | Jun 22, 2026 18:00:58 | E01 integrity verification completed | ewfverify 20140816 | Paulo Vaz |

**Integrity Hashes — Acquisition 01:**

| Artifact | Algorithm | Hash | Status |
|---|---|---|---|
| windows-target.raw (v1) | SHA-256 | `80f206ef92486901742c5c4f5a9a536d7e69b978a5ed64ba1dc0075c8f8fb08c` | Pre-acquisition reference |
| windows-target.E01 (stored) | MD5 | `41f3b630078af93662a45f37d3d6ee9b` | Generated during acquisition |
| windows-target.E01 (verified) | MD5 | `41f3b630078af93662a45f37d3d6ee9b` | Verified post-acquisition ✅ |

> **Status: INVALIDATED** — Post-acquisition analysis revealed this image captured the VDI base disk frozen at the `clean-install` snapshot state (Jun 18, 2026), before qBittorrent installation or any torrent activity. See Acquisition Revision below.

---

## Acquisition Revision — VirtualBox Snapshot Issue

### Discovery

During Phase 04 disk analysis, TSK filesystem traversal returned no qBittorrent artifacts from the acquired image despite the software having been installed and used. This triggered an investigation into the acquisition source.

### Root Cause

VirtualBox uses a differential disk architecture for snapshots. When a snapshot is taken, the base VDI is frozen and all subsequent writes go to differential VDI files in the `Snapshots/` folder. The base VDI (`windows-10-dfir-target.vdi`) was frozen at the `clean-install` state (Jun 18, 2026). All evidence — qBittorrent installation, torrent download, configuration files — resided in the snapshot differential chain, with the most recent state in `{f8e3eb22-1eef-4c0c-8d50-1ea5f2f48fd2}.vdi` (15.7 GiB, Jun 22, 2026).

### Actions Taken

**Jun 25, 2026 — Pre-consolidation backup:**
Before any modification, a full backup of the entire `windows-10-dfir-target/` directory — including all snapshot VDI files — was copied to two separate storage devices:
- External HD (NTFS, 4096-byte allocation units)
- USB drive (reformatted to exFAT to support files >4GB)

The original snapshot chain is preserved on both backup devices and remains verifiable.

**Jun 25, 2026 — Snapshot consolidation:**
VirtualBox snapshots were consolidated into the base VDI by deleting snapshots from oldest to newest. VirtualBox automatically merges each differential VDI into the next level during deletion.

> **Forensic note:** Consolidating snapshots modifies the base VDI — this alters the original evidence artifact. The correct forensic approach would have been to point `qemu-img` directly at the most recent snapshot VDI (`{f8e3eb22...}.vdi`), which resolves the differential chain automatically without modifying any files. This lesson is documented here as a real-world investigative learning outcome.
>
> This issue is specific to virtualized environments. In physical disk forensics, a hardware write blocker prevents any modification to the original evidence regardless of operations performed on the acquired copy.

**Post-consolidation VDI state:**
- Before: `windows-10-dfir-target.vdi` = 9.2 GiB (frozen at clean-install)
- After: `windows-10-dfir-target.vdi` = ~21 GiB (full consolidated state)

**TSK verification before second acquisition:**
TSK confirmed presence of expected artifacts in the consolidated raw image before proceeding with the second acquisition. Key artifacts confirmed: qBittorrent installation, `.torrent` file, logs, configuration files, Prefetch entries.

---

## Acquisition 02 — Correct Image

| Step | Timestamp | Action | Tool | Operator |
|---|---|---|---|---|
| 1 | Jun 25, 2026 ~11:28 | Consolidated VDI converted to raw | qemu-img | Paulo Vaz |
| 2 | Jun 25, 2026 ~11:44 | SHA-256 calculated on raw image v2 | sha256sum | Paulo Vaz |
| 3 | Jun 25, 2026 12:02 | TSK artifact verification performed | fls / grep | Paulo Vaz |
| 4 | Jun 25, 2026 12:02 | E01 acquisition v2 started | ewfacquire 20140816 | Paulo Vaz |
| 5 | Jun 25, 2026 12:32:34 | E01 acquisition v2 completed | ewfacquire 20140816 | Paulo Vaz |
| 6 | Jun 25, 2026 12:38 | E01 integrity verification started | ewfverify 20140816 | Paulo Vaz |
| 7 | Jun 25, 2026 12:40:58 | E01 integrity verification completed | ewfverify 20140816 | Paulo Vaz |

**Integrity Hashes — Acquisition 02:**

| Artifact | Algorithm | Hash | Status |
|---|---|---|---|
| windows-target.raw (v2) | SHA-256 | `dff2ba411365bbf36cf15306d036f2d24a0fff9bdc895e677ab6f8abab3b4e4e` | Pre-acquisition reference |
| acquisition-v2/windows-target.E01 (stored) | MD5 | `8e01029edeadbfffb36fe8516afb54df` | Generated during acquisition |
| acquisition-v2/windows-target.E01 (verified) | MD5 | `8e01029edeadbfffb36fe8516afb54df` | Verified post-acquisition ✅ |

> **Status: ACTIVE** — This is the authoritative forensic image used for all analysis in Phases 04–06.

---

## Acquisition Conditions

| Condition | Acquisition 01 | Acquisition 02 |
|---|---|---|
| Target VM powered off | ✅ Yes | ✅ Yes |
| Correct evidence source identified | ❌ No (base VDI) | ✅ Yes (consolidated VDI) |
| Pre-acquisition TSK verification | ❌ No | ✅ Yes |
| Hardware write blocker | ❌ No (lab environment) | ❌ No (lab environment) |
| Two independent hash algorithms | ✅ SHA-256 + MD5 | ✅ SHA-256 + MD5 |
| Original evidence backed up | N/A | ✅ Yes (2 devices) |

---

## Backup Registry

| Date | Action | Storage Device | Format |
|---|---|---|---|
| Jun 25, 2026 | Full backup of windows-10-dfir-target/ including all snapshot VDIs | External HD | NTFS |
| Jun 25, 2026 | Full backup of windows-10-dfir-target/ including all snapshot VDIs | USB drive | exFAT |

---

## Notes

This chain of custody document reflects a real-world investigative scenario where an initial acquisition error was identified, diagnosed, and corrected through rigorous analysis. The full troubleshooting process is preserved as evidence of investigative methodology.

In a real investigation, this document would additionally require:
- Physical signatures from all persons who handled the evidence
- Secure storage log (location, access dates, access persons)
- Court-admissible write blocker documentation
- Notarized or witnessed acquisition certification
- Formal amendment documentation for the acquisition revision
