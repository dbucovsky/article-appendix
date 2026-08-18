# Fit, Timing, Control: Rethinking a Storage Decision

Supporting repository for the article of the same name. A decade-later retrospective on a custom NOR EEPROM storage layer built in 2016 for an industrial IoT product line, TFX, re-examined against littlefs, SPIFFS, ESP-IDF NVS, and four additional embedded storage systems using real measured energy data and public defect histories.

## Contents

- **[fit-timing-control-article.md](./fit-timing-control-article.md)**
  The article itself.

- **[embedded-fs-reference-brief.md](./embedded-fs-reference-brief.md)**
  History and architecture of every comparison system: littlefs, SPIFFS, ESP-IDF NVS, FatFs+FTL, ThreadX FileX+LevelX, YAFFS2, and SEGGER emFile.

- **[generalist-fs-defect-history.md](./generalist-fs-defect-history.md)**
  Known field defects and disclosed vulnerabilities across every comparison system, sourced from each project's public issue tracker.

- **[performance-analysis.md](./performance-analysis.md)**
  Full methodology and complete scenario-by-scenario energy results behind the article's numbers.

## Note on sourcing

All figures in this repository use TFX's baseline, as-shipped storage layer. Bugs and vulnerabilities in TFX's own code are out of scope; this is a power/energy efficiency and public-defect-history comparison only. Some names and specifics are generalized throughout to protect IP; the technical substance isn't.
