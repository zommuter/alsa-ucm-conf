# CLAUDE.md — Fix HP OmniBook X Flip speaker/HDMI audio profile conflict

## Problem
When an HDMI monitor is connected, PipeWire/WirePlumber switches to a profile that includes Headphones but not Speaker. The Speaker sink vanishes completely. The two relevant profiles are mutually exclusive:
- `HiFi (HDMI1, HDMI2, HDMI3, Headphones, Mic1, Mic2)` — priority 10300
- `HiFi (HDMI1, HDMI2, HDMI3, Mic1, Mic2, Speaker)` — priority 10200

## Hardware
- HP OmniBook X Flip 14, Intel Lunar Lake-M HD Audio (00:1f.3)
- Card: sof-hda-dsp / skl_hda_dsp_generic
- alsa-ucm-conf 1.2.15.3-1

## Investigation steps

### 1. Gather diagnostic data
```bash
# Full ALSA info dump
wget -O /tmp/alsa-info.sh https://www.alsa-project.org/alsa-info.sh
bash /tmp/alsa-info.sh --no-upload
# Save the output file

# Jack detection state (key diagnostic!)
amixer -c0 contents | grep -B1 -A3 "Jack"
# Check if "Headphone Jack" shows as connected when nothing is plugged in

# UCM config being used
alsaucm -c hw:0 dump text 2>&1 | head -200

# Profile listing
pactl list cards

# WirePlumber status
wpctl status

# Kernel messages for audio
dmesg | grep -i "sof\|hda\|jack\|codec" | tail -50
```

### 2. Locate the UCM config files
```bash
# The sof-hda-dsp UCM lives here:
ls /usr/share/alsa/ucm2/Intel/sof-hda-dsp/
# Key files: HiFi.conf, sof-hda-dsp.conf

# The generic HDA analog config (included by sof-hda-dsp):
cat /usr/share/alsa/ucm2/HDA/HiFi-analog.conf
# This is where Speaker and Headphones ConflictingDevices are defined

# Check for HP-specific overrides:
ls /usr/share/alsa/ucm2/conf.d/sof-hda-dsp/
```

### 3. Understand the conflict
The `ConflictingDevices` between Speaker and Headphones in `HiFi-analog.conf` causes PipeWire to generate **separate profiles** for each. This is correct for codecs where they share a DAC, but the profile auto-selection logic should prefer Speaker when no headphones are detected.

Check:
```bash
# Is the Headphone Jack falsely reporting as plugged?
amixer -c0 cget name='Headphone Jack'
# If this shows values=on with nothing plugged in, that's the root cause
```

### 4. Potential local fixes to test

#### Fix A: Force the Speaker profile via WirePlumber
```bash
mkdir -p ~/.config/wireplumber/wireplumber.conf.d/

# Try exact profile name (get from pactl list cards):
pactl set-card-profile alsa_card.pci-0000_00_1f.3-platform-skl_hda_dsp_generic "HiFi (HDMI1, HDMI2, HDMI3, Mic1, Mic2, Speaker)"
```

If that works, make it persistent:
```
# ~/.config/wireplumber/wireplumber.conf.d/51-force-speaker.conf
monitor.alsa.rules = [
  {
    matches = [
      { device.name = "~alsa_card.pci*skl_hda_dsp*" }
    ]
    actions = {
      update-props = {
        device.profile = "HiFi (HDMI1, HDMI2, HDMI3, Mic1, Mic2, Speaker)"
      }
    }
  }
]
```

Downside: headphones won't work until you manually switch profiles.

#### Fix B: Modify UCM to remove the conflict (experimental)
```bash
# BACK UP FIRST
sudo cp -r /usr/share/alsa/ucm2/HDA /usr/share/alsa/ucm2/HDA.bak

# Edit HiFi-analog.conf
sudo nano /usr/share/alsa/ucm2/HDA/HiFi-analog.conf

# Find the Speaker device section and remove/comment out ConflictingDevices:
# SectionDevice."Speaker" {
#   ...
#   ConflictingDevices [
#     "Headphones"    <-- remove this
#   ]
# Also remove Speaker from Headphones' ConflictingDevices

# Then clear ALSA state and restart:
sudo systemctl stop alsa-state
sudo rm /var/lib/alsa/asound.state
sudo systemctl start alsa-state
systemctl --user restart wireplumber pipewire
```

This may cause both speaker and headphones to play simultaneously (no auto-mute). Check if Auto-Mute Mode in alsamixer handles switching instead.

#### Fix C: Override profile priority via WirePlumber
Make the Speaker profile higher priority than Headphones:
```
# ~/.config/wireplumber/wireplumber.conf.d/52-speaker-priority.conf
monitor.alsa.rules = [
  {
    matches = [
      { device.name = "~alsa_card.pci*skl_hda_dsp*" }
    ]
    actions = {
      update-props = {
        device.profile = "HiFi (HDMI1, HDMI2, HDMI3, Mic1, Mic2, Speaker)"
        api.alsa.use-acp = true
      }
    }
  }
]
```

### 5. File upstream bug
If local fixes work, prepare a proper bug report for https://github.com/alsa-project/alsa-ucm-conf/issues:
- Include `alsa-info.sh` output (critical — maintainers need this)
- Include `amixer -c0 contents` output
- Include `pactl list cards` with and without HDMI
- Describe which local fix worked
- Reference that v1.2.14 added "HP Omnibook X14 support" — this variant may need separate handling

## Files to track
- `/usr/share/alsa/ucm2/Intel/sof-hda-dsp/HiFi.conf`
- `/usr/share/alsa/ucm2/HDA/HiFi-analog.conf`
- `/usr/share/alsa/ucm2/conf.d/sof-hda-dsp/`
- `~/.config/wireplumber/wireplumber.conf.d/`
- `/var/lib/alsa/asound.state`
