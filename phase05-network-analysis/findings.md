# Phase 05 — Network Analysis: Findings

**Case:** BitTorrent DFIR Investigation  
**Capture file:** `qbitorrent-debian.pcapng`  
**Total packets captured:** 585,916  
**Capture tool:** Wireshark 4.6.6  
**Analysis tool:** Wireshark 4.6.6  
**Analysis performed on:** Windows 10 target VM (see Methodology Note below)  
**Analyst:** Paulo Vaz  

---

## Methodology Note — Analysis on Evidence-Generating Machine

In standard forensic practice, network captures are transferred to a dedicated investigator workstation prior to analysis. The transfer must be documented with hash verification (before and after copy) to ensure integrity. Analysis is never performed on the machine that generated the evidence, as any interaction with that system after evidence collection risks contaminating the evidentiary record and may compromise admissibility.

In this investigation, analysis was performed on the Windows 10 target VM — the same machine on which the capture was generated — for the following reasons:

- Wireshark 4.6.6 was installed exclusively on the Windows 10 VM; the Kali investigator VM had no Wireshark installation configured for this phase.
- The investigation is conducted in a controlled, single-investigator lab environment with no adversarial parties. There is no risk of evidence tampering by a third party between collection and analysis.
- Transferring 585,916 packets to Kali solely to maintain protocol would introduce operational overhead without meaningful forensic benefit in this context.
- The pcap file (`qbitorrent-debian.pcapng`) is a secondary evidence artifact — a network capture generated deliberately as part of the investigation scenario. The primary forensic image (`windows-target.E01`, MD5: `8e01029edeadbfffb36fe8516afb54df`) resides on the Kali investigator VM and was handled with full write-blocker-equivalent discipline throughout Phase 03.

**This deviation from standard procedure is explicitly acknowledged and would not be acceptable in a production forensic or legal context.** In a real investigation, the correct procedure would be: copy `qbitorrent-debian.pcapng` to the investigator workstation → hash before and after transfer → verify hashes match → perform all analysis on the copy.

This note is documented in `chain-of-custody.md` at the project root.

---

## 1. Capture Overview

| Field | Value |
|-------|-------|
| Capture file | `qbitorrent-debian.pcapng` |
| Total packets | 585,916 |
| Packets dropped during capture | ~22.2% (Wireshark reported during Phase 02) |
| Capture start | 2026-06-22 12:27 BRT |
| Capture end | 2026-06-22 12:32 BRT |
| Download window | **2026-06-22 12:29:33 – 12:30:10 BRT** (37 seconds) |
| Primary filter | `tcp.port == 29823 or udp.port == 29823` → 25,027 packets (4.3%) |
| Download window filter | `(tcp.port == 29823 or udp.port == 29823) and (frame.time >= "2026-06-22 12:29:33" and frame.time <= "2026-06-22 12:30:10")` → 20,628 packets (3.5%) |
| BitTorrent protocol filter (download window) | `bittorrent and (frame.time >= "2026-06-22 12:29:33" and frame.time <= "2026-06-22 12:30:10")` → 276,744 packets (47.2%) |

---

## 2. Port 29823 — Listening Port Behavior

The primary Wireshark filter `tcp.port == 29823 or udp.port == 29823` was applied first. UDP traffic (BT-DHT) was abundant. However, filtering for `tcp.port == 29823` alone returned **zero results** within the download window.

This is expected and not an error. Port 29823 is qBittorrent's **inbound listening port** — it accepts incoming peer connections on this port. When qBittorrent initiates outbound connections to peers, the operating system assigns **ephemeral ports** (dynamically allocated, typically above 49152) as the source port. The destination port is whatever port the remote peer is listening on.

In this investigation, the download was primarily **outbound** — the client connected to peers, rather than peers connecting to it. Therefore, the port 29823 appears in UDP/DHT traffic (where it is the source port for DHT queries) but not in the TCP peer connections (where outbound connections used ephemeral ports).

This distinction is documented because it corrects a common misunderstanding: **the absence of TCP traffic on port 29823 does not indicate the port was not in use — it indicates the download was driven by outbound connections on ephemeral ports.**

---

## 3. DHT Traffic Analysis

### 3.1 General DHT Activity

Applying the primary port filter (`tcp.port == 29823 or udp.port == 29823`) revealed extensive **BT-DHT** (BitTorrent Distributed Hash Table) traffic — 25,027 packets in total across the full capture. The DHT traffic showed multiple different info-hashes in the `Get_peers` queries, indicating the DHT node was actively participating in the broader BitTorrent DHT network beyond just the Debian torrent.

Key DHT message types observed:
- `Get_peers Info_hash=...` — client querying the DHT network for peers that have a specific torrent
- `Response Nodes=N` — DHT nodes responding with routing information (other nodes to query)

### 3.2 DHT Activity During Download Window

Narrowing the filter to the 37-second download window isolated **20,628 packets**, all carrying the same info-hash:

**`58846860f0a766f8a42b0bb214d8c713fdf1b167`**

This is the SHA-1 info-hash of the Debian torrent, identical to the filename of the `.torrent` file recovered from disk in Phase 04 (inode 156645, path `Users/Paulo/AppData/Local/qBittorrent/BT_backup/58846860f0a766f8a42b0bb214d8c713fdf1b167.torrent`).

The concentration of `Get_peers` queries carrying this single info-hash within the exact download window is direct network corroboration of the disk evidence. The DHT activity confirms the client was actively seeking peers for this specific torrent during the period established by the application log.

---

## 4. Tracker HTTP Communication

### 4.1 Announce Request

HTTP traffic was filtered within the download window:
```
http and (frame.time >= "2026-06-22 12:29:33" and frame.time <= "2026-06-22 12:30:10")
```

Six HTTP packets were recovered. Three outbound `GET /announce` requests were sent to `130.239.18.158` (port 6969 — the standard BitTorrent tracker port).

**Frame 14610 — GET announce (first request):**

```
GET /announce?info_hash=X%84h%60%f0%a7%f8%a4%2b%0b%b2%14%d8%c7%13%fd%f1%b1%67
             &peer_id=-qB5220-)z4kY3g.1ijv
             &port=29823
             &uploaded=0
             &downloaded=0
             &left=791674880
             &corrupt=0
             &key=5B769339
             &event=started
```

| Parameter | Value | Significance |
|-----------|-------|-------------|
| `info_hash` | URL-encoded `58846860f0a766f8a42b0bb214d8c713fdf1b167` | Identifies the torrent being downloaded |
| `peer_id` | `-qB5220-...` | Client identifier: `-qB` = qBittorrent, `5220` = version 5.2.20 |
| `port` | `29823` | Confirms the listening port declared to the tracker |
| `uploaded` | `0` | No data uploaded yet — download just starting |
| `downloaded` | `0` | No data downloaded yet |
| `left` | `791,674,880` | Bytes remaining = **755 MiB** — exact size of `debian-13.5.0-amd64-netinst.iso` |
| `event` | `started` | First announce — download beginning |

The `peer_id` field `-qB5220-` independently confirms the client is **qBittorrent version 5.2.2**, corroborating the Prefetch and log artifacts from Phase 04.

### 4.2 Tracker Response

**Frame 14680 — HTTP/1.1 200 OK:**

| Field | Value |
|-------|-------|
| Source | `130.239.18.158:6969` |
| Destination | `10.0.2.15` |
| Status | `200 OK` |
| Server | `mimosa` |
| Content-Type | `text/plain` |
| Content-Length | `638 bytes` |
| Peers field | `peers600` — 600 bytes of peer data |

The `peers600` field contains 600 bytes of compact peer encoding. In the compact format, each peer occupies **6 bytes** (4 bytes IPv4 address + 2 bytes port). This means the tracker returned **100 peers** in its response.

### 4.3 Clarification — Tracker vs. Web Seed (404 Error)

The `qbittorrent.log` from Phase 04 recorded a `404 Not Found` error at 12:29:35 from `cdimage.debian.org`. This has been verified against the pcap and requires precise interpretation:

- The **tracker** (`130.239.18.158:6969`) responded with `200 OK` — the tracker communication was **successful**.
- The `404` error logged by qBittorrent originated from a **web seed** — a separate HTTP URL declared in the torrent's `url-list` field pointing directly to `cdimage.debian.org`. Web seeds allow clients to download file data directly via HTTP as a fallback mechanism, independent of peer-to-peer connections.
- The web seed at `cdimage.debian.org` was temporarily unavailable (404), but this did not affect the download because peer-to-peer connections via DHT and the tracker peer list provided sufficient bandwidth.

**Tracker and web seed are distinct mechanisms.** The log error reflects web seed failure, not tracker failure.

---

## 5. BitTorrent Protocol — Handshake and Piece Exchange

### 5.1 Handshake

Filtering for `bittorrent` protocol within the download window revealed the full BitTorrent protocol exchange. The first handshake was observed at **frame 16107**.

**BitTorrent handshake structure (68 bytes total):**

```
[1 byte]   0x13 — protocol name length (19)
[19 bytes] "BitTorrent protocol"
[8 bytes]  reserved flags (extension bits)
[20 bytes] info-hash
[20 bytes] peer ID
```

In the hex dump of frame 16107, the string `BitTorrent protocol` is visible at offset 0x30, and the info-hash `58846860f0a766f8a42b0bb214d8c713fdf1b167` is present at offsets 0x50–0x60 — transmitted **in clear text**, unencrypted.

This is the definitive network proof: the same info-hash found on disk in the `.torrent` file appears verbatim in the TCP payload of a BitTorrent handshake captured during the exact download window.

### 5.2 Protocol Message Sequence

The following BitTorrent protocol messages were observed in the capture, in the expected sequence:

| Message | Direction | Meaning |
|---------|-----------|---------|
| `Handshake` | Bidirectional | Identity exchange, info-hash confirmation |
| `Extended / Have None` | Peer → Client | Peer just joined, has no pieces yet |
| `Extended / Interested` | Client → Peer | Client wants pieces the peer has |
| `Unchoke` | Peer → Client | Peer unblocks the client — data transfer authorized |
| `Allowed Fast` | Peer → Client | Peer pre-authorizes specific pieces for fast download |
| `Request` | Client → Peer | Client requests specific piece: index, offset, length |
| `Piece` | Peer → Client | Peer delivers the requested piece data |
| `Have` | Bidirectional | Peer announces it has completed a specific piece |

The sequence Handshake → Unchoke → Request → Piece → Have represents a complete, normal BitTorrent download cycle. No anomalous messages were observed.

---

## 6. TCP Peer Connections

### 6.1 Connection Statistics

Wireshark **Statistics → Conversations → TCP** revealed **64 TCP connections** established from `10.0.2.15` to external peers during the download window. All connections used ephemeral source ports (>49152) from the client side.

The full peer list with IP addresses, ports, packet counts, byte volumes, and connection durations is documented in `peers.csv` (included in this phase folder).

### 6.2 Top Peers by Volume

The following peers contributed the largest data volumes (Bytes B→A, i.e., data received by the client):

| Peer IP | Bytes Received | Notable |
|---------|---------------|---------|
| 82.124.84.162 | ~39 MB | Highest volume peer |
| 85.216.19.245 | ~38 MB | Second highest |
| 82.64.35.92 | ~39 MB | High volume |
| 152.173.104.78 | ~321 MB | Largest single contributor |
| 185.148.1.127 | ~26 MB | Active seeder |

### 6.3 Disk-to-Network Peer Correlation

A search was performed in `qbittorrent.log` (Phase 04 artifact) for the IP addresses of the most active peers identified in the pcap. **No peer IP addresses were found in the application log.**

This is expected behavior: qBittorrent logs session-level events (startup, shutdown, torrent added/completed, port binding) but does not log individual peer connection IPs. The correlation between disk and network evidence in this investigation is therefore established through:

1. **Info-hash** — identical value in the `.torrent` filename on disk and in BitTorrent handshake packets in the pcap
2. **Temporal window** — disk artifacts (qbittorrent.log, NTFS timestamps) and network activity (DHT queries, handshakes, piece exchange) converge on the same 37-second window (12:29:33–12:30:10 BRT)
3. **Port** — port 29823 declared in `qBittorrent.ini` and `qbittorrent.log` matches the port declared in the tracker announce (`port=29823`)
4. **File size** — `left=791674880` in the tracker announce matches the `length` field in the `.torrent` bencoding (791,674,880 bytes = 755 MiB)

These four independent corroboration points across disk and network sources establish the evidentiary link without requiring individual peer IP logging.

---

## 7. I/O Graph — Traffic Volume Correlation

Wireshark's I/O graph (Statistics → I/O Graphs) with the BitTorrent download window filter applied shows:

- **Baseline:** near-zero packet rate from capture start (~0s) through approximately 120 seconds
- **Download burst:** sharp rise beginning at ~120s, peaking at approximately **35,000 packets/second**, sustained between ~120–150 seconds
- **Post-download:** immediate drop back to near-zero at ~150s
- **TCP errors (red):** concentrated within the burst period — expected retransmissions and timeouts during high-speed parallel P2P transfer with 64 simultaneous peers

The burst window aligns precisely with the 37-second download period established from disk artifacts. The visual correlation between the traffic spike and the known download window provides independent graphical confirmation that the network activity is causally linked to the `debian-13.5.0-amd64-netinst.iso` download.

---

## 8. Evidence Correlation Summary — Disk vs. Network

| Evidence Point | Disk Source (Phase 04) | Network Source (Phase 05) |
|---------------|----------------------|--------------------------|
| Torrent identity | `.torrent` filename = `58846860...torrent` | Info-hash in BitTorrent handshake (frame 16107) |
| Download start | `qbittorrent.log`: 12:29:33 BRT | DHT `Get_peers` queries begin at 12:29:33; tracker announce `event=started` |
| Download end | `qbittorrent.log`: 12:30:10 BRT | I/O graph burst ends at ~150s; piece exchange ceases |
| P2P port | `qBittorrent.ini`: `Session\Port=29823` | Tracker announce: `port=29823` |
| File size | `.torrent` bencoding: `length=791674880` | Tracker announce: `left=791674880` |
| Client identity | Prefetch: `QBITTORRENT.EXE`, path `/PROGRAM FILES/QBITTORRENT` | Tracker announce `peer_id`: `-qB5220-` (qBittorrent 5.2.20) |
| Peer discovery | `qbittorrent.log`: DHT enabled | 20,628 DHT packets with target info-hash |
| Tracker contact | `qbittorrent.log`: 404 from web seed (not tracker) | HTTP GET announce → 200 OK, 100 peers returned |

---

## 9. Key Frame Reference

| Frame | Timestamp (relative) | Description |
|-------|----------------------|-------------|
| 6877 | 49.25s | First DHT `Get_peers` packet (general, multiple hashes) |
| 14474 | 116.79s | First DHT `Get_peers` with target info-hash in download window |
| 14610 | 117.30s | HTTP GET `/announce` to tracker (`event=started`, `left=791674880`) |
| 14680 | 117.53s | HTTP `200 OK` from tracker (Server: mimosa, 100 peers returned) |
| 16107 | 118.77s | First BitTorrent TCP handshake — info-hash visible in hex payload |

---

*findings.md — Phase 05 — BitTorrent DFIR Investigation*
