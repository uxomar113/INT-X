# INT 4.2 LAT% and BE% Calculation Specification — Phases 2 and 3

## Source File
Current stable source file: `INT4.2.html`.

This specification depends on the UI surfaces from Phase 1:
- `LAT%`
- `EXP%`
- `BE%`
- `EXP2%`

## Non-Negotiable Scope Rule
Do not modify current working systems unless this specification explicitly requires it.

The following systems must remain unchanged:
- Snapshot parser
- LOB block detection
- IN / OUT status classification
- Manual IN / OUT adjustment behavior
- Existing `% Req` calculation
- Existing `Heads` calculation
- Existing AUTO REQ schedule parser
- Existing AUTO REQ timed application
- Existing AVAIL timer
- Existing clear behavior
- Existing 10-second input clear behavior

The goal is to add LAT% and BE% calculation layers without breaking the stable operational calculator.

---

# Shared Definitions

## Effective IN
Use the existing current IN value after manual adjustment:

```text
effectiveIn = max(0, baseIn + adjIn)
```

## Effective OUT
Use the existing current OUT value after manual adjustment:

```text
effectiveOut = max(0, baseOut + adjOut)
```

## Effective REQ
Use the current visible REQ input value for that LOB.

```text
effectiveReq = parseFloat(req input)
```

If REQ is blank, invalid, or less than or equal to zero, LAT% / EXP% / BE% / EXP2% must display `—`.

Do not introduce a new REQ source in this phase.
Do not change AUTO REQ priority.

If AUTO REQ writes into the existing REQ input, LAT/BE calculations must naturally use that updated value.

---

# Phase 2 — LAT% and EXP% Using REQ 92%

## Goal
Add LAT% and EXP% calculation using the existing current IN and REQ values.

This phase introduces the `REQ 92` target only.

Do not add BE% in Phase 2.
BE% and EXP2% must remain `—` until Phase 3.

## Required New Target
Add a fixed LAT target:

```text
REQ_TARGET_92 = 0.92
```

This target is used only by Phase 2 LAT calculations.

Do not replace the existing `% Req` calculation.
Do not replace the existing `Heads` calculation.

## LAT% Formula
LAT% represents current staffing performance against the 92% requirement target.

Formula:

```text
requiredIn92 = effectiveReq * 0.92
LAT% = effectiveIn / requiredIn92 * 100
```

If `requiredIn92 <= 0`, display `—`.

Display format:

```text
LAT% = 00.00%
```

Use two decimals.

Example:
```text
REQ = 10
effectiveIn = 9
requiredIn92 = 10 * 0.92 = 9.2
LAT% = 9 / 9.2 * 100 = 97.83%
```

## EXP% Formula
EXP% is the expected target reference for Phase 2.

For this phase, EXP% must display:
```text
92.00%
```
when REQ is valid.

If REQ is blank, invalid, or less than or equal to zero:
```text
EXP% = —
```

## Phase 2 Display Rules
When valid REQ exists:
- `LAT%` shows calculated LAT% against REQ 92.
- `EXP%` shows `92.00%`.
- `BE%` remains `—`.
- `EXP2%` remains `—`.

When REQ is invalid:
- `LAT%` = `—`
- `EXP%` = `—`
- `BE%` = `—`
- `EXP2%` = `—`

## Phase 2 Update Triggers
LAT% and EXP% must update whenever any of these happen:
- New snapshot is parsed.
- Manual IN adjustment changes.
- Manual OUT adjustment changes if refreshRow is called.
- REQ input changes.
- AUTO REQ applies a scheduled REQ value.
- Clear LOB is used.
- Clear Master is used.

Recommended implementation:
```text
updateLatBeMetrics(id)
```

Call it from existing `refreshRow(id)` and from any existing REQ update path if needed.

## Phase 2 Safety Requirements
Do not write localStorage.
Do not add history.
Do not add interval timers.
Do not add LAT accumulation.
Do not add velocity.
Do not add manual LAT override.
Do not change existing current `% Req`.
Do not change existing current `Heads`.

---

# Phase 3 — BE% and EXP2% Using REQ 100%

## Goal
Add BE% and EXP2% calculation using the `REQ 100` target.

The user requirement:
```text
the 100 req is with BE% phase
```

This means REQ 100 belongs to Phase 3 only.

## Required New Target
Add a fixed BE target:

```text
REQ_TARGET_100 = 1.00
```

This target is used only by Phase 3 BE calculations.

Do not use 100% for LAT%.
LAT% remains based on REQ 92.

## BE% Formula
BE% represents current staffing performance against the 100% requirement target.

Formula:
```text
requiredIn100 = effectiveReq * 1.00
BE% = effectiveIn / requiredIn100 * 100
```

Because `requiredIn100 = effectiveReq`, this is equivalent to:
```text
BE% = effectiveIn / effectiveReq * 100
```

Display format:
```text
BE% = 00.00%
```

Use two decimals.

Example:
```text
REQ = 10
effectiveIn = 9
BE% = 9 / 10 * 100 = 90.00%
```

## EXP2% Formula
EXP2% is the expected target reference for Phase 3.

For this phase, EXP2% must display:
```text
100.00%
```
when REQ is valid.

If REQ is blank, invalid, or less than or equal to zero:
```text
EXP2% = —
```

## Phase 3 Display Rules
When valid REQ exists:
- `LAT%` shows calculated LAT% against REQ 92.
- `EXP%` shows `92.00%`.
- `BE%` shows calculated BE% against REQ 100.
- `EXP2%` shows `100.00%`.

When REQ is invalid:
- `LAT%` = `—`
- `EXP%` = `—`
- `BE%` = `—`
- `EXP2%` = `—`

## Phase 3 Update Triggers
BE% and EXP2% must update whenever LAT% updates:
- New snapshot parse
- Manual IN adjustment
- Manual OUT adjustment if refreshRow is called
- REQ input change
- AUTO REQ scheduled application
- Clear LOB
- Clear Master

## Styling Pass/Fail Rules
Do not change current `% Req` pass/fail colors.

LAT% tile:
- Good when `LAT% >= 100.00%`
- Bad when `LAT% < 100.00%`

BE% tile:
- Good when `BE% >= 100.00%`
- Bad when `BE% < 100.00%`

EXP% and EXP2% are reference tiles only.
They should not show pass/fail state.

## Important Distinction
Existing `% Req` is not the same as LAT%.

Existing `% Req` must remain unchanged.

New LAT% uses the 92% required staffing denominator:
```text
LAT% = effectiveIn / (REQ * 0.92) * 100
```

New BE% uses the 100% required staffing denominator:
```text
BE% = effectiveIn / (REQ * 1.00) * 100
```

## Example
Input:
```text
REQ = 10
effectiveIn = 9
effectiveOut = 4
```

Existing current `% Req` remains whatever the current app already calculates.

New Phase 2/3 metrics:
```text
LAT required = 10 * 0.92 = 9.2
LAT% = 9 / 9.2 * 100 = 97.83%
EXP% = 92.00%

BE required = 10 * 1.00 = 10
BE% = 9 / 10 * 100 = 90.00%
EXP2% = 100.00%
```

Visible result:
```text
LAT%  = 97.83%
EXP%  = 92.00%
BE%   = 90.00%
EXP2% = 100.00%
```

## Acceptance Criteria
Phase 2 is complete only if:
- LAT% is visible.
- EXP% is visible.
- LAT% updates when REQ or IN changes.
- EXP% displays 92.00% when REQ is valid.
- BE% and EXP2% remain placeholders.
- Existing current behavior is unchanged.

Phase 3 is complete only if:
- BE% is visible.
- EXP2% is visible.
- BE% updates when REQ or IN changes.
- EXP2% displays 100.00% when REQ is valid.
- LAT% remains based on 92.
- BE% is based on 100.
- Existing current behavior is unchanged.

Final acceptance:
- No parser changes.
- No REQ behavior changes.
- No AUTO REQ behavior changes.
- No manual adjustment behavior changes.
- No storage changes.
- No console errors.
- No layout overflow at 400px width.
