---
title: CANoe COM Automation via Python
impact: MEDIUM
impactDescription: Python COM automation enables scripted control of CANoe for testing and CI/CD pipelines
tags: capl, external, python, com, automation, ci-cd
---

## CANoe COM Automation via Python

Use Python's `win32com` to automate CANoe — loading configurations, controlling measurement, accessing signals, and running test modules programmatically.

**Connecting and controlling measurement:**

```python
import win32com.client
import time

app = win32com.client.Dispatch("CANoe.Application")
app.Open(r"C:\Projects\MyConfig.cfg")

measurement = app.Measurement
measurement.Start()
time.sleep(2)

if not measurement.Running:
    raise RuntimeError("Measurement failed to start")
```

**Reading and writing signal values:**

```python
bus = app.Bus
signal = bus.GetSignal(1, "EngineStatus", "EngineSpeed")

current_rpm = signal.Value
signal.Value = 3500
```

Channel index (first arg) is 1-based. Signal writes only take effect when the measurement is running and the node has transmit responsibility.

**Running test modules and collecting results:**

```python
test_env = app.Configuration.TestSetup.TestEnvironments.Item(1)
test_module = test_env.TestModules.Item(1)

test_module.Start()
while test_module.IsBusy:
    time.sleep(1)

verdict = test_module.Verdict  # 1=Pass, 2=Fail, 0=None
report_path = test_module.Report.FullName
```

**Headless execution for CI/CD:**

```bash
"C:\Program Files\Vector CANoe 17\Exec64\CANoe64.exe" -d "C:\Projects\MyConfig.cfg"
```

The `-d` flag runs CANoe in desktop (headless) mode. Combine with COM automation to start tests and wait for completion without GUI interaction.

**Error handling:**

```python
import pythoncom
from pywintypes import com_error

try:
    app = win32com.client.Dispatch("CANoe.Application")
except com_error:
    raise RuntimeError("CANoe is not installed or COM server unavailable")

try:
    app.Open(config_path)
    app.Measurement.Start()
    time.sleep(3)
    if not app.Measurement.Running:
        raise RuntimeError("Measurement did not start — check license or config")
except com_error as e:
    raise RuntimeError(f"COM call failed (timeout or disconnected): {e}")
finally:
    if app.Measurement.Running:
        app.Measurement.Stop()
```

Always verify `Measurement.Running` after `Start()` — license issues and config errors are silent. Wrap all COM calls in try/except for `com_error` to catch timeouts and RPC disconnects.

Reference: Vector CANoe Help — COM Interface / Python Examples
