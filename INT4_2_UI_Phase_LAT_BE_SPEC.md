# INT 4.2 LAT / BE Expansion — Phase 1 UI Specification

## 1. Authority and source baseline

This file is the authoritative specification for Phase 1 only.

Source application:

- Repository: `uxomar113/INT-X`
- Branch baseline: `main`
- File to modify: `INT4.2.html`
- Frozen baseline blob SHA: `161ac4d2054cc4279653f062477c0507704b2dcd`

Phase 1 prepares the existing stable interface for the later LAT and BE calculation phases. It must not implement calculation logic.

## 2. Non-negotiable scope boundary

Phase 1 may change only HTML and CSS required to display the future LAT / BE controls and metrics.

Do not modify, replace, simplify, rename, or remove any current working behavior.

The following systems are frozen:

- Snapshot paste input.
- Automatic parsing on input.
- Ten-second textarea self-clear.
- T2 / T1 / SP LOB detection and order.
- Current IN and OUT keyword classification.
- Current IN / OUT counting.
- Manual IN `+` and `-` adjustment.
- Manual OUT `+` and `-` adjustment.
- Current TOTAL calculation.
- Current REQ input.
- Current AUTO REQ panel, schedule parser, timing, and overwrite priority.
- Current `% Req` formula and display.
- Current `Heads` formula and display.
- AVAIL timer extraction and display.
- Per-LOB clear behavior.
- Master clear behavior.
- Current initialization and timers.

Do not touch `index.html` or any other application file.

## 3. Phase 1 objective

Prepare each existing LOB card for these four future metrics:

- `LAT%`
- `EXP%`
- `BE%`
- `EXP2%`

Also reserve a location for a future per-LOB `BE Req` input.

All four metric values must display `—` in Phase 1. They must not read or modify application state.

## 4. Required UI structure

Preserve the existing card hierarchy and visual direction. Do not replace the application with a new dashboard, modal system, drawer, tab system, or active-card architecture.

For every existing LOB card:

### 4.1 Existing REQ controls

Keep the current REQ line and all existing controls operational:

- `Req` label.
- Existing REQ number input.
- Existing `AUTO REQ` button.
- Existing clear button.

Add one compact, disabled Phase 1 field labelled `BE Req` near the existing REQ controls.

Required Phase 1 behavior for `BE Req`:

- It is a visual reservation only.
- It must be disabled.
- Its displayed value is blank with placeholder `—`.
- It must not have an input handler.
- It must not affect REQ, AUTO REQ, `% Req`, `Heads`, IN, OUT, TOTAL, or AVAIL.
- It will be enabled and wired only in Phase 3.

### 4.2 Existing metrics

Keep the existing metric blocks unchanged in meaning and behavior:

- IN
- OUT
- TOT
- `% Req`
- Heads

Do not rename `% Req` to LAT or change its formula.

### 4.3 New metric area

Add a compact metric area below the existing `% Req` and `Heads` row.

Use two columns so the card remains usable at the current narrow width:

First new row:

- `LAT%` with value `—`
- `EXP%` with value `—`

Second new row:

- `BE%` with value `—`
- `EXP2%` with value `—`

Every new value needs a stable per-LOB DOM id for later phases.

Recommended ids:

```text
latPct1 / latPct2 / latPct3
expPct1 / expPct2 / expPct3
bePct1 / bePct2 / bePct3
exp2Pct1 / exp2Pct2 / exp2Pct3
beReq1 / beReq2 / beReq3
```

Equivalent consistent ids are acceptable, but they must be predictable and unique.

## 5. Styling rules

- Match the current dark-card visual language.
- Reuse current CSS variables.
- Do not import another font, framework, icon set, or stylesheet.
- Do not remove the current font import in this phase.
- Do not create fixed overlays or popups.
- Do not use absolute positioning for the new metric tiles.
- Do not create horizontal page scrolling.
- Preserve the existing wrapper width and natural vertical page scrolling.
- The new metric area must remain readable at 320 px viewport width.
- Labels may use a smaller font, but values must not clip or overlap.
- Placeholder values use the current muted color.
- Do not add pass/fail colors to placeholder metrics in Phase 1.

## 6. JavaScript restrictions

Phase 1 must not add a LAT / BE calculation engine.

Do not add:

- Interval state.
- Half-hour timers.
- Weighted averages.
- Projection formulas.
- History arrays.
- localStorage.
- Manual LAT override.
- Manual BE override.
- Velocity.
- Operating-window logic.
- Stale or frozen modes.
- New AUTO REQ behavior.

The only JavaScript changes allowed are those strictly necessary to generate the new static markup inside the existing `buildRows()` flow.

The new placeholder nodes must not be written by `refreshRow()`, `updatePct()`, or AUTO REQ in Phase 1.

## 7. Regression acceptance criteria

Phase 1 passes only when all statements are true:

1. A snapshot still parses exactly as it did in the frozen baseline.
2. T2, T1, and SP still populate the same cards.
3. IN / OUT / TOT values are unchanged for identical input.
4. IN and OUT adjustment buttons still work.
5. REQ input still updates `% Req` and `Heads` immediately.
6. AUTO REQ still parses, displays, and applies schedule points exactly as before.
7. Ten-second input self-clear still keeps parsed values on screen.
8. Per-LOB clear still works.
9. Master clear still works.
10. AVAIL remains unchanged.
11. The four new metrics display `—` only.
12. `BE Req` is visible but disabled.
13. No existing function is removed or renamed.
14. No new console error is introduced.
15. No horizontal overflow or control overlap exists at 320 px, 400 px, and desktop width.

## 8. Phase 1 output contract

Codex must modify only `INT4.2.html`.

The Phase 1 pull request must contain:

- One changed application file: `INT4.2.html`.
- No Phase 2 or Phase 3 formulas.
- A concise summary of the UI-only changes.
- Static JavaScript syntax validation.
- HTML parse validation.
- `git diff --check` validation.

Phase 2 must not begin until Phase 1 is audited and approved.
