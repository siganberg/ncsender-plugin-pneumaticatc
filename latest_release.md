## What's Changed

### ✨ New Features
- **3D Probe Tool (T99)** on a new **Probe Tool** tab. The probe gets its own dock outside the rack and is picked up by the drawbar exactly like a rack tool — air-pressure check, release, descend onto the holder, clamp, seat — so `M6 T99` is fully automatic with no operator prompt. Choose **Cup** (lift straight off) or **Fork** (slide in and out) dock holding, with a slide axis independent of the rack's orientation, and optionally add **Load / Unload Probe G-code** to wake a wireless probe or switch its receiver on and off. Travel to and from the dock routes around the rack keepout and starts from where the spindle actually is, including straight after a rack tool has been put away. The probe follows your normal TLS strategy like any other tool.

  The feature is **off by default** — with it disabled, T99 keeps falling through to the manual-swap path exactly as before, so upgrading changes nothing until you turn it on.

- **`$PROBEDOCK`** console command — sends the spindle to the saved dock position at safe Z without touching the drawbar, along the same route a real `M6 T99` takes. Use it to confirm the dock coordinates before letting the tool change drive the air.
