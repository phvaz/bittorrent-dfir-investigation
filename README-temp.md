# BitTorrent DFIR Investigation

> **Status: In Progress** — This README will be finalized upon project completion.

## Overview

A simulated DFIR (Digital Forensics and Incident Response) investigation of BitTorrent activity on a Windows 10 target machine. The investigation reconstructs the full lifecycle of a torrent download — from the moment the client was opened to the last peer connection — by correlating artifacts from three independent evidence sources: disk, network capture, and Windows Registry.

## Scenario

A Windows 10 machine is suspected of having used BitTorrent software to download files. As the forensic investigator, the objective is to determine what was downloaded, when, from whom, and to produce a legally defensible forensic report with a unified timeline of events.

## Objectives

- Acquire a forensically sound disk image with verified integrity (SHA-256 + MD5)
- Locate and extract BitTorrent artifacts from the Windows filesystem and Registry
- Analyze network traffic captures to identify peers, trackers, and protocol behavior
- Correlate disk artifacts + network capture + Registry entries into a single unified timeline
- Produce a complete forensic report following professional DFIR standards

## Environment

| Component | Version |
|---|---|
| Host OS | Windows (Ryzen 7 5700X, 32GB RAM) |
| VirtualBox | 7.2.8 r173730 |
| Target VM | Windows 10 Enterprise LTSC 21H2 (10.0.19044.1288) |
| Investigator VM | Kali Linux Rolling 2026.1 x64 |
| Torrent client | qBittorrent 5.2.2 x64 |
| Wireshark | 4.6.6 x64 |

## Tools

- **ewfacquire / ewfverify** — forensic disk acquisition and verification
- **Autopsy 4.23.1** — disk image analysis and artifact extraction
- **The Sleuth Kit (TSK) 4.14.0** — filesystem-level forensic analysis
- **Wireshark 4.6.6** — network traffic capture and protocol analysis
- **qemu-img** — VDI to raw disk image conversion

## Investigation Phases

| Phase | Status |
|---|---|
| [01 — Environment Setup](phase01-environment-setup/README.md) | ✅ Complete |
| [02 — Evidence Generation](phase02-evidence-generation/README.md) | ✅ Complete |
| [03 — Forensic Acquisition](phase03-forensic-acquisition/README.md) | ✅ Complete |
| [04 — Disk Analysis](phase04-disk-analysis/README.md) | 🔄 In Progress |
| [05 — Network Analysis](phase05-network-analysis/README.md) | ⏳ Pending |
| [06 — Timeline and Report](phase06-timeline-and-report/README.md) | ⏳ Pending |

## Chain of Custody

→ [chain-of-custody.md](chain-of-custody.md)

## Key Findings

> To be documented upon project completion.

## Forensic Report

> To be published upon project completion in Phase 06.
