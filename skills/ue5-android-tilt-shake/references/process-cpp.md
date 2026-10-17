# The detector as C++

Companion to `SKILL.md` section 8. By Alex Glonti · Nomad Devs (nomaddevs.club). MIT licence.

This is a sketch for a C++ project; it has been written from the tested Blueprint logic but was **not compiled** as a separate module. Treat it as a starting point and compile-check it. API names (`PlayDynamicForceFeedback`, `GetInputMotionState`, and `SetMotionControlsEnabled`, the C++ setter of `bEnableMotionControls`; the Blueprint node is "Set Enable Motion Controls") are `APlayerController` members; **verify** signatures on your engine version.

## Detector (no engine state, easy to unit test)

```cpp
// MotionGestureDetector.h
#pragma once
#include "CoreMinimal.h"

enum class EMotionGesture : uint8 { None, TiltPositive, TiltNegative, Shake };

struct FMotionTuning
{
    int32 TiltAxis = 2;            // 0 X, 1 Y, 2 Z of RotationRate
    float TiltSign = -1.f;         // flip if the phone shows swapped directions
    float TiltThreshold = 2.5f;    // rad/s
    int32 TiltSamples = 2;         // consecutive timer ticks
    float TiltDelay = 0.25f;       // pending window, seconds
    float TiltMaxAccel = 3.0f;     // g, cancel the tilt above this
    float TiltCooldown = 1.0f;
    float ShakeThreshold = 1.8f;   // g
    float ShakeWindow = 0.5f;
    int32 ShakeStrokes = 3;
    float ShakeCooldown = 1.0f;
};

struct FMotionGestureDetector
{
    FMotionTuning Tune;

    // Returns the gesture to act on, or None. Call from a ~30 Hz timer after your gates.
    EMotionGesture Process(double Now, const FVector& Rate, const FVector& Grav, const FVector& Acc);
    void Reset();           // call whenever the gate closes or the feature is toggled
    bool InCooldown(double Now) const { return Now < CoolUntil; }

private:
    int32 Run = 0, RunDir = 0, PendDir = 0, ShakeCount = 0;
    double PendTime = 0, ShakeStart = -100.0, CoolUntil = 0;
    double PendMaxAcc = 0.0;
    FVector ShakeDir = FVector::ZeroVector;
    EMotionGesture Fire(double Now, EMotionGesture G, const FVector& Rate, const FVector& Grav);
};
```

```cpp
// MotionGestureDetector.cpp
#include "MotionGestureDetector.h"

void FMotionGestureDetector::Reset()
{
    Run = RunDir = PendDir = ShakeCount = 0;
}

EMotionGesture FMotionGestureDetector::Fire(double Now, EMotionGesture G, const FVector& Rate, const FVector& Grav)
{
    CoolUntil = Now + (G == EMotionGesture::Shake ? Tune.ShakeCooldown : Tune.TiltCooldown);
    UE_LOG(LogTemp, Log, TEXT("MotionRate x=%.2f y=%.2f z=%.2f g=(%.2f,%.2f,%.2f) axis=%d ev=fire"),
        Rate.X, Rate.Y, Rate.Z, Grav.X, Grav.Y, Grav.Z, Tune.TiltAxis);
    Reset();
    return G;
}

EMotionGesture FMotionGestureDetector::Process(double Now, const FVector& Rate, const FVector& Grav, const FVector& Acc)
{
    const double Gm = Grav.Size();
    if (Gm < 0.01) return EMotionGesture::None;          // sensors not warmed up yet
    const double Ar = Acc.Size() / Gm;                       // linear acceleration in g, any engine mode

    // SHAKE: strokes above the threshold, alternating direction, inside a window
    if (Now - ShakeStart > Tune.ShakeWindow) ShakeCount = 0;
    if (Ar > Tune.ShakeThreshold)
    {
        const FVector D = Acc.GetSafeNormal();
        if (ShakeCount == 0) { ShakeCount = 1; ShakeStart = Now; ShakeDir = D; }
        else if (FVector::DotProduct(D, ShakeDir) < -0.2f) { ++ShakeCount; ShakeDir = D; }
    }
    if (ShakeCount >= Tune.ShakeStrokes) return Fire(Now, EMotionGesture::Shake, Rate, Grav);

    // TILT pending: wait, watch the acceleration peak, then fire or cancel
    if (PendDir != 0)
    {
        PendMaxAcc = FMath::Max(PendMaxAcc, Ar);
        if (Now - PendTime >= Tune.TiltDelay)
        {
            const int32 Dir = PendDir;
            const bool bOk = PendMaxAcc < Tune.TiltMaxAccel && ShakeCount < 2;
            PendDir = 0;
            if (bOk) return Fire(Now, Dir == 1 ? EMotionGesture::TiltPositive : EMotionGesture::TiltNegative, Rate, Grav);
        }
        return EMotionGesture::None;
    }

    // TILT detect
    const double Axis = Tune.TiltAxis == 0 ? Rate.X : (Tune.TiltAxis == 1 ? Rate.Y : Rate.Z);
    const double V = Axis * Tune.TiltSign;
    if (FMath::Abs(V) > Tune.TiltThreshold)
    {
        const int32 Dir = V > 0.0 ? 1 : 2;
        Run = (Dir == RunDir) ? Run + 1 : 1;
        RunDir = Dir;
        if (Run >= Tune.TiltSamples)
        {
            PendDir = Dir; PendTime = Now; PendMaxAcc = Ar; Run = 0;
            UE_LOG(LogTemp, Log, TEXT("MotionRate x=%.2f y=%.2f z=%.2f g=(%.2f,%.2f,%.2f) axis=%d ev=pend"),
                Rate.X, Rate.Y, Rate.Z, Grav.X, Grav.Y, Grav.Z, Tune.TiltAxis);
        }
    }
    else Run = 0;
    return EMotionGesture::None;
}
```

## Hooking it up (PlayerController or a component)

Declare `MotionTimer`, `Detector`, `bMotionEnabled`, `bMatchRunning`, `OrderBus`/`EOrder` and `ShowCaption` in your PlayerController header; the names are placeholders.

```cpp
// BeginPlay
SetMotionControlsEnabled(true);   // also set bAllowIMU=True in the project settings
GetWorldTimerManager().SetTimer(MotionTimer, this, &AMyPC::MotionTick, 0.033f, true);

void AMyPC::MotionTick()
{
    const double Now = GetWorld()->GetTimeSeconds();
    float TX, TY; bool bPressed;
    GetInputTouchState(ETouchIndex::Touch1, TX, TY, bPressed);
    const bool bGateOpen = bMotionEnabled && bMatchRunning && !IsPaused() && !bPressed && !Detector.InCooldown(Now);
    if (!bGateOpen) { Detector.Reset(); return; }

    FVector Tilt, Rate, Grav, Acc;
    GetInputMotionState(Tilt, Rate, Grav, Acc);
    switch (Detector.Process(Now, Rate, Grav, Acc))
    {
        case EMotionGesture::TiltPositive: DoOrder(EOrder::Advance, /*bTilt*/true); break;
        case EMotionGesture::TiltNegative: DoOrder(EOrder::Stop,    true); break;
        case EMotionGesture::Shake:        DoOrder(EOrder::Retarget, false); break;
        default: break;
    }
}

void AMyPC::DoOrder(EOrder O, bool bTilt)
{
    const FString Result = OrderBus->Execute(O);             // your existing order system
    ShowCaption(FString::Printf(TEXT("%s: %s"), bTilt ? TEXT("TILT") : TEXT("SHAKE"), *Result));
    if (bTilt && bMotionHaptics)
    {
        PlayDynamicForceFeedback(1.0f, 0.07f, true, true, true, true, EDynamicForceFeedbackAction::Start);
    }
    UE_LOG(LogTemp, Log, TEXT("MotionGesture gesture=%d -> %s haptic=%d"), (int32)O, *Result, (bTilt && bMotionHaptics) ? 1 : 0);
}
```

The Settings UI and caption widget are yours; the skill only needs a `bMotionEnabled` flag and a Settings row that calls `SetMotionControlsEnabled(bMotionEnabled)` plus `Detector.Reset()`.

## Unit-test idea

Because `FMotionGestureDetector` has no engine dependency beyond vectors, you can feed it the same timeline as the PIE script in a simple automation test: loop `Now` in 0.033 s steps, build the vectors for each stage and `check` the returned gesture and the timestamps from `test-plan.md`.
