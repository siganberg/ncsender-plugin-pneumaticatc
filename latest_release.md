## What's Changed

### 🐛 Bug Fixes
- After the tool length set, the return to the job now leaves the keepout zone by the edge nearest to where the job was, instead of always using the fork's sliding side. On a Fork rack with the job area below the rack this removes a detour up to the far edge and back, about 240 mm of extra travel on a 12-slot Sienci rack. Slot entries and exits are unchanged.
