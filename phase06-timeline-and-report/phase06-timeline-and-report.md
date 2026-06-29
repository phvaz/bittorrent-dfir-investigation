# Phase 06 — Forensic Timeline and Report

This phase consolidates the findings from Phase 04 (disk analysis) and Phase 05 (network analysis) into a unified forensic timeline and a formal investigation report.

The formal report is available as a PDF: [`forensic-report-bittorrent-dfir.pdf`](../forensic-report-bittorrent-dfir.pdf). This document is the Markdown equivalent for repository navigation.
---

## Case Information

| Field | Value |
|---|---|
| Case Reference | BTD-2026-001 |
| Examiner | Paulo Vaz |
| Report Date | June 28, 2026 |
| Report Version | 1.0 — Final |
| Classification | Academic Portfolio |
| Evidence Image (MD5) | `8e01029edeadbfffb36fe8516afb54df` |
| Standards Referenced | ISO/IEC 27037, 27041, 27042, 27043; NIST SP 800-86 |

---

## 1. Executive Summary

This report documents the findings of a digital forensic investigation conducted on a Windows 10 Enterprise LTSC virtual machine identified as `windows-10-dfir-target`. The investigation was carried out in a controlled laboratory environment as part of an academic portfolio project.

The examination sought to determine whether BitTorrent software was installed and used on the target system, and to reconstruct the complete timeline of activity — from software acquisition to torrent download — through independent analysis of disk artifacts and network traffic. The investigation spanned six phases: environment setup, evidence generation, forensic acquisition, disk analysis, network analysis, and consolidated reporting. Two independent forensic tools were employed — Autopsy 4.23.1 and The Sleuth Kit 4.14.0 — with cross-validation between tools used as a methodological control.

**Principal findings:**

- **qBittorrent 5.2.2 installation confirmed** — Downloaded from sourceforge.net on 2026-06-19 at 10:29 BRT and first executed at 10:33:04 BRT, per Prefetch artifacts and browser download history.
- **Torrent download confirmed** — The file `debian-13.5.0-amd64-netinst.iso` (755 MiB) was downloaded via BitTorrent on 2026-06-22 between 12:29:33 and 12:30:10 BRT — 37 seconds — confirmed independently by application logs, NTFS timestamps, browser history, and network traffic.
- **Network evidence corroborates disk evidence** — The SHA-1 info-hash `58846860f0a766f8a42b0bb214d8c713fdf1b167`, recovered from the .torrent filename on disk, was independently identified in BitTorrent handshake payloads captured in the network trace during the exact download window.
- **No evidence of tampering** — NTFS timestamp analysis ($STANDARD_INFORMATION = $FILE_NAME) confirmed no timestomping on the .torrent file.
- **64 peer connections established** — The target established 64 TCP connections to external peers during the download window, consistent with normal BitTorrent swarm behaviour.

---

## 2. Objective

The investigation was conducted with the following objectives:

1. Confirm whether qBittorrent was installed on the target system and establish the installation timeline.
2. Identify and recover all disk artifacts related to BitTorrent activity: configuration files, application logs, browser history, filesystem metadata, and registry entries.
3. Analyze network traffic to independently corroborate disk findings through protocol-level evidence.
4. Reconstruct a unified, chronologically consistent timeline spanning disk and network evidence.
5. Assess artifact integrity, including anti-tampering verification of NTFS timestamps.
6. Produce a formal forensic report documenting all findings, methodology, limitations, and procedural deviations.

---

## 3. Methodology

The investigation followed a structured six-phase methodology aligned with ISO/IEC 27037 (evidence identification and collection), 27041 (assurance), 27042 (analysis and interpretation), 27043 (incident investigation), and NIST SP 800-86 (integrating forensic techniques into incident response).

| Phase | Description |
|---|---|
| Phase 01 — Environment Setup | Two-VM laboratory in VirtualBox 7.2.8: a Windows 10 Enterprise LTSC target and a Kali Linux 2026.1 investigator VM, connected via host-only network. |
| Phase 02 — Evidence Generation | Installation of qBittorrent 5.2.2 and Wireshark 4.6.6. Download of `debian-13.5.0-amd64-netinst.iso` via BitTorrent with simultaneous Wireshark capture (585,916 packets). |
| Phase 03 — Forensic Acquisition | VDI converted to raw via qemu-img, then E01 acquisition via ewfacquire. The initial acquisition was invalidated due to VirtualBox differential snapshot architecture (base VDI frozen at a pre-evidence state). Remediation and the final image (MD5 `8e01029edeadbfffb36fe8516afb54df`, 9 segments) are documented in the chain of custody. |
| Phase 04 — Disk Analysis | Analysis with Autopsy 4.23.1 and TSK 4.14.0. Autopsy ingest was interrupted twice; TSK CLI served as an independent validation method for Registry and .torrent metadata. Dual-tool validation strengthens the evidentiary value of the findings. |
| Phase 05 — Network Analysis | Analysis of the pcap with Wireshark 4.6.6, performed on the Windows target VM (Wireshark was not configured on Kali). This procedural deviation is documented with a full risk assessment in the chain of custody. |
| Phase 06 — Consolidated Report | Synthesis of Phase 04 and Phase 05 findings into a unified timeline and this report. |

---

## 4. Case Description

The target system is a Windows 10 Enterprise LTSC 21H2 virtual machine (build 10.0.19044.1288) with 8 GB RAM, 4 virtual CPUs, and a 60 GB dynamic VDI disk. The OS user account is named **Paulo** (SID S-1-5-21-2003726914-3947868171-508885183-1001). The system had two network adapters: NAT (10.0.2.15) and host-only (192.168.56.101).

The scenario involves installation and use of qBittorrent 5.2.2 to download the Debian GNU/Linux 13.5.0 network installer ISO (755 MiB) from the public BitTorrent swarm. Activity occurred across two sessions: a software installation session on June 19, 2026, and a download session on June 22, 2026.

---

## 5. Evidence and Chain of Custody

### 5.1 Primary Forensic Image

| Field | Value |
|---|---|
| Description | Windows 10 Enterprise LTSC 21H2 virtual machine disk |
| Source | `windows-10-dfir-target.vdi` (VirtualBox dynamic disk) |
| Image format | Expert Witness Format (E01), 9 segments (E01–E09) |
| Compressed size | ~13.2 GB |
| Logical size | 60 GiB (64,424,509,440 bytes) |
| Acquisition tool | ewfacquire 20140816 |
| Acquisition date | June 25, 2026 |
| MD5 (stored / verified) | `8e01029edeadbfffb36fe8516afb54df` (match) |
| Raw SHA-256 | `dff2ba411365bbf36cf15306d036f2d24a0fff9bdc895e677ab6f8abab3b4e4e` |
| Partition offset (TSK) | 104448 (vol3, NTFS) |
| Status | Active — authoritative image for all analysis |

### 5.2 Network Capture

| Field | Value |
|---|---|
| File | `qbitorrent-debian.pcapng` |
| Capture tool | Wireshark 4.6.6 |
| Total packets | 585,916 |
| Capture window | 2026-06-22 12:27 – 12:32 BRT |
| Download window | 2026-06-22 12:29:33 – 12:30:10 BRT (37 seconds) |
| Status | Secondary evidence — corroborates disk findings |

### 5.3 Chain of Custody Notes

An initial acquisition (MD5 `41f3b630078af93662a45f37d3d6ee9b`) was **invalidated** after Phase 04 analysis revealed it captured the VDI base disk frozen at the clean-install snapshot, prior to any evidence generation. Root cause: VirtualBox differential snapshot architecture — all evidence resided in snapshot differential VDIs, not the base VDI. Before remediation, the complete VM directory was backed up to two independent devices (external HD, NTFS; USB drive, exFAT). The original snapshot chain is preserved and verifiable.

> **Procedural deviation:** Network capture analysis (Phase 05) was performed on the evidence-generating machine rather than the investigator workstation, because Wireshark was not configured on Kali Linux. This deviation is explicitly acknowledged and would not be acceptable in a production forensic or legal context. A full risk assessment is recorded in the chain of custody.

---

## 6. Forensic Timeline

The following unified timeline was reconstructed from four independent evidence sources: browser download history, Windows Prefetch, the qBittorrent application log, NTFS timestamps, and network traffic analysis.

| Timestamp (BRT) | Event | Source |
|---|---|---|
| 2026-06-19 10:27 | Wireshark 4.6.6 downloaded from wireshark.org | Web Downloads |
| 2026-06-19 10:29 | qBittorrent 5.2.2 downloaded from sourceforge.net | Web Downloads |
| 2026-06-19 10:33:04 | qBittorrent installed and first launched (PID 3716, port 29823) | Prefetch, log |
| 2026-06-19 10:33:05–17 | DHT/PeX/encryption enabled; external IP 187.106.172.158 | Application log |
| 2026-06-19 10:33:18 | Session 1 ended — no torrent activity | Application log |
| 2026-06-22 12:27 | Wireshark capture started | pcap |
| 2026-06-22 12:28:15 | qBittorrent launched (PID 8060, port 29823) | Application log |
| 2026-06-22 12:29:33 | Torrent added: debian-13.5.0-amd64-netinst.iso | Log, NTFS $SI, Web Downloads |
| 2026-06-22 12:29:33 | .torrent created in BT_backup (inode 156645); $SI=$FN | TSK istat |
| 2026-06-22 12:29:33 | HTTP GET /announce to 130.239.18.158:6969 (event=started) | pcap frame 14610 |
| 2026-06-22 12:29:33 | Tracker 200 OK — 100 peers returned | pcap frame 14680 |
| 2026-06-22 12:29:33 | DHT Get_peers queries begin for target info-hash | pcap frames 14474+ |
| 2026-06-22 12:29:35 | Web seed error: cdimage.debian.org 404 (web seed, not tracker) | Application log |
| 2026-06-22 12:29:35+ | BitTorrent handshakes with peers — info-hash in payload | pcap frame 16107+ |
| 2026-06-22 12:30:10 | Download complete: 755 MiB in 37 seconds | Application log |
| 2026-06-22 12:32 | Wireshark capture stopped | pcap |

---

## 7. Technical Analysis — Disk (Phase 04)

Full detail in `phase04-disk-analysis/findings.md`. Summary of confirmed artifacts:

| Artifact | Key Value | Source |
|---|---|---|
| .torrent file | Inode 156645, `BT_backup/58846860...torrent` | TSK fls |
| Torrent metadata | debian-13.5.0-amd64-netinst.iso, 755 MiB, info-hash SHA-1 | TSK icat |
| Prefetch | QBITTORRENT.EXE first run 2026-06-19 10:33:04, count 3 | Autopsy |
| Web Downloads | Wireshark, qBittorrent, Debian ISO + Zone.Identifier (MotW) | Autopsy |
| Recent Documents | .iso.lnk and .iso.torrent.lnk (manual user interaction) | Autopsy |
| qBittorrent.ini | Port 29823, SavePath Downloads, MigrationVersion 8 | Autopsy |
| qbittorrent.log | Session 1 (06-19), Session 2 (06-22): added 12:29:33, complete 12:30:10 | Autopsy |
| Registry UserAssist | qbittorrent.exe from Program Files; .torrent file association | TSK icat |
| NTFS timestamps | $SI = $FN = 12:29:33 BRT — no timestomping | TSK istat |

**Anti-tampering verification:** NTFS maintains $STANDARD_INFORMATION ($SI, modifiable via Windows APIs) and $FILE_NAME ($FN, written only by the NTFS kernel driver). Timestomping modifies $SI while leaving $FN unchanged. On the .torrent file, $SI = $FN across all timestamp fields (2026-06-22 12:29:33 BRT) — no timestomping detected. The timestamp is corroborated by three independent sources: application log, NTFS $SI, and browser download history.

---

## 8. Technical Analysis — Network (Phase 05)

Full detail in `phase05-network-analysis/findings.md`. Summary:

| Finding | Evidence | Frame |
|---|---|---|
| DHT queries for target info-hash (download window) | 20,628 BT-DHT packets, all `58846860...` | 14474+ |
| Tracker announce (info-hash, port 29823, left=755MiB) | HTTP GET /announce to 130.239.18.158:6969 | 14610 |
| Client identified as qBittorrent 5.2.2 | peer_id `-qB5220-` | 14610 |
| Tracker returned 100 peers (200 OK, peers600) | HTTP response, Server: mimosa | 14680 |
| Info-hash in BitTorrent handshake payload | Hex offsets 0x50–0x60 | 16107 |
| 64 TCP peer connections | TCP Conversations | peers.csv |
| Download burst correlated with 37s window | I/O graph spike at ~120–150s | — |

**Port note:** Filtering for `tcp.port == 29823` returned zero results within the download window. Port 29823 is the inbound listening port; outbound connections used ephemeral ports (>49152) assigned by the OS.

**Tracker vs. web seed:** The application log recorded a 404 from cdimage.debian.org at 12:29:35. This originated from an HTTP web seed URL declared in the torrent metadata, not from the tracker. The tracker at 130.239.18.158:6969 returned 200 OK. Tracker and web seed are distinct mechanisms.

---

## 9. Disk-to-Network Correlation

Four independent data points converge between disk artifacts (Phase 04) and network evidence (Phase 05):

1. **Info-hash identity** — Disk: .torrent filename = `58846860f0a766f8a42b0bb214d8c713fdf1b167.torrent` (inode 156645). Network: same value in the BitTorrent handshake payload (frame 16107). The info-hash is a SHA-1 digest; collision is computationally infeasible.
2. **Temporal window** — Disk: log records torrent added 12:29:33, complete 12:30:10; NTFS $SI = 12:29:33. Network: DHT queries for the target info-hash begin at 12:29:33; the I/O burst aligns with the 37-second window.
3. **Port declaration** — Disk: qBittorrent.ini Session\Port = 29823; log "Listening on TCP/29823, UDP/29823". Network: tracker announce parameter port=29823.
4. **File size** — Disk: .torrent bencoding length = 791,674,880 bytes. Network: tracker announce parameter left=791,674,880.

| Evidence Point | Disk Source (Ph.04) | Network Source (Ph.05) |
|---|---|---|
| Torrent identity | .torrent filename = 58846860… | Info-hash in handshake (16107) |
| Download start | log: 12:29:33 | DHT + announce event=started |
| Download end | log: 12:30:10 | I/O burst ends; piece exchange ceases |
| P2P port | qBittorrent.ini: 29823 | announce: port=29823 |
| File size | bencoding: 791674880 | announce: left=791674880 |
| Client identity | Prefetch / log PID 3716/8060 | peer_id -qB5220- (5.2.2) |
| Peer discovery | log: DHT ENABLED | 20,628 DHT packets, target hash |
| Tracker contact | log: 404 web seed (not tracker) | HTTP 200 OK from tracker |

---

## 10. Conclusions

Based on the totality of evidence across disk analysis (Phase 04) and network analysis (Phase 05), the following conclusions are established with high confidence:

1. **qBittorrent 5.2.2 was installed on the target system.** Confirmed by browser download history, Prefetch (first run 10:33:04, count 3), installer Prefetch, application log (PID 3716), and Registry UserAssist.
2. **debian-13.5.0-amd64-netinst.iso was downloaded via BitTorrent.** Confirmed by the application log (added 12:29:33, complete 12:30:10), NTFS timestamps ($SI=$FN = 12:29:33 BRT), browser history, and .torrent bencoding.
3. **The user manually initiated the download.** Confirmed by Recent Documents .lnk files for both the .iso and .torrent files, proving direct interaction through the Windows shell.
4. **Network traffic corroborates disk evidence independently.** The info-hash `58846860f0a766f8a42b0bb214d8c713fdf1b167` was recovered from disk and independently identified in BitTorrent handshake payloads. Four convergent data points link disk and network evidence.
5. **No artifact tampering was detected.** NTFS analysis confirmed $SI = $FN on the .torrent file, ruling out timestomping. The timeline is internally consistent across all sources.
6. **The download completed successfully in 37 seconds.** Despite a web seed 404 error, the client completed the 755 MiB download via 64 peer connections, consistent with DHT-based peer discovery and normal swarm behaviour.

---

## 11. Limitations

- **No hardware write blocker.** The investigation used a virtualized environment without a physical write blocker. Integrity was maintained through hash verification and careful handling, but physical write blocking was not feasible.
- **Initial acquisition invalidated.** The first acquisition was invalidated due to VirtualBox snapshot architecture. Consolidation modified the base VDI; the original snapshot chain is preserved on two backup devices.
- **Autopsy ingest incomplete.** Ingest was interrupted twice, preventing the Registry module from completing. Registry artifacts were extracted via TSK as an alternative.
- **Network analysis on evidence machine.** Phase 05 was performed on the evidence-generating machine. Findings are corroborated by disk evidence through four convergence points, mitigating this deviation.
- **Plaso timeline incomplete.** Autopsy's Plaso timeline module errored. The timeline was reconstructed manually from individual artifact timestamps.
- **22.2% packet drop.** Wireshark reported ~22.2% packet loss during capture. Sufficient evidence was recovered for all objectives, but the capture is not a complete record.
- **Controlled lab environment.** Evidence was generated by the examiner in a single-investigator lab. Findings demonstrate methodological competence, not analysis of independently sourced evidence.

---

## 12. Recommendations

1. **Use hardware write blockers.** Connect all evidence media through a certified hardware write blocker before acquisition.
2. **Image snapshot differentials directly.** When acquiring from VirtualBox VMs, point qemu-img at the most recent snapshot differential VDI rather than the base VDI.
3. **Configure the investigator workstation before analysis.** Install and configure all analysis tools, including Wireshark, on the investigator VM before evidence collection.
4. **Hash all transfers between systems.** Accompany any evidence transfer with SHA-256 verification before and after.
5. **Configure hash sets in Autopsy.** Configuring NSRL known-good and notable hash sets would enable automatic identification of known files.

---

## 13. Examiner Declaration

I, **Paulo Vaz**, hereby declare that:

1. This report accurately reflects the findings of the digital forensic examination conducted on the evidence described herein.
2. The examination was performed using recognized forensic tools and methodologies aligned with ISO/IEC 27037, 27041, 27042, 27043 and NIST SP 800-86.
3. All procedural deviations from standard forensic practice have been documented, justified, and disclosed within this report and the chain of custody.
4. The findings, conclusions, and opinions expressed are my own and reflect an honest assessment of the evidence examined.
5. This report was prepared for academic portfolio purposes and does not constitute a legal instrument in any jurisdiction.

---

*Generated June 28, 2026. Evidence image MD5: `8e01029edeadbfffb36fe8516afb54df`. Case reference: BTD-2026-001. The formal version of this report is available as [`forensic-report-bittorrent-dfir.pdf`](../forensic-report-bittorrent-dfir.pdf).*