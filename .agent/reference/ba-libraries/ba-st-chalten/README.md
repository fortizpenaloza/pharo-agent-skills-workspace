---
name: ba-st-chalten
description: Use when writing Smalltalk code that manipulates dates, months, years, time-of-day, or datetimes - financial settlements, schedules, iteration over date ranges, multi-calendar conversions, or timezone-aware instants. Covers Chalten's immutable time model.
---

# Chalten — Immutable Time Model for Smalltalk

Chalten models time as a set of immutable value objects with strong validation: `ChaltenYear`, `ChaltenMonth`, `MonthOfYear`, `FixedDate`, `Day` (day-of-week), `TimeOfDay`, `ChaltenDateTime`, `TimeSpan`, `RelativeDate`. Multi-calendar (Gregorian, Julian, Hebrew, Islamic, Roman). Invalid values raise at construction; nothing leaks downstream.

Repo: `/home/mtabacman/Development/Repos/ba-st-skills/Chalten/`. Test suite under `source/Chalten-Core-Tests/` has 1600+ tests — mine it for examples.

## Installation

```smalltalk
Metacello new
    baseline: 'Chalten';
    repository: 'github://ba-st/Chalten:release-candidate/source';
    load.
```

Calendar sub-modules (loaded by default in `Chalten`): `Chalten-Gregorian-Calendar`, `Chalten-Julian-Calendar`, `Chalten-Hebrew-Calendar`, `Chalten-Islamic-Calendar`, `Chalten-Roman-Calendar`.

As a dependency, load only what you need:
```smalltalk
spec
    baseline: 'Chalten'
        with: [ spec
            repository: 'github://ba-st/Chalten:v{XX}/source';
            loads: #( 'Chalten-Gregorian-Calendar' ) ];
    import: 'Chalten'.
```

## Core model

| Class | Represents |
|-------|------------|
| `ChaltenYear` (`GregorianYear` etc.) | A calendar year (`2014 asGregorianYear`) |
| `ChaltenMonth` | A month concept — `January`, `February`, … exist as global instances |
| `MonthOfYear` | A specific month in a specific year |
| `FixedDate` | A specific date — `April thirtieth, 2014` |
| `Day` | A day-of-week — `Monday`, `Tuesday`, … (circular) |
| `TimeOfDay` | Hours/minutes/seconds/milliseconds within a day |
| `ChaltenDateTime` | `FixedDate` + `TimeOfDay` + `TimeZone` (an instant) |
| `TimeSpan` | A directed segment on the timeline (from-point + duration) |
| `RelativeDate` | A date relative to a `TimelineFilter` (e.g. "next business day") |

All instances are immutable value objects — `=`/`hash` compare by logical identity, no setters.

## Canonical usage

```smalltalk
"1. Fluent date construction (month globals are the idiom)"
january1_2004 := January first, 2004.
april30_2014  := April thirtieth, 2014.
dec31_2003    := December thirtyfirst, 2003.

"2. TimeOfDay with progressively finer precision"
TimeOfDay hours: 14.                                            "14:00"
TimeOfDay hours: 14 minutes: 30.                                "14:30"
TimeOfDay hours: 14 minutes: 30 seconds: 45.                    "14:30:45"
TimeOfDay hours: 14 minutes: 30 seconds: 45 milliseconds: 500.  "14:30:45.500"

"3. Combine into a DateTime (defaults to local TZ unless you pass one)"
dt := ChaltenDateTime date: (January first, 2004) timeOfDay: (TimeOfDay hours: 3).

"4. Explicit time zones (two instants comparable across zones)"
bsAs := ChaltenDateTime
    date: (April twentieth, 2014)
    timeOfDay: (TimeOfDay hours: 19 minutes: 35)
    zone: TimeZones buenosAires.
utc := ChaltenDateTime
    date: (April twentieth, 2014)
    timeOfDay: (TimeOfDay hours: 22 minutes: 35)
    zone: TimeZones greenwich.
bsAs = utc.  "=> true — same instant"

"5. Arithmetic uses Aconcagua measures (not Pharo Durations)"
tomorrow := (January first, 2004) next: 1 dayMeasure.
nextWeek := (January first, 2004) next: 7 daysMeasure.
in3Years := (2004 asGregorianYear) next: 3 yearsMeasure.

"6. Magnitude protocol for comparisons"
(January first, 2014) < (January tenth, 2014).  "=> true"
December < January.                             "=> false — calendar order"

"7. Signed distance between two points — result is a measure"
(January first, 2014) distanceTo: (January tenth, 2014).  "=> 9 daysMeasure"
(2014 asGregorianYear) distanceTo: (2016 asGregorianYear). "=> 2 yearsMeasure"

"8. Intervals and filtering — the Smalltalk idiom of #to: works on dates"
leapYears := ((2005 asGregorianYear) to: (2100 asGregorianYear))
                 select: [ :y | y isLeap ].
januarySundays := ((January first, 2014) to: (January of: 2014) lastDate)
                      select: [ :d | d is: Sunday ].

"9. Zoom in/out on the timeline"
year2014 := GregorianCalendar newYearNumber: 2014.
year2014 firstMonth.   "=> January 2014"
year2014 lastMonth.    "=> December 2014"
year2014 firstDate.    "=> January 1, 2014"
year2014 lastDate.     "=> December 31, 2014"

"10. Cross-calendar equality — compared by absolute instant"
(April first, 2014) asJulian.    "=> March 19, 2014 (Julian)"
(April first, 2014) asHebrew.    "=> Nisan 1, 5774"
(April first, 2014) asIslamic.   "=> Jumada I 30, 1435"
(April first, 2014) = (JulianMarch nineteenth, 2014).  "=> true"
```

## Integration

```smalltalk
"Current date/time"
GregorianCalendar today.   "=> FixedDate, local-zone implied"
GregorianCalendar now.     "=> ChaltenDateTime in local zone"

"TimeSpan for financial settlements etc."
ts := TimeSpan from: (April third, 2014) duration: 48 hoursMeasure.
ts to.  "=> April fifth, 2014"

"Leap-year predicate on Year"
(2004 asGregorianYear) isLeap.  "=> true"

"Interop with Pharo's built-in Date"
(January first, 2014) asSmalltalkDate.  "=> Date/DateTime (Pharo)"
```

## Chalten vs Pharo's built-in `DateAndTime`/`Date`/`Time`

| Aspect | Chalten | Pharo built-in |
|---|---|---|
| Mutability | Fully immutable | `Time` mutable; `Date`/`DateAndTime` value-ish |
| Validation | Fails at creation (`InvalidDateException`) | Silently normalizes or allows |
| First-class abstractions | `Month`, `Day`, `MonthOfYear`, `TimeSpan`, `RelativeDate` | Mostly just `Date`/`Time` |
| Calendars | Gregorian, Julian, Hebrew, Islamic, Roman | Gregorian only |
| Time zones | Explicit `TimeZones` on `ChaltenDateTime` | Fragile `DateAndTime` timezone handling |
| Arithmetic | Aconcagua measures (`daysMeasure`, `hoursMeasure`) | `Duration` |
| Leap handling | `aYear isLeap` | Compute manually |

## Gotchas

- **`January`, `February`, … are globals.** `April thirtieth, 2014` is the idiom. `FixedDate monthOfYear: ... dayNumber: ...` works but reads badly; reserve it for programmatic construction.
- **Do not mutate.** Everything returns a new instance. Hold the result.
- **Use Aconcagua measures for arithmetic**, not raw numbers: `next: 7 daysMeasure`, not `next: 7`.
- **`Day` means day-of-week** (`Monday`, `Tuesday`, …), not a specific date. Don't confuse with `FixedDate`.
- **Time zones are mandatory for cross-zone reasoning.** `ChaltenDateTime date:timeOfDay:` alone assumes local; use `zone:` when comparing/storing across zones.
- **Invalid dates raise at construction.** `February 30` throws `InvalidDateException`; test-only "allow invalid" does not exist.
- **Ranges are inclusive on both ends** via the Magnitude `#to:` protocol. Adjust by 1 day if you want half-open semantics.
- **Interop conversions are explicit.** Don't pass a `ChaltenDateTime` where a Pharo `DateAndTime` is expected — send `asSmalltalkDate` (or vice-versa with `asChaltenDate`/equivalents).

## When to reach for Chalten

Any domain where time logic is more than "print today's date": financial settlement schedules, calendars of holidays, cross-calendar display (Hebrew/Islamic dates), timezone-correct audit trails, or anything where business logic differs for leap years / month-ends / working days.
