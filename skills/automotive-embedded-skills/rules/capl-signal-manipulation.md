---
title: Signal Manipulation Library Functions
impact: HIGH
impactDescription: Ad-hoc signal manipulation leads to duplicated code, inconsistent behavior, and difficult-to-maintain test environments
tags: capl, signal, manipulation, ramp, oscillation, noise, step, library, shared
---

## Signal Manipulation Library Functions

Reusable CAPL library functions for generating signal patterns in simulation and test environments. These functions work in both CANoe simulation nodes and vTESTstudio test modules. Copy-paste ready — each function is self-contained.

---

### Ramp: Linear Increase/Decrease Over Time

**Incorrect (hardcoded ramp with magic numbers):**

```capl
variables
{
    int rampVal = 0;
}

on timer rampTimer
{
    rampVal = rampVal + 5;
    if (rampVal > 255)
        rampVal = 255;
    $EngineSpeed = rampVal;
    setTimer(rampTimer, 100);
}
```

**Correct (configurable ramp function with start, end, duration, step interval):**

```capl
variables
{
    /* SignalRamp state */
    float gRamp_currentValue;
    float gRamp_endValue;
    float gRamp_increment;
    int   gRamp_stepCount;
    int   gRamp_stepsRemaining;
    msTimer gRamp_timer;
    int   gRamp_intervalMs;
    char  gRamp_signalName[64];
}

/*
 * SignalRamp — Linear ramp from startVal to endVal over durationMs.
 *
 * Parameters:
 *   signalName   : system variable or signal path to drive
 *   startVal     : initial value at ramp start
 *   endVal       : target value at ramp end
 *   durationMs   : total ramp duration in milliseconds
 *   stepIntervalMs: time between each step update in milliseconds
 *
 * Usage:
 *   SignalRamp("EngineSpeed", 0.0, 8000.0, 5000, 50);
 *     -> ramps from 0 to 8000 over 5 seconds, updating every 50 ms
 */
void SignalRamp(char signalName[], float startVal, float endVal,
                int durationMs, int stepIntervalMs)
{
    strncpy(gRamp_signalName, signalName, elcount(gRamp_signalName));
    gRamp_currentValue = startVal;
    gRamp_endValue = endVal;
    gRamp_intervalMs = stepIntervalMs;

    if (stepIntervalMs <= 0 || durationMs <= 0)
    {
        write("SignalRamp: invalid duration (%d) or interval (%d)", durationMs, stepIntervalMs);
        return;
    }

    gRamp_stepCount = durationMs / stepIntervalMs;
    gRamp_stepsRemaining = gRamp_stepCount;

    if (gRamp_stepCount > 0)
        gRamp_increment = (endVal - startVal) / gRamp_stepCount;
    else
        gRamp_increment = 0.0;

    setSignal(gRamp_signalName, gRamp_currentValue);
    setTimer(gRamp_timer, gRamp_intervalMs);
}

on timer gRamp_timer
{
    if (gRamp_stepsRemaining > 0)
    {
        gRamp_currentValue += gRamp_increment;
        gRamp_stepsRemaining--;
        setSignal(gRamp_signalName, gRamp_currentValue);
        setTimer(gRamp_timer, gRamp_intervalMs);
    }
    else
    {
        setSignal(gRamp_signalName, gRamp_endValue);
        write("SignalRamp: complete — %s = %.2f", gRamp_signalName, gRamp_endValue);
    }
}
```

---

### Oscillation: Sine Wave

**Incorrect (approximated sine with fixed values):**

```capl
on timer sineTimer
{
    float val;
    val = 50 + 10 * sin(counter * 0.1);
    counter++;
    $Sensor1 = val;
    setTimer(sineTimer, 20);
}
```

**Correct (parameterized sine generator):**

```capl
variables
{
    /* SignalSine state */
    float gSine_amplitude;
    float gSine_frequency;
    float gSine_offset;
    float gSine_phaseRad;
    float gSine_phaseStep;
    int   gSine_intervalMs;
    msTimer gSine_timer;
    char  gSine_signalName[64];
    int   gSine_running;
}

/*
 * SignalSine — Continuous sine wave output.
 *
 * Parameters:
 *   signalName  : signal to drive
 *   amplitude   : peak deviation from offset (half of peak-to-peak)
 *   frequencyHz : oscillation frequency in Hz
 *   offset      : DC offset (center value of the wave)
 *   intervalMs  : update period in milliseconds (controls resolution)
 *
 * Output: offset + amplitude * sin(2*PI*frequency*t)
 *
 * Usage:
 *   SignalSine("SensorVoltage", 2.5, 1.0, 2.5, 10);
 *     -> 1 Hz sine, 0V to 5V, updated every 10 ms
 *
 * Stop: call SignalSineStop()
 */
void SignalSine(char signalName[], float amplitude, float frequencyHz,
                float offset, int intervalMs)
{
    strncpy(gSine_signalName, signalName, elcount(gSine_signalName));
    gSine_amplitude = amplitude;
    gSine_frequency = frequencyHz;
    gSine_offset = offset;
    gSine_intervalMs = intervalMs;
    gSine_phaseRad = 0.0;

    /* phase increment per step: 2*PI * freq * (interval/1000) */
    gSine_phaseStep = 2.0 * 3.14159265 * frequencyHz * (intervalMs / 1000.0);
    gSine_running = 1;

    setTimer(gSine_timer, gSine_intervalMs);
}

void SignalSineStop()
{
    gSine_running = 0;
    cancelTimer(gSine_timer);
}

on timer gSine_timer
{
    float val;
    if (!gSine_running) return;

    val = gSine_offset + gSine_amplitude * sin(gSine_phaseRad);
    setSignal(gSine_signalName, val);

    gSine_phaseRad += gSine_phaseStep;
    if (gSine_phaseRad > 2.0 * 3.14159265)
        gSine_phaseRad -= 2.0 * 3.14159265;

    setTimer(gSine_timer, gSine_intervalMs);
}
```

---

### Oscillation: Square Wave

**Incorrect (toggle without configurable timing):**

```capl
on timer squareTimer
{
    if (flag == 0)
    {
        $Signal1 = 5;
        flag = 1;
    }
    else
    {
        $Signal1 = 0;
        flag = 0;
    }
    setTimer(squareTimer, 100);
}
```

**Correct (parameterized square wave):**

```capl
variables
{
    float gSquare_amplitude;
    float gSquare_offset;
    int   gSquare_halfPeriodMs;
    int   gSquare_high;
    msTimer gSquare_timer;
    char  gSquare_signalName[64];
    int   gSquare_running;
}

/*
 * SignalSquare — Continuous square wave output.
 *
 * Parameters:
 *   signalName  : signal to drive
 *   amplitude   : half of peak-to-peak (wave goes offset±amplitude)
 *   frequencyHz : oscillation frequency in Hz
 *   offset      : DC center value
 *
 * Usage:
 *   SignalSquare("DigitalOut", 1.0, 5.0, 1.0);
 *     -> 5 Hz square wave toggling between 0.0 and 2.0
 */
void SignalSquare(char signalName[], float amplitude, float frequencyHz,
                  float offset)
{
    strncpy(gSquare_signalName, signalName, elcount(gSquare_signalName));
    gSquare_amplitude = amplitude;
    gSquare_offset = offset;
    gSquare_high = 1;
    gSquare_running = 1;

    if (frequencyHz <= 0.0)
    {
        write("SignalSquare: invalid frequency %.2f", frequencyHz);
        return;
    }
    gSquare_halfPeriodMs = (int)(500.0 / frequencyHz);

    setSignal(gSquare_signalName, gSquare_offset + gSquare_amplitude);
    setTimer(gSquare_timer, gSquare_halfPeriodMs);
}

void SignalSquareStop()
{
    gSquare_running = 0;
    cancelTimer(gSquare_timer);
}

on timer gSquare_timer
{
    if (!gSquare_running) return;

    if (gSquare_high)
    {
        setSignal(gSquare_signalName, gSquare_offset - gSquare_amplitude);
        gSquare_high = 0;
    }
    else
    {
        setSignal(gSquare_signalName, gSquare_offset + gSquare_amplitude);
        gSquare_high = 1;
    }
    setTimer(gSquare_timer, gSquare_halfPeriodMs);
}
```

---

### Oscillation: Triangle Wave

**Correct (parameterized triangle wave):**

```capl
variables
{
    float gTri_amplitude;
    float gTri_offset;
    float gTri_currentValue;
    float gTri_increment;
    int   gTri_intervalMs;
    msTimer gTri_timer;
    char  gTri_signalName[64];
    int   gTri_running;
    int   gTri_ascending;
}

/*
 * SignalTriangle — Continuous triangle wave output.
 *
 * Parameters:
 *   signalName  : signal to drive
 *   amplitude   : peak deviation from offset
 *   frequencyHz : full cycle frequency in Hz
 *   offset      : DC center value
 *   intervalMs  : update period in milliseconds
 *
 * Usage:
 *   SignalTriangle("ThrottlePos", 50.0, 0.5, 50.0, 20);
 *     -> 0.5 Hz triangle wave from 0 to 100, updated every 20 ms
 */
void SignalTriangle(char signalName[], float amplitude, float frequencyHz,
                    float offset, int intervalMs)
{
    float halfPeriodMs;

    strncpy(gTri_signalName, signalName, elcount(gTri_signalName));
    gTri_amplitude = amplitude;
    gTri_offset = offset;
    gTri_intervalMs = intervalMs;
    gTri_currentValue = offset - amplitude;
    gTri_ascending = 1;
    gTri_running = 1;

    if (frequencyHz <= 0.0 || intervalMs <= 0)
    {
        write("SignalTriangle: invalid parameters");
        return;
    }

    halfPeriodMs = 500.0 / frequencyHz;
    /* increment per step = full swing / steps-per-half-period */
    gTri_increment = (2.0 * amplitude) / (halfPeriodMs / intervalMs);

    setSignal(gTri_signalName, gTri_currentValue);
    setTimer(gTri_timer, gTri_intervalMs);
}

void SignalTriangleStop()
{
    gTri_running = 0;
    cancelTimer(gTri_timer);
}

on timer gTri_timer
{
    if (!gTri_running) return;

    if (gTri_ascending)
    {
        gTri_currentValue += gTri_increment;
        if (gTri_currentValue >= gTri_offset + gTri_amplitude)
        {
            gTri_currentValue = gTri_offset + gTri_amplitude;
            gTri_ascending = 0;
        }
    }
    else
    {
        gTri_currentValue -= gTri_increment;
        if (gTri_currentValue <= gTri_offset - gTri_amplitude)
        {
            gTri_currentValue = gTri_offset - gTri_amplitude;
            gTri_ascending = 1;
        }
    }

    setSignal(gTri_signalName, gTri_currentValue);
    setTimer(gTri_timer, gTri_intervalMs);
}
```

---

### Noise Injection: Random Offset Within Bounds

**Incorrect (unbounded random, no reproducibility):**

```capl
on timer noiseTimer
{
    $Sensor1 = random(1000);
    setTimer(noiseTimer, 50);
}
```

**Correct (bounded noise around a base value):**

```capl
variables
{
    float gNoise_baseValue;
    float gNoise_amplitude;
    int   gNoise_intervalMs;
    msTimer gNoise_timer;
    char  gNoise_signalName[64];
    int   gNoise_running;
}

/*
 * SignalNoise — Adds random noise to a base value.
 *
 * Parameters:
 *   signalName    : signal to drive
 *   baseValue     : center value
 *   noiseAmplitude: maximum deviation (±) from baseValue
 *   intervalMs    : update period in milliseconds
 *
 * Output: baseValue + random([-noiseAmplitude, +noiseAmplitude])
 *
 * Usage:
 *   SignalNoise("TempSensor", 85.0, 2.0, 50);
 *     -> outputs values in [83.0, 87.0] every 50 ms
 */
void SignalNoise(char signalName[], float baseValue,
                 float noiseAmplitude, int intervalMs)
{
    strncpy(gNoise_signalName, signalName, elcount(gNoise_signalName));
    gNoise_baseValue = baseValue;
    gNoise_amplitude = noiseAmplitude;
    gNoise_intervalMs = intervalMs;
    gNoise_running = 1;

    setTimer(gNoise_timer, gNoise_intervalMs);
}

void SignalNoiseStop()
{
    gNoise_running = 0;
    cancelTimer(gNoise_timer);
}

on timer gNoise_timer
{
    float noiseOffset;
    float val;
    int   randRange;

    if (!gNoise_running) return;

    /* random() returns 0..N-1; scale to [-amplitude, +amplitude] */
    randRange = (int)(gNoise_amplitude * 2000.0);
    if (randRange > 0)
        noiseOffset = (random(randRange) / 1000.0) - gNoise_amplitude;
    else
        noiseOffset = 0.0;

    val = gNoise_baseValue + noiseOffset;
    setSignal(gNoise_signalName, val);

    setTimer(gNoise_timer, gNoise_intervalMs);
}
```

---

### Step Function: Immediate Value Change at Trigger

**Incorrect (step embedded in unrelated handler):**

```capl
on message ControlMsg
{
    if (this.byte(0) == 1)
        $BrakeForce = 100;
}
```

**Correct (self-contained step function with trigger time):**

```capl
variables
{
    float gStep_newValue;
    msTimer gStep_timer;
    char  gStep_signalName[64];
}

/*
 * SignalStep — Applies an immediate value change after a delay.
 *
 * Parameters:
 *   signalName  : signal to drive
 *   triggerMs   : delay in milliseconds before the step occurs (0 = immediate)
 *   newValue    : value to set at the trigger point
 *
 * Usage:
 *   SignalStep("BrakeForce", 2000, 100.0);
 *     -> sets BrakeForce to 100.0 after 2 seconds
 *
 *   SignalStep("IgnitionState", 0, 1.0);
 *     -> sets IgnitionState to 1.0 immediately
 */
void SignalStep(char signalName[], int triggerMs, float newValue)
{
    strncpy(gStep_signalName, signalName, elcount(gStep_signalName));
    gStep_newValue = newValue;

    if (triggerMs <= 0)
    {
        setSignal(gStep_signalName, gStep_newValue);
        write("SignalStep: %s = %.2f (immediate)", gStep_signalName, gStep_newValue);
    }
    else
    {
        setTimer(gStep_timer, triggerMs);
    }
}

on timer gStep_timer
{
    setSignal(gStep_signalName, gStep_newValue);
    write("SignalStep: %s = %.2f", gStep_signalName, gStep_newValue);
}
```

---

### Sequence Playback: Predefined Value Array Over Time

**Incorrect (values inlined, not reusable):**

```capl
on timer seqTimer
{
    if (idx == 0) $Speed = 0;
    else if (idx == 1) $Speed = 30;
    else if (idx == 2) $Speed = 60;
    else if (idx == 3) $Speed = 100;
    idx++;
    setTimer(seqTimer, 500);
}
```

**Correct (array-driven sequence playback):**

```capl
variables
{
    /* SignalSequence state */
    float gSeq_values[64];
    int   gSeq_length;
    int   gSeq_index;
    int   gSeq_intervalMs;
    int   gSeq_loop;
    msTimer gSeq_timer;
    char  gSeq_signalName[64];
    int   gSeq_running;
}

/*
 * SignalSequence — Plays a predefined array of values at a fixed interval.
 *
 * Parameters:
 *   signalName  : signal to drive
 *   values[]    : array of float values to play back
 *   length      : number of entries in values[] (max 64)
 *   intervalMs  : time between each value step in milliseconds
 *   loop        : 1 = repeat from start after last value, 0 = stop after last
 *
 * Usage:
 *   float profile[5] = { 0.0, 30.0, 60.0, 100.0, 60.0 };
 *   SignalSequence("VehicleSpeed", profile, 5, 1000, 1);
 *     -> cycles through speed profile every 5 seconds
 */
void SignalSequence(char signalName[], float values[], int length,
                    int intervalMs, int loop)
{
    int i;

    strncpy(gSeq_signalName, signalName, elcount(gSeq_signalName));
    gSeq_intervalMs = intervalMs;
    gSeq_loop = loop;
    gSeq_index = 0;
    gSeq_running = 1;

    if (length > 64) length = 64;
    gSeq_length = length;

    for (i = 0; i < length; i++)
        gSeq_values[i] = values[i];

    if (gSeq_length <= 0)
    {
        write("SignalSequence: empty value array");
        return;
    }

    setSignal(gSeq_signalName, gSeq_values[0]);
    gSeq_index = 1;
    setTimer(gSeq_timer, gSeq_intervalMs);
}

void SignalSequenceStop()
{
    gSeq_running = 0;
    cancelTimer(gSeq_timer);
}

on timer gSeq_timer
{
    if (!gSeq_running) return;

    if (gSeq_index < gSeq_length)
    {
        setSignal(gSeq_signalName, gSeq_values[gSeq_index]);
        gSeq_index++;
        setTimer(gSeq_timer, gSeq_intervalMs);
    }
    else if (gSeq_loop)
    {
        gSeq_index = 0;
        setSignal(gSeq_signalName, gSeq_values[0]);
        gSeq_index = 1;
        setTimer(gSeq_timer, gSeq_intervalMs);
    }
    else
    {
        write("SignalSequence: playback complete for %s", gSeq_signalName);
        gSeq_running = 0;
    }
}
```

All functions follow consistent naming (`Signal<Pattern>`), use global state with `g<Pattern>_` prefixes for timer callbacks, include parameter validation, and provide stop mechanisms for continuous generators.

Reference: Vector CANoe CAPL Programming Guide — Signal Access, Timer Functions
