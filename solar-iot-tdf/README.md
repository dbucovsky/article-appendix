# Understanding Solar for IoT: The Tierra del Fuego Problem

A solar sizing case study for a proposed LoRaWAN gateway near Tierra del Fuego, modeled against real climate data, published soiling research, cold-battery derating, and a real off-the-shelf MPPT controller's specs, showing why a panel sized double the real industry recommendation still wasn't enough without a smarter charging strategy.

First in a series of long-form technical articles by Damian Bucovsky, President at The Shadow
on the Moon, examining the use of solar power in IoT solutions.

## About this article

This piece walks through the solar sizing analysis behind a remote LoRaWAN gateway
deployment near Tierra del Fuego, in the far south of Chile. It uses the proposal as a case
study in why simple sizing rules, even generous ones, tend to miss the specific combination of
factors that actually determines whether a solar-powered IoT deployment survives its worst
month: real weather, wind-driven soiling, cold-battery derating, real charge-controller
behavior, and the difference between a naive estimate and a fully modeled one.

The value of the piece is in the analysis itself.

## Contents

- `understanding-solar-for-iot-tdf.md` — the published article
- `tdf-solar-modeling-reference.md` — full supporting reference document: complete methodology,
  derivations, sourcing, and known limitations behind every figure quoted in the article
- `media/` — images used in the article (geometry diagrams, site photographs, data tables
  rendered as images for LinkedIn)
- `external-ref/` — local copies of cited third-party sources, kept alongside official links
  in case those links change or go offline

## Status

Draft.

## Related articles

- "BLE + LoRaWAN: The Architecture That Beats Solar in Remote Oil Fields," a related piece
  from the same Tierra del Fuego oil field project, covering the BLE-to-LoRaWAN hybrid
  architecture and hazardous-area deployment constraints
- A future piece will extend this article's multi-city solar comparison (including London)
  into its own full treatment
