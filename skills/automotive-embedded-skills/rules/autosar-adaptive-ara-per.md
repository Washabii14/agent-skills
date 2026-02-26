---
title: "ara::per — Persistency: Key-Value Storage, File Storage, and Recovery"
impact: MEDIUM
impactDescription: Incorrect persistency usage causes data loss on power failure, corrupted calibration values, or exhausted storage resources
tags: autosar, adaptive, ara-per, persistency, key-value-storage, file-storage, redundancy, recovery
---

## ara::per — Persistency: Key-Value Storage, File Storage, and Recovery

ara::per (AUTOSAR Adaptive R24-10) provides persistent data storage for Adaptive Applications via two mechanisms: key-value storage (structured data) and file storage (binary/stream data). It handles redundancy, integrity checking, and recovery from corruption — critical for data that must survive power cycles.

### Key-Value Storage

**Incorrect (using raw filesystem for persistent data):**

```cpp
#include <fstream>

void SaveCalibration(const CalData& cal)
{
    std::ofstream file("/data/calibration.bin", std::ios::binary);
    file.write(reinterpret_cast<const char*>(&cal), sizeof(cal));
    // No atomicity — power loss mid-write corrupts file
    // No integrity check — silent corruption undetected
    // No manifest integration — EM unaware of data dependency
}
```

**Correct (ara::per key-value storage with integrity):**

```cpp
#include <ara/per/key_value_storage.h>

void SaveCalibration(const CalData& cal)
{
    auto kvResult = ara::per::OpenKeyValueStorage(
        ara::core::InstanceSpecifier("MyApp/Persistency/CalibrationStore"));

    if (!kvResult.HasValue())
    {
        LogError("Failed to open KV store: ", kvResult.Error().Message());
        return;
    }

    auto& kvs = kvResult.Value();

    // Store individual key-value pairs
    auto r1 = kvs.SetValue("idle_rpm", cal.idleRpm);
    auto r2 = kvs.SetValue("max_boost", cal.maxBoost);
    auto r3 = kvs.SetValue("fuel_map_version", cal.fuelMapVersion);

    if (r1.HasValue() && r2.HasValue() && r3.HasValue())
    {
        // Persist to storage — atomic operation
        auto syncResult = kvs.SyncToStorage();
        if (!syncResult.HasValue())
        {
            LogError("Sync failed: ", syncResult.Error().Message());
        }
    }
}

void LoadCalibration(CalData& cal)
{
    auto kvResult = ara::per::OpenKeyValueStorage(
        ara::core::InstanceSpecifier("MyApp/Persistency/CalibrationStore"));

    if (!kvResult.HasValue())
    {
        cal = CalData::Defaults();
        return;
    }

    auto& kvs = kvResult.Value();

    auto r1 = kvs.GetValue<uint16_t>("idle_rpm");
    auto r2 = kvs.GetValue<uint16_t>("max_boost");
    auto r3 = kvs.GetValue<uint32_t>("fuel_map_version");

    cal.idleRpm = r1.ValueOr(800u);
    cal.maxBoost = r2.ValueOr(1500u);
    cal.fuelMapVersion = r3.ValueOr(0u);
}
```

### File Storage

```cpp
#include <ara/per/file_storage.h>

void SaveDiagnosticLog(ara::core::Span<const uint8_t> logData)
{
    auto fsResult = ara::per::OpenFileStorage(
        ara::core::InstanceSpecifier("MyApp/Persistency/DiagLogs"));

    if (!fsResult.HasValue()) return;

    auto& fs = fsResult.Value();

    // Open or create file within the managed storage
    auto fileResult = fs.OpenFileWriteOnly(
        "crash_log_001.bin",
        ara::per::OpenMode::kCreate | ara::per::OpenMode::kTruncate);

    if (fileResult.HasValue())
    {
        auto& file = fileResult.Value();

        auto writeResult = file.Write(logData);
        if (writeResult.HasValue())
        {
            file.SyncToFile();  // Flush to persistent storage
        }
    }
}

void ReadDiagnosticLog(std::vector<uint8_t>& buffer)
{
    auto fsResult = ara::per::OpenFileStorage(
        ara::core::InstanceSpecifier("MyApp/Persistency/DiagLogs"));

    if (!fsResult.HasValue()) return;

    auto& fs = fsResult.Value();

    auto fileResult = fs.OpenFileReadOnly("crash_log_001.bin");
    if (fileResult.HasValue())
    {
        auto& file = fileResult.Value();
        auto size = file.GetSize();
        buffer.resize(size.ValueOr(0));
        file.Read(ara::core::Span<uint8_t>(buffer));
    }
}
```

### Redundancy Handling

```cpp
// Persistency Manifest configuration for redundancy:
//
// PersistencyKeyValueStorage: CalibrationStore
//   PersistencyRedundancy:
//     redundancyCrc: TRUE          — CRC integrity check
//     redundancyMn: 3              — Store M copies (triple redundancy)
//     redundancyScope: kElement    — Per key-value pair redundancy
//
// ara::per internally:
//   1. Writes data to all M copies
//   2. On read: verifies CRC, uses majority vote if copies disagree
//   3. Automatically repairs corrupted copies from healthy ones

// For file storage:
// PersistencyFileStorage: DiagLogs
//   PersistencyRedundancy:
//     redundancyCrc: TRUE
//     redundancyMn: 2              — Dual copy
//     redundancyScope: kFile       — Entire file duplicated
```

### Recovery Strategies

```cpp
// Recovery on detected corruption:

void RecoverPersistency()
{
    auto kvResult = ara::per::OpenKeyValueStorage(
        ara::core::InstanceSpecifier("MyApp/Persistency/CalibrationStore"));

    if (!kvResult.HasValue())
    {
        auto error = kvResult.Error();

        // Recovery options based on error type:
        if (error == ara::per::PerErrc::kIntegrityCorrupted)
        {
            // Option 1: Reset to manifest-defined initial values
            auto resetResult = ara::per::ResetPersistency(
                ara::core::InstanceSpecifier(
                    "MyApp/Persistency/CalibrationStore"));

            if (resetResult.HasValue())
            {
                // Storage reset to initial values from manifest
                LogWarn("CalibrationStore reset to defaults after corruption");
            }
        }
        else if (error == ara::per::PerErrc::kStorageNotFound)
        {
            // First-ever startup — normal path, store will be created
        }
    }

    // Recovery for entire application persistency:
    // ara::per::ResetAllPersistency();  // Nuclear option — resets everything

    // Recover specific key:
    // kvs.RemoveKey("corrupted_key");
    // kvs.SetValue("corrupted_key", defaultValue);
    // kvs.SyncToStorage();
}

// Manifest initial values (applied on ResetPersistency):
// PersistencyKeyValueStorage: CalibrationStore
//   PersistencyKeyValuePair:
//     shortName: idle_rpm
//     initValue: 800
//   PersistencyKeyValuePair:
//     shortName: max_boost
//     initValue: 1500
```

Reference: AUTOSAR Adaptive R24-10 SWS_Persistency (ara::per), AP_SWS_Persistency
