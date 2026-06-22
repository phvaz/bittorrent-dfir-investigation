# Chain of Custody — BitTorrent DFIR Investigation

## Case Information

| Field | Value |
|---|---|
| Case name | BitTorrent DFIR Investigation |
| Evidence number | 01 |
| Examiner | Paulo Vaz |
| Date of acquisition | June 22, 2026 |
| Purpose | Academic portfolio — DFIR investigation simulation |

---

## Evidence Description

| Field | Value |
|---|---|
| Description | Windows 10 Enterprise LTSC 21H2 virtual machine disk |
| Source file | `windows-10-dfir-target.vdi` |
| Source location | `C:\Users\Paulo Vaz\VirtualBox VMs\windows-10-dfir-target\` |
| Source format | VirtualBox Dynamic Disk Image (VDI) |
| Logical size | 60 GiB (64,424,509,440 bytes) |
| Physical size | ~8.7 GiB (sparse — unallocated blocks not written) |
| Operating system | Windows 10 Enterprise LTSC 21H2 (10.0.19044.1288) |

---

## Acquisition Log

| Step | Timestamp | Action | Tool | Operator |
|---|---|---|---|---|
| 1 | Jun 22, 2026 ~14:57 | VDI converted to raw image | qemu-img | Paulo Vaz |
| 2 | Jun 22, 2026 ~15:10 | SHA-256 calculated on raw image | sha256sum | Paulo Vaz |
| 3 | Jun 22, 2026 17:49:17 | E01 acquisition started | ewfacquire 20140816 | Paulo Vaz |
| 4 | Jun 22, 2026 17:52:39 | E01 acquisition completed | ewfacquire 20140816 | Paulo Vaz |
| 5 | Jun 22, 2026 17:58:59 | E01 integrity verification started | ewfverify 20140816 | Paulo Vaz |
| 6 | Jun 22, 2026 18:00:58 | E01 integrity verification completed | ewfverify 20140816 | Paulo Vaz |

---

## Integrity Hashes

| Artifact | Algorithm | Hash | Status |
|---|---|---|---|
| windows-target.raw | SHA-256 | `80f206ef92486901742c5c4f5a9a536d7e69b978a5ed64ba1dc0075c8f8fb08c` | Pre-acquisition reference |
| windows-target.E01 (stored) | MD5 | `41f3b630078af93662a45f37d3d6ee9b` | Generated during acquisition |
| windows-target.E01 (verified) | MD5 | `41f3b630078af93662a45f37d3d6ee9b` | Verified post-acquisition ✅ |

---

## Acquisition Conditions

| Condition | Status |
|---|---|
| Target VM powered off during acquisition | ✅ Yes |
| Target disk write-protected (read-only shared folder) | ✅ Yes |
| Hardware write blocker used | ❌ No (lab environment — VM powered off as substitute) |
| Acquisition tool integrity verified | ✅ ewfacquire 20140816 |
| Two independent hash algorithms used | ✅ SHA-256 (raw) + MD5 (E01) |

---

## Output Files

| File | Size | Location |
|---|---|---|
| windows-target.E01 | 1.5 GiB | `~/forensics/phase03/` |
| windows-target.E02 | 1.5 GiB | `~/forensics/phase03/` |
| windows-target.E03 | 1.5 GiB | `~/forensics/phase03/` |
| windows-target.E04 | 535 MiB | `~/forensics/phase03/` |

All segments together constitute a single forensic image of the Windows target disk. Files are excluded from this repository due to GitHub's 100MB file size limit. Hashes above are sufficient for reproducibility verification.

---

## Notes

This chain of custody document was created in an academic lab environment simulating real-world DFIR procedures. In a real investigation, this document would additionally require:

- Physical signatures from all persons who handled the evidence
- Secure storage log (location, access dates, access persons)
- Court-admissible write blocker documentation
- Notarized or witnessed acquisition certification
