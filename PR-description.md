## Summary

On a pneumatic spindle the 3D probe could not be handled automatically. `T99` sits above any rack capacity, so it fell through to the manual-swap path and asked the operator to insert the probe by hand. But the drawbar can pick up a probe just as well as it picks up a cutter — the probe only needs a station of its own.

This adds a dedicated probe dock outside the rack. `M6 T99` now runs the same pneumatic sequence a rack tool gets, with no operator dialog on either the pick-up or the put-back.

The feature mirrors the [RapidChange ATC plugin's](https://github.com/siganberg/ncsender-plugin-rapidchangeatc) probe tool — same tool number, same enable toggle and Load/Unload G-code fields — so a job written for one machine runs on the other. The difference is that here the drawbar does the work instead of the operator.

**It is off by default.** With `addProbe` false, `T99` keeps falling through to the manual-swap path exactly as before.

## What the operator gets

A new **Probe Tool** tab in the config dialog:

| Setting | Notes |
|---|---|
| Enable Probe Tool | Registers T99 with the app and syncs `tool.probe` in `/api/settings`. Off by default |
| Dock X/Y/Z + Grab current | Where the probe rests. Z is the **engaged** height — same meaning as the rack's Engagement Z |
| Dock Holding | `Cup` (spindle lifts straight off) or `Fork` (lateral slide to engage / disengage) |
| Slide Axis / Direction / Distance / Speed | Fork docks only |
| Load Probe G-code | Runs once the probe is clamped and registered, before the TLS routine — waking a wireless probe, switching on a receiver |
| Unload Probe G-code | Runs before the spindle leaves for the dock, while the probe is still held |

Plus `$PROBEDOCK`, a console command that parks over the dock along the exact route a real `M6 T99` takes, without actuating the drawbar. It is the way to confirm the saved coordinates before letting the air fire.

## The sequence

Load reuses the rack's drawbar timing verbatim — the same `DRAWBAR_OFFSET_MM` approach height and `DRAWBAR_FEEDRATE_MMPM` seat that keep the holder on its lip while the tang is pulled up:

```
G53 G0 X300 Y150      ; route to the dock, around the rack keepout
M64 P2                ; release (skipped if an unload already opened it)
G53 G0 Z-119          ; descend to dockZ + 1 mm
M65 P2                ; clamp
G53 G1 Z-120 F300     ; seat, overlapping the pneumatic pull-up
G53 G0 Z-5
M61 Q99
<Load Probe G-code>
<TLS routine>
```

Unload mirrors it. Taper blow is honoured on both paths, so on a Sienci-style kit where the blow port is teed off the drawbar valve, the drawbar re-closes at the dock rather than venting across the whole traverse home.

Fork docks add a `G1` slide: in at depth before releasing on unload, out after clamping on load — the same order the rack's fork slots use.

## Two things worth reviewing closely

**The dock is kept out of the rack's slot math, but not out of its keepout.** A 3D probe is taller and wider than a cutter and usually lives off to the side of the magazine, so none of `slotParFor` / `slotEntryPoint` / `rackEntrance` applies to it. Travel to and from the dock still routes around the rack via `routePoint` with `freeEdge: true` — no rack slot is being engaged, so the fork's sliding-side constraint is irrelevant and the edge is picked by geometry, the same reasoning `tlsExit` already uses.

**Routing starts from where the spindle actually is.** Straight after a rack tool is put away the spindle sits at that slot, not at the pre-M6 origin. Routing the following probe load from the stale origin computed a path from somewhere the spindle no longer was, and with the rack between the two that path could cut through the keepout over the other tools. `loadOrigin` resolves the real starting point for the probe paths.

That exposed a second issue. `routePoint` defaults its edge anchor to `from`, which is correct when the destination *is* the rack (`rackEntrance`), but wrong when leaving it: anchoring on a `from` still inside the perp band picks the edge by the rack's own slide direction and can walk the spindle out the far side, along the full width of the keepout, and back down the other side — roughly 240 mm of pointless travel on a 12-slot layout. `probeRoute` anchors on whichever endpoint is clear of the band instead.

That fix is deliberately scoped to `probeRoute` and does not touch `routePoint`, `pickEntryEdge` or `tlsExit`. The same reasoning arguably applies to some existing rack-exit paths, but changing shared routing is a bigger question than this PR, so I left it alone. Happy to open it separately if you think it is worth pursuing.

## Backward compatibility

Verified rather than assumed. I generated the full `M6` program from both `main` and this branch across a matrix of rack holding × orientation × direction × slide direction × TLS mode × taper blow × pressure sensor, every current/target tool pair including 0 and out-of-rack numbers, four origins and both stored-TLO states, plus standalone `$TLS`, `$H` and `$SLOT` navigation:

- **39,424 scenarios on a config with no probe keys** (what every existing install has on disk) — **0 differences**, byte-identical output.
- **1,200 non-T99 tool changes with the probe tool enabled** — **0 differences**. Turning the feature on does not disturb ordinary rack changes.

Config migration needs nothing: `buildInitialConfig` supplies every new key, and bad or missing values fall back rather than producing NaN geometry (covered by a test).

## Testing

`node --test commands.test.js` — **131 pass, 0 fail** (109 existing, 22 new).

The new tests cover dock geometry for both holding styles, the load and unload sequences including drawbar ordering and the absence of any `M0`, chained transitions in both directions between rack and dock, the keepout-safe routing claims above, `$PROBEDOCK`, the pressure guard applying to probe pickups, and the disabled-by-default fall-through.

## Files

| File | |
|---|---|
| `commands.js` | Dock geometry, load / unload sequences, tool-change integration, `$PROBEDOCK` |
| `config.html` | Probe Tool tab, wiring, `tool.probe` sync |
| `commands.test.js` | 22 new tests |
| `manifest.json` | 0.1.34 → 0.1.35 |
| `README.md`, `latest_release.md` | Docs |

---

Built against a Fork rack with a Cup dock; I do not have a Fork dock to test on hardware, so that path has had geometry and sequence review but not a real cut. Worth a careful first run with the air disconnected if anyone is using one.
