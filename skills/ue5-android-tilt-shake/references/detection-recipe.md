# Detection recipe in detail

Companion to `SKILL.md` section 4. By Alex Glonti · Nomad Devs (nomaddevs.club).

## State variables

| Variable | Type | Role |
|---|---|---|
| Run | int | consecutive samples above the tilt threshold in one direction |
| RunDir | int | 1 = positive (value > 0), 2 = negative |
| PendDir | int | 0 = none; 1 or 2 = tilt waiting out the pending window |
| PendTime | float | game time when pending started |
| PendMaxAcc | float | peak `|Accel|/|Gravity|` seen since pending started |
| ShakeCount | int | strokes counted in the current window |
| ShakeStart | float | game time of the first stroke (init to a large negative number so the first window is "expired") |
| ShakeDir | vector | normalised direction of the last counted stroke |
| CoolUntil | float | game time before which the timer function does nothing |

Tunables (class defaults, editable per project without graph changes): `MotionEnabled`, `TiltAxis`, `TiltSign`, `TiltThreshold`, `TiltSamples`, `TiltDelay`, `TiltMaxAccel`, `TiltCooldown`, `ShakeThreshold`, `ShakeWindow`, `ShakeCooldown`, `MotionHaptics`.

## Why each piece exists

- **Timer, not Tick.** A looping timer at 0.033 s gives a fixed sample interval, so "2 samples" means about 66 ms on every device. It also makes enable/disable one `SetTimerByFunctionName` pair. The sensor itself updates around 50 Hz; polling faster only re-reads the same value.
- **Consecutive samples.** A single over-threshold sample is a sensor spike or a bump. Two in a row in the same direction is a deliberate move. Any under-threshold sample, or a direction change, restarts the count.
- **Pending window instead of firing at once.** A shake contains big rotation rates AND big linear accelerations. Firing the tilt immediately would fire a tilt for every shake. Waiting `TiltDelay` and watching the peak `ar` lets the shake detector win.
- **Two cancel conditions.** `PendMaxAcc < TiltMaxAccel` (linear acceleration stayed moderate) and `ShakeCount < 2` (at most one acceleration stroke seen). Both conditions are used.
- **Cooldown after fire.** The player rotates the phone back after a flick; without the cooldown the return flick fires the opposite command a moment later. 1 s was enough.
- **Gate resets the counters.** When the gate is closed (finger down, pause, menu) set Run, PendDir and ShakeCount to 0, otherwise stale state fires the moment the gate opens.
- **`|Gravity| < 0.01` return.** Right after enabling, before the first accelerometer sample, Gravity is zero and the ratio would divide by zero.
- **Ratio rather than absolute g.** In the default Android mode Acceleration and Gravity are both m/s^2; dividing one by the other gives g without caring which mode the engine is in.

## Order of checks inside Process

1. guard on gravity,
2. shake window expiry, shake stroke counting, shake fire (`>= 3`) and return,
3. if tilt is pending: update peak, maybe fire, return,
4. tilt detection from the chosen axis.

Shake is evaluated before tilt on purpose, so a shake in progress is never treated as a tilt candidate.

## Tuning notes

- Raise `TiltThreshold` if there are false triggers during normal play (2.5 rad/s is about a 140 degrees-per-second turn). Lower it if players report "I have to throw the phone". Check the `MotionRate ... ev=pend` log for what real flicks measure.
- If real flicks are cancelled, check `PendMaxAcc` in the log first. A hand flick can spike linear acceleration; our first 1.3 g limit was too tight, 3.0 g worked.
- Ordinary shakes with a thumb on the screen should not count: the gate already refuses while a finger is down.
- Different phones differ: a flagship's gyro is clean, a budget phone's is noisier. Test on at least two devices if you can.
- If you need more than two tilt commands, add the second axis as another pending slot; do not shorten the cooldown.

## What we did not do

- No sensor fusion, no Kalman filter: the raw gyro rate is already a clean signal for a flick.
- No per-player threshold calibration. It would be a good next step if players have very different hand speeds.
- No gesture on the Tilt (angle) output.
