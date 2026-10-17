# ue5-android-tilt-shake

**A Claude Code skill for building gyroscope tilt and shake input on Android in Unreal Engine 5.**

Flick the phone to give an order, shake it to change targets. The sensors are easy to read and easy to get wrong: the axes depend on the display rotation, the gyro is silent unless one project setting is on, and a shake looks like a tilt unless you tell them apart. This skill is the playbook a Claude agent (or you) follows: what the engine really does to the sensor data, a detection recipe with starting numbers, feedback, a no-phone test, a phone test, and a checklist to remove the debug code afterwards.

By Alex Glonti · Nomad Devs (nomaddevs.club)

Status: **v0.1 (October 2026)**, written from one real implementation; it will be revised as more devices are tested.
Tested on UE 5.8 (Launcher build) with Blueprints, Android, landscape. Items marked "verify" in the skill depend on your engine version or phone.

## Who it is for

Unreal Engine 5 developers shipping to Android who want gesture input and use Claude Code for Blueprint or C++ work. It is a recipe and a set of facts, not a plugin: nothing runs until your agent reads it.

## What is inside

The skill is `skills/ue5-android-tilt-shake/SKILL.md`, plus four reference files.

1. When to use it and what you get
2. Prerequisites: `bAllowIMU`, `SetEnableMotionControls`, `GetInputMotionState` (Tilt, RotationRate, Gravity, Acceleration) and their units on Android
3. The coordinate-frame trap: device frame vs display rotation, `Android.UnifyMotionSpace`, landscape left vs right, and the rule to make axis and sign tunables with a one-line log
4. The detection recipe: fixed-rate timer, threshold plus consecutive samples, pending window, cooldown, shake cancel, and the numbers we ended on
5. Feedback: caption and a short haptic pulse, with the Android force-feedback facts
6. Test plan: a scripted PIE sensor feed, the phone pass with `adb logcat`, a removal checklist
7. UX lessons: roll vs pitch, learning curve, a Settings switch
8. A Blueprint pseudo-graph of the core function

References (long detail):
- `references/coordinate-frames.md` the engine trace and the geometry
- `references/detection-recipe.md` state variables and tuning notes
- `references/test-plan.md` the PIE feed and the phone pass in detail
- `references/process-cpp.md` the detector as C++

## 5-minute quick start

1. Install the skill folder (all projects, or one project):

   ```
   cp -r skills/ue5-android-tilt-shake ~/.claude/skills/
   # or: cp -r skills/ue5-android-tilt-shake <your-project>/.claude/skills/
   ```

   On Windows the user folder is `C:\Users\<you>\.claude\skills\`.
2. Start a Claude Code session in your project and try a read-only first request:

   > Use the ue5-android-tilt-shake skill. Check my Android settings and PlayerController for the prerequisites in section 2 and tell me what is missing. Do not change anything.

3. Then ask it to build the detector (Blueprint or C++), with the axis and sign as class defaults and the `MotionRate` log line. Run the scripted PIE feed, then a debug-signed build on a phone, and tune with the log.

## Things worth knowing before you start

- With Allow IMU Sampling off, every motion read is zero. No error.
- Roll about the screen normal is the same in both landscapes; pitch needs the engine's legacy remap, and that is derived on paper in this repo, not yet confirmed on hardware.
- Never judge the feel after one match. Our tester needed a few.

## Licence

MIT. See `LICENSE`.
