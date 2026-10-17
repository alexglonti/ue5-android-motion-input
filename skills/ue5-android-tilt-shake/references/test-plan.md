# Test plan: scripted PIE feed, phone pass, removal checklist

Companion to `SKILL.md` section 6. By Alex Glonti · Nomad Devs (nomaddevs.club).

## Part 1. Scripted PIE feed (no phone needed)

Motion events do not fire in the editor, and setting variables on the running PIE actor from outside is unreliable (in our setup every set on the PIE actor failed). So inject the fake data **from inside the Blueprint**.

### Build the feed
1. Back up the asset AND dump the function and variable list and the timer function's flow as the "before" record.
2. Add a few temporary variables in a clearly named category (for example `MotionDebug`): `DbgT0` (float), `DbgStage` (int), plus object references to anything you spawn for the test (two player-side actors, one or more targets).
3. Add a function `MotionDbgFeed`. It computes `t = now - DbgT0` (set `DbgT0` the first call) and builds three vectors from `t`, then calls the real `Process(Rate, Grav, Acc)`:
   - `Grav = (0, 0, 1)` always (so `ar = |Acc|`),
   - `Rate` = a vector with the chosen value on the tilt axis only (for example `MakeVector 0 ry 0` for pitch on Y),
   - `Acc` = `(ax, 0, 0)` or zero.
4. In the timer function replace the call that reads `GetInputMotionState` with `MotionDbgFeed`. Keep the gate logic in place so the gates are tested too.
5. Optional spawns at t = 1 s (a few actors near a known location) if the action needs targets; spawn them right before they are needed if they fight and die.

### The script (seconds after the match starts)

| t (s) | Feed | Expected | Why |
|---|---|---|---|
| 2.0-2.2 | slow roll: rate value 2.0 on the tilt axis | no order, no pending log | below threshold |
| 4.0-4.2 | flick: rate value 6.0 | order A at about 4.3-4.45 s, cooldown set, `ev=pend` then `ev=fire` log lines | threshold crossed for 2 samples, 0.25 s pending |
| 4.5-4.7 | return flick: -6.0 | no order | inside cooldown |
| 7.0-7.2 | flick: -6.0 | order B at about 7.3-7.45 s | opposite direction |
| 10.0-10.6 | `ax` alternates +2.5 / -2.5 every 0.1 s | shake action at about 10.2 s | 3 strokes, dot < -0.2 |
| 13.0-13.4 | flick 6.0 while `ax` = 3.5 | pending set, then cancelled, no order | peak 3.5 g is over `TiltMaxAccel` 3.0, so it was a shake (at the old 1.3 g limit 2.0 g was enough; with 3.0 a 2.0 g feed would fire) |
| 16.0-16.2 | flick again | order A again | state is clean after the cancel |

Expected log count for a full run: 4 `ev=pend` lines (4.0, 7.0, 13.0, 16.0 s) and 3 `ev=fire` lines (the 13.0 s pending is cancelled), plus the shake; none during the slow roll.

### Running it
- Check that PIE is not running before every compile or save. Never compile a Blueprint during PIE.
- Start PIE, capture the output log, stop PIE. Run the script two or three times; the first run can differ if your spawned actors die early.
- Take one screenshot of the caption.
- If the signs come out reversed in PIE, do **not** flip `TiltSign` because of it: the PIE feed defines the sign by convention, only the phone decides.
- If nothing fires, print the value chosen by `TiltAxis` once to check the axis select reaches the comparison.

## Part 2. Phone pass

Use a debug-signed test build (uninstall the old one first if signatures differ). Android must have motion enabled (`bAllowIMU=True`, section 2 of the skill).

```
adb devices
adb logcat -c
adb logcat -v time | grep -E "MotionRate|MotionGesture"     # Windows: findstr /R "MotionRate MotionGesture"
```

Run:
1. **20 flicks in each direction**, at a normal casual speed. Record hits per direction. Aim for 18 of 20 or better; if the weak direction misses more, lower the threshold slightly or work on how the gesture is introduced.
2. **2 minutes of normal play** (pan, tap, drag, put the phone down and pick it up, walk with it). Target: 0 false triggers. Any false trigger: read the last `MotionRate ev=pend` line to see what crossed the threshold.
3. **10 shakes.** Every shake should fire the shake action once, and never a tilt first.
4. **Settings switch off:** nothing fires. Back on: fires again.
5. **Rotate the phone 180 degrees** (landscape left to right) mid-session and repeat a few flicks.
6. Haptics: you must be able to feel one pulse per tilt fire and none on pending or shake (if that is your choice). Check the log field `haptic=1`.

Fix-ups without a rebuild (change class defaults, repackage only if your pipeline needs it): swap `TiltSign` when directions are reversed; change `TiltAxis` when the log shows another axis moves; adjust thresholds.

## Part 3. Removal checklist for the debug code

- [ ] Restore the timer function so it reads `GetInputMotionState` again (diff the flow dump against the "before" dump from step 1).
- [ ] Delete `MotionDbgFeed` and every variable in the debug category.
- [ ] Compile: clean.
- [ ] Compare counts of functions, variables and graphs with the numbers recorded before the test: identical.
- [ ] Re-read the class defaults: axis, sign, thresholds, cooldowns, `MotionEnabled`, `MotionHaptics`. Any switch you changed for the test is back where it was.
- [ ] Save the asset by explicit path, never "save all"; confirm it is not dirty afterwards.
- [ ] Do not save the test map.
- [ ] Keep the one-line `MotionRate`/`MotionGesture` logs: they cost nothing and the next phone test needs them.
