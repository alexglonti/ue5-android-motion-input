---
name: ue5-android-tilt-shake
description: Build gyroscope tilt and shake input on Android in Unreal Engine 5 - enable the IMU, read GetInputMotionState correctly (device frame vs display rotation, Android.UnifyMotionSpace), detect a deliberate flick or shake without false triggers (fixed-rate timer, thresholds, pending window, cooldown, accel cancel), give caption and haptic feedback, test with a scripted sensor feed in PIE and then on a phone with adb logcat. Includes a Blueprint pseudo-graph and a C++ detector.
---

# Tilt and shake input for Unreal Engine 5 on Android

By Alex Glonti · Nomad Devs (nomaddevs.club). MIT licence.

Facts were learned on UE 5.8 (Launcher build), Android, one shipped touch game, landscape only (`SensorLandscape`). Anything marked **verify** depends on your engine version, phone or project and must be checked before you trust it. Numbers are **starting points**, not laws.

## 1. When to use it, what you get

Use it when a phone gesture should trigger a game action: "flick the phone right = advance, left = stop, shake = change targets". You get:
- a tilt flick detector (one axis, two directions) and a shake detector sharing one motion module;
- a gate list (when gestures must be ignored), a cooldown, and a cancel rule so a shake never fires a tilt first;
- feedback: an on-screen caption and a short haptic pulse when the action is registered;
- a way to test without a phone (scripted sensor feed in PIE) and a one-line log that lets the first phone test fix axis and sign without a rebuild;
- a Settings switch so players can turn it off.

Not covered: iOS (different sensor path, **verify** separately), VR controllers, continuous steering (this is for discrete gestures; for steering read Tilt or Gravity with a dead zone).

Design stance: route the gesture into the same order system your other input uses (we feed gestures into the same order bus as our other inputs). The gesture is only a trigger, not new game logic.

## 2. Prerequisites and what the sensors return

### 2.1 Switch the IMU on
- [ ] Project Settings > Platforms > Android > **Allow IMU Sampling** on. Ini: section `[/Script/AndroidRuntimeSettings.AndroidRuntimeSettings]`, `bAllowIMU=True`. It is written to the manifest as meta-data and read by the Java activity at start (`GameActivity.java.template` ~line 4582). With it off the listeners are not registered and motion reads return zeros, the quietest failure of the whole feature.
- [ ] At BeginPlay call the PlayerController's motion switch: Blueprint node "Set Enable Motion Controls", C++ `SetMotionControlsEnabled(true)` (setter of `bEnableMotionControls`, `PlayerController.h`). The Java side has a runtime enable (`AndroidThunkJava_EnableMotion`, ~line 7577) that registers the listeners, but we did not trace whether the C++ call reaches it on 5.8 (**verify**). Set the project switch AND call the node; do not rely on the runtime path alone.
- [ ] The engine handles sensor events only if the phone reports an accelerometer **and a magnetometer** (~line 5527). A phone without a magnetometer gives no motion data at all. Test on your lowest-end device.
- [ ] A Settings toggle calls the same switch with false (`SetMotionControlsEnabled(false)` in C++) when off (section 7).

### 2.2 Reading the sensors
`PlayerController > GetInputMotionState` returns Tilt, RotationRate, Gravity, Acceleration. Poll it from a timer (section 4). What each is on Android, from the UE 5.8 Java `onSensorChanged` (~lines 5525-5600):

| Output | What it is | Units (default mode) |
|---|---|---|
| RotationRate | raw gyroscope, device frame | rad/s, counter-clockwise positive about each axis |
| Gravity | accelerometer through a low-pass filter (decay 0.85) | m/s^2, not g |
| Acceleration | raw accelerometer minus that filtered gravity (linear acceleration) | m/s^2, not g |
| Tilt | Euler angles from `getRotationMatrix`/`getOrientation`, no display remap | radians |

Gravity and Acceleration share units, so **use the ratio** `|Acceleration| / |Gravity|` as "g". The numbers stay right whether or not `Android.UnifyMotionSpace` is on (it converts to real g).

Use RotationRate for flicks (the gyro, no filter lag) and Acceleration for shakes. Do not detect a flick from Tilt angles: the angle depends on how the phone is held and lags.

## 3. The coordinate-frame trap

Raw Android sensors use the **device frame**: X = short edge right (portrait), Y = long edge up, Z = out of the screen toward you. They do not rotate with the display. The engine applies a display-rotation fix that depends on the cvar (trace and geometry in `references/coordinate-frames.md`):

- `Android.UnifyMotionSpace = 0` (default, legacy): negates RotationRate **X** in LandscapeLeft and RotationRate **Y** in LandscapeRight. Nothing else is remapped; Gravity and Acceleration stay in the device frame.
- `1` or `2`: all four vectors are re-oriented per display rotation into one space, in g and rad/s. We shipped with the default 0. Verify behaviour on your build before switching.

Consequences:
- **Roll about the screen normal (Z)**, a steering-wheel turn, is identical in both landscapes and untouched by the legacy remap. Turn right (clockwise seen by the player) gives negative Z. It is the safest axis if you allow both landscape directions.
- **Pitch** (top edge toward or away from the face) is device Y. With the legacy remap, Y > 0 means "top edge away" in both landscapes. We derived that from the engine source and geometry; treat it as the expected default and confirm on the phone.
- Do not hard-code the axis. **Make the axis index and the sign class-default tunables** (`TiltAxis` 0/1/2, `TiltSign` +1/-1) **and ship a one-line log** printed only when a threshold is crossed:
  `MotionRate x=.. y=.. z=.. g=(gx,gy,gz) axis=N ev=pend|fire`
  The first phone test then needs only `adb logcat` and a defaults edit, not a logic rebuild. We changed the axis twice in one morning; both times it was a defaults edit.

## 4. The detection recipe

Run it on a **fixed-rate timer (about 30 Hz, 0.033 s), not Tick**, so detection does not depend on frame rate and can be switched off cleanly. Each timer call:

1. **Gate.** Do nothing, and reset the run counters, unless: feature enabled, match running, no tutorial or menu, game not paused, no finger on the screen (`GetInputTouchState`), cooldown over, and the input mode is one where the gesture makes sense.
2. Read `GetInputMotionState`, call `Process(Rate, Gravity, Accel)`.

`Process` (code in section 8, walk-through in `references/detection-recipe.md`):

- **Guard.** If `|Gravity|` is about zero, return. Never fire on the first sample.
- **g ratio** `ar = |Accel| / |Gravity|`.
- **Shake.** A sample with `ar > ShakeThreshold` is a stroke. A stroke counts when its direction (normalised Accel) has dot < -0.2 with the previous stroke. 3 strokes inside `ShakeWindow` = shake: fire, cooldown. (That is 2 direction reversals; "3 reversals" in loose talk.)
- **Tilt.** `v = Rate[TiltAxis] * TiltSign`. If `|v| > TiltThreshold` for `TiltSamples` consecutive samples in one direction, do not fire yet: enter **pending** with direction, time and current `ar`.
- **Pending window.** For `TiltDelay` seconds keep the peak `ar`. At the end fire only if the peak stayed under `TiltMaxAccel` and fewer than 2 shake strokes were seen. A shake rotates and jerks the phone, so this cancels the tilt and a shake never fires a tilt first.
- **Cooldown** after any fire. It also swallows the return flick when the player rotates back.

### Numbers we ended on (starting points, tune on your phone)

| Tunable | Value | Why |
|---|---|---|
| TiltThreshold | 2.5 rad/s | first guess 3.5 was too hard to reach; 2.5 still ignored walking and handling (inferred) |
| TiltSamples | 2 (66 ms) | one sample is a glitch, many is a slow move |
| TiltDelay | 0.25 s | long enough to see whether a shake follows |
| TiltMaxAccel | 3.0 g | first guess 1.3 cancelled real flicks; shakes go well above 3 (inferred) |
| TiltCooldown | 1.0 s | swallows the return flick |
| ShakeThreshold | 1.8 g | test feed used 2.5 g strokes; hand jitter stays low |
| ShakeWindow / strokes | 0.5 s / 3 | |
| ShakeCooldown | 1.0 s | |

Why we retuned 3.5 to 2.5 and 1.3 to 3.0 was not written down; it came from the first phone sessions (inference, not a recorded measurement). Re-tune with your own 20-flicks test. If the feature is on by default, prefer a high threshold to a hair trigger.

## 5. Feedback

- **Caption** when the action is registered, built from what the action did ("TILT: ADVANCE", "SHAKE: NEW TARGETS"), so the player learns the mapping. Put it where order captions already show, and check it is not hidden when a neighbouring feature is off.
- **Haptic pulse on fire only**, not on pending (the pulse means "registered"). On the PlayerController: `PlayDynamicForceFeedback(Intensity 1.0, Duration 0.07, all four motor flags true, Action Start)`. No asset needed. Fallback: a one-channel flat `ForceFeedbackEffect` asset played with `ClientPlayForceFeedback`.
- Android facts: force feedback reaches the phone vibrator; the merged manifest must contain `android.permission.VIBRATE` (add under Project Settings > Android > Extra Permissions if missing; read `Intermediate/Android/<arch>/AndroidManifest.xml` after a build). 70 ms is a tap, not a buzz; very short pulses may feel weak on some motors (**verify** on your device; we had not felt ours on a phone when this was written). PIE cannot vibrate, so prove the branch with a log field.
- Gate the pulse behind a class-default bool so it can be turned off without editing a graph. Decide shake separately; we gave it none.
- Log each fire: `MotionGesture gesture=N -> <action> haptic=0|1`.

## 6. Test plan

Detail in `references/test-plan.md`.

1. **Scripted PIE feed.** Motion events do not fire in the editor. Add a temporary debug function that builds fake `Rate`, `Gravity`, `Accel` from a timeline and calls `Process` directly in place of the sensor read:

| Time | Feed | Expect |
|---|---|---|
| 2.0-2.2 s | slow roll, 2.0 on the tilt axis | no order |
| 4.0-4.2 s | flick, 6.0 | order A at about 4.3 s |
| 4.5-4.7 s | return flick, -6.0 (in cooldown) | no order |
| 7.0-7.2 s | flick, -6.0 | order B at about 7.3 s |
| 10.0-10.6 s | accel alternating +/-2.5 g every 0.1 s | shake action |
| 13.0-13.4 s | flick with 3.5 g accel (at the old 1.3 g limit 2.0 g was enough; with 3.0 it would fire) | pending, cancelled, no order |
| 16.0-16.2 s | flick | order A again |

   Run it 2-3 times. The feed's sign is a convention; the phone decides the real direction.
2. **Phone pass.** Debug-signed test build, then `adb logcat -v time | grep -E "MotionRate|MotionGesture"` (`PrintString` and `UE_LOG` appear under the `UE` tag; **verify** your filter). Do 20 flicks each way, 2 minutes of normal play (pan, tap, walk) with 0 false triggers, 10 shakes, and the Settings switch off = nothing fires. Directions swapped: flip `TiltSign`. Nothing reacts: read `MotionRate` lines; if none, try `TiltAxis` 0 or 2, then re-check `bAllowIMU` and the magnetometer.
3. **Removal checklist for the debug code.** Delete the debug function and its variables; restore the timer function to the real sensor read; compile clean; function/variable counts equal the pre-test numbers; re-read the defaults (axis, sign, thresholds, switches); save; confirm the asset is clean. Keep the one-line log.

## 7. UX lessons

- Roll (left/right) and pitch (toward/away) both work. Our tester first called roll "kinda unintuitive and maybe even uncomfortable", asked for pitch, then went back to roll after a few matches and said it "intuitively offers direction". Plan for a few matches of learning before judging.
- Axis and sign as data made that switch a defaults edit. Build it that way from day one.
- Let players switch it off (a Settings row). If it is on by default, keep the threshold high. Reset detector state when toggled.
- Never fire on the first sample after enabling, a scene change or unpausing: reset run counters whenever the gate closes.
- Ignore gestures while a finger is down: a thumb sliding a phone produces the same rates.
- Name the action in the caption, teach the gesture once in a hint, not a manual.
- Mobile touch games are not worse than other platforms, they are different; motion is something only a phone has.

## 8. Core process function

Blueprint pseudo-graph (tunables are class defaults on the PlayerController or pawn):

```
Process(Rate, Grav, Acc):
  now = GetGameTimeInSeconds
  gm = VectorLength(Grav);  if gm < 0.01: return
  ar = VectorLength(Acc) / gm
  # SHAKE
  if now - ShakeStart > ShakeWindow: ShakeCount = 0
  if ar > ShakeThreshold:
      d = Normalize(Acc)
      if ShakeCount == 0: ShakeCount = 1; ShakeStart = now; ShakeDir = d
      else if Dot(d, ShakeDir) < -0.2: ShakeCount += 1; ShakeDir = d
  if ShakeCount >= 3: Fire(Shake); return
  # TILT pending
  if PendDir != 0:
      PendMaxAcc = Max(PendMaxAcc, ar)
      if now - PendTime >= TiltDelay:
          if PendMaxAcc < TiltMaxAccel and ShakeCount < 2: Fire(PendDir)
          PendDir = 0
      return
  # TILT detect
  v = Select(TiltAxis==0, Rate.X, Select(TiltAxis==1, Rate.Y, Rate.Z)) * TiltSign
  if Abs(v) > TiltThreshold:
      dir = (v > 0) ? 1 : 2
      Run = (dir == RunDir) ? Run + 1 : 1;  RunDir = dir
      if Run >= TiltSamples:
          PendDir = dir; PendTime = now; PendMaxAcc = ar; Run = 0
          Log("MotionRate x= y= z= g=(..) axis=N ev=pend")
  else: Run = 0

Fire(g): CoolUntil = now + (g == Shake ? ShakeCooldown : TiltCooldown)
         reset Run, PendDir, ShakeCount
         result = DoAction(g); Caption(result)
         if g is tilt and MotionHaptics: Pulse
         Log("MotionRate ... ev=fire"); Log("MotionGesture gesture=g -> result haptic=0|1")

MotionTick (looping timer 0.033 s): gates -> GetInputMotionState -> Process
```

A complete C++ version (struct, timer, haptic call) is in `references/process-cpp.md`.

## References

- `references/coordinate-frames.md` - engine trace, geometry per landscape, how to change the axis.
- `references/detection-recipe.md` - state variables, full walk-through, tuning notes.
- `references/test-plan.md` - scripted PIE feed in detail, phone pass, removal checklist.
- `references/process-cpp.md` - the detector as C++.
