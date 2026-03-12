## Summary

- Swap `PlaybackPriority` in `ucm2/HDA/HiFi-analog.conf`: Speaker 100→200, Headphones 200→100

## Problem

On devices where Speaker and Headphones share the same PCM (e.g. `hw:sofhdadsp`), PipeWire/ACP generates separate mutually exclusive profiles. With the current priorities, the Headphones profile (10300) is preferred over the Speaker profile (10200) whenever both are `available`.

This causes the Speaker sink to **completely vanish** when an HDMI monitor is connected: the HDMI port becoming available makes both profiles `available: yes`, and WirePlumber selects the higher-priority Headphones profile — even when the Headphone Jack reports `off` (nothing plugged in).

Without HDMI, the Headphones profile is correctly `available: no` (no ports with availability groups report available), so the bug only manifests with HDMI connected.

## Fix

Swap the priorities so Speaker (200) > Headphones (100). Built-in speakers are the natural default output. When headphones are plugged in, WirePlumber detects the Headphone Jack becoming available and switches to the Headphones profile accordingly.

## Testing

Tested on HP OmniBook X Flip 14 (Intel Lunar Lake, Realtek ALC245, sof-hda-dsp) with PipeWire 1.4.10 and WirePlumber 0.5.13:
- HDMI connected, no saved WirePlumber state, no headphones plugged in
- **Before:** Active profile = Headphones, Speaker sink missing
- **After:** Active profile = Speaker, Speaker sink present

Note: Headphone jack switching (plugging in actual headphones) has not been tested on this hardware. The expectation is that WirePlumber's jack-based profile switching handles this regardless of base priority.
