# Fit, Timing, Control: Rethinking a Storage Decision

*By Damian Bucovsky, President at The Shadow on the Moon*

Every engineering team that owns its own data storage eventually runs into the same three questions, whether they ever name them or not. Does this specific problem deserve a specific solution, or would something built for a wider audience actually serve it well enough? Does that answer hold up over time, given how quickly the available tools change? And when the chosen approach eventually fails, because everything eventually does, who is actually in a position to fix it? Most teams answer these questions once, quietly, in the middle of a design cycle, and move on. This article revisits the answers my team gave in 2016, using real measurements gathered a decade later.

The difference this piece explores is a difference in what each approach was built for, not a difference in quality. That distinction shapes everything that follows.

## The Specific Case

In the mid-to-late 2010s, I led the design of the firmware for what I'll call TFX here, a fielded industrial IoT product line built around a custom board and a specific microcontroller, running unmanned for a minimum of ten years, under tight cost and power budgets. TFX went on to cover a full family of variants, differing by sensor, hazard classification, and telematics platform, all sharing that same core set of constraints. (Some names and specifics are generalized in this piece to protect IP; the technical substance isn't.)

Buried inside TFX is a small, unglamorous piece of firmware: the code that manages how parameters, messages, and system logs get written to and read from NOR EEPROM. This article is about that storage layer specifically, not TFX as a whole. It was built entirely custom rather than adopting an existing embedded flash filesystem, open-source or otherwise, and that decision is the one being revisited here.

This isn't a guide for what you should do. It's a retrospective: what did building it ourselves actually cost and earn, now that it can be measured and compared against what the wider field ultimately converged on? Three threads run through that comparison, matching the three questions above: **fit** (whether the problem actually needed a specific solution), **timing** (whether that answer has held up as the available tools matured), and **control** (what it has actually meant, in practice, to own every line of this rather than depend on someone else's).

> **This isn't a guide for what you should do. It's a retrospective: what did building it ourselves actually cost and earn, now that it can be measured and compared against what the wider field ultimately converged on?**

To check the original decision rather than just recall it fondly, this piece draws on a twelve-scenario energy analysis comparing TFX's storage layer against three widely used embedded flash filesystems (littlefs, SPIFFS, and ESP-IDF NVS), plus a survey of each system's public defect history. The [full methodology and complete scenario-by-scenario results](https://github.com/dbucovsky/article-appendix/blob/main/fit-timing-control/performance-analysis.md) are linked for anyone who wants to go deeper; what follows here is the part that matters for the argument.

## Filesystem Maturity and Risk, at a Glance

Before getting into performance numbers, here's how these four approaches compare on age, stability, and track record. That picture sets up both the timing and control threads that follow.

| System | First released | Known for | Maturity today |
|---|---|---|---|
| ThreadX FileX + LevelX | 1997 | Safety-certified; tied to adopting the ThreadX RTOS itself | Active, Eclipse Foundation, MIT-licensed since 2023 |
| SEGGER emFile | Late 1990s (est.) | Fail-safety is a separately licensed add-on, not the default | Active, commercial; no public issue tracker |
| YAFFS2 | 2001-2002 | Long-lived, open concurrency and correctness bugs | Active, small-team maintained |
| FatFs + FTL | 2006 | Seven CVEs disclosed 2026, six still unpatched | Active (single maintainer); FTL varies by vendor |
| SPIFFS | ~2013 | Confirmed wear-leveling and corruption defects | Deprecated, unmaintained upstream |
| TFX storage layer (custom) | Designed 2016 | Internal review only, not public | Fielded, stable, single-owner |
| ESP-IDF NVS | Dec 2016 | Silent-erase and silent-non-persistence reports | Active, Espressif-owned |
| littlefs | 2017 | Ongoing corruption and bricking reports, even recently | Active, ARM-owned, ~6.4k GitHub stars |

*(Full [version history](https://github.com/dbucovsky/article-appendix/blob/main/fit-timing-control/embedded-fs-reference-brief.md) and [issue-level defect detail](https://github.com/dbucovsky/article-appendix/blob/main/fit-timing-control/generalist-fs-defect-history.md) for every system are linked below.)*

Of the systems in this comparison, littlefs and NVS were not real options in 2016; neither one existed in any usable form yet. SPIFFS was different: it existed, was already in real production use, and was a genuinely normal option at the time, not an immature or fringe one. Its absence from serious consideration in 2016 is a question of fit, not timing, and that distinction matters for how the rest of this piece reads. FatFs, ThreadX FileX+LevelX, YAFFS2, and emFile were mature options too, each with its own strings attached (a variable, unstandardized FTL for FatFs, a commercial RTOS dependency for FileX, licensing and support models that looked very different from an internal engineering decision), but the real reason none of them were seriously evaluated for performance at the time is that all four are built around block- or chunk-sized writes meant for large, sequential files. That mismatch against small, frequent records was obvious from how they operate, before any benchmark was run, and it's exactly what the data confirms now: see the numbers under Fit below.

## Fit: Specialization vs. Generality

There would have been no decision to make at all if generalist embedded flash filesystems were actually specific to problems like this one. They aren't, and that's not a flaw, it's a consequence of the job they're built to do. Generalist embedded flash filesystems succeed by serving a wide, unknown market. That's the job, and it requires breadth: unknown file counts, unknown access patterns, unknown concurrency, all defended against up front because the next person to adopt the library might need any of it.

TFX's storage layer had a different job. It grew to serve an entire product line, several fielded variants differing in sensor, hazard classification, and telematics platform, but every variant shared the same core constraints and the same known, bounded data shape: parameters that were almost always four bytes or smaller, with a handled exception path for the rare larger value or string, and messages and system log entries following a consistent structure. This storage layer never had any external users at all. TFX is an industrial product: customers buy finished hardware, usually in volume, not firmware, and the code inside is closed, invisible to anyone who buys the device. Its only effect a customer could ever perceive is indirect: what it costs, how long the battery lasts, whether the thing works. A published filesystem is the opposite kind of thing entirely. Its actual "market" is every developer who might adopt it into their own project, for their own unknown data, which is exactly the open-ended range of needs TFX's storage layer never had to serve. That's the real comparison this piece is making: firmware built around one team's own known, specific requirements against filesystems built for external adoption into projects nobody involved in writing them will ever see. It's a market-shape difference, not a quality difference, and that framing matters more than it might first appear, since it's easy to read a performance comparison as a verdict on engineering quality when it's really a verdict on fit.

TFX's storage layer never had to serve an unknown audience, so it got to assume things a generalist library can't: a bounded ID set, a known access pattern, one thread at a time, no concurrency guards needed. It's also worth being precise about what "TFX's storage layer" actually is. It isn't one general-purpose store that happens to be custom, it's three separate, independently specialized subsystems: a parameter store, a message buffer, and a system log, each shaped around its own access pattern rather than one layer trying to serve all three adequately. A single embedded flash filesystem underneath all three would have had to serve the worst-fit case for all of them at once. Building separately let each piece be shaped to exactly what it needed and nothing else.

Two concrete tricks show where that specificity actually paid for itself. Parameter records are packed six bytes at a time, two bytes of ID plus four bytes of value, and adjacent records share space within the same 32-bit NOR words using AND-only programming: writing a new record only ever clears bits, never sets them, so two records can occupy the same word without an erase cycle as long as their bit patterns don't collide. And when a parameter update's new value is a strict subset of the old one's bits (formally, `old AND new == old`), the update happens in place, in that same word, with no new slot and no erase at all. Neither trick is available to a generalist filesystem, because neither one holds in general. A filesystem has to assume the next write could be anything.

This wasn't guesswork. The requirement, from the outset, was a fixed, well-understood set of data with no major structural changes expected over the product's ten-year field life, running unmanned, under a tight cost and power budget. This project never had to deal with either of the things a generalist layer is built to handle: change, or the unknown. That's also worth stating plainly in the other direction: generalist embedded flash filesystems are genuinely excellent at the job they're built for, broad adoption across products whose needs aren't known yet. TFX's storage layer was built for a narrower, different job. Neither one has to be wrong for the other to be right, and if a mature, sufficiently efficient generalist option had existed in 2016, it likely would have been used instead of building custom (more on that in the next section).

> **Generalist embedded flash filesystems are genuinely excellent at the job they're built for: broad adoption across products whose needs aren't known yet.**

Fit and generality don't really coexist, not unless you build the thing yourself. NVS's 15-character key cap and SPIFFS's two-page floor per object aren't oversights, they're artifacts of designing for a broad, unpredictable audience rather than any one deployment's actual shape. And the reverse holds too: a "generalist" tool built in-house rarely stays general in practice, your own constraints leak into it whether you intend that or not. The real decision was never "custom versus generalist" in the abstract. It was whether the product needed efficiency now or malleability for undetermined future growth, and being honest with yourself about which one you actually need is most of the work.

> **Fit and generality don't really coexist, not unless you build the thing yourself.**

That efficiency argument is not unconditional, and it shouldn't be presented as if it were. The parameter store's write-avoidance logic, which skips a write entirely if the incoming value matches what's already stored, only pays for itself above roughly a 40-50% avoided-write rate. Below that threshold, TFX's own two-step search-then-fetch API costs more than littlefs's simpler, unconditional-write model, and littlefs wins the isolated parameter comparison outright. That's not a weak spot to hide, it's the credibility the rest of this argument depends on: the framework works in both directions, not just as an advertisement for building your own.

### The Numbers

Energy comparisons across this whole piece are computed one specific way: total joules are summed across scenarios first, and only then converted to a percentage. Averaging the percentage ratios directly, which is the more obvious way to do it, gives a materially different and, on inspection, misleading answer, because it lets a handful of low-energy scenarios carry the same weight as the highest-energy ones. Every table below uses the energy-first method.

**Isolated, short duration (avg per scenario, energy-first), mJ (% of TFX)**

| System | Messages | Parameters | Log |
|---|---|---|---|
| TFX (storage layer) | 🟢 41.92 mJ (100.0%) | 177.58 mJ (100.0%) | 22.49 mJ (100.0%) |
| littlefs | 44.65 mJ (106.5%) | 🟢 158.32 mJ (89.2%) | 22.82 mJ (101.5%) |
| NVS | 104.60 mJ (249.5%) | 505.65 mJ (284.7%) | 🟢 21.63 mJ (96.2%) |
| SPIFFS | 266.18 mJ (635.1%) | n/a | 155.02 mJ (689.3%) |

**Isolated, long duration (~10 years, avg per scenario, energy-first), J (% of TFX)**

| System | Messages | Parameters | Log |
|---|---|---|---|
| TFX (storage layer) | 🟢 18.625 J (100.0%) | 142.524 J (100.0%) | 41.951 J (100.0%) |
| littlefs | 21.380 J (114.8%) | 🟢 119.130 J (83.6%) | 48.135 J (114.7%) |
| NVS | 60.652 J (325.7%) | 368.530 J (258.6%) | 🟢 33.544 J (80.0%) |
| SPIFFS | 130.839 J (702.4%) | n/a | 287.125 J (684.4%) |

**Combined, pooled, long duration (the actual full device, ~10 years), J total (% of TFX)**

| System | Total energy | % of TFX |
|---|---|---|
| TFX (storage layer) | 🟢 563.258 J | 100.0% |
| littlefs | 605.763 J | 107.5% |
| NVS | 1,200.888 J | 213.2% |
| SPIFFS | 7,010.654 J | 1,244.7% |

That last table is the real headline. Over a full ten-year device lifetime, with all three subsystems drawing on shared flash capacity the way they actually do in the field, littlefs comes within 7.5% of TFX's own storage layer. NVS costs a bit over double. SPIFFS costs roughly 12.4 times as much.

### A Brief Note on FAT-Based and NAND-Native Options

Four more systems, FatFs with a flash translation layer, ThreadX FileX with LevelX, SEGGER's emFile, and YAFFS2, were tested for completeness. None were seriously evaluated back in 2016, since the fit problem (block- or chunk-sized writes against 6-48 byte records) was obvious from their architecture alone. All four land one to two orders of magnitude worse than TFX's storage layer, worse than SPIFFS. See the appendix for the numbers.

## Timing: Maturity and the Cost of Transition

Fit isn't the whole story, and the calendar matters more than it might seem. Of the three main comparison systems, littlefs didn't exist at all in 2016, its repository wasn't created until early 2017. NVS was first published in December 2016, essentially the same month TFX's design window closed, with no track record, no field history, and tied to a different microcontroller family than the one TFX was already committed to. Neither was a real option, and that's a straightforward timing fact rather than a judgment call. SPIFFS is a different case, and worth being precise about, since it existed, was already in real production use, and was a genuinely normal option at the time. SPIFFS wasn't skipped because it wasn't ready. It was skipped because it didn't fit the needs, and that belongs to the fit argument in the previous section, not to timing. Collapsing all three comparison systems into "nothing was available yet" would overstate the timing argument and understate the fit one.

That timing fact cuts in an interesting direction when you look at it from today. littlefs is the only comparison system that gets close to TFX's storage layer on the pooled long-term numbers, 107.5%, and actually cheaper than TFX in the hourly-cadence scenarios. Had this project started a year or two later, once littlefs cleared its 2019-2020 breakout-adoption period, the build-versus-buy call might reasonably have gone the other way. Worth saying plainly rather than leaving implied: made today, or any time after roughly 2020, this decision would probably favor adopting littlefs over building custom, given how close the energy numbers land and the added engineering cost of building your own. That's not a reversal of the previous section's argument. It's the same argument with a second variable added: fit determines whether specialization is worth pursuing at all, and timing determines whether a sufficiently mature generalist option exists yet to make pursuing it unnecessary.

> **The calendar, not the code, is what would have changed the outcome.** Had this project started a year or two later, once littlefs cleared its 2019-2020 breakout-adoption period, the build-versus-buy call might reasonably have gone the other way.

It's worth resisting a tempting shortcut here, though: adopting a generalist embedded flash filesystem doesn't automatically mean de-risking, and treating "off-the-shelf" as a synonym for "safe" is its own kind of mistake. A lot of modern software, across the industry generally, ships release-then-patch rather than test-then-release. Adopting an immature dependency can carry more risk than building your own, not less, and "it's maintained by someone else" is not the same claim as "it's been proven under load."

Once a product is actually fielded, the door narrows further, and this constraint holds regardless of whether a better option exists. With units already deployed and running unattended for a ten-year lifespan, any change to the storage layer has to be fully backward-compatible with data already sitting on flash. That's true even for a filesystem that did exist and was already mature at the time; a mature but incompatible replacement is not a real option once real devices are in the field with real data on them.

There's a second, less obvious cost buried in that same constraint, and it shows up far more often than switching filesystems entirely. Even evaluating a single bug fix in someone else's storage code is genuinely hard: reading the change, understanding the exact failure mode it closes, and reasoning honestly about whether your own usage pattern was ever exposed to it takes real effort, on code you didn't write. Now multiply that by a release that bundles several fixes at once. At that point, few teams actually do the work several times over, and taking the upgrade stops being an act of engineering and becomes an act of faith in the maintainer's overall diligence. Swapping one third-party system for another compounds the same problem further, since now it's two sets of unwritten assumptions instead of one. At least when you build it yourself, one side of that equation is something you actually understand.

> **Taking the upgrade stops being an act of engineering and becomes an act of faith.**

## Control: Debuggability When Things Fail in the Field

Every system compared in this piece has real defects, including TFX's own storage layer. That's worth stating plainly before going any further, because it would be easy to read a section about field failures as "ours is clean, theirs isn't," and that isn't the argument. TFX's parameter store has a non-atomic append-and-invalidate window; its message buffer wipes an entire region rather than a single record when it detects corruption. Those are documented, internal trade-offs, not secrets, and they sit in the same failure category as everything below: torn writes, corruption on power events, silent data loss. Every one of the systems in this comparison, including the custom one, has real exposure here.

What differs isn't the presence of defects. It's who controls the fix. A defect in TFX's own roughly two thousand lines of storage code is something that can be read, root-caused, and patched on whatever schedule the team decides. A defect in someone else's dependency puts the timeline, and sometimes the outcome, in someone else's hands entirely.

> **What differs isn't the presence of defects. It's who controls the fix.**

Espressif marked an NVS issue where `nvs_erase_all()` reports success without actually erasing the underlying flash bytes, meaning "deleted" data remains recoverable from a raw flash dump, as "Won't Do." A reporter on littlefs's own tracker described repeated field corruption and bricked devices from an ordinary file-creation workload, "idling in an assert," and was left without a resolution. This isn't a criticism of the people maintaining littlefs, SPIFFS, or NVS. It's a structural reality of depending on any external project with its own priorities, its own resourcing, and its own timeline, however well-run that project is. The point is about where control sits, not about competence.

Field failures in third-party code also tend to be stranger than the ones in your own. For a lot of deployed devices, field debugging after the fact is difficult or outright impossible, there's no console, no easy way to reproduce a corruption pattern that took months of real-world power cycling to surface. Third-party corruption in this comparison shows up again and again as exactly that kind of phantom failure: no clear trigger, no reliable reproduction, just a device that stopped working correctly somewhere in the field.

It's worth being concrete about what recovering from that actually looks like, because it's rarely fast and rarely clean. Root-causing a field failure depends on a customer who is already frustrated, often already losing money because of the outage, being willing to schedule a technician, on their own timeline rather than anyone else's, to physically reach the device, recover it, and replace it, frequently at a cost that exceeds the price of the device itself. The failed unit then makes its way back to the engineering team, if it makes it back at all, weeks or months after the failure, sometimes never. Every day in between is a day the device wasn't operating correctly. And by the time the EEPROM actually reaches someone who can read it, it has often been erased or overwritten by normal operation in the meantime, quietly destroying the exact evidence that would explain what went wrong in the first place. Code you wrote yourself, having been exercised and understood far more thoroughly before it ever shipped, tends to fail in ways that can actually be traced back to a cause.

And even when a third party does fix something, the picture you get is rarely complete. Most maintainers do disclose real detail, this isn't a claim that they're hiding anything, but a fix rarely arrives alone. Releases routinely carry small additional changes alongside the actual fix, some undocumented, that can affect what you need to update in your own integration without ever telling you that you need to. You can read every line of the changelog and still miss the one behavioral shift that happens to matter for your specific usage pattern, because it was never framed as something that would.

That risk gets sharper for any device capable of over-the-air updates. An OTA-capable fleet means upgrading in a semi-blind fashion: you find out whether the update actually worked only after it's already done, when the device either comes back online or doesn't. For units that were already running the buggy version but hadn't tripped the failure condition yet, quietly working fine in the field, that's real exposure, not a hypothetical one. A fix meant to protect a handful of units that already failed can end up destabilizing a fleet that was working. That's not just an engineering risk, it's a fast way to lose a customer's trust entirely: a handful of units fail, you ship the fix, and half the fleet goes down at once.

Without OTA capability, the update itself is far more controlled, there's no semi-blind mass push to worry about, but the logistics that replace it are their own kind of expensive. Applying a fix means physically recovering assets, including the ones that never failed, installing a replacement device in the field so the customer stays operational, and only later getting the original unit back to actually perform the upgrade before it can be recirculated as a spare. For any meaningful fleet size, that cycle runs weeks to months, and it costs real money at every step: the truck roll, the temporary replacement hardware, the labor, twice, once to swap and once to swap back. Neither path is free. OTA trades that cost for a different kind of risk; no OTA trades the risk for the cost.

Neither of these risks disappears with your own code, but they change shape. You're not depending on someone else's changelog to tell you what actually shifted, and you're not upgrading blind against assumptions you didn't write.

**A few concrete examples, drawn from the comparison systems' public issue trackers:**

| System | Example |
|---|---|
| ESP-IDF NVS | `nvs_erase_all()` reports success without physically erasing the data; marked "Won't Do." |
| littlefs | Field-reported directory corruption and bricked devices from ordinary file creation, left unresolved. |
| SPIFFS | A confirmed wear-leveling defect where fully-written blocks never participate in wear leveling at all. |
| FatFs | Seven CVEs disclosed in 2026; six remain unpatched, and the maintainer hasn't responded to coordinated disclosure. |

*(Full sourcing, issue numbers, and additional examples for every system in this comparison are in the [complete defect history](https://github.com/dbucovsky/article-appendix/blob/main/fit-timing-control/generalist-fs-defect-history.md).)*

## Conclusion

Every team that owns its own data storage eventually faces three questions, whether they name them or not. Does this specific problem deserve a specific solution? Does that answer hold up as the available tools evolve? And who is actually in control when the chosen approach eventually fails?

Most teams never really get to the first question. They reach for whatever generalist option is already available, evaluate it fairly within that limited frame, and, where it doesn't quite fit, adjust their own requirements to match the tool rather than the other way around. That's often the right call, for entirely legitimate reasons: real resource constraints, no appetite for funding an engineering effort upfront, or an honest read that a team without deep flash experience isn't going to out-engineer a project with years of field hardening behind it. But sometimes it's something else wearing pragmatism's clothes: if I didn't write it, I'm not the one who gets blamed when it fails. Either way, the question of whether the problem actually deserved a specific solution never really gets asked.

The opposite failure is less common, and it's the one this piece has spent most of its time examining: a team that does ask the first question, answers it honestly, and then stops there. Fit gets taken seriously, and building custom looks like the right call, but the second and third questions never come up, even when a mature, well-supported alternative already exists or will soon enough. That team ends up building something of its own that a suitable, even excellent, generalist option would have handled just as well. Both failures come from the same place: treating one question as the whole decision.

I've been guilty of the second failure more than once myself: assuming, without really checking, that nothing on the market would fit, and building my own anyway on the strength of that assumption alone. It's a familiar enough pattern in this field to have its own name, not-invented-here syndrome, and knowing the name doesn't make you immune to it.

TFX's storage layer answers the first question clearly. The fit was real: parameters almost always four bytes or smaller, messages and log entries following a consistent structure, a single-threaded execution model, and no external adopters to design around, because this code was never going anywhere beyond the devices it shipped in. Given that, specialization paid for itself, and the two flash-physics tricks at the center of it, sub-word packing and in-place narrowing updates, are the direct evidence of that payoff. None of that makes littlefs, SPIFFS, or NVS worse tools. Generalist filesystems aren't worse. They're built for a different job.

> **Generalist filesystems aren't worse. They're built for a different job.**

The second question is where this piece earns its keep as a retrospective rather than a defense. Made today, or any time after roughly 2020 once littlefs cleared its own breakout-adoption period, this decision would probably come out differently. The pooled long-term number, 107.5% of TFX's own cost, is close enough that the engineering investment of building custom likely wouldn't be worth it now. That doesn't undercut the original 2016 decision. It confirms the actual argument: the decision was correct for its moment, and moments expire. The right call can flip entirely because the calendar moved, without the underlying problem changing at all.

The third question doesn't resolve as cleanly, and it isn't supposed to. Every system in this comparison has real defects, TFX's own storage layer included. What differs is who's holding the timeline when one surfaces, and that's not something a benchmark can fully capture. It's a bet on where you'd rather the risk sit.

A few things this piece isn't saying, worth stating directly rather than leaving to inference: this isn't an argument that generalist filesystems are worse or poorly engineered, they're excellent at the job they're actually built for. It isn't a claim that building custom is the right call for most teams, most of the time, TFX's specific constraints, cost, power, a ten-year unmanned field life, a known and bounded product family, are exactly why it made sense here, and most projects don't share all four. And it isn't a criticism of the people maintaining littlefs, SPIFFS, or NVS, whose defect histories look entirely ordinary for actively used infrastructure carrying real adoption.

What's left is the actual takeaway, and it travels well beyond NOR flash: fit, timing, and control are three separate questions, not one. Getting the first one right doesn't mean the second one stays right forever, and getting both right doesn't tell you who's holding the fix when something eventually breaks. The right answer can change because the calendar moved, not because the problem did. That's worth checking on, every so often, for any decision like this one, long after it was made.

> **Fit, timing, and control are three separate questions, not one. Getting one right doesn't mean the others stay right forever.**

---

## Appendix: FAT-Based and NAND-Native Options

Four more systems are included here for completeness: FatFs paired with a flash translation layer, ThreadX FileX with LevelX, SEGGER's commercial emFile, and YAFFS2. None of these were seriously evaluated for performance back in 2016, not because of licensing or dependency friction, but because the mismatch was obvious from how they operate: all four are block- or chunk-oriented, writing in units of 512 bytes to 1 KB, against records that average 6 to 48 bytes. That was clear from the architecture alone, long before any measurement, and the current data simply confirms it. Across the combined ten-year, device-wide scenarios, even the cheapest of the four (FileX+LevelX or emFile, with directory-sync writes batched rather than flushed on every update) costs roughly 11 to 24 times TFX's storage layer, depending on scenario. The most expensive, FatFs with no directory batching, costs up to 285 times as much.

None of this changes the central comparison; littlefs, NVS, and SPIFFS remain the only systems close enough to matter. But it's worth including briefly, because a reader coming from general-purpose or larger-systems backgrounds might reasonably ask why a "real" filesystem like FAT wasn't just used here. The honest answer is in the numbers rather than an assertion: FAT and NAND-native filesystems were built for large, sequential files, and they don't bend gracefully toward tiny, frequent records, no matter how mature or well-engineered the implementation is.

| System | Best-case ratio to TFX | Worst-case ratio to TFX |
|---|---|---|
| ThreadX FileX + LevelX / SEGGER emFile | ~11.0x | ~42.1x |
| YAFFS2 | ~20.8x | ~46.8x |
| FatFs + FTL | ~72.4x | ~285.0x |

*(These four systems were modeled on dedicated storage regions, consistent with TFX's own architecture, rather than a shared-pool configuration. Full scenario-by-scenario source data is in the [complete performance analysis](https://github.com/dbucovsky/article-appendix/blob/main/fit-timing-control/performance-analysis.md).)*

---

**Further reading, full supporting data:**
- [Embedded filesystem history and architecture](https://github.com/dbucovsky/article-appendix/blob/main/fit-timing-control/embedded-fs-reference-brief.md), littlefs, SPIFFS, ESP-IDF NVS, FatFs+FTL, ThreadX FileX+LevelX, YAFFS2, and SEGGER emFile
  https://github.com/dbucovsky/article-appendix/blob/main/fit-timing-control/embedded-fs-reference-brief.md
- [Known defect history](https://github.com/dbucovsky/article-appendix/blob/main/fit-timing-control/generalist-fs-defect-history.md), sourced issue trackers for every comparison system
  https://github.com/dbucovsky/article-appendix/blob/main/fit-timing-control/generalist-fs-defect-history.md
- [Full performance analysis](https://github.com/dbucovsky/article-appendix/blob/main/fit-timing-control/performance-analysis.md), complete methodology and scenario-by-scenario results
  https://github.com/dbucovsky/article-appendix/blob/main/fit-timing-control/performance-analysis.md

**Acknowledgments:** Thanks to [Iván Andrés León Vásquez](https://www.linkedin.com/in/ivanleonv/) and Rodrigo Garbi, who were integral to developing TFX's EEPROM drivers, and to Pablo Gomez Martino, who helped perform the quantitative analysis and was a key editor and reviewer throughout. This article would not be what it is without their contributions.

#IoT #EmbeddedSystems #Firmware #IIoT #FlashMemory
