# Digital-forensic-
Digital forensics lab — recovering files from a disk image with a missing/corrupted partition table using TestDisk and fiwalk, including manual filesystem identification and verifying recovered files were intact.

FAT16 File Carving & Data Recovery from a Corrupted Disk Image (Digital Forensics Lab)

`TestDisk` · `fiwalk` · `The Sleuth Kit` · `Kali Linux` · `CompTIA Labs` · `File Carving`

## Overview
This lab was hands-on practice with **recovering data from a disk image that had no readable partition table**. Unlike a normal disk image where `fdisk` or `mmls` can immediately hand you a partition layout, this image (`11-carve-fat.dd`) came back with a blank/zeroed disk identifier and no partition entries at all, meaning every standard tool that depends on reading the MBR had nothing to work with. The exercise was about recognizing that situation for what it is and pivoting to manual filesystem identification and carving instead of assuming the image was unrecoverable.

## Objective
Take a raw disk image with a missing or corrupted partition table, confirm that automated tools can't parse it, manually identify and mount the underlying filesystem type, and recover the files it contains, then verify the recovered files are actually intact and usable rather than just present.

## Environment
- **Attacker box:** Kali Linux (VM, root shell)
- **Target image:** `11-carve-fat.dd` (61.97 MiB, 126,913 sectors, 512-byte sectors), delivered as `11-carve-fat.zip` alongside `11-carve-fat.html`, `COPYING-GNU.txt`, and `README.txt`
- **Lab platform:** CompTIA Learning Platform, hosted lab environment via LabClient (labondemand.com)

## Tools I Used

| Tool | What It Does | Why I Used It |
|------|--------------|----------------|
| **fdisk** | Partition table reader | First check on the image; confirmed the disklabel type was reported as `dos` but the disk identifier was blank (`0x00000000`) with no partition entries listed |
| **mount** | Filesystem mount | Attempted a direct mount to see if the kernel could identify the filesystem on its own |
| **TestDisk** | Partition recovery utility | Manually specified the partition type (FAT16) when automatic detection failed, then browsed and copied recovered files out of the image |
| **fiwalk** | Filesystem metadata walker (Sleuth Kit) | Attempted automated metadata extraction directly against the raw image to cross-check TestDisk's findings |
| **GNOME Image Viewer** | Image viewer | Opened recovered `.jpg` files to visually confirm they were intact, valid images and not corrupted fragments |
| **unzip** | Archive extraction | Unpacked the lab's provided `.zip` into a working directory before analysis |

## What I Did

### Confirming the Partition Table Was Unreadable
1. Unzipped the lab archive and ran a basic partition check:
```bash
fdisk -l 11-carve-fat.dd
```
Result: `Disk 11-carve-fat.dd: 61.97 MiB, 64979456 bytes, 126913 sectors`, `Disklabel type: dos`, `Disk identifier: 0x00000000`, with no partition entries printed underneath. A disklabel type without any partitions listed was the first sign this wasn't a normal, cleanly imaged disk.
2. Tried mounting the image directly to see if the kernel could figure it out on its own:
```bash
mkdir /mnt/temp11
mount 11-carve-fat.dd /mnt/temp11
```
Result: `mount: /mnt/temp11: wrong fs type, bad option, bad superblock on /dev/loop2, missing codepage or helper program, or other error.` This confirmed there was no valid, auto-detectable superblock at the start of the image, not just a missing partition table entry.

### Manually Identifying the Filesystem with TestDisk
1. Since automated detection had nothing to go on, I ran TestDisk against the image directly and let it analyze the disk.
2. TestDisk's own partition type search came back as `P Unknown`, meaning it also couldn't automatically classify the filesystem. Rather than give up there, I used TestDisk's manual partition type selection screen and stepped through the list (BeFS, btrfs, CramFS, exFAT, ext2/3/4, FAT12/16/32, FreeBSD, f2fs, GFS2, HFS, and many more) to specify **FAT16** by hand, based on prior knowledge that these lab images are typically small FAT volumes.
3. With the type manually set to FAT16 and `Proceed` selected, TestDisk was able to walk the filesystem structure and list its contents, which confirmed the manual identification was correct.

### Recovering Files
1. Browsed into the recovered FAT16 structure inside TestDisk and selected files to copy out of the image.
2. TestDisk reported a successful copy operation (`Copy done! 1 ok, 0 failed`) and the directory listing showed a healthy mix of recovered file types dated `9-Mar-2005`: `2003_document.doc`, `_OMOPERS.WMV`, `enterprise.wav`, `haxor2.jpg`, `holly.xls`, `lin_1.2.pdf`, `nlin_14.pdf`, `paul.jpg`, `pumpkin.jpg`, `shark.jpg`, `sm1.gif`, `surf.mov`, `surf.wmv`, `_EST.PPT`, and `wword60t.zip`, a realistic mix of documents, spreadsheets, video, audio, and images, exactly what you'd expect to carve off a real user's drive.

### Cross-Checking with fiwalk
1. To compare TestDisk's manual approach against a fully automated tool, I ran `fiwalk` directly against the raw image:
```bash
fiwalk 11-carve-fat.dd
```
2. Instead of walking the filesystem metadata cleanly, fiwalk returned a long, repeating stream of errors, one per sector, all reading `TSK_Error 'Possible encryption detected (High entropy (7.92–7.93))'`, starting at sector 0 and continuing sector by sector through the image.
3. This was a useful negative result rather than a dead end: it confirmed that fiwalk, which relies on being able to locate valid filesystem structures to walk, had nothing to anchor on without the partition information TestDisk had to supply manually. The high-entropy warning is fiwalk's way of flagging data it can't otherwise make sense of, which lined up with what `fdisk` and `mount` had already shown.

### Verifying the Recovery Was Real
1. Rather than trusting the recovered file listing at face value, I opened a few of the recovered `.jpg` files in the GNOME Image Viewer to confirm they weren't corrupted fragments.
2. The images opened cleanly and displayed as expected, a black-and-white photo of someone playing guitar (`00019717.jpg`, 396x461, 29.9 kB), a cat next to a pumpkin, and a shark photo, confirming the carved files were intact, viewable images and not just correctly-named garbage data.

## What's in This Repo

```
fat-carving-lab/
├── README.md                          # This file
├── image/
│   └── 11-carve-fat.dd                # Raw disk image with missing/corrupted partition table
├── recovered/
│   ├── 2003_document.doc
│   ├── _OMOPERS.WMV
│   ├── enterprise.wav
│   ├── haxor2.jpg
│   ├── holly.xls
│   ├── lin_1.2.pdf
│   ├── nlin_14.pdf
│   ├── paul.jpg
│   ├── pumpkin.jpg
│   ├── shark.jpg
│   ├── sm1.gif
│   ├── surf.mov
│   ├── surf.wmv
│   ├── _EST.PPT
│   └── wword60t.zip
└── screenshots/
    ├── 01-fdisk-blank-partition-table.png
    ├── 02-mount-failure-bad-superblock.png
    ├── 03-testdisk-manual-partition-type-selection.png
    ├── 04-testdisk-recovered-file-listing.png
    ├── 05-fiwalk-high-entropy-errors.png
    └── 06-recovered-images-verified-in-viewer.png
```

## Skills I Picked Up
- **Recognizing when a partition table is truly gone versus just unusual.** A `dos` disklabel type with zero partition entries and a blank disk identifier is a specific signature worth knowing, it's different from a disk that's simply unpartitioned or has an exotic layout.
- **Manually specifying a filesystem type when auto-detection fails.** TestDisk's automatic search isn't the end of the road; stepping through its manual type list and applying prior knowledge about the likely filesystem (FAT16, in this case) let recovery proceed anyway.
- **Reading a tool's failure output as diagnostic information, not just an error to dismiss.** Fiwalk's wall of "possible encryption detected" messages wasn't actually about encryption, it was a symptom of the same missing filesystem structure that made `fdisk` and `mount` fail, and recognizing that pattern connected all three tools' outputs into one coherent story.
- **Verifying recovered data instead of trusting a file listing.** A successful copy operation and a plausible filename aren't proof a file is intact; opening the recovered images to confirm they rendered correctly closed the loop on whether the recovery actually worked.

## How This Applies in the Real World
Corrupted or missing partition tables come up constantly in real digital forensics and incident response work, whether from a failing drive, an anti-forensic wipe attempt, or simple file system damage. An examiner who only knows how to run `fdisk -l` and read the output stops cold the moment that output comes back empty. Being able to fall back to manual filesystem identification and carving tools like TestDisk, and knowing how to interpret an automated tool's failure (like fiwalk's entropy warnings) as a clue rather than a dead end, is exactly the kind of persistence that separates a surface-level scan from an actual recovery effort.

## Where I'm Coming From
I'm making the jump into cybersecurity from a background in **healthcare**. It's a different field on paper, but a lot of the muscle memory carries over: following procedures carefully, protecting sensitive information, staying calm and methodical when something isn't working the way it's supposed to. I'm currently studying for **CompTIA Security+** and building labs like this one to get real hands-on reps in, since that's what I'm missing on paper right now compared to my experience.

## What I Want to Learn Next
- Using `blkls` and `foremost`/`scalpel` for signature-based carving directly against unallocated space, as an alternative to TestDisk's guided recovery
- Practicing with images that have partially overwritten or fragmented FAT structures, rather than just a missing partition table
- Comparing TestDisk's manual recovery output against `tsk_recover` and `fls`/`icat` on the same image once a filesystem type is confirmed, to build a repeatable cross-validation habit
- Digging into what specifically causes fiwalk's entropy-based "possible encryption" heuristic to trigger, so I can tell that failure mode apart from an image that's actually encrypted

## Limitations & What I'd Do Differently in Production
- **The FAT16 filesystem type was applied based on prior context about the lab, not purely from evidence in the image itself.** In a real case without that context, I'd need to validate the guess against file signatures in the carved data before trusting it.
- **Only a handful of recovered files were manually verified by opening them.** A full engagement would need every recovered file hash-checked and, where possible, opened or validated against known file signatures at scale, not spot-checked.
- **No hashing or chain-of-custody documentation was captured in this pass.** In a real recovery scenario the original image and every recovered file would need to be hashed (MD5/SHA-256) immediately, with those hashes logged before and after recovery.
- **Single tool for recovery.** In production I'd cross-validate TestDisk's output against `scalpel` or `foremost` carving the same image to confirm nothing was missed or misidentified.

## References
- [TestDisk Documentation](https://www.cgsecurity.org/testdisk.html)
- [The Sleuth Kit — fiwalk](https://wiki.sleuthkit.org/index.php?title=Fiwalk)
- [CompTIA Security+ (SY0-701) Exam Objectives](https://www.comptia.org/certifications/security)
- Kali Linux, attacker VM environment used throughout
