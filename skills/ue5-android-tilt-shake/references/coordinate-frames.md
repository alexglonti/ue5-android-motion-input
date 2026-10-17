# Coordinate frames: what the engine does to Android sensor data

Companion to `SKILL.md` section 3. By Alex Glonti · Nomad Devs (nomaddevs.club).

Sources read (paths are inside an engine install):
- UE 5.8: `Engine/Build/Android/Java/src/com/epicgames/unreal/GameActivity.java.template` (the Java activity template; read directly).
- UE 5.2: `Engine/Source/Runtime/ApplicationCore/Private/Android/AndroidInputInterface.cpp`, `QueueMotionData` (~lines 1744-1825) and the cvar `Android.UnifyMotionSpace` (~line 58). The Launcher build of 5.8 ships no `Private` C++ sources, so the C++ remap was read from 5.2. **verify** it on 5.8 (check the cvar exists with `Android.UnifyMotionSpace` in the console on a device, and test the axes on a phone). The 5.8 Java template and the 5.2 C++ agreed with each other in every call we traced.

## The chain, sensor to Blueprint

1. **Java `onSensorChanged`** (5.8 template ~lines 5525-5600). Registered only when `bAllowIMU` is true, with `SENSOR_DELAY_GAME` (~50 Hz class rate; it is a hint, not a guarantee). It needs an accelerometer AND a magnetometer (`accelerometer != null && magnetometer != null`).
   - accelerometer sample: copied, then low-pass filtered into a gravity estimate: `g = g_prev * 0.85 + sample * 0.15` (`SampleDecayRate = 0.85f`, ~line 280).
   - gyroscope sample: `current_rotation_rate = current_gyroscope`, i.e. the raw Android gyro vector, device frame, rad/s, counter-clockwise positive (Android convention).
   - linear acceleration: `current_acceleration = current_accelerometer - current_gravity`. Raw m/s^2.
   - tilt: from `getRotationMatrix` + `getOrientation`, reordered `(pitch, roll, azimuth)`. No `remapCoordinateSystem` call exists in the 5.8 template (we grepped for it), so Tilt is not corrected for display rotation here.
   - `AndroidThunkJava_PushSensorEvents` forwards `(tilt, rotation_rate, gravity, acceleration)` to native `nativeHandleSensorEvents` only when new data arrived.
2. **C++ `FAndroidInputInterface::QueueMotionData`** (5.2): applies the display-rotation fix.
   - `Android.UnifyMotionSpace = 0` (default, "legacy"): only RotationRate changes: **X negated in LandscapeLeft, Y negated in LandscapeRight**. Tilt, Gravity, Acceleration pass through raw.
   - `= 1`: Tilt, RotationRate, Gravity and Acceleration all re-oriented into Unreal's left-handed Z-up space per rotation, rotation rate in rad/s and acceleration/gravity divided by 9.8 into g.
   - `= 2`: like 1 but right-handed (swaps x and y) and the rotation rate is negated.
3. **`UPlayerInput::InputMotion` / `APlayerController::GetInputMotionState`** (5.2 `PlayerInput.cpp` ~460, `PlayerController.cpp` ~5615): the vectors are passed through unchanged. Nothing more to correct in Blueprint.

## Device frame cheat sheet (portrait reference)

- X: toward the right short edge.
- Y: toward the top long edge.
- Z: out of the screen toward the player.

Roll about Z (a steering-wheel turn) keeps the same meaning in every display rotation because Z always points out of the screen. That is why a roll gesture needs no landscape-left/right special case and why the legacy remap leaves Z alone.

## Why pitch is Y > 0 = "top edge away" in both landscapes (derivation)

Landscape means the phone is held with its long edge horizontal. Screen "up" for the player is then a device X direction, screen "right" a device Y direction.

- **LandscapeLeft** (Android `ROTATION_90`, the phone turned counter-clockwise from portrait): screen-up = device +X, screen-right = device -Y. The top edge of the screen is the device +X edge. Moving that edge away from the player is a movement toward -Z. A positive rotation about +Y (right-hand rule) moves a point on +X toward -Z. So `omega_Y > 0` = top edge away. The legacy remap does not touch Y in this orientation.
- **LandscapeRight** (`ROTATION_270`): screen-up = device -X, so the top edge is the device -X edge. Moving it away (toward -Z) needs omega_Y < 0 in the raw device frame. The legacy remap negates Y in LandscapeRight, giving `omega_Y > 0`.

Result: with `UnifyMotionSpace = 0`, RotationRate.Y > 0 means "top edge of the screen moving away from the player" in both landscape orientations, and the same flick maps to the same sign without any per-orientation code. The mapping of the engine's LandscapeLeft/LandscapeRight names to Android's `ROTATION_90`/`ROTATION_270` is also an assumption (**verify**). This was derived on paper from the source above and was **not** confirmed on hardware when this was written: confirm on a phone with the `MotionRate` log line before relying on it. If a phone shows the opposite, flip `TiltSign`.

Same derivation for roll: turning the phone clockwise (as the player sees it) rotates it about Z in the negative direction (Android gyro is counter-clockwise positive), so right-hand tilt = negative Z, hence `TiltSign = -1` for "right = positive command" when the axis is Z.

## Changing the axis without touching logic

Keep these as class-default (Blueprint) or UPROPERTY (C++) values:

| Tunable | Meaning |
|---|---|
| `TiltAxis` | 0 = RotationRate.X, 1 = .Y, 2 = .Z |
| `TiltSign` | +1 or -1, multiplies the chosen component |

and log one line the moment a flick crosses the threshold:

```
MotionRate x=0.4 y=5.9 z=-0.2 g=(0.1,-0.3,9.7) axis=1 ev=pend
```

First phone test checklist:
1. Flick the way you want to be "command A". Which component of the log line is largest and what sign? If it is the axis you chose and the command is wrong, flip the sign.
2. If the log never prints, the threshold was never crossed on the chosen axis: try the other axes, then check `bAllowIMU` and that the phone has a magnetometer.
3. Do the same for the opposite direction; the sign must be opposite.

## Display rotation changes mid-game

If you allow both landscape orientations (`SensorLandscape`), the player can rotate the phone 180 degrees and the engine's `GetDeviceOrientation()` changes. With the legacy remap the roll axis (Z) is unaffected, and the pitch axis (Y) is compensated. Test both by flipping the phone during a session. Your gate should also reset the run counters on an orientation change.
