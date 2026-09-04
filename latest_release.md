## What's Changed

### ✨ New Features
- **Measure All Tools** on the TLS tab (available with the "Use tool library offset" strategy). Loads every tool that has a rack slot in slot order, probes each on the tool setter and saves the length to the tool library whether or not a value is already stored, then puts the last tool away and returns to where the machine started. A confirmation lists the tools and their stored lengths, the progress view shows each tool as it is measured with an estimated time remaining and a Cancel that feed-holds then soft-resets, and a summary shows the old and new length of every tool. Make sure all settings are configured and you have done at least one normal tool change before using it, and do not close or refresh the page while it runs.
