# Known Defect History: SPIFFS, littlefs, ESP-IDF NVS, FatFs+FTL, ThreadX FileX+LevelX, YAFFS2, SEGGER emFile

Supporting reference material for the article "Fit, Timing, Control: Rethinking a Storage Decision." Sourced from each project's public issue tracker and release notes.

These are not obscure or contrived failures. Every one below falls into the same failure class flagged in a code-quality review of TFX's own storage subsystems: torn writes, corruption on power/reset events, and silent data loss. The point isn't that these filesystems are poorly engineered, SPIFFS and littlefs in particular are widely used and actively maintained, the point is that maturity and wide adoption reduce but do not eliminate this risk class, and when it does surface, the fix timeline and priority sit with the maintainer, not the team depending on it. The FatFs entry is a useful outlier worth flagging separately: it's a live, unpatched, actively disclosed 2026 security issue, not historical stability trivia, and the maintainer's unresponsiveness during coordinated disclosure is itself a sharp illustration of that same control question.

---

## SPIFFS (pellepl/spiffs)

| # | Issue | Description | Source |
|---|---|---|---|
| 1 | #145 | Confirmed wear-leveling defect: fully-written blocks never participate in wear leveling under certain write/delete patterns, meaning some blocks erase far more often than others despite the wear-leveling feature existing specifically to prevent that. | github.com/pellepl/spiffs/issues/145 |
| 2 | #88 | Lock/unlock asymmetry bug (fixed in 0.3.7 release line). | github.com/pellepl/spiffs/releases (0.3.7 notes) |
| 3 | (unnumbered) | Double-write of the same data under certain cache configurations (fixed in 0.3.7 line, no isolated issue number found, noted in release history). | github.com/pellepl/spiffs/releases |
| 4 | #90 | Files not visible via `SPIFFS_readdir` under certain conditions. | github.com/pellepl/spiffs/releases (0.3.7 notes) |
| 5 | (maintainer FAQ) | The project's own maintainer removed the mount-time (`SPIFFS_check`) integrity check entirely in later versions because it had become too slow on larger flash sizes, pushing corruption detection/recovery from an automatic mount-time step onto a manual, user-invoked function. | github.com/pellepl/spiffs/wiki/FAQ |

---

## littlefs (littlefs-project/littlefs)

| # | Issue | Description | Source |
|---|---|---|---|
| 1 | #1177 | Filesystem corruption when a file open for writing under one handle is truncated via a second handle, reported and reproduced on a build from January 2026, i.e. an actively-maintained, currently-supported version. | github.com/littlefs-project/littlefs/issues/1177 |
| 2 | #953 | Superblock corruption bricking two field devices with no associated power-loss event or other obvious trigger; devices became unresponsive/unmountable on reboot. | github.com/littlefs-project/littlefs/issues/953 |
| 3 | #986 | Multiple independent field reports of directory-pair corruption and bad blocks on NOR flash under ordinary file-creation workloads (creating a few hundred files); reporter explicitly states devices became "bricked, idling in an assert," and that littlefs seemed "too unstable and unreliable" for production use in their configuration. | github.com/littlefs-project/littlefs/issues/986 |
| 4 | #1057 | Read-path cache bug: `lfs_file_read`/`lfs_file_seek` returned incorrect data under specific cache-size/offset combinations, confirmed on v2.10, a recent stable release. | github.com/littlefs-project/littlefs/issues/1057 |
| 5 | #268 | File corruption after `lfs_file_truncate`, with no error surfaced during read/write, data silently replaced with runs of 0xFF. | github.com/littlefs-project/littlefs/issues/268 (ARMmbed/littlefs predecessor repo) |
| 6 | #394 | "LFS Corruption", reporter's filesystem was silently reformatted due to an `LFS_ERR_CORRUPT` result they could not root-cause after a week of investigation. | github.com/ARMmbed/littlefs/issues/394 |
| 7 | #1194 (fixed) | Data corruption from multiple open write handles to the same file, fixed in a later release, meaning it shipped and was live for some period before the fix. | littlefs-project/littlefs release notes |

---

## ESP-IDF / NVS (espressif/esp-idf)

| # | Issue | Description | Source |
|---|---|---|---|
| 1 | #13828 | Entire NVS partition silently erased to zero under a specific button-hold/reset sequence; confirmed as a reproducible bug internally by Espressif ("Type: Bug, bugs in IDF"). | github.com/espressif/esp-idf/issues/13828 |
| 2 | hardeman/esp32-nvs-erase-after-idf-update | NVS partition erased specifically as a side effect of an ESP-IDF framework version upgrade (4.2 → 4.4.8), a defect triggered purely by updating the dependency, not by any application-level action. | github.com/hardeman/esp32-nvs-erase-after-idf-update |
| 3 | #9149 | Intermittent failures where `nvs_set_*` calls report success but the value is not actually persisted; once triggered, the partition becomes effectively read-only, recoverable only via a full flash erase and firmware reinstall. | github.com/espressif/esp-idf/issues/9149 |
| 4 | #17561 | `nvs_erase_all()` reports success and NVS itself treats the data as gone, but the underlying flash bytes are not physically erased, data remains recoverable from a raw flash dump. Marked "Won't Do" by Espressif. | github.com/espressif/esp-idf/issues/17561 |
| 5 | #5747 | Corrupted NVS key-partition (`ESP_ERR_NVS_CORRUPT_KEY_PART`) under flash-encryption configurations, with the reporter unable to determine the cause. | github.com/espressif/esp-idf/issues/5747 |
| 6 | #7837 | Corrupted NVS blob causes a boot-time crash from an unhandled assertion, with no way for the application to catch the error and recover (e.g. by clearing NVS), the device simply fails to boot. | github.com/espressif/esp-idf/issues/7837 |

---

## FatFs + FTL (elm-chan.org / runZero disclosure, 2026)

| # | Issue | Description | Source |
|---|---|---|---|
| 1 | CVE-2026-6685 | Unsigned-subtraction wraparound in dirty-cache handling on fragmented volumes; can cause out-of-bounds memory effects and silent data corruption. CVSS 6.1. Unpatched. | github.com/runZeroInc/vulns-2026-fatfs-chance; NVD CVE-2026-6685 |
| 2 | CVE-2026-6683 | exFAT divide-by-zero in sync/write paths, triggerable via crafted media; reliable device crash, and can brick hardware if triggered mid-firmware-update. CVSS 4.6. Unpatched. | NVD CVE-2026-6683 |
| 3 | CVE-2026-6686 | Extending a file past its end exposes uninitialized cluster data, leaking stale content from previously deleted files. CVSS 4.6. Unpatched. | NVD CVE-2026-6686 |
| 4 | CVE-2026-6684 | GPT partition-scan infinite loop from a malformed partition table; can hang the device at mount time with no watchdog recovery on bare-metal targets. CVSS 4.6. **The only one of the seven fixed upstream**, in FatFs R0.16. | NVD CVE-2026-6684 |
| 5 | (three more, CVE-2026-6682/6687/6688) | Additional memory-safety and information-disclosure issues from the same 2026 disclosure batch, full technical detail in the runZero repository. | github.com/runZeroInc/vulns-2026-fatfs-chance |

Context worth keeping: this is a **2026 disclosure**, not old history. FatFs is bundled into Espressif ESP-IDF, STMicroelectronics STM32Cube, Zephyr RTOS, MicroPython, ArduPilot, RT-Thread, TizenRT, and SWUpdate. runZero reported it could not get a response from the maintainer after repeated attempts, even after routing through JPCERT/CC, leaving downstream vendors to patch independently.

---

## ThreadX FileX + LevelX (eclipse-threadx/levelx, eclipse-threadx/filex)

| # | Issue | Description | Source |
|---|---|---|---|
| 1 | (release notes) | A LevelX release "addresses several bugs, one of which could lead to memory corruption," per the Eclipse Foundation's own release notes. | github.com/eclipse-threadx/levelx/releases |
| 2 | #15 | `yaffs_flush_inodes`-equivalent pattern in LevelX: unsynchronized access to a shared list from multiple contexts, a concurrency bug open on the tracker since mid-2025. | github.com/eclipse-threadx/levelx/issues/15 |
| 3 | #67, #55, #53, #52, #51, #49 | Multiple additional issues labeled "bug" by maintainers, open as of early 2026, spanning the current release line. | github.com/eclipse-threadx/levelx/issues |
| 4 | (mailing list) | Field report on the ThreadX users mailing list: after repeated power-loss testing on a real product (STM32H563, custom SPI-serial-flash driver), LevelX produced **two simultaneously valid sector mappings for the same logical sector**, exactly the failure mode its wear-leveling and fault-tolerance design is meant to prevent. | eclipse.org/lists/threadx-users, "Issue with LevelX Wear-Leveling on STM32H563" |

---

## YAFFS2 (Aleph-One-Ltd/yaffs2)

| # | Issue | Description | Source |
|---|---|---|---|
| 1 | #18 | The `inband_is_shrink` object-header field is never actually written, forcing `is_shrink` to be treated as always-true on scan when inband tags are used, a correctness bug in the exact mechanism YAFFS2 relies on to track file truncation across power loss. | github.com/Aleph-One-Ltd/yaffs2/issues/18 |
| 2 | #15 | `yaffs_flush_inodes` accesses the shared `sb_inodes` list without protection, a concurrency/race bug. | github.com/Aleph-One-Ltd/yaffs2/issues/15 |
| 3 | #6 | `yaffs_get_obj_name` may `memset` past the buffer size if `sizeof(YCHAR)` isn't 1, a potential buffer overflow. | github.com/Aleph-One-Ltd/yaffs2/issues/6 |
| 4 | #5 | Open question (unresolved) about whether context allocated with `kmalloc` in `yaffs_internal_read_super` is ever freed, potential memory leak. | github.com/Aleph-One-Ltd/yaffs2/issues/5 |

All four are long-lived, acknowledged-but-open issues on the currently maintained fork, not one-off or already-patched reports, consistent with a small commercial team (Aleph One) maintaining a mature but resource-constrained codebase.

---

## SEGGER emFile (commercial, no public issue tracker)

| # | Issue | Description | Source |
|---|---|---|---|
| 1 | Forum thread #4453 | File corruption on NAND under frequent power cycling; SEGGER's own support response confirms plain FAT "is not fail-safe by design" and that the separately licensed Journaling add-on is required for fail-safety, on top of a fail-safe device driver. | forum.segger.com, Thread 4453 |
| 2 | Forum thread #4782 | Systematic file corruption traced to inadequate ECC configuration after a NAND part substitution, the replacement part required stronger error correction (8 bits/512 bytes vs. 1 bit/512 bytes) than the driver was configured for. | forum.segger.com, Thread 4782 |
| 3 | Forum thread #6241 | Filesystem corruption ("too few clusters allocated to file") tied to NAND bus timing after a flash part substitution, requiring manual EMC timing adjustment to resolve. | forum.segger.com, Thread 6241 |

Because emFile has no public source repository or issue tracker, this list is necessarily thinner and less independently verifiable than the open-source systems above, visibility depends entirely on what surfaces in SEGGER's own support forum. That opacity is itself a relevant data point for the control/transparency argument, separate from raw defect count.

---

## Context

Every system compared in the article this document supports, including TFX's own custom storage layer, has real exposure in this same failure category (the parameter store's non-atomic append-and-invalidate window, and the message buffer's whole-region wipe on single-record corruption, are the comparable TFX-side findings). The point isn't "who has more bugs," it's **who controls the fix**: a defect in a team's own storage code is something that can be read, root-caused, and patched on that team's own schedule; a defect in someone else's dependency (see NVS #17561, marked "Won't Do," the littlefs #986 reporter left without a resolution, or FatFs's unresponsive maintainer during a 2026 coordinated disclosure) means the timeline, and sometimes the outcome, belongs to someone else. The commercial systems (ThreadX FileX+LevelX, SEGGER emFile) add a further wrinkle worth naming explicitly: commercial licensing and safety certifications don't exempt a system from this same failure class, and in emFile's case, the lack of a public issue tracker makes independent verification of the defect history harder, not easier, than for the open-source systems.
