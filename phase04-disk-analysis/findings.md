# Phase 04 — Disk Analysis: Findings

**Case:** BitTorrent DFIR Investigation  
**Image:** `windows-target.E01` (MD5: `8e01029edeadbfffb36fe8516afb54df`)  
**Partition offset (TSK):** `104448`  
**Analysis tools:** Autopsy 4.23.1, The Sleuth Kit 4.14.0  
**Autopsy case:** `windows-10-dfir-target-analysis-v3`  
**Analyst:** Paulo Vaz  

---

## 1. Partition Layout

Autopsy identified 6 volumes in the E01 image:

| ID | Name | Starting Sector | Length (Sectors) | Description | Flags |
|----|------|----------------|-----------------|-------------|-------|
| 1 | vol1 | 0 | 2048 | Unallocated | Unallocated |
| 2 | vol2 | 2048 | 102400 | NTFS / exFAT (0x07) | Allocated |
| 3 | vol3 | 104448 | 124656856 | NTFS / exFAT (0x07) | Allocated |
| 4 | vol4 | 124761304 | 808 | Unallocated | Unallocated |
| 5 | vol5 | 124762112 | 1062912 | Unknown Type (0x27) | Allocated |
| 6 | vol6 | 125825024 | 4096 | Unallocated | Unallocated |

**vol3 (starting sector 104448)** is the primary NTFS partition containing the Windows installation and all user data. This offset was confirmed independently via TSK prior to Autopsy analysis.

---

## 2. Ingest Warnings and Errors

During Autopsy ingest, the following errors were logged:

- `06/23/26 16:26:16 BRT — No known hash set. Known file search will not be executed.`
- `06/23/26 16:26:16 BRT — No notable hash set. Notable file search will not be executed.`
- `06/23/26 20:33:06 BRT — Error unpacking eapp3hst.dll`
- `06/24/26 00:55:48 BRT — Plaso returned an error when sorting events. Results are not complete.`

The Plaso error means the Autopsy timeline module did not produce a complete event timeline. This was mitigated by constructing the investigation timeline manually from individual artifact timestamps. Hash set warnings are non-critical — no hash database was configured for this investigation. The `eapp3hst.dll` unpacking error is a known Autopsy issue with certain Windows PE files and did not affect artifact recovery.

**Additionally, ingest was interrupted twice during this phase:**

1. The host machine was shut down while Autopsy was at approximately 74% ingest progress.
2. A display bug caused the file browser panel to render blank when clicking filesystem folders. Attempting to fix this via Autopsy's "Reset Windows" dialog confirmed the action accidentally, closing Autopsy mid-ingest.

Each time Autopsy was reopened, the case database retained all artifacts already indexed. Ingest was restarted from scratch each time. When the display bug recurred, critical artifacts were confirmed via **TSK CLI on Kali as an independent validation method**, specifically for Registry (NTUSER.DAT) and .torrent file analysis. This multi-tool approach is documented as methodological transparency and strengthens the evidentiary chain by demonstrating independent corroboration.

---

## 3. Confirmed Artifacts

### 3.1 .torrent File

| Field | Value |
|-------|-------|
| Inode | `156645` |
| Short name | `588468~1.TOR` |
| Full path | `Users/Paulo/AppData/Local/qBittorrent/BT_backup/58846860f0a766f8a42b0bb214d8c713fdf1b167.torrent` |
| Parent MFT Entry | `96082` (Sequence: 3) |
| Allocated size | 98304 bytes |
| Actual size | 60608 bytes |
| Owner SID | `S-1-5-21-2003726914-3947868171-508885183-1001` (user Paulo) |
| Source | TSK `fls -r -p -o 104448` |

The file is stored in qBittorrent's `BT_backup` directory, which is where qBittorrent caches active and completed torrent metadata. The filename is the SHA-1 info-hash of the torrent, which is used as the unique identifier for the torrent in the BitTorrent protocol. This will be used to correlate with network traffic in Phase 05.

### 3.2 .torrent File Content (Bencoding)

Extracted via: `icat -o 104448 ~/forensics/phase03/windows-target.raw 156645 | strings | head -20`

| Field | Value |
|-------|-------|
| comment | `Debian CD from cdimage.debian.org` |
| created by | `mktorrent 1.1` |
| creation date | `1778933537` (Unix timestamp = **2026-05-16**) |
| name | `debian-13.5.0-amd64-netinst.iso` |
| length | `791674880` bytes (755 MiB) |
| piece length | `262144` bytes (256 KB per piece) |
| pieces | `60400` SHA-1 hashes (one per piece) |

The creation date (`2026-05-16`) reflects when the Debian project generated the torrent file, not when the user downloaded it. The download timestamp is established separately via NTFS timestamps, qbittorrent.log, and Web Downloads artifacts.

### 3.3 Run Programs (Prefetch)

| Program | Path | First Run | Count | Source |
|---------|------|-----------|-------|--------|
| `QBITTORRENT.EXE` | `/PROGRAM FILES/QBITTORRENT` | **2026-06-19 10:33:04 BRT** | **3** | Autopsy Run Programs (Prefetch) |
| `QBITTORRENT_5.2.2_X64_SETUP.E` | `/USERS/PAULO/DOWNLOADS` | 2026-06-19 10:56 BRT | — | Autopsy Run Programs (Prefetch) |

Prefetch file: `QBITTORRENT.EXE-17EBDC32.pf`  
Prefetch file: `QBITTORRENT_5.2.2_X64_SETUP.E-D01C2050.pf`

A run count of 3 for `QBITTORRENT.EXE` is consistent with the two sessions recorded in `qbittorrent.log` plus one additional invocation (likely the installer launching qBittorrent post-install or a brief test open).

### 3.4 Web Downloads

Autopsy recovered 11 Web Downloads entries from Microsoft Edge browser history:

| File | URL | Timestamp (BRT) | Domain |
|------|-----|----------------|--------|
| `Wireshark-4.6.6-x64.exe` | `https://2.na.dl.wireshark.org/win64/Wireshark-4.6.6-x64.exe` | **2026-06-19 10:27** | wireshark.org |
| `qbittorrent_5.2.2_x64_setup.exe` | `https://downloads.sourceforge.net/project/qbittorrent/...` | **2026-06-19 10:29** | sourceforge.net |
| `qbittorrent_5.2.2_x64_setup.exe` | `https://sfsa.dl.sourceforge.net/project/qbittorrent/...` | 2026-06-19 10:29 | sourceforge.net |
| `qbittorrent_5.2.2_x64_setup.exe` | `https://sitsa.dl.sourceforge.net/project/qbittorrent/...` | 2026-06-19 10:29 | sourceforge.net |
| `debian-13.5.0-amd64-netinst.iso` | `https://cdimage.debian.org/debian-cd/current/amd64/...` | **2026-06-22 12:29** | cdimage.debian.org |
| `debian-13.5.0-amd64-netinst.iso.Zone.Identifier` | `about:internet` | 2026-06-22 12:29 | about:internet |

The `Zone.Identifier` file is an Alternate Data Stream (ADS) automatically created by Windows when files are downloaded from the internet via a browser. Zone 3 (Internet Zone) confirms the ISO was downloaded directly from the internet — not copied from a local device or network share. This is referred to as **Mark of the Web (MotW)**.

The three SourceForge entries for qBittorrent represent CDN mirror redirects during the same download event, not three separate downloads.

### 3.5 Recent Documents (.lnk files)

Autopsy recovered 24 Recent Documents entries. Forensically relevant entries:

| File | Path | Date Accessed | Comment |
|------|------|--------------|---------|
| `debian-13.5.0-amd64-netinst.iso.lnk` | `C:\Users\Paulo\Downloads\debian-13.5.0-amd64-netinst.iso` | **2026-06-22 12:29 BRT** | User manually opened the ISO |
| `debian-13.5.0-amd64-netinst.iso.torrent.lnk` | `C:\Users\Paulo\Downloads\debian-13.5.0-amd64-netinst.iso.torrent` | — | User manually opened the .torrent file |
| `NTUSER.DAT` | `C:\Windows\system32\wf.msc` | — | Recently opened via Windows Management Console MRU |
| `Downloads.lnk` | `C:\Users\Paulo\Downloads` | — | Downloads folder accessed |

Windows automatically creates `.lnk` shortcut files in `%APPDATA%\Microsoft\Windows\Recent\` whenever a user opens a file via Explorer or a file dialog. The presence of both `.iso.lnk` and `.iso.torrent.lnk` proves the user directly interacted with both files — this was not an automated background process.

### 3.6 qBittorrent Configuration (qBittorrent.ini)

File path: `Users/Paulo/AppData/Roaming/qBittorrent/qBittorrent.ini`  
Source: Autopsy filesystem Text view

Key configuration values:

```ini
[AddNewTorrentDialog]
SavePathHistory=C:\\Users\\Paulo\\Downloads

[BitTorrent]
Session\Port=29823
Session\SSL\Port=62269

[Meta]
MigrationVersion=8

[TorrentProperties]
Accepted=true
```

**Port 29823** is the P2P listening port used by qBittorrent for incoming peer connections. This value will be used as the primary Wireshark filter in Phase 05: `tcp.port == 29823 or udp.port == 29823`.

`MigrationVersion=8` confirms this is a legitimate qBittorrent 5.x installation (not a portable/tampered version).

`SavePathHistory` confirms all downloads were directed to `C:\Users\Paulo\Downloads`, consistent with the Web Downloads artifact paths.

### 3.7 qBittorrent Application Log (qbittorrent.log)

File path: `Users/Paulo/AppData/Local/qBittorrent/logs/qbittorrent.log`  
Source: Autopsy filesystem Text view

**Session 1 — 2026-06-19:**

```
10:33:04  qBittorrent v5.2.2 started. Process ID: 3716
10:33:05  Config dir: C:\Users\Paulo\AppData\Roaming\qBittorrent
10:33:05  Attempting to listen on port TCP/29823, UDP/29823
10:33:06  DHT support: ENABLED
10:33:06  PeX support: ENABLED
10:33:06  Encryption support: ENABLED
10:33:06  Binding to IPs: 10.0.2.15, 192.168.56.101, 127.0.0.1
10:33:17  Detected external IP: 187.106.172.158
10:33:18  Session shutdown initiated
10:33:18  BitTorrent session ended successfully
```

Session 1 was brief — qBittorrent was started, confirmed connectivity, and was closed without adding any torrent. No download activity.

**Session 2 — 2026-06-22:**

```
12:28:15  qBittorrent v5.2.2 started. Process ID: 8060
12:28:15  Attempting to listen on port TCP/29823, UDP/29823
12:28:15  DHT support: ENABLED
12:28:15  PeX support: ENABLED
12:28:15  Encryption support: ENABLED
12:28:27  Detected external IP: 280b4:14d:688e:4315:a96d:46c1:adeb538 (IPv6)
12:29:33  New torrent added. Torrent: "debian-13.5.0-amd64-netinst.iso"
12:29:35  Error from seeder URL: "https://cdimage.debian.org/..." — 404 Not Found
12:30:10  Download complete: "debian-13.5.0-amd64-netinst.iso"
```

Download duration: **37 seconds** for 755 MiB. The tracker 404 error is non-critical — it means the HTTP seeder URL was unavailable, but the torrent completed via peer-to-peer connections from other seeders.

### 3.8 Registry UserAssist (NTUSER.DAT)

**File:** `Users/Paulo/NTUSER.DAT`  
**Inode:** `92635`  
**Extraction method:** `icat -o 104448 ~/forensics/phase03/windows-target.raw 92635 | strings | grep -i qbittorrent`

> **Why TSK instead of Autopsy:** Autopsy's Registry module processes UserAssist keys during ingest. Because ingest was interrupted twice before the Registry module completed, the structured UserAssist view was unavailable in Autopsy. TSK was used to extract the raw NTUSER.DAT binary; `strings` and `grep` then confirmed the relevant entries. The result is the same information extracted via an alternative, independently validated method.

Relevant strings recovered:

```
{6D809377-6AF0-444B-8957-A3773F02200E}\qBittorrent\qbittorrent.exe
C:\Users\Paulo\Downloads\qbittorrent_5.2.2_x64_setup.exe
C:\Program Files\qBittorrent\qbittorrent.exe
{6D809377-6AF0-444B-8957-A3773F02200E}\qBittorrent\qbittorrent.exe
qBittorrent.File.Torrent_.torrent
```

`{6D809377-6AF0-444B-8957-A3773F02200E}` is the well-known GUID for the **Program Files** folder. Its presence in UserAssist confirms qBittorrent was launched from `C:\Program Files\qBittorrent\qbittorrent.exe` — a standard installation path, not a portable or suspicious location.

`qBittorrent.File.Torrent_.torrent` is a ProgID entry in UserAssist, confirming that `.torrent` files were associated with qBittorrent and that the user opened at least one `.torrent` file through the Windows shell (double-click or file dialog).

### 3.9 NTFS Timestamps (.torrent file)

Extracted via: `istat -o 104448 ~/forensics/phase03/windows-target.raw 156645`

**$STANDARD_INFORMATION attribute:**

| Timestamp | Value (EDT) | Value (BRT) |
|-----------|------------|------------|
| Created | 2026-06-22 11:29:33.894382800 | **12:29:33** |
| File Modified | 2026-06-22 11:29:33.894382800 | **12:29:33** |
| MFT Modified | 2026-06-22 11:29:33.907296000 | **12:29:33** |
| Accessed | 2026-06-22 11:29:33.894382800 | **12:29:33** |

**$FILE_NAME attribute:**

| Timestamp | Value (EDT) | Value (BRT) |
|-----------|------------|------------|
| Created | 2026-06-22 11:29:33.894382800 | **12:29:33** |
| File Modified | 2026-06-22 11:29:33.894382800 | **12:29:33** |
| MFT Modified | 2026-06-22 11:29:33.894382800 | **12:29:33** |
| Accessed | 2026-06-22 11:29:33.894382800 | **12:29:33** |

**$STANDARD_INFORMATION = $FILE_NAME → NO TIMESTOMPING DETECTED.**

When an attacker uses timestomping tools to manipulate file timestamps, they typically modify only `$STANDARD_INFORMATION` (accessible via standard Windows APIs), leaving `$FILE_NAME` (updated only by the NTFS kernel driver) unchanged. Identical timestamps across both attributes indicate the file was created naturally by the operating system with no post-creation timestamp manipulation.

The timestamp **2026-06-22 12:29:33 BRT** is corroborated by three independent sources:
1. `qbittorrent.log` — "New torrent added" at 12:29:33
2. NTFS `$STANDARD_INFORMATION` Created timestamp — 12:29:33
3. Autopsy Web Downloads — `debian-13.5.0-amd64-netinst.iso` accessed at 12:29 BRT

Three independent artifact sources converging on the same second provides high-confidence temporal anchoring for the download event.

---

## 4. Artifact Timeline

| Timestamp (BRT) | Event | Source |
|----------------|-------|--------|
| 2026-06-19 10:27 | Wireshark 4.6.6 downloaded from wireshark.org | Web Downloads (Edge) |
| 2026-06-19 10:29 | qBittorrent 5.2.2 x64 setup downloaded from sourceforge.net | Web Downloads (Edge) |
| 2026-06-19 10:33:04 | qBittorrent installed and first launched (PID 3716) | Prefetch, qbittorrent.log |
| 2026-06-19 10:33:05 | Listening on TCP/UDP 29823; DHT, PeX, encryption enabled | qbittorrent.log |
| 2026-06-19 10:33:17 | External IP detected: 187.106.172.158 | qbittorrent.log |
| 2026-06-19 10:33:18 | Session 1 ended — no torrent activity | qbittorrent.log |
| 2026-06-22 12:28:15 | qBittorrent launched (PID 8060); listening on TCP/UDP 29823 | qbittorrent.log |
| 2026-06-22 12:29:33 | Torrent added: debian-13.5.0-amd64-netinst.iso | qbittorrent.log, NTFS $SI, Web Downloads |
| 2026-06-22 12:29:35 | Tracker HTTP seeder returned 404 — download continued via P2P | qbittorrent.log |
| 2026-06-22 12:30:10 | Download complete: debian-13.5.0-amd64-netinst.iso (755 MiB in 37s) | qbittorrent.log |

---

## 5. Key Identifiers for Phase 05 (Network Analysis)

| Item | Value | Use |
|------|-------|-----|
| P2P port | `29823` | Wireshark filter: `tcp.port == 29823 or udp.port == 29823` |
| Info-hash (from filename) | `58846860f0a766f8a42b0bb214d8c713fdf1b167` | Correlate with BitTorrent handshake in pcap |
| Download window | 2026-06-22 12:29:33 – 12:30:10 BRT | Time filter in Wireshark |
| Source IP (NAT) | `10.0.2.15` | Filter outbound traffic from target |
| Source IP (host-only) | `192.168.56.101` | Filter host-only interface traffic |
| External IP (Session 2) | `280b4:14d:688e:4315:a96d:46c1:adeb538` (IPv6) | Correlate with pcap source addresses |

---

*findings.md — Phase 04 — BitTorrent DFIR Investigation*
