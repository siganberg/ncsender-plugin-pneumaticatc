## What's Changed

### 🐛 Bug Fixes
- Fixed a false "Air Pressure Low" fault right after the drawbar opened. The check after a release now waits up to 2 seconds for pressure to recover from the brief dip while the cylinder fills, so a healthy air supply no longer pauses the tool change. Pressure that stays low still stops the job as before.

### 🔧 Improvements
- Removed the "Invert reading" toggle added in v0.1.29. The problem it was meant for was the timing above, not inverted wiring. If your switch really reads the other way round, invert the port with grblHAL's $370 setting.
