# CLAUDE.md — alsa-ucm-conf

## Project overview
Fork of https://github.com/alsa-project/alsa-ucm-conf — ALSA UCM (Use Case Manager) configuration files for sound cards. UCM configs define audio profiles, devices, and routing for PipeWire/WirePlumber.

## Key paths
- `ucm2/HDA/HiFi-analog.conf` — Generic HDA analog I/O devices (Speaker, Headphones, Line, Mic). Defines PlaybackPriority values that determine profile selection.
- `ucm2/HDA/HiFi-mic.conf` — Generic HDA microphone devices
- `ucm2/Intel/sof-hda-dsp/` — Intel SOF HDA DSP card config
- `ucm2/conf.d/sof-hda-dsp/` — Device-specific overrides for sof-hda-dsp

## Hardware context
Testing on HP OmniBook X Flip 14 (Intel Lunar Lake, Realtek ALC245, sof-hda-dsp).

## Remotes
- `origin` — upstream https://github.com/alsa-project/alsa-ucm-conf
- `zommuter` — fork git@github.com:zommuter/alsa-ucm-conf.git

## How UCM priorities work
- `PlaybackPriority` per device sums into profile priorities (e.g. HDMI 500+600+700 + Speaker 200 + mics = 10300)
- When Speaker and Headphones share the same PCM, PipeWire/ACP splits them into separate mutually exclusive profiles
- WirePlumber selects the highest-priority `available` profile
- Profile availability is determined by port availability groups (jack detection)
