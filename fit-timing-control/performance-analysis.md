# Performance Analysis: TFX Storage Layer vs. Seven Comparison Systems

Supporting reference material for the article "Fit, Timing, Control: Rethinking a Storage Decision." This document presents the full energy analysis behind that article: methodology, per-scenario results, and the complete comparison across TFX's storage layer, littlefs, SPIFFS, ESP-IDF NVS, and four additional systems (FatFs+FTL, ThreadX FileX+LevelX, SEGGER emFile, YAFFS2) added for completeness.

All figures use TFX's baseline, as-shipped storage layer. This analysis does not evaluate any hypothetical optimized variant of TFX's own code; every TFX figure below reflects the storage layer exactly as fielded. Bugs and vulnerabilities in any system are out of scope; this is a power/energy efficiency study only.

---

## Methodology

### Hardware and energy model

The analysis is derived from real datasheet figures for the actual hardware: an EFM32PG1B MCU (Silicon Labs) running at 38.4MHz with its DC-DC converter disabled, driving an S25FL132K0XMFI041 SPI NOR flash chip (Infineon/Cypress) at a 3.84MHz SPI bus clock, 256-byte page size, 4096-byte sector size.

Every operation is modeled in two phases where applicable: a **transfer phase** (the MCU clocks command, address, and data over SPI, both chips active) and, for writes and erases, a **busy phase** (the flash executes internally over its programming or erase time, with the MCU assumed to stay active polling status rather than sleeping, the single largest assumption in this model). A nominal 3.0V is used for all energy conversions, the mid-range of the flash's 2.7-3.6V supply spec.

Baseline per-operation energy costs (bare payload, before any per-system framing overhead):

| Operation | Typical cost |
|---|---|
| Read (single SPI transfer) | 1.55 µJ (message-sized) |
| Write (transfer + ~0.7ms program-busy phase) | 28.7 µJ (message-sized) |
| Erase (4KB sector, transfer + ~50ms erase-busy phase) | 3,731.7 µJ |

Parameter and log records use their own record-size-adjusted figures (parameters: 0.554 µJ read / 27.66 µJ write per 6-byte register; log entries: ~29.55 µJ write per 48-byte entry). NVS uses a single physically-derived entry cost (29.05 µJ write / 2.0 µJ read) applied uniformly to every entry it writes, since NVS always writes a fixed 32-byte structure regardless of the logical payload it's carrying.

### littlefs's real on-flash framing

littlefs does not store a bare payload. Every record it commits carries a 4-byte tag header and is sealed by an 8-byte CRC entry validating everything written since the last commit, 12 bytes of framing on top of whatever payload it's storing, on every write, with no batching assumed anywhere in this analysis (every write, on every system including TFX's own, is modeled as its own isolated, immediate flash transaction, matching the single-variable calling convention TFX's own parameter code uses). This changes littlefs's per-write energy only slightly (the flash's fixed program-busy phase dominates regardless of a few extra transfer bytes), but it meaningfully reduces how many records fit per physical region, which is what actually drives littlefs's erase frequency in every suite below.

### Periodic restart cost

Every scenario longer than 28 days includes a periodic controlled device restart, a clean reboot that loses no data but requires every system to rebuild its in-memory pointers and indexes from what's actually on flash. This is priced identically across every system in a given scenario, as a full read-back of every physical page in the region(s) that system uses, at the flash's native 256-byte page granularity. This is a deliberately conservative, system-agnostic simplification (a real system's mount algorithm, NVS's in particular, likely costs less in practice), and because the cost depends only on total physical capacity rather than which system holds the data, it never changes a ranking, it adds a small, flat, common tax (well under 1% of most totals at the 100-day tier, a few percent at 10 years).

### Duration tiers

Each of the first three suites (messages, parameters, log) runs the same workload pattern at three duration tiers: a short tier (5-13 days depending on suite), a medium tier (100-177 days), and a 10-year tier. The combined suite (Suite 4) uses the exact composite duration stated in its own scenario definitions (3,727 days for hourly-cadence scenarios, 3,653 days for 15-minute-cadence scenarios) rather than a rounded figure.

### Energy-first averaging

Wherever this document or the article averages results across scenarios, the average is computed by summing total joules first and only then converting to a percentage. Averaging the percentage ratios directly gives a different and, on inspection, misleading answer, because it lets low-energy scenarios carry the same statistical weight as the highest-energy ones, systematically understating how bad the worst-case systems actually are once weighted by where the real energy is spent.

---

## Suite 1: Messages

Twelve scenarios testing message-buffer storage cost across two buffer sizes (24 vs. 8,000-message capacity), two wake cadences (hourly vs. 15-minute), and three duration tiers (13/8 days, 103/177 days, 10 years), against a fixed connectivity-outage pattern. Message mix: one INFO message (27B), READING messages (24B), and periodic CONFIG messages (18B).

**Full results, all twelve scenarios (energy):**

| Scenario | Days | TFX | littlefs | SPIFFS | NVS |
|---|---|---|---|---|---|
| S1 | 13 | 22.98 mJ | 25.08 mJ | 131.19 mJ | 58.63 mJ |
| S2 | 13 | 22.98 mJ | 24.43 mJ | 131.22 mJ | 58.78 mJ |
| S3 | 8 | 60.85 mJ | 68.12 mJ | 395.99 mJ | 149.68 mJ |
| S4 | 8 | 60.85 mJ | 60.97 mJ | 406.33 mJ | 151.29 mJ |
| S5 | 177 | 380.85 mJ | 430.07 mJ | 2.545 J | 1.156 J |
| S6 | 177 | 380.85 mJ | 429.41 mJ | 2.545 J | 1.156 J |
| S7 | 103 | 818.21 mJ | 943.31 mJ | 5.774 J | 2.639 J |
| S8 | 103 | 818.21 mJ | 936.16 mJ | 5.785 J | 2.641 J |
| S9 | 3,727 | 8.045 J | 9.126 J | 54.736 J | 25.512 J |
| S10 | 3,727 | 8.045 J | 9.126 J | 54.736 J | 25.512 J |
| S11 | 3,653 | 29.205 J | 33.638 J | 206.936 J | 95.791 J |
| S12 | 3,653 | 29.205 J | 33.631 J | 206.946 J | 95.793 J |

**Ratio to TFX:**

| Scenario | littlefs | SPIFFS | NVS |
|---|---|---|---|
| S1 | 1.09x | 5.71x | 2.55x |
| S2 | 1.06x | 5.71x | 2.56x |
| S3 | 1.12x | 6.51x | 2.46x |
| S4 | 1.00x | 6.68x | 2.49x |
| S5 | 1.13x | 6.68x | 3.04x |
| S6 | 1.13x | 6.68x | 3.04x |
| S7 | 1.15x | 7.06x | 3.23x |
| S8 | 1.14x | 7.07x | 3.23x |
| S9 | 1.13x | 6.80x | 3.17x |
| S10 | 1.13x | 6.80x | 3.17x |
| S11 | 1.15x | 7.09x | 3.28x |
| S12 | 1.15x | 7.09x | 3.28x |

A stable ranking across every scenario: TFX cheapest, littlefs a close second (0.2-15.3% more expensive), NVS third, SPIFFS most expensive throughout (5.7-7.1x TFX), driven by SPIFFS's page-pair floor (two full page writes to create a message record, two more to delete it) and by a fixed 164-message thrashing tail in the two scenarios with no discard valve (S4/S8/S12), where a 72-hour dead-first outage exhausts SPIFFS's message headroom 40 hours before connectivity returns.

### FAT-family and YAFFS2 (Suite 1)

| Scenario | FatFs+FTL (nobatch) | FatFs+FTL (deferred) | FileX+LevelX/emFile (nobatch) | FileX+LevelX/emFile (deferred) | YAFFS2 |
|---|---|---|---|---|---|
| S1 | 4.426 J | 3.186 J | 611.43 mJ | 425.86 mJ | 855.01 mJ |
| S2 | 4.243 J | 3.003 J | 583.72 mJ | 398.15 mJ | 802.29 mJ |
| S3 | 11.640 J | 8.679 J | 1.707 J | 1.259 J | 2.433 J |
| S4 | 9.622 J | 6.662 J | 1.402 J | 953.77 mJ | 1.854 J |
| S5 | 65.944 J | 49.064 J | 9.960 J | 7.402 J | 14.041 J |
| S6 | 65.761 J | 48.881 J | 9.932 J | 7.374 J | 13.989 J |
| S7 | 151.964 J | 113.851 J | 22.996 J | 17.219 J | 32.634 J |
| S8 | 149.947 J | 111.834 J | 22.691 J | 16.914 J | 32.054 J |
| S9 | 1,397.611 J | 1,042.170 J | 212.305 J | 158.433 J | 299.471 J |
| S10 | 1,397.428 J | 1,041.987 J | 212.277 J | 158.406 J | 299.418 J |
| S11 | 5,395.719 J | 4,043.991 J | 818.589 J | 613.724 J | 1,161.214 J |
| S12 | 5,393.698 J | 4,041.970 J | 818.284 J | 613.419 J | 1,160.634 J |

Ratio to TFX ranges across all twelve scenarios: FatFs+FTL 109.5x-192.6x, FileX+LevelX/emFile 15.7x-28.1x, YAFFS2 30.5x-40.0x. One to two orders of magnitude worse than every other system in this suite, in every scenario.

---

## Suite 2: Parameters

Twelve scenarios testing parameter-store cost. Every scenario writes 75 parameters at boot, then reads 6 and writes 10 parameters every 15 minutes thereafter. What varies is the update pattern: scenarios cycle through 0%, 39%, 51%, and 71% "avoided-write" rates (a write that turns out to be a same-value no-op or a narrowing update that can be applied in place, rather than a full new write), at three duration tiers (5 days, 100 days, 10 years).

SPIFFS is excluded from this suite specifically: its page-pair floor (established in Suite 1) is already known to be a poor fit for records smaller than any message in Suite 1, and every parameter here is smaller still. SPIFFS is folded back in for the combined Suite 4 comparison. In its place, this suite adds a **ping-pong baseline** (the classic two-page vendor EEPROM-emulation pattern), the closest real-world structural relative of TFX's own scheme, with no same-value or narrowing-update capability of its own.

**Full results, all twelve scenarios (energy):**

| Scenario | TFX | Ping-pong | littlefs | NVS |
|---|---|---|---|---|
| S1 | 276.16 mJ | 136.44 mJ | 158.32 mJ | 505.65 mJ |
| S2 | 179.24 mJ | 136.44 mJ | 158.32 mJ | 505.65 mJ |
| S3 | 147.38 mJ | 136.44 mJ | 158.32 mJ | 505.65 mJ |
| S4 | 107.55 mJ | 136.44 mJ | 158.32 mJ | 505.65 mJ |
| S5 | 6.104 J | 3.310 J | 3.263 J | 10.097 J |
| S6 | 3.894 J | 3.310 J | 3.263 J | 10.097 J |
| S7 | 3.189 J | 3.310 J | 3.263 J | 10.097 J |
| S8 | 2.257 J | 3.310 J | 3.263 J | 10.097 J |
| S9 | 223.901 J | 121.933 J | 119.130 J | 368.530 J |
| S10 | 144.006 J | 121.933 J | 119.130 J | 368.530 J |
| S11 | 117.968 J | 121.933 J | 119.130 J | 368.530 J |
| S12 | 84.219 J | 121.933 J | 119.130 J | 368.530 J |

Ping-pong, littlefs, and NVS are all flat within each duration tier, since none of the three has any dedup capability of its own, avoided-write rate has zero effect on their cost. Only TFX's cost changes across the four scenarios within a tier.

**littlefs ÷ TFX, the direct measure of the dedup optimization's payoff:**

| Scenario group | Avoided-write rate | littlefs ÷ TFX |
|---|---|---|
| S1 / S5 / S9 | 0% | 0.57x / 0.53x / 0.53x, littlefs cheaper |
| S2 / S6 / S10 | 39% | 0.88x / 0.84x / 0.83x, littlefs cheaper |
| S3 / S7 / S11 | 51% | 1.07x / 1.02x / 1.01x, TFX cheaper |
| S4 / S8 / S12 | 71% | 1.47x / 1.45x / 1.41x, TFX cheaper |

TFX's dedup optimization has a real break-even point, consistently sitting between the 39% and 51% avoided-write tiers across all three durations tested. Below it, littlefs's unconditional-write simplicity wins; above it, TFX's optimization pays for itself. TFX also pays a real cost for its own split search-then-fetch API (two reads per access attempt, where a merged design would need one), which is why it doesn't win outright at low dedup rates the way its optimization might suggest it should.

NVS is the most expensive system in every one of the twelve scenarios, by 1.8-4.4x over TFX, for reasons unrelated to dedup, its fixed 32-byte entry against a 4-byte parameter payload is an 87.5% overhead by construction, and that structural mismatch dominates its cost regardless of update pattern.

### FAT-family and YAFFS2 (Suite 2)

| Scenario | FatFs+FTL (nobatch) | FatFs+FTL (deferred) | FileX+LevelX/emFile (nobatch) | FileX+LevelX/emFile (deferred) | YAFFS2 |
|---|---|---|---|---|---|
| S1 | 37.226 J | 20.464 J | 5.585 J | 3.045 J | 5.828 J |
| S2 | 37.226 J | 20.464 J | 5.585 J | 3.045 J | 5.828 J |
| S3 | 37.226 J | 20.464 J | 5.585 J | 3.045 J | 5.828 J |
| S4 | 37.226 J | 20.464 J | 5.585 J | 3.045 J | 5.828 J |
| S5 | 733.588 J | 403.710 J | 111.162 J | 61.166 J | 116.127 J |
| S6 | 733.588 J | 403.710 J | 111.162 J | 61.166 J | 116.127 J |
| S7 | 733.588 J | 403.710 J | 111.162 J | 61.166 J | 116.127 J |
| S8 | 733.588 J | 403.710 J | 111.162 J | 61.166 J | 116.127 J |
| S9 | 26,755.701 J | 14,725.171 J | 4,056.434 J | 2,233.086 J | 4,237.878 J |
| S10 | 26,755.701 J | 14,725.171 J | 4,056.434 J | 2,233.086 J | 4,237.878 J |
| S11 | 26,755.701 J | 14,725.171 J | 4,056.434 J | 2,233.086 J | 4,237.878 J |
| S12 | 26,755.701 J | 14,725.171 J | 4,056.434 J | 2,233.086 J | 4,237.878 J |

These four are flat within every duration tier for the same reason ping-pong/littlefs/NVS are: no dedup capability of their own. Ratio to TFX ranges from 10.0x (FileX+LevelX/emFile deferred, best case) to 346.1x (FatFs+FTL nobatch, worst case), and unlike every other comparison system in this suite, **the ratio widens, not narrows, as avoided-write rate rises**, since TFX's own cost keeps falling with dedup while these four stay flat. This is the sharpest illustration anywhere in this analysis of what having no dedup capability costs against a workload of small, frequent, individually-written records.

---

## Suite 3: System Log

Twelve scenarios testing the system log, structurally the simplest of the three subsystems (a pure append-only ring, no read-before-write and no consumption step). Reading-failure rate varies across the four scenarios within each of the three duration tiers (0% to 75% failure), since a failed reading logs three lines instead of one.

**A modeling note stated up front:** SPIFFS and NVS neither one has a native concept of "this is old, overwrite it" the way an append-only ring gets for free. Both are granted the same age-based circular-reclaim behavior TFX's own ring gets natively, so all four systems produce comparable totals. This is the single most consequential assumption in this suite, and it is generous to both SPIFFS and NVS relative to how either would actually behave unmodified; modeled faithfully, SPIFFS in particular would exhaust its real headroom and stop logging well before most of these scenarios complete.

**Full results, all twelve scenarios (energy):**

| Scenario | Days | NVS | TFX | littlefs | SPIFFS |
|---|---|---|---|---|---|
| S1 | 5 | 13.97 mJ | 14.51 mJ | 14.75 mJ | 28.43 mJ |
| S2 | 5 | 16.76 mJ | 17.40 mJ | 17.68 mJ | 63.94 mJ |
| S3 | 5 | 20.93 mJ | 21.76 mJ | 22.09 mJ | 139.61 mJ |
| S4 | 5 | 34.85 mJ | 36.27 mJ | 36.77 mJ | 388.08 mJ |
| S5 | 100 | 368.02 mJ | 513.78 mJ | 625.99 mJ | 4.851 J |
| S6 | 100 | 479.76 mJ | 657.70 mJ | 789.33 mJ | 5.860 J |
| S7 | 100 | 649.25 mJ | 871.68 mJ | 1.034 J | 7.374 J |
| S8 | 100 | 1.212 J | 1.584 J | 1.855 J | 12.415 J |
| S9 | 3,650 | 22.235 J | 27.662 J | 31.649 J | 185.839 J |
| S10 | 3,650 | 26.350 J | 32.857 J | 37.645 J | 222.671 J |
| S11 | 3,650 | 32.516 J | 40.652 J | 46.637 J | 277.918 J |
| S12 | 3,650 | 53.073 J | 66.632 J | 76.608 J | 462.071 J |

A clean, stable ranking in every scenario: NVS cheapest (3.7% cheaper than TFX at 5 days, 20-28% cheaper at 100 days and 10 years, once its log capacity is correctly scaled to its actual 256KB region), TFX second, littlefs third (1.4-22% more expensive than TFX, its own per-record tag/CRC framing outweighing the two rollover writes TFX's own ring pays that littlefs doesn't), SPIFFS a distant fourth throughout, and by S12 the single most expensive cell in the entire three-suite comparison (462 J) even with its generous circular-reclaim grant.

### FAT-family and YAFFS2 (Suite 3)

| Scenario | FatFs+FTL (nobatch) | FatFs+FTL (deferred) | FileX+LevelX/emFile (nobatch) | FileX+LevelX/emFile (deferred) | YAFFS2 |
|---|---|---|---|---|---|
| S1 | 3.693 J | 3.693 J | 320.84 mJ | 320.84 mJ | 820.33 mJ |
| S2 | 4.427 J | 4.062 J | 435.64 mJ | 380.16 mJ | 925.81 mJ |
| S3 | 5.531 J | 4.618 J | 602.31 mJ | 463.61 mJ | 1.084 J |
| S4 | 9.209 J | 6.468 J | 1.158 J | 741.69 mJ | 1.611 J |
| S5 | 73.727 J | 73.727 J | 10.975 J | 10.975 J | 20.911 J |
| S6 | 88.461 J | 81.138 J | 13.209 J | 12.099 J | 23.022 J |
| S7 | 110.562 J | 92.253 J | 16.558 J | 13.783 J | 26.187 J |
| S8 | 184.229 J | 129.302 J | 27.723 J | 19.398 J | 36.738 J |
| S9 | 2,691.045 J | 2,691.045 J | 409.248 J | 409.248 J | 771.906 J |
| S10 | 3,228.865 J | 2,961.527 J | 490.760 J | 450.242 J | 848.928 J |
| S11 | 4,035.598 J | 3,367.253 J | 613.030 J | 511.735 J | 964.461 J |
| S12 | 6,724.703 J | 4,719.667 J | 1,020.591 J | 716.707 J | 1,349.571 J |

Ratio to TFX ranges from 10.8x (FileX+LevelX/emFile deferred, best case) to 254.5x (FatFs+FTL, worst case). Unlike SPIFFS and NVS, none of these four systems needed a favorable modeling assumption to land in last place here, their cost comes purely from documented block/chunk granularity against a 48-byte log record.

---

## Suite 4: Combined (Full Device)

Twelve scenarios combining messages, parameters, and the system log on one device sharing a single wake cycle, plus a periodic restart, the closest approximation in this analysis to how the device actually behaves in the field. Every comparison filesystem (littlefs, SPIFFS, NVS) is run two ways: **Run A**, three dedicated regions sized exactly as Suites 1-3 sized them (matching how TFX itself is built), and **Run B**, one pooled region hosting all three logical streams together (the way a team choosing "just use one filesystem for everything" would actually deploy it). TFX has no Run B, its three subsystems are separate hand-rolled structures by design, not a general filesystem that could plausibly be pooled.

**Full results, all twelve scenarios (energy), TFX baseline vs. littlefs (architecturally identical between Run A and Run B) vs. SPIFFS and NVS (both runs):**

| Scenario | Days | TFX | littlefs (A=B) | SPIFFS A | SPIFFS B | NVS A | NVS B |
|---|---|---|---|---|---|---|---|
| S1 | 13 | 224.35 mJ | 147.65 mJ | 4.608 J | 2.067 J | 334.72 mJ | 272.37 mJ |
| S2 | 13 | 163.96 mJ | 149.65 mJ | 4.654 J | 2.113 J | 337.37 mJ | 275.02 mJ |
| S3 | 8 | 354.29 mJ | 386.61 mJ | 11.905 J | 5.664 J | 866.32 mJ | 670.72 mJ |
| S4 | 8 | 321.91 mJ | 411.19 mJ | 12.461 J | 6.222 J | 897.99 mJ | 728.52 mJ |
| S5 | 177 | 3.596 J | 2.485 J | 66.486 J | 31.895 J | 5.455 J | 5.356 J |
| S6 | 177 | 2.670 J | 2.559 J | 66.948 J | 32.359 J | 5.506 J | 5.408 J |
| S7 | 103 | 5.741 J | 6.227 J | 157.147 J | 76.647 J | 13.094 J | 12.878 J |
| S8 | 103 | 5.536 J | 7.087 J | 162.493 J | 82.078 J | 13.689 J | 13.479 J |
| S9 | 3,727 | 80.838 J | 57.259 J | 1,406 J | 677.541 J | 122.019 J | 120.008 J |
| S10 | 3,727 | 61.731 J | 58.789 J | 1,416 J | 687.113 J | 123.073 J | 121.064 J |
| S11 | 3,653 | 214.220 J | 229.850 J | 5,585 J | 2,730 J | 477.236 J | 469.573 J |
| S12 | 3,653 | 206.469 J | 259.865 J | 5,770 J | 2,916 J | 497.826 J | 490.243 J |

**Ratio to TFX:**

| Scenario | littlefs | SPIFFS A | SPIFFS B | NVS A | NVS B |
|---|---|---|---|---|---|
| S1 | 0.66x | 20.54x | 9.21x | 1.49x | 1.21x |
| S2 | 0.91x | 28.39x | 12.89x | 2.06x | 1.68x |
| S3 | 1.09x | 33.60x | 15.99x | 2.45x | 1.89x |
| S4 | 1.28x | 38.71x | 19.33x | 2.79x | 2.26x |
| S5 | 0.69x | 18.49x | 8.87x | 1.52x | 1.49x |
| S6 | 0.96x | 25.07x | 12.12x | 2.06x | 2.03x |
| S7 | 1.08x | 27.37x | 13.35x | 2.28x | 2.24x |
| S8 | 1.28x | 29.35x | 14.82x | 2.47x | 2.43x |
| S9 | 0.71x | 17.39x | 8.38x | 1.51x | 1.48x |
| S10 | 0.95x | 22.93x | 11.13x | 1.99x | 1.96x |
| S11 | 1.07x | 26.07x | 12.74x | 2.23x | 2.19x |
| S12 | 1.26x | 27.94x | 14.12x | 2.41x | 2.37x |

littlefs splits cleanly along wake cadence, not buffer size or dedup rate: it beats TFX in exactly the six hourly-cadence scenarios (0.66-0.96x) and loses in exactly the six 15-minute-cadence scenarios (1.07-1.28x). At the faster cadence, littlefs's own smaller per-region capacity gets exercised four times as often, and its per-record framing overhead compounds into more frequent erasing than TFX's own ring structures.

**Pooling's effect on SPIFFS and NVS (Run A to Run B):**

| Scenario | SPIFFS A | SPIFFS B | Change | NVS A | NVS B | Change |
|---|---|---|---|---|---|---|
| S1 | 4.608 J | 2.067 J | -55.1% | 334.72 mJ | 272.37 mJ | -18.6% |
| S2 | 4.654 J | 2.113 J | -54.6% | 337.37 mJ | 275.02 mJ | -18.5% |
| S3 | 11.905 J | 5.664 J | -52.4% | 866.32 mJ | 670.72 mJ | -22.6% |
| S4 | 12.461 J | 6.222 J | -50.1% | 897.99 mJ | 728.52 mJ | -18.9% |
| S5 | 66.486 J | 31.895 J | -52.0% | 5.455 J | 5.356 J | -1.8% |
| S6 | 66.948 J | 32.359 J | -51.7% | 5.506 J | 5.408 J | -1.8% |
| S7 | 157.147 J | 76.647 J | -51.2% | 13.094 J | 12.878 J | -1.6% |
| S8 | 162.493 J | 82.078 J | -49.5% | 13.689 J | 13.479 J | -1.5% |
| S9 | 1,406 J | 677.541 J | -51.8% | 122.019 J | 120.008 J | -1.6% |
| S10 | 1,416 J | 687.113 J | -51.5% | 123.073 J | 121.064 J | -1.6% |
| S11 | 5,585 J | 2,730 J | -51.1% | 477.236 J | 469.573 J | -1.6% |
| S12 | 5,770 J | 2,916 J | -49.5% | 497.826 J | 490.243 J | -1.5% |

Pooling helps both systems substantially, not because sharing space is inherently better, but because each system's dedicated parameter capacity (128 objects for SPIFFS, 2,016 entries for NVS) was drastically undersized relative to how often parameters actually get written. Pooling gives parameters access to the full combined capacity instead of a small dedicated slice. littlefs is architecturally unaffected by pooling (Run A = Run B exactly), since its reclaim is scoped per-file internally regardless of whether files share a region.

Even pooled, no comparison filesystem approaches TFX except littlefs in its six winning scenarios: SPIFFS remains 8.4-19.3x more expensive even pooled, NVS remains 1.2-2.5x more expensive even pooled.

**The article's headline table** (energy-first sum across S9-S12, the combined 10-year device-wide picture, using Run B for littlefs/SPIFFS/NVS):

| System | Total energy (S9-S12) | % of TFX |
|---|---|---|
| TFX | 563.258 J | 100.0% |
| littlefs | 605.763 J | 107.5% |
| NVS | 1,200.888 J | 213.2% |
| SPIFFS | 7,010.654 J | 1,244.7% |

### FAT-family and YAFFS2 (Suite 4, Run A only)

A pooled Run B was not modeled for these four systems, this addition is scoped to Run A (dedicated regions) only, consistent with TFX's own architecture.

| Scenario | FatFs+FTL (nobatch) | FatFs+FTL (deferred) | FileX+LevelX/emFile (nobatch) | FileX+LevelX/emFile (deferred) | YAFFS2 |
|---|---|---|---|---|---|
| S1 | 34.189 J | 20.483 J | 4.828 J | 2.754 J | 5.525 J |
| S2 | 34.668 J | 20.633 J | 4.901 J | 2.778 J | 5.569 J |
| S3 | 85.817 J | 51.703 J | 12.659 J | 7.484 J | 14.495 J |
| S4 | 91.745 J | 53.683 J | 13.557 J | 7.785 J | 15.054 J |
| S5 | 458.021 J | 277.899 J | 69.193 J | 41.895 J | 79.578 J |
| S6 | 464.544 J | 281.090 J | 70.183 J | 42.380 J | 80.488 J |
| S7 | 1,098.260 J | 663.329 J | 166.194 J | 100.277 J | 190.226 J |
| S8 | 1,174.174 J | 700.504 J | 177.701 J | 105.912 J | 200.808 J |
| S9 | 9,632.410 J | 5,850.004 J | 1,462.411 J | 889.151 J | 1,682.497 J |
| S10 | 9,769.699 J | 5,918.957 J | 1,483.220 J | 899.600 J | 1,702.131 J |
| S11 | 38,931.847 J | 23,518.998 J | 5,904.061 J | 3,568.092 J | 6,757.290 J |
| S12 | 41,623.199 J | 24,871.539 J | 6,311.965 J | 3,773.082 J | 7,142.431 J |

**Ratio to TFX:**

| Scenario | FatFs+FTL (nobatch) | FatFs+FTL (deferred) | FileX+LevelX/emFile (nobatch) | FileX+LevelX/emFile (deferred) | YAFFS2 |
|---|---|---|---|---|---|
| S1 | 152.4x | 91.3x | 21.5x | 12.3x | 24.6x |
| S2 | 211.4x | 125.8x | 29.9x | 16.9x | 34.0x |
| S3 | 242.2x | 145.9x | 35.7x | 21.1x | 40.9x |
| S4 | 285.0x | 166.8x | 42.1x | 24.2x | 46.8x |
| S5 | 127.4x | 77.3x | 19.2x | 11.7x | 22.1x |
| S6 | 174.0x | 105.3x | 26.3x | 15.9x | 30.1x |
| S7 | 191.3x | 115.5x | 28.9x | 17.5x | 33.1x |
| S8 | 212.1x | 126.5x | 32.1x | 19.1x | 36.3x |
| S9 | 119.2x | 72.4x | 18.1x | 11.0x | 20.8x |
| S10 | 158.3x | 95.9x | 24.0x | 14.6x | 27.6x |
| S11 | 181.7x | 109.8x | 27.6x | 16.7x | 31.5x |
| S12 | 201.6x | 120.5x | 30.6x | 18.3x | 34.6x |

Taking the true minimum and maximum across all twelve scenarios and both sync variants: **FatFs+FTL ranges 72.4x-285.0x**, **ThreadX FileX+LevelX/SEGGER emFile ranges 11.0x-42.1x**, **YAFFS2 ranges 20.8x-46.8x**. This device-wide total compounds all three subsystems' own mismatches at once (24-byte messages, 6-byte parameters, 48-byte log entries, all against a 512B-1KB write floor), which is why these combined-suite ratios run higher than any single standalone suite's ratios for the same systems, and higher than SPIFFS's own worst case in this suite (38.71x, Run A).

---

## Assumptions and known simplifications

- **No write batching anywhere.** Every write, on every system in every suite, is modeled as its own isolated, immediate flash transaction. This is the only calling convention established anywhere in TFX's own source (a single-variable write call with no documented batch path), and it's applied consistently rather than assuming batching for some systems and not others. A firmware team that deliberately batched writes when porting to littlefs, for instance, would recover a meaningful part of littlefs's disadvantage at faster wake cadences.
- **SPIFFS and NVS's log-suite reclaim is modeled as an idealized age-based circular reclaim**, which neither system actually implements natively. This is the single most consequential assumption in Suite 3, and it's generous to both systems relative to their real, unmodified behavior.
- **The periodic restart cost uses one conservative, system-agnostic model** (a full page-level read-back of every region a system uses) rather than each system's own possibly cheaper real mount algorithm; this is very likely conservative for NVS in particular, which in reality reads only small per-page state headers on mount.
- **Suite 4's Run B pooling model for SPIFFS uses a stated conservative lower bound** for the three scenarios with no discard valve (S4/S8/S12); a full event-ordered simulation of pooled reclaim under those conditions was not attempted.
- **The FAT-family and YAFFS2 addition (all four suites) is modeled on dedicated storage regions only** (Suite 4's Run A), consistent with TFX's own architecture; a pooled configuration for these four systems was not modeled.
- **Ten-year duration** uses a flat 3,650-day convention in Suites 1 and 3, and the exact composite scenario timeline (3,727 / 3,653 days) in Suite 4, per that suite's own scenario definitions.
