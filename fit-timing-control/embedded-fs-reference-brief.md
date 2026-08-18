# Embedded Flash Filesystems: History, Architecture, and Maturity

Supporting reference material for the article "Fit, Timing, Control: Rethinking a Storage Decision." This document covers the history and internal design of seven embedded storage systems referenced in that piece: littlefs, SPIFFS, ESP-IDF NVS, FatFs+FTL, ThreadX FileX+LevelX, YAFFS2, and SEGGER emFile.

---

## 1. littlefs

### History
- Developed primarily by Christopher Haster (GitHub handle "geky"), with copyright assigned to ARM Limited. Haster was affiliated with the University of Texas while doing the core work, and is credited as architect and maintainer. Development began in early 2017, and it became the filesystem used in ARM's Mbed OS for IoT devices.
- It ships as the `LittleFileSystem` class inside Mbed OS, alongside `FATFileSystem`; for external flash on microcontrollers it's positioned as outperforming Mbed's other filesystems in RAM, ROM, wear, and runtime, while FAT is recommended when PC-readable SD card portability matters.
- Actively maintained and versioned (v2.x branch, with v3.x in progress as of this writing), with commits continuing well past its original release; licensed BSD-3-Clause and written in C99.
- Not available in mid-to-late 2016, littlefs postdates that design window by several months to about a year, meaning it did not exist as an option at the time.

### Adoption timeline
- **2017, origin.** The GitHub repository was created in early 2017 (widely cited as February 2017), matching the "development began in early 2017" figure from Phoronix's original coverage above. Early users were primarily engineers experimenting with fail-safe embedded flash filesystems.
- **2018-2019, early traction.** Visibility grew within the embedded community specifically around its fail-safe (power-loss-resilient) design, wear leveling, and bounded RAM footprint, the same three properties highlighted in ARM's own Mbed OS documentation above.
- **2019-2020, breakout adoption.** Major frameworks began integrating littlefs directly (Mbed OS, community ESP-IDF/STM32 ports). Notably, developers migrating away from SPIFFS specifically cited SPIFFS's wear-leveling and corruption issues (the same class of defect catalogued in the companion defect-history document) as their reason for moving to littlefs.
- **2020-2023, standardization.** littlefs became a default or commonly-recommended choice for IoT devices, NOR/NAND flash storage, and resource-constrained MCUs requiring power-loss resilience.
- **2024-2026, mature and actively maintained.** As of this writing, the project stands at roughly **6.4k GitHub stars and 956 forks** (confirmed via the live repository), with continued releases across the v2.x line and v3.x in progress, including further work on wear leveling, metadata handling, and power-loss recovery.

This timeline is relevant to a build-vs-buy question that's time-bound rather than fixed: TFX's design window (mid-to-late 2016) predates littlefs entirely, and predates littlefs's breakout-adoption period by roughly three years, the point at which a team starting fresh would have had a mature, widely-vetted, actively-maintained option actually available to evaluate.

### How it works
- At a high level, littlefs is a block-based filesystem that uses small logs to store metadata and larger copy-on-write (COW) structures to store file data, described as a "two-layered cake" of metadata logs plus COW data structures.
- Core structural unit is the **metadata pair**: two blocks acting as a small circular-buffer log, chosen specifically because flash can only be programmed incrementally but must be erased a full block at a time, so a single-block log can't be updated atomically without a second block to fall back on. Each metadata pair independently tracks its own two block addresses, which keeps block-address bookkeeping simple even though the underlying blocks themselves get erased and swapped during writes.
- File data itself is stored using **CTZ (count trailing zeros) skip-lists**, chosen as a compact way to represent file contents as chains of data blocks, functioning like an inode structure anchored in a metadata pair. The design docs note this scheme has weaknesses for very small files, which is directly relevant when comparing against small, fixed-format records.
- Design goals explicitly include bounded RAM/ROM (no unbounded recursion, statically-configurable buffers), power-loss resilience via strong copy-on-write guarantees so on-disk state is always valid, and dynamic wear leveling for systems that can't fit a full flash translation layer, with the option to layer littlefs on top of an FTL for more aggressive static wear leveling.
- Additional properties: detects and works around bad blocks, thread-safe in the Mbed integration, and RAM usage stays flat as the filesystem grows (no growth-dependent memory scaling).

---

## 2. SPIFFS

### History
- Originally authored by Peter Andersson, who first worked on it targeting STM32 with external SPI flash before it became strongly associated with the ESP8266 ecosystem.
- Became the default flash filesystem for the Arduino ESP8266 core, where it was used for storing sketch data, config files, and web server content on the same SPI flash chip as the firmware.
- Licensed MIT ("use, modify, sell, print it out, roll it and smoke it"). A 0.4.0 rewrite was planned that would change the underlying structure and break compatibility with earlier versions, though the widely-deployed line stayed on the 0.3.x series (last tagged release 0.3.7).
- Now deprecated in the ESP8266 Arduino core in favor of LittleFS, with SPIFFS no longer actively supported by upstream maintainers. Espressif's ESP-IDF likewise moved new projects toward LittleFS/FAT+wear-leveling, keeping SPIFFS mainly for legacy compatibility.
- Design lineage: SPIFFS was explicitly inspired by YAFFS, though YAFFS targets NAND flash on larger-RAM systems, and SPIFFS borrowed ideas while adapting them to SPI NOR flash constraints.

### How it works
- Designed around specific SPI NOR flash characteristics: only large areas (blocks) can be erased, erasing resets all bits to ones, writing pulls ones to zeroes, and zeroes cannot be pulled back to ones without a full erase, all under the assumption of a memory-constrained embedded target.
- The storage area is divided into blocks, and each block is further divided into pages of a fixed count; individual pages cannot be erased on their own, only whole blocks can be erased. One file update typically consumes two pages (metadata page + data page).
- It targets NOR flash generally, not only SPI flash, and the docs note it could theoretically run on the embedded flash of a microprocessor rather than just external SPI parts, and multiple independent SPIFFS instances/configurations can coexist on the same physical device.
- Flat namespace only: it produces no real directory hierarchy, so creating a file at "tmp/myfile.txt" just creates a file literally named "tmp/myfile.txt" rather than a file inside a "tmp" directory.
- Wear leveling is achieved via a garbage-collection scheme with tunable heuristics (`SPIFFS_GC_MAX_RUNS`, erase-age scoring). A worked example from the project's own FAQ: on a 1 MB part with 64 KB blocks and 1 KB pages, roughly 256 file updates exhaust one block's free pages, and cross-block wear leveling extends usable life into tens of millions of update cycles before flash endurance limits are hit.
- Known constraints called out by the maintainers: poor scalability beyond roughly 128 MB (a direct consequence of minimizing RAM use), no bad-block detection or handling, single fixed configuration per binary (no generic build that adapts at runtime to different configs), and it is explicitly not a real-time stack since one write can take much longer than another.
- Power-loss handling relies on an optional consistency check (`SPIFFS_check`) rather than the strict transactional guarantees littlefs makes; the FAQ documents patterns like battery-backed dirty bits for detecting a check-worthy power loss.

---

## 3. ESP-IDF NVS (Non-Volatile Storage)

### History
- Developed by Espressif as part of ESP-IDF for the ESP32 family (and later chips), positioned specifically for small key-value configuration data rather than file-style storage.
- Has evolved across IDF versions (structure described consistently from v4.0 through current/v6.x docs), with encryption support and stricter type/size handling added over time.
- Explicitly scoped as the "sidecar" to a real filesystem: the docs themselves recommend that if you need large blobs or strings, you should use a FAT filesystem layered on top of ESP-IDF's separate wear-levelling library instead of NVS. This makes it functionally the closest of the three to what a dedicated "storage module for specific formatted data" is trying to do, rather than a general-purpose file abstraction.

### How it works
- Operates purely on key-value pairs: keys are ASCII strings currently capped at 15 characters, values can be any of the standard fixed-width integer types, plus strings and blobs. String values are capped at 4000 bytes including the null terminator, and blob values are capped at 508,000 bytes or 97.6% of the partition size minus 4000 bytes, whichever is smaller.
- Internally built from two entities: pages and entries. A page is a logical structure holding part of the overall log, and maps one-to-one onto a physical flash sector; pages in use carry a sequence number that orders them, with higher numbers meaning more recently created.
- Each page moves through defined states: empty (all 0xFF, unused, no sequence number), active (header written, valid sequence number, has free entries available for writing, with the constraint that no more than one page can be in this state at a time), and full (consistent, entirely filled with key-value pairs). This state machine is what gives NVS its power-loss safety, since a page is only ever in one write-eligible state system-wide.
- Writes are explicitly buffered until commit: after setting values, `nvs_commit()` must be called to guarantee the change reaches flash, since individual implementations may flush at other times but that isn't guaranteed by the API contract.
- Namespacing is built in via `nvs_open`, letting different modules keep keys in separate namespaces to avoid collisions, and NVS uses whatever partitions in the partition table are marked with type "data" and subtype "nvs."
- Wear leveling is handled internally by NVS's own erase-write balancing logic rather than by ESP-IDF's separate wear_levelling component, evenly distributing writes across blocks to extend flash life; a worked example in Espressif's own FAQ assumes a 4096-byte sector holding roughly 126 32-byte entries after accounting for a 32-byte page header and a 32-byte entry-state bitmap.
- No filesystem semantics at all: no paths, no directories, no streaming reads, it is a structured record store tuned for lots of small, typed values rather than arbitrary file content. That's a meaningfully different design target than littlefs or SPIFFS, and the closest architectural peer to a custom key-value parameter store.

---

## 4. FatFs (+ Flash Translation Layer)

### History
- Authored by a Japanese developer known only by the handle "ChaN," first released February 26, 2006. Now on the R0.x line, with R0.16 released July 2025.
- Not a flash filesystem in its own right: FatFs implements the FAT/exFAT format and is completely separated from the physical storage layer, so on raw NOR/NAND it must be paired with a separate Flash Translation Layer (FTL) that emulates a block device underneath it. There's no single standard FTL; implementations vary by vendor and are often hand-rolled per project.
- Extremely widely deployed. It underpins storage support in Espressif ESP-IDF, STMicroelectronics STM32Cube, Zephyr RTOS, MicroPython, ArduPilot, RT-Thread, Samsung TizenRT, and SWUpdate, among others, almost always hidden behind a vendor SDK or RTOS abstraction layer rather than used directly.
- Licensed under ChaN's own permissive, BSD-like terms, free for commercial and non-commercial use.

### How it works
- FatFs is a pure filesystem-format library: it understands FAT12/16/32 and exFAT layout, directory structures, and long filenames, but has no concept of flash-specific concerns like wear leveling, bad-block handling, or power-loss atomicity. Those are entirely the FTL's responsibility, if the FTL provides them at all.
- Because FAT was designed for magnetic media, using it directly on raw NAND/NOR without a wear-leveling FTL underneath is explicitly discouraged in the embedded community: FAT will write repeatedly to the same physical sectors (the FAT tables themselves, directory entries) unless something below it remaps logical to physical addresses and spreads wear.
- Designed to be minimal and portable: 2-10 KB of RAM in its smallest configuration, ANSI C, no OS dependency, ported to 8051, PIC, AVR, ARM, and more.

### Recent defect history (2026)
- In March 2026, security researchers at runZero revisited FatFs (previously lightly audited in 2017) using AI-assisted fuzzing and disclosed **seven CVEs** (CVE-2026-6682 through CVE-2026-6688), spanning memory corruption, silent data corruption, denial-of-service, and information disclosure, all reachable via a crafted FAT/exFAT/GPT volume delivered over USB, SD card, or an OTA firmware-update image.
- Only **one of the seven** (CVE-2026-6684, a GPT partition-scan infinite loop) has been fixed upstream, in R0.16. The rest remain unpatched at the library level as of this writing.
- Notably, runZero reported it could not get a response from the maintainer after repeated outreach, even after routing through Japan's JPCERT/CC coordination center, leaving downstream vendors (who bundle FatFs into ESP-IDF, STM32Cube, Zephyr, and others) to patch independently or remain exposed.

---

## 5. ThreadX FileX + LevelX

### History
- ThreadX itself was first released in 1997 by Express Logic (San Diego), founded and led by William Lamie. FileX (FAT-compatible filesystem) and LevelX (NAND/NOR wear-leveling layer) are companion components in the same suite.
- Ownership has changed twice: Microsoft acquired Express Logic in 2019 and rebranded the whole suite Azure RTOS; in November 2023 Microsoft donated the codebase to the Eclipse Foundation under the MIT license, and it's now known as Eclipse ThreadX.
- Despite the current open-source licensing, this is functionally a commercial-pedigree stack: it has been deployed on more than 12 billion devices, and FileX carries safety certifications including IEC 61508 SIL 4, ISO 26262 ASIL D, and IEC 62304 Class C, positioning it for safety-critical use in a way none of littlefs/SPIFFS/NVS claim.

### How it works
- FileX supports FAT12/16/32 and exFAT, and includes an optional fault-tolerance module (`fx_fault_tolerant_enable`) that journals write operations so file and directory updates survive power loss, distinct from FatFs, which has no such feature built in.
- LevelX sits underneath FileX (or can be used standalone) as a dedicated wear-leveling layer for raw NAND and NOR flash. It presents a logical sector interface, mapped internally to physical flash, and selects which block to erase and reuse based primarily on erase count, with a secondary heuristic favoring blocks with more obsolete mappings to reduce copy overhead.
- Designed for fault tolerance at the flash-update level: sector writes happen in a multi-step process that can be safely interrupted at any step and recovered automatically on the next operation, similar in spirit to NVS's page state machine.

### Known defect history
- The Eclipse Foundation's own release notes describe a LevelX update that "addresses several bugs, one of which could lead to memory corruption."
- Open, maintainer-labeled `bug` issues on the public tracker include an unprotected access to `sb_inodes` (a shared list touched without synchronization, a concurrency bug) and other confirmed defects spanning multiple release cycles (issues opened as recently as February 2026).
- A field report on the ThreadX users mailing list describes LevelX producing **two simultaneously valid sector mappings for the same logical sector** after repeated power-loss testing on a real product (STM32H563 with a custom SPI-serial-flash driver), exactly the kind of "wear leveling was supposed to handle this" failure that undercuts its core fault-tolerance claim under real-world power-cycling conditions.

---

## 6. YAFFS2 (Yet Another Flash File System)

### History
- Originally written by Charles Manning starting in 2001/2002, sponsored by Toby Churchill Ltd and Brightstar Engineering, for the company that became Aleph One Ltd. YAFFS1 targeted the small (512-byte) NAND pages common at the time; YAFFS2 followed to support larger (2KB+) pages and modern MLC/TLC NAND.
- Dual-licensed: GPLv2 (to be Linux-compatible) or a separate commercial license from Aleph One, similar in structure to the SPIFFS/generalist split, free to use but the company monetizes commercial licensing and support.
- Widely used historically in embedded Linux and was notably one of the flash filesystems used in early Android devices, alongside a broad base of consumer electronics, avionics, and critical-infrastructure deployments per Aleph One's own description.

### How it works
- Log-structured, similar in family to JFFS2, but written specifically for NAND (unlike JFFS2, which predates common NAND use). Each file is split into fixed-size chunks matching the flash page size, with a chunk 0 acting as a header (type, permissions, size, etc.) and subsequent chunks numbered sequentially.
- Every flash page write carries out-of-band (OOB) tags: file ID, chunk number, and a write serial/sequence number, which is what allows YAFFS2 to reconstruct current-vs-stale state after an unclean shutdown by scanning and sorting blocks by sequence number.
- YAFFS2 cannot rely on YAFFS1's simpler deletion-marker scheme because larger modern NAND typically restricts how many times a page can be rewritten before erase ("write-once" style constraints per page), so YAFFS2 uses shrink headers and chronological log-scanning instead, at the cost of a more complex garbage collector.
- Garbage collection reclaims blocks by copying still-live chunks out of a "dirty" block into the currently-allocating block, then erasing the source; YAFFS2's own documentation acknowledges this scan-and-sort approach makes mount-time recovery more complex than YAFFS1's.

### Known defect history
- Open, unresolved issues on the maintained GitHub mirror include: an object-header field (`inband_is_shrink`) that is never actually written, forcing `is_shrink` to be treated as always true on scan when inband tags are used (a correctness bug in exactly the shrink-tracking mechanism YAFFS2 relies on); an unprotected, unsynchronized access to the shared inode list (`yaffs_flush_inodes`), a concurrency bug; and a potential buffer overflow in `yaffs_get_obj_name` when the character type isn't a single byte.
- These are long-lived, acknowledged-but-unfixed issues on the current maintained fork, not one-off reports, consistent with a mature but resource-constrained maintenance model (a small commercial company rather than a well-funded foundation).

---

## 7. SEGGER emFile

### History
- Developed by SEGGER Microcontroller GmbH, founded in 1992 in Germany. emFile itself is described by SEGGER's own partner materials as having "over 28 years of continuous development," placing its origin in the late 1990s, alongside the company's other early middleware products (embOS, emWin).
- Purely commercial: royalty-free one-time licensing after purchase, with a free evaluation license for non-commercial and educational use. No public source repository or public issue tracker, unlike every other system in this comparison, defect visibility depends entirely on SEGGER's own forum and support channels.
- Documented in production use since at least 2012 (SEGGER's own case study cites BYK-Gardener's spectrophotometer product line).

### How it works
- Supports two filesystem formats: FAT12/16/32/exFAT (for PC interoperability) and SEGGER's own proprietary Embedded File System (EFS), layered over a common storage abstraction that supports NAND, NOR, SD/MMC/eMMC, CompactFlash, and USB mass storage through a common driver interface.
- Organized in five layers (API, filesystem, storage, device, hardware), with SEGGER's marketing materials describing all access operations as atomic by design.
- Critically, and unlike SEGGER's own marketing headline claims, **fail-safety is not automatic**: SEGGER's own support forum confirms that plain FAT is not fail-safe by design, and that the separately licensed Journaling add-on is required to make the FAT/EFS filesystem itself fail-safe, on top of a fail-safe-by-design device driver. A team evaluating emFile purely from the product page could reasonably miss that the core fail-safety claim depends on an additional purchased component.
- NAND driver includes configurable ECC bit-error correction, but the correction strength must be manually matched to the specific NAND part's datasheet requirements.

### Known defect history
- Multiple real field reports on SEGGER's own support forum describe file/filesystem corruption under frequent power cycling on NAND, consistent with SEGGER's own explanation that plain FAT (without the Journaling add-on purchased separately) is not fail-safe and can't survive an ill-timed reset.
- A separate forum thread documents corruption traced to inadequate ECC configuration after a NAND part substitution: the replacement NAND part required a higher bit-error correction capability (8 bits/512 bytes vs. 1 bit/512 bytes) than the emFile driver was configured for, causing systematic corruption once the substitution went into production.
- Because there's no public issue tracker, this defect history is necessarily thinner and less structured than the open-source systems above; that's itself a relevant data point about transparency and independent verifiability, not just about defect count.

---

## Comparison Table

| | littlefs | SPIFFS | ESP-IDF NVS | FatFs + FTL | ThreadX FileX + LevelX | YAFFS2 | SEGGER emFile |
|---|---|---|---|---|---|---|---|
| First available | ~2017 | Pre-ESP8266 era, popularized ~2014-2015 | With ESP-IDF, ESP32 era onward | 2006 (FatFs); FTL varies by vendor | 1997 (ThreadX/FileX); LevelX later | 2001-2002 (YAFFS1), YAFFS2 later | Late 1990s (est.) |
| Abstraction | Real filesystem (files, some dir support) | Filesystem, flat namespace only | Key-value store, no file abstraction | Real filesystem (FAT/exFAT) | Real filesystem (FAT/exFAT) | Real filesystem (POSIX-like) | Real filesystem (FAT/exFAT or proprietary EFS) |
| License | Open source (BSD-3) | Open source (MIT) | Open source (Apache 2.0, part of ESP-IDF) | Free, ChaN's own permissive license | Open source (MIT, since 2023) | Dual: GPLv2 or commercial | Commercial, royalty-free after purchase |
| Atomicity model | COW + 2-block metadata log | GC-based, optional consistency check | Page state machine (empty/active/full) | None built in; entirely dependent on the FTL | Optional journaling module (FileX); multi-step recoverable writes (LevelX) | Log-structured, sequence-number scan on mount | Atomic by design claim; fail-safety requires separately licensed Journaling add-on |
| Wear leveling | Dynamic, bad-block aware, optional FTL layering | GC with tunable heuristics | Built-in erase-write balancing | Entirely dependent on the FTL underneath | Built into LevelX, erase-count based | Built-in via garbage collection | Built into NAND/NOR drivers |
| Current status | Active, ARM-maintained | Deprecated in favor of LittleFS | Active, Espressif-maintained | Active, single-maintainer; 7 unpatched CVEs disclosed 2026 | Active, Eclipse Foundation-maintained | Active but small-team maintained (Aleph One) | Active, commercial, closed defect-tracking |

---

## Sources
- littlefs: GitHub littlefs-project/littlefs, ARM Mbed docs and DESIGN.md, Phoronix coverage, Christopher Haster's GitHub/LinkedIn
- SPIFFS: GitHub pellepl/spiffs (README, TECH_SPEC, wiki FAQ), ESP8266 Arduino core docs
- ESP-IDF NVS: Espressif ESP-IDF Programming Guide (multiple versions), Espressif ESP-FAQ
- FatFs: elm-chan.org official site and patch notes, Wikipedia, runZero's 2026 vulnerability disclosure (blog and GitHub PoC repo), The Hacker News, Security Affairs, GBHackers coverage
- ThreadX FileX + LevelX: Eclipse Foundation GitHub (eclipse-threadx/levelx, /filex, /rtos-docs), Wikipedia/HandWiki, CNX Software and Hackster.io coverage of the Eclipse Foundation transition, Eclipse ThreadX mailing list archives
- YAFFS2: yaffs.net official documentation ("How YAFFS Works," Yaffs2 Specification), GitHub Aleph-One-Ltd/yaffs2 (README, issues), Wikipedia/Grokipedia
- SEGGER emFile: segger.com product pages and User Guide, SEGGER support forum threads, STMicroelectronics partner page, Wikipedia/Grokipedia on SEGGER company history
