# PROGRESS — Speaker/HDMI profile fix

## Completed
- [x] Diagnose Speaker sink disappearing when HDMI connected
- [x] Root cause: Headphones PlaybackPriority (200) > Speaker (100), HDMI hotplug makes both profiles available
- [x] Confirm Fix B (remove ConflictingDevices) not applicable — no ConflictingDevices exist
- [x] Implement fix: swap PlaybackPriority in ucm2/HDA/HiFi-analog.conf
- [x] Verify fix works standalone (Fix A disabled, saved state cleared, HDMI connected)
- [x] Gather diagnostic data (alsa-info, pactl, amixer, wpctl) for bug report
- [x] Complete bug report draft
- [x] Push fix to zommuter fork
- [x] Submit upstream PR #721

## Pending
- [ ] Test 3.5mm headphone jack behavior with the fix (need cable)
- [ ] Test Bluetooth headset default routing still works after fix
- [ ] Await upstream review/feedback on PR #721
- [ ] Remove WirePlumber workaround (51-force-speaker.conf) once fix is merged upstream and packaged
