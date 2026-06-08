---
name: ba-st-aconcagua
description: Use when writing Smalltalk code that handles quantities with units - currencies, distances, temperatures, durations, physical measures - and needs arithmetic, comparison, or conversion between units. Covers Aconcagua's Measure/Unit/MeasureBag API.
---

# Aconcagua — Measures as First-Class Objects

Aconcagua models quantities as `(amount, unit)` value objects. Arithmetic, comparison, and unit conversion all work on `Measure` instances, so unit mistakes become impossible rather than silently numeric.

Repo: `/home/mtabacman/Development/Repos/ba-st-skills/Aconcagua/`. Read the tests in `source/Aconcagua-*-Tests/` — they double as API documentation.

## Installation

```smalltalk
Metacello new
    baseline: 'Aconcagua';
    repository: 'github://ba-st/Aconcagua:release-candidate/source';
    load: 'Deployment'.
```

Groups: `Deployment` (core), `Tests`, `CI`, `Development`. Pin with `:v{XX}/source` in production.

As a dependency in your baseline:
```smalltalk
spec
    baseline: 'Aconcagua'
    with: [ spec
        repository: 'github://ba-st/Aconcagua:v{XX}/source';
        loads: #('Deployment') ];
    import: 'Aconcagua'.
```

## Core classes

- **`Measure`** — a number + unit. Entry point for arithmetic/comparison. Constructed with `Measure amount: aNumber unit: aUnit` (or the more idiomatic `aUnit with: aNumber`).
- **`BaseUnit`** — atomic unit (converts 1:1 to itself). `BaseUnit named: 'meter' sign: 'm'`.
- **`ProportionalDerivedUnit`** — linear conversion factor from a base unit. `cm = m × 1/100`.
- **`NotProportionalDerivedUnit`** — non-linear conversion via `conversionBlock:` / `reciprocalConversionBlock:`. Needed for Celsius/Fahrenheit (offsets, not just scaling).
- **`MultipliedUnit` / `DividedUnit`** — compound units produced by `*` and `/` between measures (e.g. `meter/second`).
- **`MeasureBag`** — heterogeneous bag produced automatically by `+` on incompatible measures (e.g. `10 peso + 20 dollar`). Has no single `amount`/`unit`; you must iterate or split it.

## Defining your own unit system

```smalltalk
"Linear domain: distance"
meter        := BaseUnit named: 'meter' sign: 'm'.
centimeter   := ProportionalDerivedUnit
                    baseUnit: meter conversionFactor: 1/100
                    named: 'centimeter' sign: 'cm'.
kilometer    := ProportionalDerivedUnit
                    baseUnit: meter conversionFactor: 1000
                    named: 'kilometer' sign: 'km'.

"Non-linear domain: temperature"
kelvin  := BaseUnit named: 'kelvin' sign: 'K'.
celsius := NotProportionalDerivedUnit
               baseUnit: kelvin
               conversionBlock:           [ :k | k - 273.15 ]
               reciprocalConversionBlock: [ :c | c + 273.15 ]
               named: 'celsius' sign: '°C'.
```

Most ba-st projects keep units as class-side methods on a domain class (`Currencies class >> peso`, `Distances class >> meter`) that lazily build and cache the unit instance.

## Usage examples

```smalltalk
"1. Build a measure (two equivalent forms)"
oneKm := Measure amount: 1 unit: kilometer.
oneKm := kilometer with: 1.

"2. Same-domain addition auto-normalizes to the base unit"
(kilometer with: 1) + (meter with: 500).
"=> 1500 meter"

"3. Multiplication creates a compound (product) unit"
(meter with: 3) * (meter with: 4).
"=> 12 meter*meter  (area)"

"4. Division creates a compound (quotient) unit"
(meter with: 100) / (second with: 10).
"=> 10 meter/second  (speed)"

"5. Explicit conversion between units"
(meter with: 1) convertTo: centimeter.
"=> 100 centimeter"

"6. Comparison auto-converts to the base unit"
(kilometer with: 1) > (meter with: 500).
"=> true"

"7. Heterogeneous addition yields a MeasureBag (no error raised)"
bag := (peso with: 10) + (dollar with: 20).
bag isMeasureBag "=> true".

"8. Zero/nothing across units is equal"
(peso with: 0) = (dollar with: 0) "=> true  — 'no quantity' is unit-agnostic"
```

## Integration with Chalten

Date arithmetic in Chalten uses Aconcagua measures for durations:
```smalltalk
(January first, 2014) next: 7 daysMeasure.
(2004 asGregorianYear)  next: 3 yearsMeasure.
```
Don't invent your own duration class if you're already on ba-st; use `daysMeasure`, `hoursMeasure`, `yearsMeasure`, etc.

## Gotchas

- **`=` compares base-unit values, not surface units.** `1 km = 1000 m` is `true`. If you want "same unit exactly", compare both `amount` and `unit` manually.
- **`MeasureBag` creation is silent.** `10 peso + 20 dollar` does NOT raise — it yields a bag. Send `isMeasureBag` before calling `amount`/`unit`, or handle the mixed case explicitly.
- **Non-proportional units have offsets.** `0°C` is not `0K`. Convert before comparing numeric `amount`s across such units.
- **`amount` is the stored amount, not base-unit value.** Use `convertAmountToBaseUnit` if you need the base-unit numeric quantity.
- **Integer division/modulo (`//`, `\\`) operate after base conversion** — results can surprise with fractional factors.
- **Ship units as singletons** (class-side lazy accessors on a domain class). Creating fresh `BaseUnit` instances inside methods breaks `=`/`hash` expectations.

## When NOT to use Aconcagua

For purely dimensionless numeric work (counts, ratios, percentages that never convert), plain numbers are fine. Reach for Aconcagua when *units matter* — prices, physics, time spans, anything where the answer to "10 + 20" depends on what kind of 10 and 20 they are.
