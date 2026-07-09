# INT 4.2 LAT/BE Expansion Specification — UI Phase

## Source File
Current stable source file: `INT4.2.html`.

This specification is intentionally limited to the UI preparation required for future LAT% and BE% calculation phases.

## Non-Negotiable Scope Rule
Do not modify, rewrite, rename, remove, or weaken any current working behavior in the existing file.

The following existing systems must remain functionally unchanged:
- Snapshot paste behavior
- Auto-parse behavior
- 10-second input clear behavior
- Current LOB parsing
- Current IN / OUT counting
- Current manual IN adjustment
- Current manual OUT adjustment
- Current REQ input behavior
- Current AUTO REQ panel behavior
- Current AUTO REQ schedule parsing
- Current AUTO REQ timed application
- Current `% Req` calculation
- Current `Heads` calculation
- Current AVAIL timer
- Current clear buttons
- Current visual style direction

This phase may only add UI surfaces needed for future LAT% and BE% values.

Do not implement LAT% math in this phase.
Do not implement BE% math in this phase.
Do not add storage in this phase.
Do not change the parser in this phase.
Do not change current formulas in this phase.

---

# Phase 1 — UI Preparation for LAT% / EXP% / BE% / EXP2%

## Goal
Prepare the existing stable UI so the application has visible, stable locations for:
- LAT%
- EXP%
- BE%
- EXP2%

without changing any current working calculation or behavior.

## Required UI Additions
For each LOB card, add a new metrics area below the current `% Req / Heads` row.

The new area must display four tiles:
1. `LAT%`
2. `EXP%`
3. `BE%`
4. `EXP2%`

Initial display values:
- `LAT%` = `—`
- `EXP%` = `—`
- `BE%` = `—`
- `EXP2%` = `—`

These values are placeholders only in Phase 1.

## Naming Rules
Use these exact labels:
- `LAT%`
- `EXP%`
- `BE%`
- `EXP2%`

Do not rename them to Expected LAT, Expected, BE LAT, BE Expected, EXP LAT, or EXP BE.

## UI Placement
The new metric row must appear after the existing stat row that currently contains `% Req` and `Heads`.

Recommended structure:

```text
Current:
[ % Req ] [ Heads ]

Add below:
[ LAT% ] [ EXP% ]
[ BE%  ] [ EXP2% ]
```

For narrow mobile screens, use a 2-column grid.
For wider screens, a 4-column grid is acceptable only if it does not squeeze or break the current layout.

## Styling Rules
Reuse the current design language:
- Same card background
- Same border style
- Same font family
- Same rounded tile style
- Same compact spacing

Do not introduce a new design system.
Do not add animations.
Do not add glow effects.
Do not add modal windows.
Do not add new popups.

## DOM / ID Requirements
Each LOB must have stable IDs for the four new values.

Recommended value IDs:
```text
latPct1 / expPct1 / bePct1 / exp2Pct1
latPct2 / expPct2 / bePct2 / exp2Pct2
latPct3 / expPct3 / bePct3 / exp2Pct3
```

Recommended tile IDs:
```text
latTile1 / expTile1 / beTile1 / exp2Tile1
latTile2 / expTile2 / beTile2 / exp2Tile2
latTile3 / expTile3 / beTile3 / exp2Tile3
```

Do not reuse existing IDs.
Do not rename existing IDs.

## JavaScript Requirements
Add safe render helpers only.

Recommended helper:
```text
renderLatBePlaceholders(id)
```

Behavior in Phase 1:
- Writes `—` into LAT%, EXP%, BE%, and EXP2%.
- Does not read or change current REQ values.
- Does not read or change current IN/OUT values.
- Does not change `% Req`.
- Does not change `Heads`.
- Does not change AUTO REQ.
- Does not write localStorage.

Call it from `refreshRow(id)` only after the existing current values are rendered.

## Acceptance Criteria
Phase 1 is complete only if:
- Current parser still works.
- Current IN/OUT counts still work.
- Current manual IN +/- still works.
- Current manual OUT +/- still works.
- Current REQ input still updates `% Req`.
- Current Heads calculation remains unchanged.
- Current AUTO REQ remains unchanged.
- New LAT% / EXP% / BE% / EXP2% tiles are visible.
- New tiles show `—`.
- No console errors.
- No broken IDs.
- No layout overflow on 360px mobile width.
- No existing UI behavior changes.
