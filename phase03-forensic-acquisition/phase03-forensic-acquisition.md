# Phase 03 — Forensic Acquisition

## Objective

Create a forensically sound image of the Windows target disk, verify its integrity through cryptographic hashing, and document the full chain of custody. The resulting E01 image will be used as the primary evidence source for all subsequent analysis phases.

---

## Environment

- **Investigator machine:** kali-linux-2026.1-virtualbox-amd64
- **Target machine:** windows-10-dfir-target (powered off during acquisition)
- **Acquisition tool:** ewfacquire 20140816
- **Verification tool:** ewfverify 20140816
- **Conversion tool:** qemu-img (QEMU disk image utility)
- **Source file:** `windows-10-dfir-target.vdi` (VirtualBox dynamic disk)
- **Output format:** EnCase 6 (.E01)

---

## Pre-Acquisition Checklist

- [x] Windows target VM powered off — disk is static, no writes occurring
- [x] VDI file accessible via VirtualBox shared folder at `/media/sf_windows-10-dfir-target/`
- [x] `ewfacquire` and `qemu-img` confirmed installed on Kali
- [x] Output directory created: `~/forensics/phase03/`

---

## Procedure

### Step 1 — Convert VDI to raw image

The VirtualBox `.vdi` format is not directly supported by `ewfacquire`. The disk was first converted to a raw image using `qemu-img`:

```bash
mkdir -p ~/forensics/phase03
qemu-img convert -f vdi -O raw \
  /media/sf_windows-10-dfir-target/windows-10-dfir-target.vdi \
  ~/forensics/phase03/windows-target.raw
```

The resulting raw image is a sparse file — logical size 60GB, physical size ~8.7GB. The difference represents unallocated disk space (zero blocks), which the filesystem stores implicitly without writing physical zeros.

### Step 2 — Calculate SHA-256 hash of raw image

Before any further processing, a SHA-256 hash was calculated over the raw image to document its state prior to E01 conversion:

```bash
sha256sum ~/forensics/phase03/windows-target.raw
```

**SHA-256 (raw):**
```
80f206ef92486901742c5c4f5a9a536d7e69b978a5ed64ba1dc0075c8f8fb08c
```

This hash serves as an independent pre-acquisition integrity reference, separate from the MD5 generated internally by `ewfacquire`.

### Step 3 — Acquire E01 image

```bash
ewfacquire \
  -t ~/forensics/phase03/windows-target \
  -f encase6 \
  -C "BitTorrent DFIR Investigation" \
  -D "Windows 10 DFIR Target VM" \
  -e "Paulo Vaz" \
  -m fixed \
  -M logical \
  ~/forensics/phase03/windows-target.raw
```

**Acquisition parameters:**

| Parameter | Value |
|---|---|
| Case number | BitTorrent DFIR Investigation |
| Description | Windows 10 DFIR Target VM |
| Evidence number | 01 |
| Examiner name | Paulo Vaz |
| Notes | Windows 10 LTSC 21H2 VM - qBittorrent BitTorrent activity |
| Media type | Fixed disk |
| EWF format | EnCase 6 (.E01) |
| Compression | deflate / fast |
| Segment size | 1.4 GiB |
| Bytes per sector | 512 |
| Block size | 64 sectors |
| Retries on read error | 2 |

**Acquisition result:**

```
Acquiry completed at: Jun 22, 2026 17:52:39
Written: 60 GiB (64424509628 bytes) in 3 minute(s) and 22 second(s)
Speed: 304 MiB/s (318933215 bytes/second)
MD5 hash calculated over data: 41f3b630078af93662a45f37d3d6ee9b
ewfacquire: SUCCESS
```

The image was automatically segmented into four files due to the 1.4GiB segment size limit:

| Segment | Size |
|---|---|
| windows-target.E01 | 1.5 GiB |
| windows-target.E02 | 1.5 GiB |
| windows-target.E03 | 1.5 GiB |
| windows-target.E04 | 535 MiB |

### Step 4 — Verify E01 integrity

```bash
ewfverify ~/forensics/phase03/windows-target.E01
```

**Verification result:**

```
Verify completed at: Jun 22, 2026 18:00:58
Read: 60 GiB (64424509440 bytes) in 1 minute(s) and 59 second(s)
Speed: 516 MiB/s (541382432 bytes/second)
MD5 hash stored in file:        41f3b630078af93662a45f37d3d6ee9b
MD5 hash calculated over data:  41f3b630078af93662a45f37d3d6ee9b
ewfverify: SUCCESS
```

MD5 hashes match — image integrity confirmed.

---

## Hash Summary

| Artifact | Algorithm | Hash |
|---|---|---|
| windows-target.raw | SHA-256 | `80f206ef92486901742c5c4f5a9a536d7e69b978a5ed64ba1dc0075c8f8fb08c` |
| windows-target.E01 (stored) | MD5 | `41f3b630078af93662a45f37d3d6ee9b` |
| windows-target.E01 (verified) | MD5 | `41f3b630078af93662a45f37d3d6ee9b` |

---

## Screenshots

### SHA-256 hash of raw image

![SHA-256 raw image](screenshots/01-sha256-raw-image.png)

### ewfacquire — parameters and start

![ewfacquire parameters](screenshots/02-ewfacquire-parameters.png)

### ewfacquire — completion and MD5

![ewfacquire success](screenshots/03-ewfacquire-progress.png)

### ewfverify — verification in progress

![ewfverify progress](screenshots/04-ewfverify-progress.png)

### ewfverify — SUCCESS and MD5 match

![ewfverify success](screenshots/05-ewfverify-success.png)

---

## Forensic Rationale

**Why power off the VM before acquisition?**
An active Windows system writes constantly — swap files, prefetch, registry, log files. Acquiring a live disk introduces write contamination that alters the evidence state. Powering off freezes the disk at the moment the suspect activity occurred.

**Why convert VDI to raw first?**
`ewfacquire` does not natively support VDI format. The raw conversion preserves all data bit-for-bit and provides a format that any forensic tool can read. The SHA-256 hash of the raw image documents the pre-conversion state.

**Why two hash algorithms (SHA-256 and MD5)?**
The SHA-256 was calculated independently on the raw source before acquisition. The MD5 is generated internally by `ewfacquire` during the E01 creation and verified by `ewfverify`. Using two independent hashes from two different tools over two different stages creates multiple verification layers — if any single hash is contested, the other still holds.

**Why does the E01 come in four segments?**
The 1.4GiB segment size is standard EnCase behavior and ensures compatibility with filesystems that have file size limits (e.g. FAT32 at 4GB). The segments are logically one image — `ewfverify` treats them as a single unit.

**Why not use a hardware write blocker?**
In a real investigation, a hardware write blocker would be mandatory — it physically prevents any write commands from reaching the evidence disk during acquisition. In this lab environment, powering off the VM and accessing the VDI file read-only via shared folder serves the same purpose: the target disk received no writes during acquisition.

---

## Evidence Files

Large forensic image files exceed GitHub's 100MB limit and are excluded from this repository via `.gitignore`. Original hashes are documented above and in `../../evidences/hashes.md` for reproducibility.

```
~/forensics/phase03/windows-target.E01  (excluded — 1.5 GiB)
~/forensics/phase03/windows-target.E02  (excluded — 1.5 GiB)
~/forensics/phase03/windows-target.E03  (excluded — 1.5 GiB)
~/forensics/phase03/windows-target.E04  (excluded — 535 MiB)
~/forensics/phase03/windows-target.raw  (excluded — 60 GiB)
```

---

## Chain of Custody

Full chain of custody documentation for this investigation is maintained at the repository root:

→ [Chain of Custody](../chain-of-custody.md)

---

## Next Phase

With a verified forensic image in hand, Phase 04 begins: mounting the E01 in Autopsy, running ingest modules, and locating BitTorrent artifacts on the Windows disk — torrent files, qBittorrent history, Registry entries, and NTFS timestamps.

→ [Phase 04 — Disk Analysis](../phase04-disk-analysis/README.md)
