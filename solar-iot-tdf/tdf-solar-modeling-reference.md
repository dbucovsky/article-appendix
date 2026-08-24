# Reference: Solar Viability Modeling for a Remote LoRaWAN Gateway
## Supporting analysis for "Understanding Solar for IoT: The Tierra del Fuego Problem"

This document holds the full modeling behind the article. It is written for readers
who want the underlying math, sources, and methodology rather than the narrative
version. Figures quoted in the article are rounded for readability; figures here are
given to more precision where useful.

**Note on local source copies:** citations below reference both an official source URL
and a local copy path under `external-ref/` in this repository, for every source the
author has archived there. Local copies protect these citations from link rot even if
the official pages move or go offline. Not every source below has a local copy (a few
are general reference sites without a single canonical document, e.g. the climate-data
aggregators and battery-derating references); those are cited by URL only.

## 1. The scenario

Location: an oil field wellhead in the far south of Chile, near Tierra del Fuego,
modeled at 52.3846S, 69.7371W. Equipment: an 8-channel MultiTech Conduit LoRaWAN
gateway (standard indoor-rated variant, not the IP67 outdoor SKU), a 175W solar panel,
and a 100Ah 12V lead-acid battery.

### 1.1 Panel size rationale

175W represents a documented sizing decision, worth explaining on its own terms
rather than presented without justification. Fifty to eighty watts is roughly the
range real commercial vendors size around for an outdoor gateway in this class.
RAKwireless, for example, bundles its commercial-grade WisGate Edge Pro outdoor
gateway with an 80W solar kit as a complete, currently-sold product. Voltaic Systems,
a solar-for-IoT vendor that states it ships roughly one million panels a year,
publishes a simpler rule for a comparable MultiTech gateway: panel output should
target 1.5x expected daily consumption. Applied to this document's load figure
(Section 2), that rule implies a panel in the same 53-66W range.

The sizing decision called for a bit more than double the upper end of that
real-world range, 175W against an approximately 80W reference point, as an added
safety margin. This document's own modeling (Sections 3-11) finds that even this
doubled figure is not sufficient on its own once real weather, soiling, cold-battery
derating, and real charge-controller behavior are all accounted for together; only
the mixed-strategy charging fix (Section 11) closes the gap; see Section 11 for the
result.

## 2. Load derivation

MultiTech publishes measured input current for the Conduit LoRaWAN gateway (MTAC-LORA
accessory card, installed in a Conduit device) at 9.0V, 20.0V, and 32.0V, but not at
12V directly. The published table includes three columns: "Typical Power, No
Connection," "Typical Power w/Ethernet Cable Attached," and "Max Power, Passing Data
Stream." This document uses Typical Power, No Connection, the more representative
figure for continuous day-to-day operation:

- 9.0V measured (typical, no connection): 0.607A -> 5.463W
- 20.0V measured (typical, no connection): 0.300A -> 6.0W
- Interpolated at 12V: 5.61W -> 467mA

Working load: **134.6 Wh/day** (5.61W x 24h continuous, no meaningful idle mode).
Source: MultiTech Systems, MTAC-LORA Power Draw developer documentation.

This figure is independently corroborated: Voltaic Systems, in a presentation hosted
on multitech.com covering solar sizing for MultiTech gateways, states the MultiTech
IP67 Gateway (a different specific model, including their PoE injector) draws 5.6W
continuous, 134 Wh/day, nearly identical to the figure derived here for a different
model in the same product family. Two independent sources landing within half a
percent of each other is strong support that this load figure is not a significant
over- or under-estimate.

This load figure is used unchanged throughout every calculation in this document and
in the article. It was deliberately kept simple (a flat daily figure derived from
published specs) since that is the level of rigor a non-specialist team would
realistically apply at an early sizing stage, and the article's early sections are
built to show what that level of rigor produces.

## 3. Geometry model

### 3.0 Fixed tilt angle selection

The 49-degree fixed tilt used throughout this document was found numerically, not
analytically: for each candidate tilt angle from 0 to 90 degrees, in 1-degree steps,
this document's full-year clear-sky irradiance model (Section 3.2 method) was run to
completion, and the tilt angle producing the highest total annual output was kept. A
fixed, single tilt angle is the relevant case here because the deployment uses a
permanently mounted panel, not a seasonally adjustable or tracking mount. For a
Southern Hemisphere site, the panel faces north (azimuth 0 degrees), toward the
equator, the same convention used for a Northern Hemisphere site facing south.

This numerical search, rather than a closed-form formula, is used because the
optimal fixed tilt for annual output depends on the full irradiance model (including
diffuse sky irradiance, not just the direct beam), which does not reduce to a single
clean equation. A common rule of thumb, optimal tilt approximately equal to
latitude, gives 52.4 degrees for this site; the numerically found 49 degrees is close
to that rule of thumb but not identical, which is expected since the rule of thumb
ignores diffuse irradiance and the asymmetry between summer and winter day lengths.
The optimal tilt angle depends on latitude alone, not on panel wattage, so this
result is unaffected by the panel size used throughout this document, described in Section 1.1.

Clear-sky irradiance modeled with pvlib (Ineichen model), using a uniform Linke
turbidity of 3.0 across all calculations in this document, deliberately chosen to keep
atmospheric clarity assumptions consistent and location-neutral rather than defaulting
to pvlib's built-in location-specific turbidity values, which encode real average
atmospheric clarity for a given place and would otherwise silently contaminate a
"geometry-only" calculation with real weather information.

### 3.1 Solar-noon geometry, June solstice

The general formula for solar altitude angle at any time of day, at any latitude, is:

```
sin(altitude) = sin(phi) * sin(delta) + cos(phi) * cos(delta) * cos(H)
```

where phi is latitude, delta is solar declination (the sun's angle relative to the
equatorial plane, which varies through the year from about -23.44 degrees at the
December solstice to +23.44 degrees at the June solstice), and H is the hour angle
(0 degrees at solar noon, increasing through the day).

At solar noon specifically, H = 0, so cos(H) = 1, and the formula simplifies to:

```
sin(altitude) = sin(phi) * sin(delta) + cos(phi) * cos(delta)
```

This is exactly the cosine angle-difference identity, cos(A - B) = cos(A)cos(B) +
sin(A)sin(B), rearranged. So:

```
sin(altitude) = cos(phi - delta)
```

Since sin(90 - x) = cos(x), this means:

```
altitude = 90 - |phi - delta|
```

and, since zenith angle is simply 90 degrees minus altitude:

```
theta_z = |phi - delta|
```

That is the shortcut formula used throughout this document. It only holds at solar
noon; at any other hour, the full formula above (with the cos(H) term reinstated) is
needed. This result depends only on latitude and date, not on panel size, so it is
unaffected by the panel size used throughout this document, described in Section 1.1.

Substituting Tierra del Fuego's latitude and the June solstice declination:

```
phi = -52.4 degrees (Tierra del Fuego)
delta = +23.4 degrees (June solstice, Northern Hemisphere summer, Southern Hemisphere winter)

theta_z = |-52.4 - 23.4| = 75.8 degrees
cos(75.8 deg) = 0.244
```

Direct-beam intensity scales with cos(theta_z), so a fixed panel at this latitude
receives roughly a quarter of the intensity it would see with the sun directly
overhead, purely from geometry, before any tilt optimization, weather, or hardware
factors are considered. This is the physical mechanism illustrated in the article's
geometry diagram.

### 3.2 June harvest, geometry alone, clear sky

Note: Section 3.1 above computes a single instant, solar noon on one specific day,
using the sun's angle relative to a horizontal surface. That calculation illustrates
the underlying physical mechanism (why winter sun delivers less intensity) but is not
itself the method used to produce the Wh/day figures used throughout this document.

The actual harvest figures are computed differently: full-day numerical integration,
at 10-minute resolution, of plane-of-array irradiance on the panel's own 49-degree
fixed tilt (not the horizontal reference used in 3.1), using pvlib's irradiance model
to account for the sun's changing position throughout the day, not just at noon, and
including both direct and diffuse irradiance components. This is a numerical
integration, not a formula that reduces to a hand-computable equation the way 3.1
does; it is run in code (pvlib.irradiance.get_total_irradiance combined with
pvlib.solarposition.get_solarposition), and the result is summed across the day and
converted to Wh using the 175W panel rating and instantaneous plane-of-array
irradiance relative to the 1000 W/m2 standard test condition.

Using this method, the 49-degree fixed tilt, clear sky (no real weather derate), no
soiling, ideal 98% conversion efficiency, no charge-controller self-consumption:

**June harvest: 422.1 Wh/day** (+214% versus the 134.6 Wh/day load)

## 4. Real annual insolation (idealized, no hardware losses)

For the "honest annual average" figure quoted early in the article, real monthly
weather (percent of possible sunshine, from cited climate normals) is applied across
all 12 months, with the same 49-degree tilt and ideal 98% conversion, no soiling or
self-consumption yet:

| Month | % possible sunshine | Daily harvest (Wh) |
|---|---|---|
| Jan | 45 | 597.5 |
| Feb | 45 | 576.2 |
| Mar | 42 | 480.6 |
| Apr | 38 | 332.5 |
| May | 33 | 192.6 |
| Jun | 32 | 135.1 |
| Jul | 33 | 164.6 |
| Aug | 35 | 268.2 |
| Sep | 38 | 402.8 |
| Oct | 45 | 559.7 |
| Nov | 47 | 619.7 |
| Dec | 47 | 626.6 |

**Annual average: 413.0 Wh/day (3.07x margin over load)**

Note June's own figure in this table, 135.1 Wh, essentially exactly breaks even
against the 134.6 Wh/day load, +0.3%. This is a useful internal consistency check:
the article's "Where the math breaks" section states that isolating the worst month
from this same idealized calculation lands at "barely breaking even," and this table
confirms that claim numerically.

## 5. Monthly percent-of-possible-sunshine values used

"Percent of possible sunshine" is a standard meteorological metric: the ratio of
actual recorded sunshine hours in a period to the maximum sunshine hours
astronomically possible for that period (i.e., the day length the sun is above the
horizon at all, regardless of cloud). It is reported this way, rather than as raw
sunshine hours, specifically because it separates the day-length effect (a function
of latitude and season alone) from the cloud-cover effect (actual atmospheric
conditions), which is exactly the distinction this document's geometry-then-weather
structure depends on.

### 5.1 Source data

No single source publishes a clean percent-of-possible-sunshine table for this
specific location. The monthly figures used here were constructed from independently
reported raw sunshine-hour data for Punta Arenas, the nearest station with reliable
climate normals to the Tierra del Fuego site, cross-checked against day-length
figures computed directly via solar position calculations (pvlib) for the same
latitude.

Anchor data points, from independent published sources:

- Annual average: the sun shines approximately 42-43% of daylight hours at this
  latitude (climatestotravel.com, Punta Arenas climate normals, 1991-2020 baseline).
- Sunniest months (October, November, December): average approximately 7 hours 29
  minutes of recorded sunshine per day (nomadseason.com, Punta Arenas monthly
  climate data).
- Least sunny months (May, June, July): average approximately 3 hours 26 minutes of
  recorded sunshine per day (same source).
- Total annual sunshine: reported in the range of roughly 1,780 to 2,090 hours per
  year across several independent climate-data aggregators (climatestotravel.com,
  climate-data.org, weather-and-climate.com, nomadseason.com), reflecting normal
  variation in methodology and averaging period between sources rather than a
  disagreement about the underlying climate.

### 5.2 Construction of the monthly table

Each month's raw sunshine-hour figure (interpolated between the anchor points above
where a source did not report that specific month directly) was divided by that
month's astronomically possible day length, computed from solar position for
52.3846S, calculated directly rather than taken from a secondary source:

```
percent_possible_sunshine = recorded_sunshine_hours / possible_daylight_hours
```

This produces the monthly table used throughout this document:

```
Jan 45, Feb 45, Mar 42, Apr 38, May 33, Jun 32,
Jul 33, Aug 35, Sep 38, Oct 45, Nov 47, Dec 47
```

June and July, the two lowest months, correspond to the shortest, cloudiest, and
least-sunny stretch of the year, consistent with the anchor data above (the reported
May-July low of roughly 3h26m/day). The annual average of this monthly table works
out to approximately 40%, consistent with the 42-43% figure reported directly for the
full year; the small difference reflects the interpolation and rounding involved in
reconstructing a 12-point monthly curve from a smaller set of published anchor
points, rather than a disagreement with the source data.

This construction is a reasonable, defensible estimate built from multiple
independent, cross-checked sources, but it is not a single authoritative published
table. Anyone reproducing this analysis with access to a direct percent-of-possible-
sunshine dataset for this specific station should prefer that source over the
reconstruction here.

Annual average approximately 40%, matching the figure cited in the article. June, the
site's worst month, sits at 32%, the lowest value in the annual cycle.

## 6. Temperature data used

Monthly mean ambient temperature (proxy for battery temperature, see limitations in
Section 12), from Punta Arenas climate normals (multiple independent sources,
cross-checked: climatestotravel.com, weather-atlas.com, nomadseason.com):

```
Jan 11.3, Feb 11.0, Mar 9.5, Apr 7.5, May 4.5, Jun 2.5,
Jul 2.0, Aug 3.0, Sep 5.0, Oct 7.5, Nov 9.5, Dec 10.5 (deg C)
```

June's mean of 2.5C (36.5F) is itself a generous simplification. Published climate
normals for the region report roughly 43 days per year below 0C (32F) and rare
extremes approaching -9C (16F) (weather-atlas.com and climatestotravel.com Punta
Arenas climate data, 1991-2020 baseline), concentrated in the June-August window. The
daily-mean figure used throughout this model does not capture those overnight
extremes; see Section 12.

## 7. Soiling

Applied uniformly as an 8% loss on harvested energy, representing wind-driven dust and
grit exposure at an exposed, unmaintained wellhead site. This figure was originally an
engineering estimate (2019), made deliberately conservative given the absence of
site-specific soiling data at the time. It has since been checked against two NREL-
affiliated peer-reviewed sources:

- Deceglie, M.G., Micheli, L., and Muller, M., "Quantifying Soiling Loss Directly From
  PV Yield," IEEE Journal of Photovoltaics, Vol. 8, No. 2, 2018.
  Official source: https://www.osti.gov/servlets/purl/1419409
  Local copy: external-ref/deceglie-micheli-muller-2018-quantifying-soiling-loss.pdf
  (Manuscript received Oct 19, 2017; accepted Dec 8, 2017; publicly available well
  before the 2019 site visit that produced the original estimate.)

- Micheli, L., Deceglie, M.G., and Muller, M., "Mapping Photovoltaic Soiling Using
  Spatial Interpolation Techniques," IEEE Journal of Photovoltaics, published online
  Oct 11, 2018.
  Official source: https://www.osti.gov/pages/biblio/1479873
  Local copy: external-ref/micheli-deceglie-muller-2018-mapping-soiling.pdf
  This paper's 83-site US dataset found soiling losses concentrated in the southwestern
  US, with the worst-affected counties attributed specifically to high particulate
  matter concentration and long dry periods, the same mechanism (wind-driven dust,
  low precipitation) present at the Tierra del Fuego site.

Both studies are US-based. The 8% figure used here is not drawn directly from either
paper's measured values; it generalizes the underlying mechanism (dry, wind-exposed
sites see above-average soiling loss) to a different geography with similar exposure
characteristics. This is presented in the article explicitly as an estimate that holds
up under later scrutiny, not as a number pulled directly from the cited literature.

June harvest after soiling: **388.3 Wh/day** (down from 422.1 Wh/day, geometry alone)

## 8. Battery model

### 8.0 Nameplate capacity

100Ah x 12V = 1200 Wh nameplate capacity at 25C, the reference figure all temperature
derating and clipping calculations in this section scale against.

### 8.1 Capacity-vs-temperature derating

Lead-acid AGM capacity fraction versus temperature (cited, solarsizecalculator.com
Battery Temperature Calculator, cross-referenced against PVEducation.org's simpler
"~1% capacity loss per degree below 20C" rule of thumb):

```
25C (77F): 100%
10C (50F): 80%
0C (32F): 55%
-10C (14F): 25%
```

Linearly interpolated between points; floored at 10% for any extrapolation colder than
-10C (not reached in this model's monthly-mean temperature inputs, though likely
reached on individual cold nights, see Section 12).

### 8.2 Clipping mechanics

Battery state of charge is simulated day by day across a full year (365 individual
daily updates, not a single monthly-resolution shortcut). Each day: add that month's
harvest figure, subtract the 134.6 Wh/day load, clip the result at that month's
temperature-derated ceiling (not the full 1200 Wh nameplate capacity), floor at zero.
Battery starts fully charged (1200 Wh) on January 1.

A key finding from this modeling: cold-derated capacity, on its own, with clipping
*not* enforced (i.e., only capped at the full 1200 Wh nameplate regardless of
temperature), produces no visible effect on the simulation at all, since a battery
large enough relative to the load never approaches even its derated ceiling before
clipping is turned on. The cold-derating effect only becomes visible once combined
with the clipping constraint that prevents summer surplus from being banked forward
past a shrunken winter ceiling. This is why the article presents cold-derating and
clipping as one combined step rather than two separable ones.

June harvest unchanged at 388.3 Wh/day (cold-derating and clipping affect the storage
side, not the harvest rate); battery minimum state of charge across the year drops to
**60%** once clipping is enforced against the derated ceiling.

## 9. Real charge controller

Product: Renogy Rover 20A MPPT Solar Charge Controller (RNG-CTRL-RVR20). Widely
available retail (approximately $70-100), rated for up to 260W at 12V, comfortably
covering the 175W panel. Published specifications, confirmed against both the current
product page and the official Rover Li Series user manual (Version B1, March 2024):

- Peak conversion efficiency: 98%
- Tracking efficiency: up to 99%
- Power consumption: <100mA at 12V (<58mA at 24V)

This model's own official specification, confirmed independently on both Renogy's
current product listing and in the printed user manual, is under 100mA at 12V,
approximately 1.2W. That figure is used throughout this document and the article.

Modeled as: `charge_W = panel_dc_W * 0.98 - 1.2`, applied continuously (the 1.2W draw
does not pause at night; when panel output is zero, this becomes a net drain on the
battery).

June harvest with this real controller (soiling and cold/clipping already in place):
**359.5 Wh/day** (+167% margin, before real cloud cover is introduced)

## 10. Real cloud cover

The calculations in Sections 3-9 use a hypothetical clear sky for every month, by
design, to isolate what geometry, soiling, cold-battery effects, and charge-controller
behavior each individually contribute before the dominant real-world factor is added.
Substituting June's actual recorded weather (32% of possible sunshine, per Section 5)
in place of the assumed clear sky:

June harvest: **95.5 Wh/day** (-29%, a real deficit)
Battery minimum state of charge across the year: **0%**
Dead-battery days across the year: **42**, concentrated in June

This is the single largest change in the entire model. Three real, independently
verified problems, soiling, cold-battery derating, and charge-controller
self-consumption, stacked together, still leave the system at a 167% margin under
clear skies. Real cloud cover alone erases that margin entirely and pushes the system
into deficit.

### 10.1 What a "dead-battery day" actually represents

This model tracks battery state at daily-net resolution: each day, total harvest for
that day is added, total load for that day is subtracted, the result is clipped at
the temperature-derated ceiling and floored at zero. A day flagged as "dead" means the
running multi-day balance reached zero by end of day; the model has no sub-daily
visibility into when during the day that happened, or whether the gateway had any
working hours at all.

The physically reasonable picture, inferred from what a net-negative day means rather
than measured directly, is: the gateway is genuinely down overnight, since there is no
reserve and no charging occurring in the dark. Once the sun rises and panel output
climbs past the continuous load, the gateway comes back and operates for some real
stretch of the day. As afternoon output falls back below the load, the gateway goes
dark again, having picked up too little net charge to survive the coming night, and
the cycle repeats the next day. What varies is the width of that daytime working
window, driven by how far in deficit that particular day's harvest actually was, not
whether the general pattern holds.

Two things this document cannot claim with the same confidence: the exact width or
timing of that daytime window (which would require sub-daily, not daily-net,
simulation), and the effect of real charge-controller low-voltage disconnect
behavior. The Renogy Rover's default settings cut the load off at 11.0-11.1V and do
not reconnect it until the battery recovers to 12.6V, a deliberate hysteresis gap
meant to protect the battery from shallow-cycling. A real deployed system would
likely stay dark somewhat later into the morning, and come back somewhat later in the
day, than a pure energy-balance picture alone would suggest, since the controller
will not re-energize the load the instant panel output first exceeds the load, only
once the battery has recovered a meaningfully larger charge than that.

"Dead-battery day," used throughout this document and the article, should be read as
"zero net reserve carried into the next day, with real but unmeasured partial-day
operation," not as 24 continuous hours without power.

## 11. The fix: mixed MPPT and pump-and-dump charging

Modeled as an automatic, sensing-based switch between two charging modes at each
timestep, selecting whichever mode nets more charge given the panel's instantaneous
output:

```
mppt_W = clip(panel_dc_W * 0.95 - 0.18, lower=-0.18)
pumpdump_W = panel_dc_W * 0.88
charge_W = max(mppt_W, pumpdump_W)
```

- MPPT mode: 95% conversion efficiency, 0.18W fixed self-consumption (a
  purpose-built, low-quiescent-current MPPT stage, distinct from the generic
  off-the-shelf Rover controller modeled in Section 9)
- Pump-and-dump mode: a flat 88% conversion efficiency, independent of instantaneous
  power level, representing a burst-transfer charging strategy that avoids paying
  continuous tracking overhead during low-light conditions

Both the 0.18W self-consumption figure and the 88% pump-and-dump efficiency are
engineering estimates, not manufacturer specifications for a named commercial part,
unlike the Renogy Rover figures in Section 9. The 0.18W figure is directionally
supported by published research on purpose-built, ultra-low-quiescent-current MPPT
architectures for photovoltaic energy harvesting, which report average tracking
efficiency around 96% using specifically designed low-power circuitry (Ibrahim, M.A.A.
et al., "An ultra-low-power MPPT architecture for photovoltaic energy harvesting
systems," IEEE, 2017). That research describes a purpose-built integrated circuit
design for micro-scale harvesting applications, not an off-the-shelf product
comparable to the Renogy Rover, so the 0.18W and 88% figures used here should be read
as a plausible, directionally-supported engineering estimate for a well-selected or
custom low-power charging stage, not as a value drawn directly from that paper's
measured results.

With this strategy, soiling, cold-derating and clipping, and real June cloud cover all
still in the model:

June harvest: **129.7 Wh/day** (-4%, still fractionally short of the load on its own)
Battery minimum state of charge across the year: **49%**
Dead-battery days: **0**

June alone does not fully break even under the mixed strategy. The battery survives
because the smarter charging captures enough real surplus in the months surrounding
June to carry a reserve into the shortfall rather than entering it empty.

## 12. Known limitations

- **Monthly, not daily, weather resolution.** Each month's harvest is computed once,
  from a representative day at that month's midpoint, then applied identically to
  every day within the month. The battery's day-by-day response to that repeated
  value is tracked accurately (see Section 8.2), but real day-to-day weather variance
  within a month, including short, unusually severe cloudy stretches, is not
  captured. This model is likely somewhat optimistic in its true worst case, not
  pessimistic.

- **Daily-net battery resolution, not sub-daily.** See Section 10.1. "Dead-battery
  days" mean zero net reserve at end of day, not necessarily 24 continuous hours
  without power; real charge-controller reconnect hysteresis likely makes actual
  downtime somewhat worse than the pure energy-balance picture implies.

- **Battery temperature assumed equal to monthly mean ambient air temperature.**
  Real battery temperature lags ambient air temperature (thermal mass) and likely
  runs colder than the monthly mean during actual overnight lows, which is also when
  no charging is occurring at all. Using the monthly mean likely understates the
  severity of cold-derating during the specific hours it matters most.

- **Soiling and snow-season losses are environment-based estimates**, not
  site-specific field measurements. See Section 7 for sourcing and reasoning.

- **The 0.18W MPPT self-consumption and 88% pump-and-dump efficiency figures used in
  the fix (Section 11) are engineering estimates**, not sourced datasheet values for
  a specific named product, unlike the Renogy Rover figures used in Sections 9-10.

- **No backhaul radio power is included.** This model covers the LoRaWAN gateway's
  own draw only. If the deployed hardware also carries a cellular or satellite
  backhaul radio on the same power budget, real load would be higher than modeled
  here, worsening every result in this document.

- **Uniform atmospheric turbidity (3.0) is used throughout**, deliberately, to avoid
  contaminating geometry-only calculations with location-specific atmospheric clarity
  data. Real atmospheric clarity at this specific site has not been independently
  verified against this assumption.

## 13. Minimum justification for the multi-city comparison

The article states that a fixed-tilt panel in Tierra del Fuego, averaged across a
year under clear-sky conditions, outperforms one in London. This section shows the
minimum calculation behind that specific claim; a fuller comparison across multiple
locations (including real weather, soiling, and charge-controller behavior for each)
is being developed as a separate piece of writing and is not reproduced here.

Using the same method as Section 3.0 (numerical search across candidate fixed tilt
angles, clear-sky annual irradiance, uniform Linke turbidity 3.0), applied to London
(51.5072N, -0.1276) exactly as it was applied to Tierra del Fuego:

| Location | Latitude | Best fixed tilt (numerical search) | Annual clear-sky insolation |
|---|---|---|---|
| Tierra del Fuego | 52.4S | 49 deg | 2298.1 kWh/m2/year |
| London | 51.5N | 44 deg | 1944.1 kWh/m2/year |

Tierra del Fuego's annual total is approximately 18% higher than London's under this
method (2298.1 / 1944.1 = 1.182). Both sites use the same clear-sky model, the same
turbidity assumption, and the same numerical tilt-search method described in Section
3.0; only latitude, and the resulting optimal tilt and day-length/sun-angle balance
through the year, differ between them. This ratio is a per-square-meter irradiance
comparison and is unaffected by the panel size used throughout this document, described in Section 1.1.

### 13.1 Each site's worst month, geometry versus real cloud cover

Using each site's real monthly weather data (percent of possible sunshine, sourced
the same way as Section 5), the worst month for each site, June for Tierra del
Fuego, December for London, is isolated the same way Sections 3.2 and 10 isolate
Tierra del Fuego's June: geometry alone (clear sky, that site's own optimal fixed
tilt), then real recorded cloud cover for that month layered on top, nothing else
added at this stage (no soiling, no battery, no charge-controller behavior). Both
sites use the corrected 175W panel and 134.6 Wh/day load.

**Tierra del Fuego, worst month June (32% of possible sunshine):**

| Stage | June harvest | Excess energy |
|---|---|---|
| Geometry alone (clear sky, 49-degree tilt) | 422.1 Wh | +214% |
| + real June cloud cover | 135.1 Wh | +0.3% |

**London, worst month December (21% of possible sunshine):**

| Stage | December harvest | Excess energy |
|---|---|---|
| Geometry alone (clear sky, 44-degree tilt) | 434.9 Wh | +223% |
| + real December cloud cover | 91.3 Wh | -32.1% |

The pattern is the same at both sites: geometry alone looks comfortable, even
generous, and real cloud cover alone is what erodes that margin. The swing is more
severe at London: a comparable clear-sky margin (+223% versus +214%) collapses to a
worse real-world outcome once actual weather is applied. At this same stage of
analysis, before any soiling or hardware effects are added, Tierra del Fuego's June
lands almost exactly at breakeven, +0.3%. London's December, under the identical
method, is already a real deficit at -32.1%. This is consistent with, and is the
minimum evidence behind, the article's broader point that extreme latitude is not a
reliable predictor of which site actually struggles once real weather, not just sun
angle, is accounted for.

This does not account for soiling or hardware behavior at either site, the same
caveat that applies to the geometry-only figures used earlier in this document for
Tierra del Fuego (Sections 3.2 and 4). It is a geometry-and-weather comparison, not a
claim about which site's a real deployed system would actually survive; that fuller
question, including a real off-the-shelf controller and a full battery simulation for
both sites, is exactly what the forthcoming separate piece addresses.

### 13.2 A third data point: Stockholm

Voltaic Systems' own presentation on solar sizing for MultiTech gateways (Section 1.1;
also cited in Section 2) includes a December sizing table across four cities: Munich,
Malaga, London, and Stockholm. For Stockholm specifically, their table lists 0.44
hours of sun in December, producing 44 Wh from a 100W panel and 88 Wh from a 200W
panel, their largest tabled option, against their own stated load of 134 Wh/day for
the MultiTech IP67 Gateway. Even 200W falls well short there. Munich and Malaga are
comfortably covered by their tabled options; London is not; Stockholm is not, even at
the largest panel size they show. This is real, independently published vendor data
showing that no single fixed panel wattage solves every site; some real locations do
not close the gap even at panel sizes well above what this document models for
Tierra del Fuego. Stockholm is not modeled in this document; it is noted here as a
forward pointer to the separate multi-city piece referenced in Section 13, where it
is intended to be treated with the same rigor as Tierra del Fuego and London.

## 14. The bigger-panel alternative: wind loading

The article discusses a larger fixed panel as a genuine alternative to the mixed
charging strategy in Section 11, simpler to deploy, but with its own real tradeoff:
more surface area means more wind loading, which is a material consideration at an
exposed, wind-heavy site like this one.

An international research team (UAE and Singapore-based, lead author Sagarika Kumar)
investigated wind-induced vibration in PV modules and found that torsional stress
from wind can produce microcracks, misalignment, and mechanical failure over time,
with larger panels experiencing meaningfully greater stress than smaller ones under
the same wind conditions. This is a mechanism-level finding, not a study of this
specific site or panel size, and is generalized here the same way the soiling
research in Section 7 is: the underlying physical relationship (more area, more wind
force, more stress) is not location-specific, even though the study itself did not
model Tierra del Fuego directly.

Source: Emiliano Bellini, "The impact of wind-induced vibrations on solar modules,"
pv magazine International, January 14, 2025
(pv-magazine.com/2025/01/14/the-impact-of-wind-induced-vibrations-on-solar-modules/).

## 15. Sources

- MultiTech Systems, MTAC-LORA Power Draw developer documentation
  Official source: multitech.net/developer/products/multiconnect-conduit-platform/accessory-cards/mtac-lora/mtac-lora-power-draw/
  Local copy: external-ref/mtac-lora-power-draw.pdf
- Voltaic Systems, "Solar Power for LoRaWAN Gateways" presentation
  Official source: multitech.com/wp-content/uploads/Voltaic-Solar-Systems-for-Multitech-2024-04-11-1.pdf
  Local copy: external-ref/Voltaic-Solar-Systems-for-Multitech-2024-04-11-1.pdf
- Renogy Rover 20A MPPT Solar Charge Controller product specifications; cross-checked
  against the Rover Li Series MPPT Solar Charge Controller User Manual, Version B1,
  March 2024 (RNG-CTRL-RVR20/RVR30/RVR40)
  Official source: renogy.com/collections/all/products/rover-li-20-amp-mppt-solar-charge-controller
  Local copy (manual): external-ref/RVR203040-Manual.pdf
- RAKwireless, WisGate Edge Pro Solar bundle product listing, for the 80W commercial
  solar kit figure cited in Section 1.1
  Official source: store.rakwireless.com/products/wisgate-edge-pro-battery-plus-solar-panel-kit
  Local copy: external-ref/accessories-solar-panel-kit-80w-solar-panel-for-battery-plus-datasheet.pdf
- Bellini, Emiliano, "The impact of wind-induced vibrations on solar modules," pv
  magazine International, January 14, 2025, covering research led by Sagarika Kumar,
  cited in Section 14
  Official source: pv-magazine.com/2025/01/14/the-impact-of-wind-induced-vibrations-on-solar-modules/
  Local copy: external-ref/the-impact-of-wind-induced-vibrations-on-solar-modules-pv-magazine-global.pdf
- Deceglie, Micheli, and Muller, "Quantifying Soiling Loss Directly From PV Yield,"
  IEEE Journal of Photovoltaics, 2018
  Official source: osti.gov/servlets/purl/1419409
  Local copy: external-ref/deceglie-micheli-muller-2018-quantifying-soiling-loss.pdf
- Micheli, Deceglie, and Muller, "Mapping Photovoltaic Soiling Using Spatial
  Interpolation Techniques," IEEE Journal of Photovoltaics, 2018
  Official source: osti.gov/pages/biblio/1479873
  Local copy: external-ref/micheli-deceglie-muller-2018-mapping-soiling.pdf
- Ibrahim, M.A.A., et al., "An ultra-low-power MPPT architecture for photovoltaic
  energy harvesting systems," IEEE, 2017
  Official source: ieeexplore.ieee.org/abstract/document/8011105/
  (No local copy archived; paywalled IEEE conference paper, cited for its abstract-
  level finding only, see Section 11.)
- solarsizecalculator.com, Battery Temperature Calculator (lead-acid AGM capacity
  fractions)
  (No local copy; a live calculator tool, not a fixed document.)
- PVEducation.org, Characteristics of Lead Acid Batteries (cross-check reference for
  temperature derating)
  (No local copy; a general reference page, not a fixed document.)
- Climate normals for Punta Arenas / Tierra del Fuego, compiled from multiple
  independent climate-data sources (climate-data.org, weather-and-climate.com,
  climatestotravel.com, nomadseason.com, weather-atlas.com)
  (No single local copy; figures are cross-checked across several live aggregator
  sites rather than one canonical document, see Section 5.1 and Section 6.)
