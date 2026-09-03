## What's Changed

### ✨ New Features
- Added a **Taper blow / cone clean** option on the Advanced tab. Turn it on when the spindle's taper-blow port is plumbed off the drawbar valve, as on the Sienci kit. The tool change then closes the drawbar as soon as it lifts off the unloaded holder, so air is no longer venting during the whole move to the next slot, opens it again directly above the next holder, waits 0.8 s for the blow to clear the taper, and feeds the last 20 mm down slowly before clamping. On the Sienci ATC profile this is always on; on Generic it defaults to off, which keeps the previous behaviour.
