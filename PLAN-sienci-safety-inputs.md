# Plan: Sienci ATC safety-input parity

**Status:** partly SHIPPED 2026-09-24 — drawbar + tool-in-spindle are implemented.
Rack-mount detect and the spindle-nose holder probe are still unbuilt.

## What shipped (2026-09-24)

Not what this document originally proposed. The plan below put `_tc_input_db`
(drawbar) under "Explicitly out of scope" and proposed rack-mount detect as
Feature A; the actual request was the opposite pair — drawbar + tool sensor,
no rack detect. What went in:

| Config field | Sienci var | Checked where |
|---|---|---|
| `drawbarInput` | `_tc_input_db` | after every release (must read open) and after every clamp (must read closed) |
| `toolSensorInput` | `_tc_input_tis` | after a drop-off lift (spindle must be empty) and after a clamp (tool must be present) |

- Guards: `drawbarGuard()` / `toolGuard()` over a shared `sensorGuard()`, beside
  `pressureGuard()` in `commands.js`.
- Read convention: `L4` waits for LOW (asserted), `L3` waits for HIGH
  (de-asserted); a timeout is `#5399 == -1` either way, so there is exactly one
  fault test in the file. Invert in firmware with `$370`.
- Wired into all four automated paths: `buildUnloadTool`, `buildLoadTool`,
  `buildProbeUnload`, `buildProbeLoad` (cup and fork branches each).
  Manual-station paths are untouched — the operator is standing there with a
  dialog in front of them.
- Ordering rule, and the reason the checks are worth having: every check sits
  **before** the motion that depends on it. The drawbar-closed and tool-present
  checks precede the fork slide-out, so a tool the drawbar never gripped is
  never dragged sideways out of the fork.
- No unrolled retry (unlike pressure). A compressor recovers over time and is
  worth re-reading; a drawbar either actuated or it did not. The dialogs say
  plainly that the sensor is not re-read, and their button is "Continue", not
  "Re-check".
- o-words: `120/130/140/150` pressure (existing), `200/201` rack unload,
  `210/211/212` rack load, `220/221` probe unload, `230/231/232` probe load.
- `auxInputConfigured()` now gates all three guards including pressure. The old
  `settings.x < 0` test passed for `undefined`, which would have emitted a
  literal `M66 Pundefined` from any unsanitized settings object.

### Sienci pin map (authoritative)

Taken from Sienci's own ATC plugin, `Sienci-Labs/sienci-atci-plugin` — use it
rather than the `_tc_*` variable names, which are easy to misread:

| Aux input | Sensor | ATCI report code |
|---|---|---|
| `AUXINPUT0` | drawbar — "detects drawbar position" | `B` Drawbar Open |
| `AUXINPUT1` | tool — "detects if a tool is in the collet" | `L` Tool Loaded |
| `AUXINPUT2` | pressure | `P` Air Pressure |
| `AUXINPUT7` | rack — rack physically mounted | `I` Rack Installed |

The trap: `_tc_input_th` (tool HOLDER, rack-slot occupancy) does share pin 0
with the drawbar, and an in-repo comment said so. That is a different sensor
from tool-in-spindle, which has its own input at P1. An earlier pass here
conflated them and left the Sienci profile's tool pin unset; it is `1`.

`AUXINPUT7` (rack presence) is the input Feature A below would use.

## Still unbuilt

- Rack-mount detect (`_tc_input_rack`) — Feature A below, never started.
- Spindle-nose tool-holder probe (`_tc_input_th`) — needs hardware and the
  rack-probing routines; its placeholder UI row was deleted rather than left
  as a dead control.

---

## Original plan (2026-08-13), kept for the Sienci audit table

**Instruction to future assistant:** the audit below is still the record of
what Sienci's macros do — the source (`~/Downloads/microsd/`) has since been
deleted. Treat the scope sections as historical, not current.

---

## Context

Sienci's stock ATC macro set (source: `/Users/francis/Downloads/microsd/`, files `TC.macro`, `P501.macro`, `P502.macro`, `ATCI.macro`, `P100.macro`) uses five aux inputs, each gated by an enable flag:

| Sienci var | Sensor | Where Sienci uses it |
|---|---|---|
| `_tc_input_pres` | air-pressure switch | `G65 P501` before every unload/load |
| `_tc_input_rack` | rack-mount detect | `TC.macro:59` — decides in-rack routing vs passthrough |
| `_tc_input_tis` | tool-in-spindle (pullstud sense) | `TC.macro:272 / 432` verify after unclamp / clamp; `P502.macro` pre-flight sync |
| `_tc_input_th` | tool-holder touch probe (spindle-nose sensor) | `TC.macro:212` occupancy check + `P301/302/510` rack probing |
| `_tc_input_db` | drawbar-open reed switch (on cylinder) | `TC.macro:252 / 415` post-aux verify |

We currently implement only the first (pressure). This plan closes rack-detect and tool-in-spindle. The other two need hardware most users don't have and stay unplanned.

## What ships (scope)

Two features, both toggled off by setting their aux input to `-1` — matches how `pressureInput` already works (`commands.js:112` `sanitizeAuxInput`; `commands.js:196` config sanitize; `commands.js:958` early-return).

### Feature A — Rack-mount detection

One `M66` read at the top of `buildToolChangeProgram()` (`commands.js:1149`). Wrong reading → dialog → M0.

- Config field: `rackInput` (aux picker, default `-1`).
- Emit site: at the very start of `buildToolChangeProgram`, before any other g-code assembly.
- Emit shape:
  ```
  M66 P<rackInput> L4 Q0.01
  o<N> if [#5399 EQ -1]
    (MSG, PLUGIN_PNEUMATICATC:RACK_NOT_MOUNTED)
    M0
  o<N> endif
  ```
- o-word base: `110` (pressureGuard already uses `120/130/140/150` — leave a gap).
- If `settings.rackInput < 0` → emit empty string (mirror `pressureGuard`'s early return at `commands.js:958`).

**Interpretation of polarity:** copy Sienci exactly. In `TC.macro:59` they treat `#5399 == -1` as "no rack present." Match that: `EQ -1` triggers the dialog. Operators invert with `$370` if their sensor is wired the other way.

### Feature B — Tool-in-spindle verify (post-drawbar-action only)

One helper function `tisGuard(settings, oNum, expect)`. Called at four sites — the two rack-path drawbar transitions and the two manual-path ones — to catch a drawbar that mechanically failed.

- Config field: `toolInSpindleInput` (aux picker, default `-1`).
- Helper signature:
  ```js
  function tisGuard(settings, oNum, expect /* 'empty' | 'gripped' */) {
    if (settings.toolInSpindleInput < 0) return '';
    const read = `M66 P${settings.toolInSpindleInput} L4 Q0.5\n    G4 P0.1`;
    // Sienci convention (TC.macro:272 unload-verify, TC.macro:432 load-verify):
    //   #5399 == -1  →  sensor didn't trip within the wait   →  "empty spindle"
    //   #5399 != -1  →  sensor tripped                        →  "tool present"
    // So:
    //   expect === 'empty'   → FAULT when #5399 NE -1   → dialog TOOL_STILL_GRIPPED
    //   expect === 'gripped' → FAULT when #5399 EQ -1   → dialog TOOL_FAILED_TO_SEAT
    const badCondition = expect === 'empty' ? 'NE -1' : 'EQ -1';
    const msgId = expect === 'empty' ? 'TOOL_STILL_GRIPPED' : 'TOOL_FAILED_TO_SEAT';
    return `
      ${read}
      o${oNum} if [#5399 ${badCondition}]
        (MSG, PLUGIN_PNEUMATICATC:${msgId})
        M0
      o${oNum} endif`.trim();
  }
  ```
- Emit sites — four, all in `commands.js`:

  | Site | File location (current line, approx) | Existing anchor | New guard call |
  |---|---|---|---|
  | Rack unload — after unclamp + pressure check | `commands.js:1028` inside `buildUnloadTool` (fork path). Also inside the Cup branch of `buildUnloadTool` near `pressureGuard(settings, 140)` sibling call at ~`1014`. | Right after each existing `${pressureGuard(settings, ...)}` **in the unload paths only** | `tisGuard(settings, 160, 'empty')` and `tisGuard(settings, 162, 'empty')` |
  | Rack load — after clamp | `commands.js` inside `buildLoadTool`, immediately after `${auxLineFor(settings, 'clamp')}${drawbarSeat}` — the Cup branch at ~`1109` and the Fork branch at ~`1122` | Right after the `G4 P0.5` that follows the clamp (before the pulloff / Z-safe move) | `tisGuard(settings, 164, 'gripped')` and `tisGuard(settings, 166, 'gripped')` |

  Interleave inside the existing template literal exactly the way `pressureGuard` is already threaded. Each site gets its own `oNum` — spread by `+2` to be safe (guard block uses one `o<N>` label, `+2` leaves headroom if any variant grows to two).

- o-word bases used across the plugin after this change: `110` rack, `120/130/140/150` pressure (existing), `160/162/164/166` TIS.

## Config UI wiring (`config.html`)

The Advanced tab already has a "Tool Sensors" card at `config.html:1511` labelled **Coming soon**, with two disabled selects: `pa-adv-input-tool` (line 1522) and `pa-adv-input-holder` (line 1532).

- **Reuse `pa-adv-input-tool`** for `toolInSpindleInput`. Update the label copy to "Tool In Spindle" and update the help tip to describe the pullstud sensor.
- **Do not** reuse `pa-adv-input-holder` — that placeholder was for `_tc_input_th` (spindle-nose touch probe), a separate feature we're not building. Delete the `pa-adv-input-holder` row entirely to avoid a dead control.
- **Add a new row** in the same card for `rackInput`, `id="pa-adv-input-rack"`, label "Rack Mount".
- **Remove the "Coming soon" `<sup>`** on the card title (`config.html:1512`); this card ships live.
- **Wire persistence** the same way `pressureInput` is:
  - `initialConfig` read at `config.html:2497` — add analogous `savedTis`, `savedRack` reads.
  - Save block at `config.html:2598` — add analogous `toolInSpindleInput` and `rackInput` fields into the config object, using the same "empty string → omit" pattern the pressure select uses.
- **Populate the dropdowns** from the same `[AUX IO:..]` controller report that already feeds `pa-adv-input-pres`. Grep for the population code in `config.html`; add both new selects to the same iteration (they share `.pa-adv-pin` class already).

## Manifest dialogs (`manifest.json`)

Add three entries to the `messages` block (after `PRESSURE_FAULT_UNVERIFIED` at line 38 is the cleanest anchor):

```json
"RACK_NOT_MOUNTED": {
  "title": "Rack Not Mounted",
  "message": "The rack-mount sensor is reporting no rack on the machine, so the tool change cannot safely continue.<br><br>Mount the ATC rack and click <em>\"Re-check\"</em> — the macro reads the sensor again and only carries on if it now reports mounted. Click <em>\"Abort\"</em> to stop the tool change.",
  "continueLabel": "Re-check"
},
"TOOL_STILL_GRIPPED": {
  "title": "Tool Did Not Release",
  "message": "The tool-in-spindle sensor still detects a tool after the drawbar was commanded open. The drawbar may not have actuated, or the pullstud is stuck.<br><br>Check air pressure, inspect the drawbar and pullstud, and free the tool manually if needed. Click <em>\"Re-check\"</em> to read the sensor again, or <em>\"Abort\"</em> to stop the tool change.",
  "continueLabel": "Re-check"
},
"TOOL_FAILED_TO_SEAT": {
  "title": "Tool Failed to Seat",
  "message": "The tool-in-spindle sensor doesn't detect a tool after the drawbar was commanded closed. The slot may be empty, the holder mis-seated, or the drawbar didn't grip.<br><br>Inspect the slot and holder, seat the tool by hand if needed, and click <em>\"Re-check\"</em> to read the sensor again. Click <em>\"Abort\"</em> to stop the tool change.",
  "continueLabel": "Re-check"
}
```

Copy tone/structure from `PRESSURE_FAULT` at `manifest.json:33` so all three fault dialogs feel like one family.

Note that "Re-check" on a Sienci-style ncSender dialog just resumes the program — control returns to grblHAL and re-executes the very next line. Because our guard is `M0` immediately *after* the M66 read, "Re-check" resumes past the M0 into the next block **without re-running the M66**. If future testing shows operators want a real re-read, wrap each guard in the same 3-pass unrolled pattern `pressureGuard` uses. Ship without it first.

## sanitizeConfig (`commands.js:190` area)

Add two lines beside the existing `pressureInput` line (`commands.js:196`):

```js
rackInput: sanitizeAuxInput(raw.rackInput),
toolInSpindleInput: sanitizeAuxInput(raw.toolInSpindleInput),
```

## Tests (`commands.test.js`)

Add these cases, mirroring the shape of the existing pressure-guard tests:

1. `rackInput === -1` → `buildToolChangeProgram` output contains no `M66 P` for the rack pin and no `RACK_NOT_MOUNTED`.
2. `rackInput === 3` → output starts (before any motion) with `M66 P3 L4 Q0.01`, an `o110 if [#5399 EQ -1]`, `(MSG, PLUGIN_PNEUMATICATC:RACK_NOT_MOUNTED)`, `M0`, `o110 endif`.
3. `toolInSpindleInput === -1` → no `TOOL_STILL_GRIPPED` / `TOOL_FAILED_TO_SEAT` anywhere in output.
4. `toolInSpindleInput === 1`, unload path (any tool → 0) → output contains `M66 P1 L4 Q0.5` followed by `o160 if [#5399 NE -1]` + `TOOL_STILL_GRIPPED` in the unload block only.
5. `toolInSpindleInput === 1`, load path (0 → any tool) → output contains `M66 P1 L4 Q0.5` followed by `o164 if [#5399 EQ -1]` + `TOOL_FAILED_TO_SEAT` in the load block only.
6. `toolInSpindleInput === 1`, swap (Tm → Tn) → both `TOOL_STILL_GRIPPED` (unload leg) and `TOOL_FAILED_TO_SEAT` (load leg) appear, in that order.
7. Site placement: `TOOL_STILL_GRIPPED` appears **after** the `${pressureGuard(settings, 140)}` (or 130 for cup) block, not before it. `TOOL_FAILED_TO_SEAT` appears **after** the `${auxLineFor(settings, 'clamp')}${drawbarSeat}` block.

## Total footprint

- **2 new config fields** wired end-to-end (UI + sanitizeConfig).
- **1 new emit** at the top of `buildToolChangeProgram` (rack).
- **4 new emit call sites** inside `buildUnloadTool` (2, one per hold style) and `buildLoadTool` (2, one per hold style) (TIS).
- **3 new manifest messages** (`RACK_NOT_MOUNTED`, `TOOL_STILL_GRIPPED`, `TOOL_FAILED_TO_SEAT`).
- **~7 new tests** in `commands.test.js`.

## Explicitly out of scope (do not add)

- `_tc_input_db` — drawbar-open reed-switch verify (`TC.macro:252 / 415`).
- `_tc_input_th` — spindle-nose tool-holder probe (`TC.macro:212`, `P301/302/510`).
- 3-pass unrolled retry for either new guard (that pattern only lives on pressure because compressor recovery takes real time; a drawbar either grips or it doesn't).
- P502-equivalent pre-flight sync check (comparing `_current_tool` with sensor state on every M6). If someone wants it later, ship it as an explicit `sync-spindle-state` plugin command rather than baking it into `buildToolChangeProgram`.
- "Passthrough" fallback that rewrites in-rack routing to manual dialogs when the rack sensor reports absent. Our version aborts with a dialog and expects the operator to use the plugin's existing Manual Tool command.
- Rack re-check inside individual pickups (`P300.macro:31`). Single top-of-macro check is enough.
- Any change to `_tc_input_th`-related UI other than deleting the placeholder row at `config.html:1527-1535`.

## Decision gate before implementing

The user was on the fence when this plan was written. Real triggers to un-fence:

1. A customer hits a "would have caught it" event on the field (rack forgotten, drawbar failed, tool dropped mid-swap).
2. We add one of these sensors on our own machine and want to use it in anger.

Absent (1) or (2), keep this doc as-is and ship nothing. When the trigger arrives, re-read this doc top to bottom and implement without re-scoping.
