---
title: CANoe COM Automation via C#
impact: MEDIUM
impactDescription: C# COM interop enables type-safe CANoe automation integrated with .NET test frameworks
tags: capl, external, csharp, com, automation, dotnet
---

## CANoe COM Automation via C#

Use .NET COM interop to automate CANoe from C# — controlling measurement, accessing signals, and running test modules with full type safety.

**Setting up COM interop:**

Add the CANoe type library as a COM reference in your project: right-click References → Add COM Reference → select "Vector CANoe Type Library". This generates the `CANoe` interop namespace.

```csharp
using CANoe;

var app = new CANoe.Application();
app.Open(@"C:\Projects\MyConfig.cfg");
```

Alternatively, use late binding for environments where the type library isn't registered:

```csharp
dynamic app = Activator.CreateInstance(Type.GetTypeFromProgID("CANoe.Application"));
app.Open(@"C:\Projects\MyConfig.cfg");
```

**Measurement control:**

```csharp
app.Measurement.Start();
Thread.Sleep(2000);

if (!app.Measurement.Running)
    throw new InvalidOperationException("Measurement failed to start");

// ... perform test actions ...

app.Measurement.Stop();
```

**Signal access:**

```csharp
var signal = app.Bus.GetSignal(1, "EngineStatus", "EngineSpeed");

double rpm = signal.Value;
signal.Value = 3500.0;
```

**Running test modules and retrieving results:**

```csharp
var testEnv = app.Configuration.TestSetup.TestEnvironments[1];
var testModule = testEnv.TestModules[1];

testModule.Start();
while (testModule.IsBusy)
    Thread.Sleep(500);

int verdict = testModule.Verdict;  // 1=Pass, 2=Fail, 0=None
string reportPath = testModule.Report.FullName;
```

**Integration with NUnit:**

```csharp
[TestFixture]
public class CanoeIntegrationTests
{
    private CANoe.Application _app;

    [OneTimeSetUp]
    public void Setup()
    {
        _app = new CANoe.Application();
        _app.Open(@"C:\Projects\MyConfig.cfg");
        _app.Measurement.Start();
        Thread.Sleep(2000);
        Assert.That(_app.Measurement.Running, Is.True, "Measurement must be running");
    }

    [Test]
    public void EngineSpeed_WhenThrottleApplied_ShouldIncrease()
    {
        var throttle = _app.Bus.GetSignal(1, "DriverInput", "ThrottlePosition");
        var speed = _app.Bus.GetSignal(1, "EngineStatus", "EngineSpeed");

        throttle.Value = 80.0;
        Thread.Sleep(1000);

        Assert.That(speed.Value, Is.GreaterThan(1000.0));
    }

    [OneTimeTearDown]
    public void Teardown()
    {
        if (_app?.Measurement.Running == true)
            _app.Measurement.Stop();
    }
}
```

Wrap COM interactions in `try/catch` for `COMException`. Always stop measurement in teardown — orphaned CANoe instances lock licenses. For xUnit, use `IClassFixture<T>` with a shared fixture instead of `[OneTimeSetUp]`.

Reference: Vector CANoe Help — COM Interface / .NET Integration
