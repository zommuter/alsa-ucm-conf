# Bug Report: Speaker sink disappears when HDMI monitor connected

**Repository:** https://github.com/alsa-project/alsa-ucm-conf

**Title:** sof-hda-dsp: Speaker profile deprioritized when HDMI connected — Headphones profile selected despite no headphones plugged in

## Environment

- **Device:** HP OmniBook X Flip Laptop 14 (Intel Core Ultra 7 258V, Lunar Lake)
- **OS:** Manjaro Linux, kernel 6.18.12-1-MANJARO
- **Packages:**
  - alsa-ucm-conf 1.2.15.3-1
  - sof-firmware 2025.12.2-1
  - PipeWire 1.4.10
  - WirePlumber 0.5.13
- **Audio controller:** Intel Lunar Lake-M HD Audio Controller (00:1f.3)
- **Codec:** Realtek ALC245 (subsystem 103c:8d9f)
- **Card:** sof-hda-dsp / skl_hda_dsp_generic
- **Card longname:** `HP-HPOmniBookXFlipLaptop14_fm0xxx-Type1ProductConfigId-8D9F`

## Problem

When an HDMI monitor is connected, the Speaker sink completely disappears. The card has two mutually exclusive HiFi profiles (Speaker and Headphones share the same PCM `hw:sofhdadsp`):

```
HiFi (HDMI1, HDMI2, HDMI3, Headphones, Mic1, Mic2): priority 10300, available: yes
HiFi (HDMI1, HDMI2, HDMI3, Mic1, Mic2, Speaker):    priority 10200, available: yes
```

When HDMI is connected, both profiles become `available: yes` (because HDMI1 is available in both). WirePlumber selects the higher-priority Headphones profile (10300) — even though the Headphone Jack reports `off` and no headphones are plugged in. This removes the Speaker sink entirely.

Without HDMI connected, the Headphones profile is correctly `available: no` (no ports with availability groups report as available), so the Speaker profile is selected.

## Expected behavior

The Speaker sink should remain available regardless of HDMI connection state. When no headphones are plugged in, the Speaker profile should be preferred.

## Steps to reproduce

1. Clear WirePlumber saved state: `sed -i '/alsa_card.pci.*skl_hda_dsp/d' ~/.local/state/wireplumber/default-profile`
2. Boot or restart WirePlumber with no HDMI connected
3. Verify Speaker sink is present: `pactl list short sinks` — Speaker sink exists
4. Connect an HDMI monitor
5. `pactl list short sinks` — Speaker sink is gone, replaced by Headphones

Note: WirePlumber persists profile selection in `~/.local/state/wireplumber/default-profile`. If the Speaker profile was previously manually selected, the bug won't reproduce until this saved state is cleared.

## Diagnostic data

Full `alsa-info.sh` output is attached as `alsa-info-output.txt`.

### Jack status (HDMI connected, no headphones plugged in)

```
HDMI/DP,pcm=3 Jack: values=on    (monitor connected)
HDMI/DP,pcm=4 Jack: values=off
HDMI/DP,pcm=5 Jack: values=off
Headphone Jack:     values=off    (nothing plugged in)
Mic Jack:           values=off
Speaker Phantom Jack: values=on   (always on)
```

### Profile and port listing (HDMI connected — broken state)

```
Profiles:
  off: (priority: 0, available: yes)
  HiFi (HDMI1, HDMI2, HDMI3, Headphones, Mic1, Mic2): (priority: 10300, available: yes)
  HiFi (HDMI1, HDMI2, HDMI3, Mic1, Mic2, Speaker):    (priority: 10200, available: yes)
  pro-audio: (priority: 1, available: yes)

Active Profile: HiFi (HDMI1, HDMI2, HDMI3, Headphones, Mic1, Mic2)   ← WRONG

Ports:
  [Out] HDMI3:      priority 700, not available
  [Out] HDMI2:      priority 600, not available
  [Out] HDMI1:      priority 500, available         (monitor connected)
  [Out] Speaker:    priority 100, availability unknown
  [In]  Mic2:       priority 200, not available
  [In]  Mic1:       priority 100, availability unknown
  [Out] Headphones: priority 200, not available      ← jack is off, nothing plugged in
```

### Profile and port listing (no HDMI — working state)

```
Profiles:
  HiFi (HDMI1, HDMI2, HDMI3, Headphones, Mic1, Mic2): (priority: 10300, available: no)   ← correctly unavailable
  HiFi (HDMI1, HDMI2, HDMI3, Mic1, Mic2, Speaker):    (priority: 10200, available: yes)

Active Profile: HiFi (HDMI1, HDMI2, HDMI3, Mic1, Mic2, Speaker)   ← CORRECT
```

### UCM device dump

```
Verb.HiFi {
  Device.Headphones {
    PlaybackPriority 200
    PlaybackPCM "hw:sofhdadsp"
    JackControl "Headphone Jack"
  }
  Device.Speaker {
    PlaybackPriority 100
    PlaybackPCM "hw:sofhdadsp"
    # No JackControl — uses Speaker Phantom Jack (always on)
  }
}
```

### wpctl status (broken state, HDMI connected)

```
Audio
 ├─ Sinks:
 │      HDMI / DisplayPort 2 Output [vol: 1.00]
 │  *   HDMI / DisplayPort 1 Output [vol: 1.00]
 │      Headphones [vol: 1.00]                    ← present despite nothing plugged in
 │      HDMI / DisplayPort 3 Output [vol: 1.00]
 │                                                 ← Speaker is MISSING
```

## Root cause analysis

Speaker and Headphones share the same PCM device (`hw:sofhdadsp`), so PipeWire/ACP correctly generates separate mutually exclusive profiles. The UCM config in `ucm2/HDA/HiFi-analog.conf` assigns:

- **Headphones**: `PlaybackPriority 200`
- **Speaker**: `PlaybackPriority 100`

This results in profile priorities of 10300 (Headphones) vs 10200 (Speaker). The profile availability logic works as follows:

1. **Without HDMI**: All HDMI ports are `not available`, Headphones port is `not available`. The Headphones profile has no available ports (ports with `availability unknown` like Mic1 don't make the profile available), so the profile is `available: no`. Speaker profile is selected correctly.

2. **With HDMI**: HDMI1 becomes `available`. Since HDMI1 is shared by both profiles, both become `available: yes`. WirePlumber then selects purely by priority: Headphones (10300) wins over Speaker (10200), **ignoring that the Headphones port itself is `not available`**.

There are no `ConflictingDevices` declarations in the UCM config — the profile split is entirely due to the shared PCM.

## Suggested fix

**Swap `PlaybackPriority` values** in `ucm2/HDA/HiFi-analog.conf` so Speaker (200) > Headphones (100). This makes the Speaker profile the default (priority 10200 → 10300), while the Headphones profile would only be preferred when the Headphone Jack reports as plugged (WirePlumber boosts priority for profiles with available ports).

This seems correct as a general default: built-in speakers should be the default output, with headphones taking over only when detected via jack. When headphones are plugged in, the Headphone Jack becomes `available`, which should cause WirePlumber to re-evaluate and switch to the Headphones profile based on port availability — regardless of the lower base priority.

Note: headphone jack switching behavior has not been tested on this hardware (no 3.5mm cable available). Feedback from others with similar hardware welcome.

Alternatively, this could be addressed in WirePlumber's profile selection logic by weighting port availability more heavily — but the UCM priority swap is the simpler and more targeted fix.

## Workaround

Force the Speaker profile via WirePlumber:

```lua
-- ~/.config/wireplumber/wireplumber.conf.d/51-force-speaker.conf
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

Or manually: `pactl set-card-profile alsa_card.pci-0000_00_1f.3-platform-skl_hda_dsp_generic "HiFi (HDMI1, HDMI2, HDMI3, Mic1, Mic2, Speaker)"`

Note: Headphone behavior when actually plugging in headphones has not been tested with this workaround (no 3.5mm cable available at time of report).
