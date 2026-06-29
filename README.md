# BitTorrent DFIR Investigation

End-to-end digital forensic investigation of BitTorrent activity on a Windows 10 Enterprise system. The investigation reconstructs the complete activity timeline — from software installation to torrent download — through independent analysis of disk artifacts and network traffic, producing a formal forensic report as the final deliverable.

---

## Overview

| Field | Value |
|---|---|
| Case reference | BTD-2026-001 |
| Target system | Windows 10 Enterprise LTSC 21H2 (10.0.19044.1288) |
| Evidence image | `windows-target.E01` — MD5 `8e01029edeadbfffb36fe8516afb54df` |
| Forensic tools | Autopsy 4.23.1 · The Sleuth Kit 4.14.0 · Wireshark 4.6.6 |
| Standards | ISO/IEC 27037, 27041, 27042, 27043 · NIST SP 800-86 |
| Report | [`forensic-report-bittorrent-dfir.pdf`](./forensic-report-bittorrent-dfir.pdf) |

---

## What this investigation demonstrates

**Forensic acquisition under non-standard conditions.** The initial acquisition was invalidated after disk analysis revealed the evidence resided in VirtualBox snapshot differentials, not the base VDI. The root cause was diagnosed, the correct acquisition performed, and the full troubleshooting process documented in the chain of custody — including what should have been done differently and why.

**Multi-tool disk analysis with independent validation.** Autopsy ingest was interrupted twice due to application failures. Rather than treating this as a setback, TSK CLI was used as an independent validation method to confirm all findings — specifically for Registry (NTUSER.DAT / UserAssist) and .torrent metadata extraction. Two tools reaching the same conclusions independently strengthens the evidentiary value of the findings.

**Disk-to-network correlation.** The SHA-1 info-hash `58846860f0a766f8a42b0bb214d8c713fdf1b167`, recovered from the `.torrent` filename on disk, was independently identified in BitTorrent TCP handshake payloads in the network capture. Four convergent data points — info-hash identity, temporal window, port declaration, and file size — link disk and network evidence without relying on individual peer IP logging.

**Honest documentation of procedural deviations.** Two deviations from standard forensic practice occurred: snapshot consolidation modified the original evidence artifact, and network analysis was performed on the evidence-generating machine. Both are documented with root cause analysis, risk assessment, and what correct procedure would have looked like. In forensic work, hiding problems is worse than having them.

---

## Key findings

- qBittorrent 5.2.2 was installed on the target system on **2026-06-19 at 10:33:04 BRT**, confirmed by Prefetch, browser download history, application log, and Registry UserAssist.
- `debian-13.5.0-amd64-netinst.iso` (755 MiB) was downloaded via BitTorrent on **2026-06-22 between 12:29:33 and 12:30:10 BRT** — 37 seconds — confirmed independently by the application log, NTFS timestamps, browser history, and network traffic.
- NTFS timestamp analysis (`$STANDARD_INFORMATION = $FILE_NAME`) confirmed no timestomping on the `.torrent` file.
- The BitTorrent client established **64 TCP peer connections** during the 37-second download window.
- Network traffic analysis confirmed the tracker announce (`130.239.18.158:6969`) returned `200 OK` with 100 peers. The `404` error logged by qBittorrent originated from an HTTP web seed, not the tracker — a distinction documented in detail.

---

## Investigation structure

```
bittorrent-dfir-investigation/
├── chain-of-custody.md               ← acquisition history, deviations, hash registry
├── forensic-report-bittorrent-dfir.pdf  ← formal report (final deliverable)
├── phase01-environment-setup/
│   ├── README.md
│   └── screenshots/
├── phase02-evidence-generation/
│   ├── README.md
│   └── screenshots/
├── phase03-forensic-acquisition/
│   ├── README.md                     ← two acquisition attempts documented
│   └── screenshots/
├── phase04-disk-analysis/
│   ├── README.md
│   ├── findings.md
│   └── screenshots/
├── phase05-network-analysis/
│   ├── README.md
│   ├── findings.md
│   └── peers.csv                     ← 64 TCP connections exported from Wireshark
└── phase06-timeline-and-report/
    └── README.md                     ← unified timeline + report summary
```

---

## Artifact summary

### Disk artifacts (Phase 04)

| Artifact | Key value | Source |
|---|---|---|
| `.torrent` file | Inode 156645 · `BT_backup/58846860...torrent` | TSK `fls` |
| Torrent metadata | `debian-13.5.0-amd64-netinst.iso` · 755 MiB · info-hash SHA-1 | TSK `icat` |
| Prefetch | `QBITTORRENT.EXE` first run 2026-06-19 10:33:04 · count 3 | Autopsy |
| Web Downloads | Wireshark · qBittorrent · Debian ISO · Zone.Identifier (MotW) | Autopsy |
| Recent Documents | `.iso.lnk` and `.iso.torrent.lnk` — manual user interaction | Autopsy |
| qBittorrent.ini | Port 29823 · SavePath Downloads · MigrationVersion 8 | Autopsy |
| qbittorrent.log | Session 1 (06-19) · Session 2: added 12:29:33, complete 12:30:10 | Autopsy |
| Registry UserAssist | `qbittorrent.exe` from Program Files · `.torrent` file association | TSK `icat` |
| NTFS timestamps | `$SI = $FN = 12:29:33 BRT` — no timestomping | TSK `istat` |

### Network artifacts (Phase 05)

| Finding | Evidence | Frame |
|---|---|---|
| DHT queries for target info-hash | 20,628 BT-DHT packets · all `58846860...` | 14474+ |
| Tracker announce | HTTP GET `/announce` · port=29823 · left=791674880 | 14610 |
| Client identified as qBittorrent 5.2.2 | `peer_id: -qB5220-` | 14610 |
| Tracker 200 OK | Server: mimosa · 100 peers returned | 14680 |
| Info-hash in TCP payload | Hex offsets 0x50–0x60 of BitTorrent handshake | 16107 |
| 64 peer connections | See `peers.csv` | — |

---

## Unified timeline

| Timestamp (BRT) | Event | Source |
|---|---|---|
| 2026-06-19 10:27 | Wireshark 4.6.6 downloaded from wireshark.org | Web Downloads |
| 2026-06-19 10:29 | qBittorrent 5.2.2 downloaded from sourceforge.net | Web Downloads |
| 2026-06-19 10:33:04 | qBittorrent first launched — PID 3716, port 29823 | Prefetch · log |
| 2026-06-19 10:33:18 | Session 1 ended — no torrent activity | Application log |
| 2026-06-22 12:27 | Wireshark capture started | pcap |
| 2026-06-22 12:28:15 | qBittorrent launched — PID 8060, port 29823 | Application log |
| 2026-06-22 12:29:33 | Torrent added: `debian-13.5.0-amd64-netinst.iso` | Log · NTFS $SI · Web Downloads |
| 2026-06-22 12:29:33 | `.torrent` created in BT_backup — inode 156645 · $SI=$FN | TSK istat |
| 2026-06-22 12:29:33 | Tracker announce sent — event=started · left=791674880 | pcap frame 14610 |
| 2026-06-22 12:29:33 | Tracker 200 OK — 100 peers returned | pcap frame 14680 |
| 2026-06-22 12:29:33 | DHT Get_peers queries begin for target info-hash | pcap frames 14474+ |
| 2026-06-22 12:29:35 | Web seed 404 from cdimage.debian.org (web seed, not tracker) | Application log |
| 2026-06-22 12:29:35+ | BitTorrent handshakes — info-hash in TCP payload | pcap frame 16107+ |
| 2026-06-22 12:30:10 | Download complete — 755 MiB in 37 seconds | Application log |
| 2026-06-22 12:32 | Wireshark capture stopped | pcap |

---

## Examiner

Paulo Vaz · [GitHub](https://github.com/phvaz) · [LinkedIn](https://linkedin.com/in/phvaz)
