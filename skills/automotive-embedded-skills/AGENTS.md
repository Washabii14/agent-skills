# Automotive Embedded C/C++/CAPL Best Practices

**Version 2.0.0**  
Automotive Embedded Engineering  
February 2026

> **Note:**  
> This document is mainly for agents and LLMs to follow when maintaining,  
> generating, or refactoring automotive embedded C/C++ and CAPL codebases.  
> Humans may also find it useful, but guidance here is optimized for automation  
> and consistency by AI-assisted workflows in the embedded automotive domain.

---

## Abstract

Comprehensive coding guidelines for automotive embedded software in C, C++, and CAPL. Contains 180+ rules across 27 sections, prioritized by safety impact from critical (memory safety, MISRA/AUTOSAR compliance, functional safety) to incremental (testing patterns, tool integration). Covers the full automotive communication stack: CAN/CAN FD, LIN, Automotive Ethernet (TCP, UDP, DoIP, ARP, ICMP, VLAN/QoS, SOME/IP, SOME/IP-SD, TSN time sync, traffic shaping, switch configuration), UDS diagnostics, and ISO 21434 cybersecurity. Extended coverage includes MISRA grouped rule topics (type system, deviations, pointer safety, concurrency), AUTOSAR Classic BSW module patterns (EcuM, COM, NvM, BswM, DCM/DEM, PduR, CanIf/CanTp, OS), AUTOSAR Adaptive Platform (ara::com, ara::core, ara::exec, ara::diag, ara::log, ara::per, ara::phm), ECU boot sequences, NVM management, power management, compiler/static analysis tooling, vTESTstudio CAPL testing, advanced CANoe simulation, fault injection (CAN/LIN/Ethernet), signal manipulation libraries, CAPL DLL integration, and CI/CD automation. Each rule includes detailed explanations, real-world examples comparing incorrect vs. correct implementations, relevant standard references (MISRA C:2012, AUTOSAR C++14, ISO 26262, ISO 21434), and specific impact metrics to guide automated code generation and refactoring for safety-critical automotive ECU software.

---

## Table of Contents

1. [Memory Safety & Management](#1-memory-safety--management) — **CRITICAL**
   - 1.1 [Prefer Stack Allocation Over Heap](#11-prefer-stack-allocation-over-heap)
   - 1.2 [Use Static Allocation for Deterministic Memory](#12-use-static-allocation-for-deterministic-memory)
   - 1.3 [Always Validate Buffer Boundaries](#13-always-validate-buffer-boundaries)
   - 1.4 [Memory Pool Pattern for Dynamic-Like Allocation](#14-memory-pool-pattern-for-dynamic-like-allocation)
   - 1.5 [Never Use malloc/free in Real-Time Critical Paths](#15-never-use-mallocfree-in-real-time-critical-paths)
   - 1.6 [RAII for Resource Management in C++](#16-raii-for-resource-management-in-c)
   - 1.7 [Use volatile Correctly](#17-use-volatile-correctly)
   - 1.8 [Ensure Proper Data Structure Alignment](#18-ensure-proper-data-structure-alignment)
   - 1.9 [Always Initialize Variables](#19-always-initialize-variables)
2. [MISRA C/C++ Compliance](#2-misra-cc-compliance) — **CRITICAL**
   - 2.1 [Avoid Implicit Type Conversions](#21-avoid-implicit-type-conversions)
   - 2.2 [Single Exit Point for Critical Functions](#22-single-exit-point-for-critical-functions)
   - 2.3 [No Dynamic Memory Allocation](#23-no-dynamic-memory-allocation)
   - 2.4 [No Recursion in Embedded Context](#24-no-recursion-in-embedded-context)
   - 2.5 [Always Include Default Case in Switch](#25-always-include-default-case-in-switch)
   - 2.6 [No goto Except for Error Cleanup in C](#26-no-goto-except-for-error-cleanup-in-c)
   - 2.7 [Explicit Boolean Comparisons](#27-explicit-boolean-comparisons)
   - 2.8 [Restrict Pointer Arithmetic](#28-restrict-pointer-arithmetic)
   - 2.9 [No Side Effects in Conditional Expressions](#29-no-side-effects-in-conditional-expressions)
3. [AUTOSAR C++14 Guidelines](#3-autosar-c14-guidelines) — **CRITICAL**
   - 3.1 [Smart Pointers for Ownership](#31-smart-pointers-for-ownership)
   - 3.2 [No Exceptions in Real-Time Contexts](#32-no-exceptions-in-real-time-contexts)
   - 3.3 [const-Correctness Throughout Interfaces](#33-const-correctness-throughout-interfaces)
   - 3.4 [Always Use override and final](#34-always-use-override-and-final)
   - 3.5 [Use enum class Over Plain enum](#35-use-enum-class-over-plain-enum)
   - 3.6 [Avoid Unions, Use std::variant](#36-avoid-unions-use-stdvariant)
   - 3.7 [Prefer Braced Initialization](#37-prefer-braced-initialization)
   - 3.8 [Use [[nodiscard]] for Important Return Values](#38-use-nodiscard-for-important-return-values)
4. [Safety & Functional Safety — ISO 26262](#4-safety--functional-safety--iso-26262) — **HIGH**
   - 4.1 [Defensive Programming at Module Boundaries](#41-defensive-programming-at-module-boundaries)
   - 4.2 [Error Detection and Plausibility Checks](#42-error-detection-and-plausibility-checks)
   - 4.3 [Redundant Checks for Critical Control Paths](#43-redundant-checks-for-critical-control-paths)
   - 4.4 [Watchdog Monitoring Patterns](#44-watchdog-monitoring-patterns)
   - 4.5 [State Machine Integrity Protection](#45-state-machine-integrity-protection)
   - 4.6 [CRC Validation for Critical Data](#46-crc-validation-for-critical-data)
   - 4.7 [Always Define and Reach Safe State on Failure](#47-always-define-and-reach-safe-state-on-failure)
   - 4.8 [ASIL Decomposition Patterns](#48-asil-decomposition-patterns)
5. [Real-Time & Timing Constraints](#5-real-time--timing-constraints) — **HIGH**
   - 5.1 [Deterministic Execution in Cyclic Tasks](#51-deterministic-execution-in-cyclic-tasks)
   - 5.2 [Design with WCET in Mind](#52-design-with-wcet-in-mind)
   - 5.3 [Never Block in ISRs](#53-never-block-in-isrs)
   - 5.4 [Prevent Priority Inversion](#54-prevent-priority-inversion)
   - 5.5 [Cyclic Scheduling Patterns](#55-cyclic-scheduling-patterns)
   - 5.6 [Minimize Interrupt Latency](#56-minimize-interrupt-latency)
   - 5.7 [Deadline Monitoring for Critical Tasks](#57-deadline-monitoring-for-critical-tasks)
6. [Communication Protocols](#6-communication-protocols) — **HIGH**
   - 6.1 [CAN Message Layout and DBC Conventions](#61-can-message-layout-and-dbc-conventions)
   - 6.2 [CAN Bus-Off Recovery and Error Handling](#62-can-bus-off-recovery-and-error-handling)
   - 6.3 [CAN FD Extended Data Handling](#63-can-fd-extended-data-handling)
   - 6.4 [LIN Schedule Table and Response Handling](#64-lin-schedule-table-and-response-handling)
   - 6.5 [Signal Timeout Monitoring](#65-signal-timeout-monitoring)
   - 6.6 [Network Management State Machine](#66-network-management-state-machine)
   - 6.7 [TCP Socket Lifecycle Management](#67-tcp-socket-lifecycle-management)
   - 6.8 [UDP Datagram Handling](#68-udp-datagram-handling)
   - 6.9 [Diagnostics over IP (DoIP)](#69-diagnostics-over-ip-doip)
   - 6.10 [ARP Table Management](#610-arp-table-management)
   - 6.11 [ICMP Handling for Network Diagnostics](#611-icmp-handling-for-network-diagnostics)
   - 6.12 [VLAN and QoS Priority Mapping](#612-vlan-and-qos-priority-mapping)
   - 6.13 [DHCP and AutoIP Address Configuration](#613-dhcp-and-autoip-address-configuration)
   - 6.14 [SOME/IP Serialization](#614-someip-serialization)
   - 6.15 [SOME/IP Service Discovery](#615-someip-service-discovery)
   - 6.16 [UDS Diagnostic Service Handling](#616-uds-diagnostic-service-handling)
   - 6.17 [Gateway Message Routing](#617-gateway-message-routing)
7. [Concurrency & RTOS Patterns](#7-concurrency--rtos-patterns) — **MEDIUM-HIGH**
   - 7.1 [Single Responsibility Task Design](#71-single-responsibility-task-design)
   - 7.2 [Minimize Critical Section Duration](#72-minimize-critical-section-duration)
   - 7.3 [Correct Mutex Usage](#73-correct-mutex-usage)
   - 7.4 [Prefer Message Queues Over Shared Memory](#74-prefer-message-queues-over-shared-memory)
   - 7.5 [Priority Inheritance Protocols](#75-priority-inheritance-protocols)
   - 7.6 [Defer ISR Processing to Task Context](#76-defer-isr-processing-to-task-context)
   - 7.7 [Task Stack Sizing](#77-task-stack-sizing)
8. [CAPL Scripting Best Practices](#8-capl-scripting-best-practices) — **MEDIUM-HIGH**
   - 8.1 [Message Handler Structure](#81-message-handler-structure)
   - 8.2 [Timer Patterns](#82-timer-patterns)
   - 8.3 [Test Case Structure](#83-test-case-structure)
   - 8.4 [Signal Access via Database Symbols](#84-signal-access-via-database-symbols)
   - 8.5 [Error Frame Handling in Simulation](#85-error-frame-handling-in-simulation)
   - 8.6 [Environment Variables for Panel Interaction](#86-environment-variables-for-panel-interaction)
   - 8.7 [Diagnostic Testing Patterns](#87-diagnostic-testing-patterns)
   - 8.8 [Node Simulation with State Machines](#88-node-simulation-with-state-machines)
9. [Code Organization & Architecture](#9-code-organization--architecture) — **MEDIUM**
   - 9.1 [Hardware Abstraction Layer](#91-hardware-abstraction-layer)
   - 9.2 [Clean Module Interfaces](#92-clean-module-interfaces)
   - 9.3 [Table-Driven State Machines](#93-table-driven-state-machines)
   - 9.4 [Callback Patterns for Layer Decoupling](#94-callback-patterns-for-layer-decoupling)
   - 9.5 [Separate Configuration from Logic](#95-separate-configuration-from-logic)
   - 9.6 [Layered Architecture Pattern](#96-layered-architecture-pattern)
10. [Performance Optimization](#10-performance-optimization) — **MEDIUM**
    - 10.1 [Loop Optimization for Embedded Targets](#101-loop-optimization-for-embedded-targets)
    - 10.2 [Lookup Tables Over Runtime Computation](#102-lookup-tables-over-runtime-computation)
    - 10.3 [Bitwise Operations for Flags and Registers](#103-bitwise-operations-for-flags-and-registers)
    - 10.4 [Cache-Friendly Data Organization](#104-cache-friendly-data-organization)
    - 10.5 [Inline Critical Functions](#105-inline-critical-functions)
    - 10.6 [Fixed-Point Arithmetic](#106-fixed-point-arithmetic)
    - 10.7 [DMA for Bulk Data Transfers](#107-dma-for-bulk-data-transfers)
11. [Build, Compilation & Static Analysis](#11-build-compilation--static-analysis) — **MEDIUM**
    - 11.1 [Compiler Warnings as Errors](#111-compiler-warnings-as-errors)
    - 11.2 [Static Analysis Integration](#112-static-analysis-integration)
    - 11.3 [Safety-Optimized Compiler Flags](#113-safety-optimized-compiler-flags)
    - 11.4 [Link-Time Optimization](#114-link-time-optimization)
    - 11.5 [Reproducible Builds](#115-reproducible-builds)
12. [Testing & Verification](#12-testing--verification) — **MEDIUM**
    - 12.1 [Unit Test Patterns for Embedded](#121-unit-test-patterns-for-embedded)
    - 12.2 [Mock Hardware Dependencies](#122-mock-hardware-dependencies)
    - 12.3 [Boundary Value Testing](#123-boundary-value-testing)
    - 12.4 [Coverage Targets per ASIL Level](#124-coverage-targets-per-asil-level)
    - 12.5 [Integration Testing Patterns](#125-integration-testing-patterns)
    - 12.6 [HIL/SIL Test Patterns](#126-hilsil-test-patterns)
13. [Security & Cybersecurity — ISO 21434](#13-security--cybersecurity--iso-21434) — **HIGH**
    - 13.1 [Secure Boot Chain Verification](#131-secure-boot-chain-verification)
    - 13.2 [Secure In-Vehicle Communication (TLS/DTLS)](#132-secure-in-vehicle-communication-tlsdtls)
    - 13.3 [Cryptographic Key Management](#133-cryptographic-key-management)
    - 13.4 [Secure UDS Authentication](#134-secure-uds-authentication)
    - 13.5 [Input Sanitization from External Interfaces](#135-input-sanitization-from-external-interfaces)
    - 13.6 [Secure OTA/Reflash with Signature Verification](#136-secure-otareflash-with-signature-verification)
    - 13.7 [Security Domain Access Control](#137-security-domain-access-control)
    - 13.8 [Correct Cryptographic Primitive Usage](#138-correct-cryptographic-primitive-usage)
14. [Tool Integration](#14-tool-integration) — **MEDIUM**
    - 14.1 [A2L/ASAP2 Calibration Descriptions](#141-a2lasap2-calibration-descriptions)
    - 14.2 [ODX/PDX Diagnostic Descriptions](#142-odxpdx-diagnostic-descriptions)
    - 14.3 [FIBEX Network Descriptions](#143-fibex-network-descriptions)
    - 14.4 [DBC/ARXML and Code Synchronization](#144-dbcarxml-and-code-synchronization)
    - 14.5 [XCP Measurement and Calibration Protocol](#145-xcp-measurement-and-calibration-protocol)
    - 14.6 [AUTOSAR ARXML Configuration](#146-autosar-arxml-configuration)
15. [MISRA Grouped Topics](#15-misra-grouped-topics) — **HIGH**
    - 15.1 [MISRA Type System — Conversions, Casting, Essential Type Model](#151-misra-type-system--conversions-casting-essential-type-model)
    - 15.2 [MISRA Deviation Process](#152-misra-deviation-process)
    - 15.3 [Additional MISRA Rule Groups](#153-additional-misra-rule-groups)
16. [AUTOSAR Classic BSW](#16-autosar-classic-bsw) — **HIGH**
    - 16.1 [EcuM — ECU State Manager](#161-ecum--ecu-state-manager)
    - 16.2 [COM — Signal Packing and Transmission Modes](#162-com--signal-packing-and-transmission-modes)
    - 16.3 [Additional Classic BSW Modules](#163-additional-classic-bsw-modules)
17. [AUTOSAR Adaptive Platform](#17-autosar-adaptive-platform) — **HIGH**
    - 17.1 [ara::com — Service-Oriented Communication](#171-aracom--service-oriented-communication)
    - 17.2 [ara::core — Result, ErrorCode, Future/Promise](#172-aracore--result-errorcode-futurepromise)
    - 17.3 [Additional Adaptive Platform APIs](#173-additional-adaptive-platform-apis)
18. [ECU Boot Sequence](#18-ecu-boot-sequence) — **HIGH**
    - 18.1 [Bare-Metal Boot Sequence](#181-bare-metal-boot-sequence)
    - 18.2 [Secure Boot Chain of Trust](#182-secure-boot-chain-of-trust)
    - 18.3 [AUTOSAR Classic Boot Sequence](#183-autosar-classic-boot-sequence)
    - 18.4 [Additional Boot Topics](#184-additional-boot-topics)
19. [NVM Management](#19-nvm-management) — **HIGH**
    - 19.1 [AUTOSAR NvM Block Configuration](#191-autosar-nvm-block-configuration)
    - 19.2 [Flash Wear Leveling Strategies](#192-flash-wear-leveling-strategies)
    - 19.3 [Fee and Ea Abstraction Layers](#193-fee-and-ea-abstraction-layers)
    - 19.4 [Additional NVM Topics](#194-additional-nvm-topics)
20. [Power Management](#20-power-management) — **MEDIUM-HIGH**
    - 20.1 [EcuM Sleep and Wakeup Cycle](#201-ecum-sleep-and-wakeup-cycle)
    - 20.2 [MCU Low-Power Mode Selection](#202-mcu-low-power-mode-selection)
    - 20.3 [Partial Networking and Selective Wakeup](#203-partial-networking-and-selective-wakeup)
    - 20.4 [Additional Power Management Topics](#204-additional-power-management-topics)
21. [Automotive Ethernet Deep-Dive](#21-automotive-ethernet-deep-dive) — **HIGH**
    - 21.1 [TSN Time Synchronization (IEEE 802.1AS / gPTP)](#211-tsn-time-synchronization-ieee-8021as--gptp)
    - 21.2 [TSN Traffic Shaping (IEEE 802.1Qbv)](#212-tsn-traffic-shaping-ieee-8021qbv)
    - 21.3 [Automotive Ethernet Switch Configuration](#213-automotive-ethernet-switch-configuration)
    - 21.4 [Additional Ethernet Topics](#214-additional-ethernet-topics)
22. [Compiler & Static Analysis](#22-compiler--static-analysis) — **MEDIUM-HIGH**
    - 22.1 [GCC Warning Flags for Automotive](#221-gcc-warning-flags-for-automotive)
    - 22.2 [Polyspace Bug Finder and Code Prover](#222-polyspace-bug-finder-and-code-prover)
    - 22.3 [Additional Compiler & Analysis Tools](#223-additional-compiler--analysis-tools)
23. [vTESTstudio CAPL](#23-vtestudio-capl) — **MEDIUM**
    - 23.1 [Test Unit and Fixture Structure](#231-test-unit-and-fixture-structure)
    - 23.2 [Data-Driven Testing](#232-data-driven-testing)
    - 23.3 [XML Test Module Integration](#233-xml-test-module-integration)
    - 23.4 [Additional vTESTstudio Topics](#234-additional-vtestudio-topics)
24. [CAPL Advanced Simulation](#24-capl-advanced-simulation) — **HIGH**
    - 24.1 [Multi-Channel Bus Simulation](#241-multi-channel-bus-simulation)
    - 24.2 [Reactive Rest Bus Simulation](#242-reactive-rest-bus-simulation)
    - 24.3 [Gateway Simulation and Routing Patterns](#243-gateway-simulation-and-routing-patterns)
25. [CAPL Fault Injection](#25-capl-fault-injection) — **HIGH**
    - 25.1 [CAN/CAN FD Fault Injection Patterns](#251-cancan-fd-fault-injection-patterns)
    - 25.2 [LIN Fault Injection Patterns](#252-lin-fault-injection-patterns)
    - 25.3 [Ethernet Fault Injection Patterns](#253-ethernet-fault-injection-patterns)
26. [CAPL Signal Manipulation](#26-capl-signal-manipulation) — **MEDIUM-HIGH**
    - 26.1 [Signal Manipulation Library Functions](#261-signal-manipulation-library-functions)
27. [CAPL External Integration](#27-capl-external-integration) — **HIGH**
    - 27.1 [CAPL DLL Integration](#271-capl-dll-integration)
    - 27.2 [CI/CD Integration for CANoe Test Automation](#272-cicd-integration-for-canoe-test-automation)
    - 27.3 [Additional External Integration Topics](#273-additional-external-integration-topics)

---

## 1. Memory Safety & Management

**Impact: CRITICAL**

Memory errors are the #1 cause of embedded system failures. In automotive ECUs with no MMU, a buffer overflow corrupts adjacent memory silently. Deterministic memory usage is mandatory for safety-critical systems.

### 1.1 Prefer Stack Allocation Over Heap

**Impact: CRITICAL (deterministic, no fragmentation)**

In embedded systems, stack allocation is deterministic in both time and space. Heap allocation introduces non-deterministic latency and fragmentation risk.

**Incorrect: heap allocation in periodic function**

```c
void ProcessSensorData(void)
{
    float *buffer = (float *)malloc(SENSOR_COUNT * sizeof(float));
    if (buffer == NULL)
    {
        /* Error: no memory — but damage may already be done */
        return;
    }
    ReadSensors(buffer, SENSOR_COUNT);
    ProcessBuffer(buffer, SENSOR_COUNT);
    free(buffer);
}
```

**Correct: stack allocation with known bounds**

```c
void ProcessSensorData(void)
{
    float buffer[SENSOR_COUNT]; /* Stack-allocated, deterministic */
    ReadSensors(buffer, SENSOR_COUNT);
    ProcessBuffer(buffer, SENSOR_COUNT);
}
```

For larger buffers that cannot fit on stack, use statically allocated module-level buffers.

Reference: MISRA C:2012 Rule 21.3 — The memory allocation and deallocation functions of `<stdlib.h>` shall not be used.

### 1.2 Use Static Allocation for Deterministic Memory

**Impact: CRITICAL (predictable at compile time)**

All memory usage should be determinable at compile time. Static allocation ensures the linker can verify total memory requirements fit within the target MCU.

**Incorrect: runtime-determined buffer**

```c
void InitComStack(uint8_t numChannels)
{
    ComChannel_t *channels = malloc(numChannels * sizeof(ComChannel_t));
    /* ... */
}
```

**Correct: compile-time-determined buffer**

```c
#define COM_MAX_CHANNELS (8U)

static ComChannel_t g_comChannels[COM_MAX_CHANNELS];
static uint8_t g_numActiveChannels = 0U;

Std_ReturnType InitComStack(uint8_t numChannels)
{
    if (numChannels > COM_MAX_CHANNELS)
    {
        return E_NOT_OK;
    }
    g_numActiveChannels = numChannels;
    (void)memset(g_comChannels, 0, sizeof(g_comChannels));
    return E_OK;
}
```

### 1.3 Always Validate Buffer Boundaries

**Impact: CRITICAL (prevents memory corruption, security vulnerabilities)**

Every buffer access must be validated against the buffer size. In automotive ECUs without MMU protection, out-of-bounds writes silently corrupt adjacent memory.

**Incorrect: no bounds checking**

```c
void CopyDtcData(uint8_t *dest, const uint8_t *src, uint16_t len)
{
    (void)memcpy(dest, src, len); /* len could exceed dest capacity */
}
```

**Correct: bounds-checked copy**

```c
Std_ReturnType CopyDtcData(uint8_t *dest, uint16_t destSize,
                            const uint8_t *src, uint16_t srcLen)
{
    if ((dest == NULL) || (src == NULL))
    {
        return E_NOT_OK;
    }
    if (srcLen > destSize)
    {
        return E_NOT_OK;
    }
    (void)memcpy(dest, src, srcLen);
    return E_OK;
}
```

### 1.4 Memory Pool Pattern for Dynamic-Like Allocation

**Impact: HIGH (deterministic dynamic allocation)**

When you need allocation/deallocation flexibility, use a statically-backed memory pool with fixed-size blocks.

**Implementation:**

```c
#define POOL_BLOCK_SIZE  (64U)
#define POOL_BLOCK_COUNT (32U)

typedef struct
{
    uint8_t data[POOL_BLOCK_SIZE];
    boolean inUse;
} PoolBlock_t;

static PoolBlock_t g_memPool[POOL_BLOCK_COUNT];

void *MemPool_Alloc(void)
{
    uint16_t idx;
    for (idx = 0U; idx < POOL_BLOCK_COUNT; idx++)
    {
        if (g_memPool[idx].inUse == FALSE)
        {
            g_memPool[idx].inUse = TRUE;
            return (void *)g_memPool[idx].data;
        }
    }
    return NULL;
}

void MemPool_Free(void *ptr)
{
    uint16_t idx;
    for (idx = 0U; idx < POOL_BLOCK_COUNT; idx++)
    {
        if ((void *)g_memPool[idx].data == ptr)
        {
            g_memPool[idx].inUse = FALSE;
            return;
        }
    }
}
```

### 1.5 Never Use malloc/free in Real-Time Critical Paths

**Impact: CRITICAL (non-deterministic latency, heap fragmentation)**

`malloc()` has non-deterministic execution time. In a 10ms cyclic task, a single `malloc()` can cause deadline overrun due to heap fragmentation and searching.

Reference: MISRA C:2012 Rule 21.3

### 1.6 RAII for Resource Management in C++

**Impact: HIGH (automatic cleanup, exception-safe)**

In C++ embedded code, use RAII to ensure resources are released even when execution paths change.

**Incorrect: manual resource management**

```cpp
void TransmitFrame(const uint8_t* data, size_t len)
{
    auto* buffer = HwBufferPool::Acquire();
    if (buffer == nullptr) { return; }

    std::memcpy(buffer->data, data, len);
    HwDriver::Send(buffer);
    /* If Send() throws or an early return is added, buffer leaks */
    HwBufferPool::Release(buffer);
}
```

**Correct: RAII wrapper**

```cpp
class ScopedHwBuffer
{
public:
    ScopedHwBuffer() : m_buffer(HwBufferPool::Acquire()) {}
    ~ScopedHwBuffer() { if (m_buffer) { HwBufferPool::Release(m_buffer); } }

    ScopedHwBuffer(const ScopedHwBuffer&) = delete;
    ScopedHwBuffer& operator=(const ScopedHwBuffer&) = delete;

    HwBuffer* Get() { return m_buffer; }
    bool IsValid() const { return m_buffer != nullptr; }

private:
    HwBuffer* m_buffer;
};

void TransmitFrame(const uint8_t* data, size_t len)
{
    ScopedHwBuffer buffer;
    if (!buffer.IsValid()) { return; }

    std::memcpy(buffer.Get()->data, data, len);
    HwDriver::Send(buffer.Get());
}
```

### 1.7 Use volatile Correctly

**Impact: CRITICAL (prevents compiler optimization of hardware access)**

`volatile` tells the compiler the value may change outside the program flow. Required for: hardware registers, shared variables modified by ISRs, and memory-mapped I/O.

**Incorrect: compiler may optimize away the read**

```c
uint32_t WaitForFlag(void)
{
    uint32_t *statusReg = (uint32_t *)0x40020010U;
    while ((*statusReg & 0x01U) == 0U)
    {
        /* Compiler may read statusReg once and loop forever */
    }
    return *statusReg;
}
```

**Correct: volatile forces re-read**

```c
uint32_t WaitForFlag(void)
{
    volatile uint32_t *statusReg = (volatile uint32_t *)0x40020010U;
    while ((*statusReg & 0x01U) == 0U)
    {
        /* Each iteration re-reads the hardware register */
    }
    return *statusReg;
}
```

**Warning:** `volatile` does NOT provide atomicity or memory ordering. For shared data between ISR and task, also use critical sections or atomic operations.

### 1.8 Ensure Proper Data Structure Alignment

**Impact: MEDIUM (prevents hard faults, optimizes access speed)**

Misaligned access causes hard faults on ARM Cortex-M and performance degradation on other architectures.

**Incorrect: packed struct with misaligned fields**

```c
typedef struct __attribute__((packed))
{
    uint8_t  id;
    uint32_t timestamp;  /* Misaligned at offset 1 */
    uint16_t value;
} SensorData_t;
```

**Correct: naturally aligned struct**

```c
typedef struct
{
    uint32_t timestamp;  /* Offset 0: aligned */
    uint16_t value;      /* Offset 4: aligned */
    uint8_t  id;         /* Offset 6 */
    uint8_t  reserved;   /* Padding made explicit */
} SensorData_t;
```

Use `__attribute__((packed))` only for wire protocol structures, and access fields through `memcpy()` to avoid alignment faults.

### 1.9 Always Initialize Variables

**Impact: HIGH (prevents undefined behavior in safety-critical code)**

Uninitialized variables produce undefined behavior. In safety-critical automotive code, every variable must have a defined initial value.

**Incorrect: uninitialized local**

```c
Std_ReturnType ReadSensor(uint16_t *value)
{
    Std_ReturnType ret;  /* Uninitialized — UB if no branch sets it */
    uint16_t rawVal;

    if (HAL_ADC_Read(&rawVal) == HAL_OK)
    {
        *value = rawVal;
        ret = E_OK;
    }
    /* If HAL_ADC_Read fails, ret is garbage */
    return ret;
}
```

**Correct: initialized with safe defaults**

```c
Std_ReturnType ReadSensor(uint16_t *value)
{
    Std_ReturnType ret = E_NOT_OK;
    uint16_t rawVal = 0U;

    if (value == NULL)
    {
        return E_NOT_OK;
    }

    if (HAL_ADC_Read(&rawVal) == HAL_OK)
    {
        *value = rawVal;
        ret = E_OK;
    }
    else
    {
        *value = 0U;
    }
    return ret;
}
```

Reference: MISRA C:2012 Rule 9.1 — The value of an object with automatic storage duration shall not be read before it has been set.

---

## 2. MISRA C/C++ Compliance

**Impact: CRITICAL**

MISRA C:2012 and MISRA C++:2023 are the de facto coding standards for automotive embedded software. Compliance is required by ISO 26262 for safety-critical code.

### 2.1 Avoid Implicit Type Conversions

**Impact: HIGH (prevents data loss and unexpected behavior)**

Implicit conversions between signed/unsigned, narrowing conversions, and integer promotion can cause subtle bugs.

**Incorrect: implicit narrowing**

```c
uint16_t adc_value = ReadAdc();  /* Returns uint32_t */
int8_t temperature = (adc_value - 500) / 10;  /* Signed/unsigned mix, narrowing */
```

**Correct: explicit casting with range check**

```c
uint32_t raw_adc = ReadAdc();
int32_t temp_calc = ((int32_t)raw_adc - 500) / 10;

if ((temp_calc >= INT8_MIN) && (temp_calc <= INT8_MAX))
{
    temperature = (int8_t)temp_calc;
}
else
{
    temperature = TEMP_DEFAULT_VALUE;
    ReportError(ERR_TEMP_RANGE);
}
```

Reference: MISRA C:2012 Rules 10.1–10.8 (Essential type model)

### 2.2 Single Exit Point for Critical Functions

**Impact: MEDIUM (improves traceability and code review)**

For safety-critical functions, a single exit point makes control flow analysis tractable for formal verification tools.

**Incorrect: multiple return points**

```c
Std_ReturnType ValidateFrame(const CanFrame_t *frame)
{
    if (frame == NULL) return E_NOT_OK;
    if (frame->dlc > 8U) return E_NOT_OK;
    if (frame->id > 0x7FFU) return E_NOT_OK;
    return E_OK;
}
```

**Correct: single exit with result variable**

```c
Std_ReturnType ValidateFrame(const CanFrame_t *frame)
{
    Std_ReturnType result = E_NOT_OK;

    if (frame != NULL)
    {
        if ((frame->dlc <= 8U) && (frame->id <= 0x7FFU))
        {
            result = E_OK;
        }
    }

    return result;
}
```

Reference: MISRA C:2012 Rule 15.5 (Advisory) — A function should have a single point of exit at the end.

### 2.3 No Dynamic Memory Allocation

**Impact: CRITICAL (eliminates heap fragmentation, deterministic memory)**

Dynamic memory allocation shall not be used in production embedded automotive code. The heap introduces non-determinism and fragmentation.

Reference: MISRA C:2012 Rule 21.3 — The memory allocation and deallocation functions of `<stdlib.h>` shall not be used.

### 2.4 No Recursion in Embedded Context

**Impact: CRITICAL (prevents stack overflow)**

Recursion makes stack usage unpredictable. In embedded systems with limited stack, this can cause hard faults.

**Incorrect: recursive tree traversal**

```c
uint32_t SumTree(const TreeNode_t *node)
{
    if (node == NULL) { return 0U; }
    return node->value + SumTree(node->left) + SumTree(node->right);
}
```

**Correct: iterative with explicit stack**

```c
uint32_t SumTree(const TreeNode_t *root)
{
    uint32_t sum = 0U;
    const TreeNode_t *stack[MAX_TREE_DEPTH];
    int16_t top = -1;

    if (root != NULL)
    {
        stack[++top] = root;
    }

    while (top >= 0)
    {
        const TreeNode_t *node = stack[top--];
        sum += node->value;

        if ((node->right != NULL) && (top < (MAX_TREE_DEPTH - 1)))
        {
            stack[++top] = node->right;
        }
        if ((node->left != NULL) && (top < (MAX_TREE_DEPTH - 1)))
        {
            stack[++top] = node->left;
        }
    }
    return sum;
}
```

Reference: MISRA C:2012 Rule 17.2 — Functions shall not call themselves, either directly or indirectly.

### 2.5 Always Include Default Case in Switch

**Impact: MEDIUM (catches unexpected values, defensive programming)**

Every switch statement must have a default case to handle unexpected values, especially for values received from external interfaces.

**Incorrect: missing default**

```c
void HandleState(SystemState_t state)
{
    switch (state)
    {
        case STATE_INIT:    DoInit();    break;
        case STATE_RUNNING: DoRunning(); break;
        case STATE_STOP:    DoStop();    break;
    }
}
```

**Correct: with default and error handling**

```c
void HandleState(SystemState_t state)
{
    switch (state)
    {
        case STATE_INIT:    DoInit();    break;
        case STATE_RUNNING: DoRunning(); break;
        case STATE_STOP:    DoStop();    break;
        default:
            ReportError(ERR_INVALID_STATE);
            EnterSafeState();
            break;
    }
}
```

Reference: MISRA C:2012 Rule 16.4 — Every switch statement shall have a default label.

### 2.6 No goto Except for Error Cleanup in C

**Impact: LOW-MEDIUM (maintains readability)**

In C, `goto` is acceptable only for forward jumps to a common cleanup label at the end of a function. All other uses of `goto` shall be avoided.

**Acceptable: error cleanup pattern**

```c
Std_ReturnType ProcessMessage(const uint8_t *data, uint16_t len)
{
    Std_ReturnType ret = E_NOT_OK;
    MsgBuffer_t *buf = NULL;

    buf = MsgPool_Alloc();
    if (buf == NULL) { goto cleanup; }

    if (DecodeMessage(buf, data, len) != E_OK) { goto cleanup; }
    if (ValidateMessage(buf) != E_OK) { goto cleanup; }

    ret = RouteMessage(buf);

cleanup:
    if (buf != NULL) { MsgPool_Free(buf); }
    return ret;
}
```

Reference: MISRA C:2012 Rule 15.1 — The goto statement should not be used.

### 2.7 Explicit Boolean Comparisons

**Impact: LOW (improves readability and prevents implicit conversion bugs)**

Use explicit boolean comparisons rather than relying on implicit truthiness.

**Incorrect: implicit boolean**

```c
if (ptr)           { /* ... */ }
if (!errorCount)   { /* ... */ }
if (flags & MASK)  { /* ... */ }
```

**Correct: explicit comparisons**

```c
if (ptr != NULL)          { /* ... */ }
if (errorCount == 0U)     { /* ... */ }
if ((flags & MASK) != 0U) { /* ... */ }
```

Reference: MISRA C:2012 Rule 14.4 — The controlling expression of an if/while shall have essentially Boolean type.

### 2.8 Restrict Pointer Arithmetic

**Impact: MEDIUM (prevents out-of-bounds access)**

Pointer arithmetic should only be used for array indexing. Never compare pointers from different objects.

**Incorrect: arbitrary pointer arithmetic**

```c
void ProcessData(uint8_t *data, uint16_t offset)
{
    uint8_t *target = data + offset;  /* Could point anywhere */
    *target = 0xFFU;
}
```

**Correct: array indexing with bounds check**

```c
void ProcessData(uint8_t data[], uint16_t dataSize, uint16_t offset)
{
    if (offset < dataSize)
    {
        data[offset] = 0xFFU;
    }
}
```

Reference: MISRA C:2012 Rule 18.1–18.4

### 2.9 No Side Effects in Conditional Expressions

**Impact: MEDIUM (prevents order-dependent bugs)**

Conditional expressions shall not contain side effects. The evaluation order of operands is often implementation-defined.

**Incorrect: side effect in condition**

```c
if ((ReadSensor(&value) == E_OK) && (value > threshold++))
{
    TriggerAlarm();
}
```

**Correct: separate side effects from conditions**

```c
Std_ReturnType readResult = ReadSensor(&value);
if (readResult == E_OK)
{
    if (value > threshold)
    {
        threshold++;
        TriggerAlarm();
    }
}
```

Reference: MISRA C:2012 Rule 13.5 — The right hand operand of a logical && or || shall not contain persistent side effects.

---

## 3. AUTOSAR C++14 Guidelines

**Impact: CRITICAL**

AUTOSAR C++14 Coding Guidelines define safe C++ subset for automotive use. These rules ensure type safety, determinism, and maintainability in Adaptive AUTOSAR platforms.

### 3.1 Smart Pointers for Ownership

**Impact: HIGH (prevents resource leaks)**

Use `std::unique_ptr` for exclusive ownership and `std::shared_ptr` only when shared ownership is genuinely needed. Never use raw owning pointers.

**Incorrect: raw owning pointer**

```cpp
class DiagSession
{
public:
    void Start()
    {
        m_handler = new UdsHandler();
    }
    ~DiagSession() { delete m_handler; }  /* Easily forgotten or doubled */

private:
    UdsHandler* m_handler = nullptr;
};
```

**Correct: unique_ptr**

```cpp
class DiagSession
{
public:
    void Start()
    {
        m_handler = std::make_unique<UdsHandler>();
    }

private:
    std::unique_ptr<UdsHandler> m_handler;
};
```

Reference: AUTOSAR C++14 Rule A18-1-1

### 3.2 No Exceptions in Real-Time Contexts

**Impact: HIGH (deterministic error handling)**

C++ exceptions have non-deterministic cost due to stack unwinding. In hard real-time contexts, use return codes or `ara::core::Result<T,E>`.

**Incorrect: throwing in cyclic task**

```cpp
void Cyclic10ms()
{
    auto data = sensorDriver.Read();
    if (!data.isValid)
    {
        throw std::runtime_error("Sensor failure");  /* Non-deterministic */
    }
}
```

**Correct: Result type**

```cpp
ara::core::Result<SensorData> Cyclic10ms()
{
    auto result = sensorDriver.Read();
    if (!result.HasValue())
    {
        return ara::core::Result<SensorData>::FromError(
            SensorErrc::kReadFailure);
    }
    return result;
}
```

Reference: AUTOSAR C++14 Rule A15-0-1

### 3.3 const-Correctness Throughout Interfaces

**Impact: MEDIUM (enables compiler optimization, documents intent)**

Apply `const` to all parameters, member functions, and return values that should not be modified.

**Incorrect: missing const**

```cpp
class SensorManager
{
public:
    float GetTemperature()     { return m_temp; }
    void ProcessData(float* data, int count);
};
```

**Correct: const-correct**

```cpp
class SensorManager
{
public:
    float GetTemperature() const { return m_temp; }
    void ProcessData(const float* data, std::size_t count);
};
```

Reference: AUTOSAR C++14 Rule A7-1-1

### 3.4 Always Use override and final

**Impact: MEDIUM (catches signature mismatches at compile time)**

**Incorrect: silent signature mismatch**

```cpp
class SensorBase
{
public:
    virtual void Init(uint8_t channel);
};

class TempSensor : public SensorBase
{
public:
    virtual void Init(int channel);  /* Different signature — new function, not override */
};
```

**Correct: compiler catches the error**

```cpp
class TempSensor : public SensorBase
{
public:
    void Init(uint8_t channel) override;  /* Compiler verifies base signature */
};
```

Reference: AUTOSAR C++14 Rule A10-3-1

### 3.5 Use enum class Over Plain enum

**Impact: MEDIUM (prevents implicit conversion, scoped names)**

**Incorrect: plain enum pollutes scope**

```c
enum Color { RED, GREEN, BLUE };
enum TrafficLight { RED, YELLOW, GREEN };  /* Compilation error: RED/GREEN conflict */
```

**Correct: scoped enum**

```cpp
enum class Color : uint8_t { Red, Green, Blue };
enum class TrafficLight : uint8_t { Red, Yellow, Green };

auto c = Color::Red;  /* No conflict, no implicit conversion */
```

Reference: AUTOSAR C++14 Rule A7-2-2

### 3.6 Avoid Unions, Use std::variant

**Impact: MEDIUM (type-safe, no UB from active member confusion)**

**Incorrect: union with type confusion risk**

```cpp
union SensorValue
{
    float    temperature;
    uint32_t pressure;
    int16_t  rawAdc;
};
```

**Correct: type-safe variant**

```cpp
using SensorValue = std::variant<float, uint32_t, int16_t>;

SensorValue val = 25.5f;
if (auto* temp = std::get_if<float>(&val))
{
    ProcessTemperature(*temp);
}
```

Reference: AUTOSAR C++14 Rule A9-5-1

### 3.7 Prefer Braced Initialization

**Impact: LOW-MEDIUM (prevents narrowing conversions)**

**Incorrect: allows narrowing**

```cpp
uint8_t channel = 300;  /* Silently truncated to 44 */
```

**Correct: braces catch narrowing at compile time**

```cpp
uint8_t channel{300};  /* Compilation error: narrowing conversion */
```

Reference: AUTOSAR C++14 Rule A8-5-2

### 3.8 Use [[nodiscard]] for Important Return Values

**Impact: MEDIUM (prevents ignoring error codes)**

**Incorrect: error code silently ignored**

```cpp
Std_ReturnType WriteEeprom(uint16_t addr, const uint8_t* data, uint16_t len);

void SaveConfig()
{
    WriteEeprom(0x100, configData, sizeof(configData));  /* Return value ignored */
}
```

**Correct: [[nodiscard]] enforced**

```cpp
[[nodiscard]] Std_ReturnType WriteEeprom(uint16_t addr, const uint8_t* data,
                                          uint16_t len);

void SaveConfig()
{
    Std_ReturnType ret = WriteEeprom(0x100, configData, sizeof(configData));
    if (ret != E_OK)
    {
        ReportError(ERR_EEPROM_WRITE);
    }
}
```

Reference: AUTOSAR C++14 Rule A0-1-2

---

## 4. Safety & Functional Safety — ISO 26262

**Impact: HIGH**

ISO 26262 defines functional safety requirements for automotive systems. Software measures vary by ASIL level (A through D). These patterns implement the software safety requirements.

### 4.1 Defensive Programming at Module Boundaries

**Impact: HIGH (catches interface violations early)**

Validate all inputs at public function boundaries. Never trust data from external modules or communication interfaces.

**Incorrect: trusting external data**

```c
void ApplyTorqueRequest(const TorqueMsg_t *msg)
{
    SetMotorTorque(msg->requestedTorque);
}
```

**Correct: defensive validation**

```c
void ApplyTorqueRequest(const TorqueMsg_t *msg)
{
    if (msg == NULL)
    {
        EnterSafeState();
        return;
    }

    if ((msg->requestedTorque < TORQUE_MIN) ||
        (msg->requestedTorque > TORQUE_MAX))
    {
        ReportError(ERR_TORQUE_RANGE);
        SetMotorTorque(TORQUE_SAFE_VALUE);
        return;
    }

    if (msg->messageCounter == g_lastMsgCounter)
    {
        ReportError(ERR_MSG_STALE);
        return;
    }
    g_lastMsgCounter = msg->messageCounter;

    SetMotorTorque(msg->requestedTorque);
}
```

### 4.2 Error Detection and Plausibility Checks

**Impact: HIGH (detects sensor/actuator failures)**

Implement plausibility checks on sensor readings and actuator feedback to detect hardware failures.

```c
typedef struct
{
    float currentValue;
    float previousValue;
    float maxDeltaPerCycle;
    float minValid;
    float maxValid;
} PlausibilityCheck_t;

boolean IsPlausible(PlausibilityCheck_t *check, float newValue)
{
    float delta;

    if ((newValue < check->minValid) || (newValue > check->maxValid))
    {
        return FALSE;  /* Out of physical range */
    }

    delta = newValue - check->previousValue;
    if (delta < 0.0f) { delta = -delta; }

    if (delta > check->maxDeltaPerCycle)
    {
        return FALSE;  /* Gradient too steep — likely sensor fault */
    }

    check->previousValue = check->currentValue;
    check->currentValue = newValue;
    return TRUE;
}
```

### 4.3 Redundant Checks for Critical Control Paths

**Impact: CRITICAL (ASIL C/D requirement)**

For ASIL C and D, critical control outputs must be validated by diverse redundant checks.

```c
void SetBrakePressure(float requested)
{
    /* Primary calculation */
    float primary = CalculateBrakePressure_Primary(requested);

    /* Diverse redundant calculation */
    float secondary = CalculateBrakePressure_Secondary(requested);

    float diff = primary - secondary;
    if (diff < 0.0f) { diff = -diff; }

    if (diff > BRAKE_TOLERANCE)
    {
        ReportError(ERR_BRAKE_REDUNDANCY);
        ApplyEmergencyBrake();
        return;
    }

    HwDriver_SetBrakePressure(primary);
}
```

### 4.4 Watchdog Monitoring Patterns

**Impact: HIGH (detects task deadlocks and runaway code)**

Implement alive monitoring and logical supervision to detect software malfunctions.

```c
typedef struct
{
    uint32_t expectedPeriodMs;
    uint32_t toleranceMs;
    uint32_t lastAliveTimestamp;
    uint16_t sequenceCounter;
    uint16_t expectedSequence;
    boolean  isActive;
} WdgSupervisor_t;

Std_ReturnType Wdg_CheckAlive(WdgSupervisor_t *sup, uint32_t currentTimeMs,
                               uint16_t seqCounter)
{
    uint32_t elapsed;

    if (!sup->isActive) { return E_NOT_OK; }

    elapsed = currentTimeMs - sup->lastAliveTimestamp;

    if (elapsed > (sup->expectedPeriodMs + sup->toleranceMs))
    {
        return E_NOT_OK;  /* Deadline missed */
    }

    if (seqCounter != sup->expectedSequence)
    {
        return E_NOT_OK;  /* Wrong execution order */
    }

    sup->lastAliveTimestamp = currentTimeMs;
    sup->expectedSequence = (seqCounter + 1U) % UINT16_MAX;
    return E_OK;
}
```

### 4.5 State Machine Integrity Protection

**Impact: HIGH (prevents illegal state transitions)**

Protect state machine transitions from corruption caused by memory faults or software bugs.

```c
#define STATE_MAGIC (0xA5A5U)

typedef struct
{
    uint16_t magic;
    SystemState_t currentState;
    SystemState_t previousState;
    uint32_t transitionCount;
} ProtectedStateMachine_t;

Std_ReturnType SM_Transition(ProtectedStateMachine_t *sm,
                              SystemState_t newState)
{
    if (sm->magic != STATE_MAGIC)
    {
        EnterSafeState();
        return E_NOT_OK;
    }

    if (!IsTransitionValid(sm->currentState, newState))
    {
        ReportError(ERR_INVALID_TRANSITION);
        return E_NOT_OK;
    }

    sm->previousState = sm->currentState;
    sm->currentState = newState;
    sm->transitionCount++;

    return E_OK;
}
```

### 4.6 CRC Validation for Critical Data

**Impact: HIGH (detects data corruption)**

Protect critical data stored in RAM or transmitted over communication with CRC.

```c
typedef struct
{
    float   safetyRelevantParam;
    uint8_t controlFlags;
    uint32_t crc;
} ProtectedData_t;

Std_ReturnType WriteProtectedData(ProtectedData_t *data, float param,
                                   uint8_t flags)
{
    data->safetyRelevantParam = param;
    data->controlFlags = flags;
    data->crc = Crc_CalculateCRC32(
        (const uint8_t *)data,
        offsetof(ProtectedData_t, crc),
        CRC_INITIAL_VALUE);
    return E_OK;
}

boolean ValidateProtectedData(const ProtectedData_t *data)
{
    uint32_t computed = Crc_CalculateCRC32(
        (const uint8_t *)data,
        offsetof(ProtectedData_t, crc),
        CRC_INITIAL_VALUE);
    return (computed == data->crc);
}
```

### 4.7 Always Define and Reach Safe State on Failure

**Impact: CRITICAL (fundamental safety requirement)**

Every safety-relevant module must define a safe state and transition to it on any detected failure.

```c
void EnterSafeState(void)
{
    DisableActuators();
    SetDefaultOutputs();
    NotifyDiagnostic(DTC_SAFE_STATE_ENTERED);
    SetSystemState(STATE_SAFE);
}
```

### 4.8 ASIL Decomposition Patterns

**Impact: HIGH (enables mixed-criticality systems)**

When decomposing ASIL requirements, ensure freedom from interference between partitions.

```c
/* ASIL D partition — protected memory region */
#pragma section ".ASIL_D_DATA"
static BrakeControl_t g_brakeData;
#pragma section

/* QM partition — separate memory region */
#pragma section ".QM_DATA"
static InfotainmentData_t g_infoData;
#pragma section
```

Reference: ISO 26262 Part 6, Table 5 — Methods for software architectural design

---

## 5. Real-Time & Timing Constraints

**Impact: HIGH**

Automotive ECUs execute cyclic tasks with hard deadlines (1ms, 5ms, 10ms). Missing a deadline can lead to safety-relevant failures.

### 5.1 Deterministic Execution in Cyclic Tasks

**Impact: CRITICAL (deadline compliance)**

Cyclic task execution time must be bounded and deterministic. Avoid data-dependent loops and dynamic allocation.

**Incorrect: unbounded loop in cyclic task**

```c
void Task_10ms(void)
{
    while (MessageQueue_HasPending())  /* Unbounded — could overrun */
    {
        ProcessNextMessage();
    }
}
```

**Correct: bounded processing per cycle**

```c
#define MAX_MESSAGES_PER_CYCLE (5U)

void Task_10ms(void)
{
    uint8_t count = 0U;
    while ((MessageQueue_HasPending()) && (count < MAX_MESSAGES_PER_CYCLE))
    {
        ProcessNextMessage();
        count++;
    }
}
```

### 5.2 Design with WCET in Mind

**Impact: HIGH (schedulability analysis)**

All code paths must have analyzable worst-case execution time for schedulability proofs.

### 5.3 Never Block in ISRs

**Impact: CRITICAL (system responsiveness)**

ISRs must execute quickly and never wait for resources. Defer complex processing to task context.

**Incorrect: blocking in ISR**

```c
void CAN_RxISR(void)
{
    CanFrame_t frame = CAN_ReadMailbox();
    ProcessFrame(&frame);    /* Could take >100us */
    WriteToFlash(&frame);    /* Blocking flash write in ISR! */
}
```

**Correct: minimal ISR, deferred processing**

```c
void CAN_RxISR(void)
{
    CanFrame_t frame = CAN_ReadMailbox();
    RingBuffer_Put(&g_canRxBuffer, &frame);
    OS_SetEvent(TASK_CAN_PROCESS, EVT_CAN_RX);
}
```

### 5.4 Prevent Priority Inversion

**Impact: HIGH (prevents low-priority task blocking high-priority task)**

Use priority inheritance or priority ceiling protocol when high-priority tasks share resources with lower-priority tasks.

### 5.5 Cyclic Scheduling Patterns

**Impact: MEDIUM (consistent timing)**

Distribute workload evenly across cycles using phase offsets.

```c
void Task_10ms(void)
{
    static uint8_t phase = 0U;

    ReadSensors();  /* Every 10ms */

    switch (phase)
    {
        case 0U: RunDiagnostics();     break;
        case 1U: UpdateNvm();          break;
        case 2U: ProcessBackgroundCalc(); break;
        default: /* No additional work */ break;
    }

    phase = (phase + 1U) % 3U;
}
```

### 5.6 Minimize Interrupt Latency

**Impact: HIGH (system responsiveness)**

Keep ISR execution time to a minimum. Read hardware, set flags, return.

### 5.7 Deadline Monitoring for Critical Tasks

**Impact: HIGH (ASIL compliance)**

Monitor that critical tasks complete within their allocated time budget.

```c
void Task_5ms(void)
{
    uint32_t startTime = Timer_GetCurrentUs();

    /* Task body */
    ExecuteControlLoop();

    uint32_t elapsed = Timer_GetCurrentUs() - startTime;
    if (elapsed > TASK_5MS_DEADLINE_US)
    {
        ReportError(ERR_DEADLINE_OVERRUN);
    }
}
```

---

## 6. Communication Protocols

**Impact: HIGH**

CAN, CAN FD, LIN, Automotive Ethernet (TCP/UDP/DoIP/SOME-IP), and diagnostic protocols (UDS) form the backbone of automotive communication. Modern vehicles use 100+ ECUs communicating over multiple bus technologies.

### CAN / LIN Bus Protocols

### 6.1 CAN Message Layout and DBC Conventions

**Impact: MEDIUM (interoperability, maintainability)**

Follow DBC-defined signal layout. Use signal names from database, not magic offsets.

**Incorrect: magic numbers for signal extraction**

```c
uint16_t GetSpeed(const uint8_t data[8])
{
    return ((uint16_t)data[2] << 8) | data[3];  /* What signal is this? */
}
```

**Correct: named signal access aligned with DBC**

```c
uint16_t GetVehicleSpeed_kmph(const VehicleSpeed_Msg_t *msg)
{
    uint16_t rawSpeed = msg->VehicleSpeed_Raw;
    return (uint16_t)(rawSpeed * VEHICLE_SPEED_FACTOR + VEHICLE_SPEED_OFFSET);
}
```

### 6.2 CAN Bus-Off Recovery and Error Handling

**Impact: HIGH (communication resilience)**

Implement proper bus-off recovery with backoff strategy.

```c
void CAN_ErrorCallback(Can_ErrorType error)
{
    switch (error)
    {
        case CAN_ERROR_BUSOFF:
            g_canBusOffCount++;
            if (g_canBusOffCount < MAX_BUSOFF_RECOVERY)
            {
                Timer_Start(TIMER_BUSOFF_RECOVERY, BUSOFF_DELAY_MS);
            }
            else
            {
                ReportDtc(DTC_CAN_BUSOFF_PERMANENT);
                EnterDegradedMode();
            }
            break;

        case CAN_ERROR_PASSIVE:
            ReportDtc(DTC_CAN_ERROR_PASSIVE);
            break;

        default:
            break;
    }
}
```

### 6.3 CAN FD Extended Data Handling

**Impact: MEDIUM (next-gen CAN communication)**

CAN FD supports up to 64 bytes per frame (vs. 8 for Classic CAN) and higher bit rates in the data phase. Handle DLC-to-length mapping correctly since CAN FD DLC values above 8 are non-linear (12→16, 13→20, 14→24, 15→32, etc.).

**Incorrect: assuming DLC == data length**

```c
void ProcessCanFdFrame(const CanFd_Frame_t *frame)
{
    uint8_t buffer[64];
    (void)memcpy(buffer, frame->data, frame->dlc);  /* DLC 15 != 15 bytes */
}
```

**Correct: DLC-to-length conversion**

```c
static const uint8_t g_canFdDlcToLen[] =
{
    0U, 1U, 2U, 3U, 4U, 5U, 6U, 7U, 8U,
    12U, 16U, 20U, 24U, 32U, 48U, 64U
};

uint8_t CanFd_DlcToLength(uint8_t dlc)
{
    if (dlc > 15U) { return 64U; }
    return g_canFdDlcToLen[dlc];
}

void ProcessCanFdFrame(const CanFd_Frame_t *frame)
{
    uint8_t dataLen = CanFd_DlcToLength(frame->dlc);
    uint8_t buffer[64];
    (void)memcpy(buffer, frame->data, dataLen);
    ProcessPayload(buffer, dataLen);
}
```

Reference: ISO 11898-1:2015

### 6.4 LIN Schedule Table and Response Handling

**Impact: MEDIUM (LIN master/slave communication)**

Implement LIN schedule tables with proper header/response separation and error detection.

```c
typedef struct
{
    uint8_t frameId;
    uint8_t direction;     /* LIN_TX or LIN_RX */
    uint8_t dataLength;
    uint16_t slotTimeMs;
} LinScheduleEntry_t;

static const LinScheduleEntry_t g_linSchedule[] =
{
    { 0x10U, LIN_TX, 8U, 10U },  /* Master request */
    { 0x11U, LIN_RX, 4U, 10U },  /* Slave response */
    { 0x3CU, LIN_TX, 8U, 10U },  /* Diagnostic request (MasterReq) */
    { 0x3DU, LIN_RX, 8U, 10U },  /* Diagnostic response (SlaveResp) */
};
```

### 6.5 Signal Timeout Monitoring

**Impact: HIGH (detects communication loss)**

Monitor received signal freshness and substitute default values on timeout. Applies to all bus types.

```c
typedef struct
{
    uint32_t lastRxTimestamp;
    uint16_t timeoutMs;
    boolean  isTimedOut;
    float    defaultValue;
} SignalMonitor_t;

float GetMonitoredSignal(SignalMonitor_t *mon, float rxValue,
                          uint32_t currentTime)
{
    if (mon->isTimedOut)
    {
        return mon->defaultValue;
    }

    uint32_t elapsed = currentTime - mon->lastRxTimestamp;
    if (elapsed > mon->timeoutMs)
    {
        mon->isTimedOut = TRUE;
        ReportDtc(DTC_SIGNAL_TIMEOUT);
        return mon->defaultValue;
    }

    mon->lastRxTimestamp = currentTime;
    return rxValue;
}
```

### 6.6 Network Management State Machine

**Impact: MEDIUM (NM compliance)**

Follow AUTOSAR NM state machine transitions correctly (Bus-Sleep ↔ Prepare Bus-Sleep ↔ Network Mode).

### Automotive Ethernet / IP Stack

### 6.7 TCP Socket Lifecycle Management

**Impact: HIGH (reliable in-vehicle Ethernet communication)**

Manage TCP socket lifecycle correctly: connection establishment, keepalive monitoring, graceful shutdown, and error recovery. Automotive Ethernet TCP connections must handle ECU sleep/wake transitions and network partitioning.

**Incorrect: no error handling, no timeout, resource leak**

```c
void SendDiagData(const uint8_t *data, uint16_t len)
{
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    connect(sock, (struct sockaddr *)&serverAddr, sizeof(serverAddr));
    send(sock, data, len, 0);
    /* Socket never closed — resource leak */
}
```

**Correct: full lifecycle with timeout and cleanup**

```c
typedef struct
{
    int       fd;
    uint32_t  connectTimeoutMs;
    uint32_t  sendTimeoutMs;
    uint32_t  keepaliveIntervalMs;
    boolean   isConnected;
} TcpConnection_t;

Std_ReturnType Tcp_Connect(TcpConnection_t *conn,
                            const struct sockaddr_in *serverAddr)
{
    conn->fd = socket(AF_INET, SOCK_STREAM, 0);
    if (conn->fd < 0)
    {
        return E_NOT_OK;
    }

    struct timeval timeout;
    timeout.tv_sec  = conn->connectTimeoutMs / 1000U;
    timeout.tv_usec = (conn->connectTimeoutMs % 1000U) * 1000U;
    (void)setsockopt(conn->fd, SOL_SOCKET, SO_SNDTIMEO,
                     &timeout, sizeof(timeout));

    int keepalive = 1;
    (void)setsockopt(conn->fd, SOL_SOCKET, SO_KEEPALIVE,
                     &keepalive, sizeof(keepalive));

    if (connect(conn->fd, (const struct sockaddr *)serverAddr,
                sizeof(*serverAddr)) < 0)
    {
        close(conn->fd);
        conn->fd = -1;
        return E_NOT_OK;
    }

    conn->isConnected = TRUE;
    return E_OK;
}

Std_ReturnType Tcp_Send(TcpConnection_t *conn,
                         const uint8_t *data, uint16_t len)
{
    if (!conn->isConnected || conn->fd < 0)
    {
        return E_NOT_OK;
    }

    ssize_t totalSent = 0;
    while (totalSent < (ssize_t)len)
    {
        ssize_t sent = send(conn->fd, &data[totalSent],
                            len - (uint16_t)totalSent, MSG_NOSIGNAL);
        if (sent <= 0)
        {
            conn->isConnected = FALSE;
            return E_NOT_OK;
        }
        totalSent += sent;
    }
    return E_OK;
}

void Tcp_Disconnect(TcpConnection_t *conn)
{
    if (conn->fd >= 0)
    {
        (void)shutdown(conn->fd, SHUT_RDWR);
        (void)close(conn->fd);
        conn->fd = -1;
    }
    conn->isConnected = FALSE;
}
```

**Key considerations for automotive TCP:**
- Always set send/receive timeouts to prevent indefinite blocking
- Use `SO_KEEPALIVE` to detect broken connections during ECU idle
- Handle `EPIPE`/`ECONNRESET` for ungraceful peer disconnection
- Implement reconnection logic with exponential backoff
- Close sockets properly on ECU shutdown to release resources

### 6.8 UDP Datagram Handling

**Impact: HIGH (service discovery, streaming, low-latency communication)**

UDP is used for SOME/IP-SD (Service Discovery), DoIP vehicle identification, and real-time sensor streaming where low latency matters more than guaranteed delivery.

**Incorrect: no buffer size check, no source validation**

```c
void UdpReceive(int sock)
{
    uint8_t buf[256];
    recv(sock, buf, sizeof(buf), 0);  /* Ignores sender, no length check */
    ProcessData(buf);
}
```

**Correct: validated reception with sender identification**

```c
Std_ReturnType Udp_Receive(int sock, uint8_t *buf, uint16_t bufSize,
                            uint16_t *receivedLen,
                            struct sockaddr_in *senderAddr)
{
    socklen_t addrLen = sizeof(*senderAddr);
    ssize_t n = recvfrom(sock, buf, bufSize, 0,
                         (struct sockaddr *)senderAddr, &addrLen);

    if (n < 0)
    {
        *receivedLen = 0U;
        return E_NOT_OK;
    }

    *receivedLen = (uint16_t)n;

    if (!IsAuthorizedSender(senderAddr))
    {
        *receivedLen = 0U;
        return E_NOT_OK;
    }

    return E_OK;
}
```

**UDP multicast for SOME/IP-SD:**

```c
Std_ReturnType Udp_JoinMulticast(int sock, const char *multicastIp,
                                  const char *localIp)
{
    struct ip_mreq mreq;
    mreq.imr_multiaddr.s_addr = inet_addr(multicastIp);
    mreq.imr_interface.s_addr = inet_addr(localIp);

    if (setsockopt(sock, IPPROTO_IP, IP_ADD_MEMBERSHIP,
                   &mreq, sizeof(mreq)) < 0)
    {
        return E_NOT_OK;
    }
    return E_OK;
}
```

Reference: AUTOSAR SWS Ethernet (SWS_Eth), SOME/IP Protocol Specification (PRS_SOMEIP)

### 6.9 Diagnostics over IP (DoIP)

**Impact: HIGH (ISO 13400 compliance, remote diagnostics)**

DoIP enables UDS diagnostics over TCP/IP. Implement proper vehicle identification, routing activation, and diagnostic message handling per ISO 13400.

**DoIP activation and routing:**

```c
#define DOIP_PORT                  (13400U)
#define DOIP_HEADER_LEN            (8U)
#define DOIP_TYPE_ROUTING_ACT_REQ  (0x0005U)
#define DOIP_TYPE_ROUTING_ACT_RESP (0x0006U)
#define DOIP_TYPE_DIAG_MSG         (0x8001U)
#define DOIP_TYPE_DIAG_MSG_ACK     (0x8002U)
#define DOIP_TYPE_VEHICLE_ID_REQ   (0x0001U)
#define DOIP_TYPE_VEHICLE_ID_RESP  (0x0004U)

typedef struct
{
    uint8_t  protocolVersion;
    uint8_t  inverseVersion;
    uint16_t payloadType;
    uint32_t payloadLength;
} DoIp_Header_t;

Std_ReturnType DoIp_HandleRoutingActivation(
    const uint8_t *request, uint16_t len,
    uint8_t *response, uint16_t *respLen,
    uint16_t sourceTesterAddr)
{
    if (len < (DOIP_HEADER_LEN + 7U))
    {
        return E_NOT_OK;
    }

    uint16_t testerAddr = ((uint16_t)request[DOIP_HEADER_LEN] << 8U) |
                           request[DOIP_HEADER_LEN + 1U];

    if (testerAddr != sourceTesterAddr)
    {
        DoIp_SendRoutingResponse(response, respLen, testerAddr,
                                  DOIP_ROUTING_DENIED_UNKNOWN_SA);
        return E_NOT_OK;
    }

    if (g_activeConnections >= DOIP_MAX_CONNECTIONS)
    {
        DoIp_SendRoutingResponse(response, respLen, testerAddr,
                                  DOIP_ROUTING_DENIED_ALL_SOCKETS_ACTIVE);
        return E_NOT_OK;
    }

    DoIp_RegisterConnection(testerAddr);
    DoIp_SendRoutingResponse(response, respLen, testerAddr,
                              DOIP_ROUTING_SUCCESS);
    return E_OK;
}
```

**Vehicle identification via UDP broadcast (port 13400):**

```c
Std_ReturnType DoIp_HandleVehicleIdentRequest(
    uint8_t *response, uint16_t *respLen)
{
    DoIp_Header_t header;
    header.protocolVersion = DOIP_PROTOCOL_VERSION;
    header.inverseVersion  = (uint8_t)~DOIP_PROTOCOL_VERSION;
    header.payloadType     = DOIP_TYPE_VEHICLE_ID_RESP;

    uint16_t offset = DOIP_HEADER_LEN;
    (void)memcpy(&response[offset], g_vin, VIN_LENGTH);
    offset += VIN_LENGTH;
    response[offset++] = (uint8_t)(g_logicalAddr >> 8U);
    response[offset++] = (uint8_t)(g_logicalAddr & 0xFFU);
    (void)memcpy(&response[offset], g_eid, EID_LENGTH);
    offset += EID_LENGTH;
    (void)memcpy(&response[offset], g_gid, GID_LENGTH);
    offset += GID_LENGTH;

    header.payloadLength = offset - DOIP_HEADER_LEN;
    (void)memcpy(response, &header, DOIP_HEADER_LEN);
    *respLen = offset;
    return E_OK;
}
```

Reference: ISO 13400-2:2019 — DoIP transport protocol and network layer services

### 6.10 ARP Table Management

**Impact: MEDIUM (deterministic Ethernet address resolution)**

In automotive Ethernet networks, use static ARP entries for known ECU addresses to ensure deterministic communication startup. Dynamic ARP introduces non-deterministic latency at first communication.

```c
typedef struct
{
    uint32_t ipAddr;
    uint8_t  macAddr[6];
    boolean  isStatic;
    uint32_t lastSeenMs;
    uint16_t timeoutMs;
} ArpEntry_t;

#define ARP_TABLE_SIZE (32U)
static ArpEntry_t g_arpTable[ARP_TABLE_SIZE];

Std_ReturnType Arp_AddStaticEntry(uint32_t ip, const uint8_t mac[6])
{
    for (uint8_t i = 0U; i < ARP_TABLE_SIZE; i++)
    {
        if (g_arpTable[i].ipAddr == 0U)
        {
            g_arpTable[i].ipAddr = ip;
            (void)memcpy(g_arpTable[i].macAddr, mac, 6U);
            g_arpTable[i].isStatic = TRUE;
            return E_OK;
        }
    }
    return E_NOT_OK;  /* Table full */
}

const uint8_t *Arp_Resolve(uint32_t ip)
{
    for (uint8_t i = 0U; i < ARP_TABLE_SIZE; i++)
    {
        if (g_arpTable[i].ipAddr == ip)
        {
            return g_arpTable[i].macAddr;
        }
    }
    Arp_SendRequest(ip);  /* Dynamic resolution fallback */
    return NULL;
}
```

**Best practices:**
- Pre-populate static ARP entries for all known ECUs at boot
- Implement ARP timeout for dynamic entries (typically 60-300s in automotive)
- Handle duplicate IP detection via gratuitous ARP
- Log ARP conflicts as DTCs for network diagnostics

### 6.11 ICMP Handling for Network Diagnostics

**Impact: MEDIUM (network reachability and diagnostics)**

Use ICMP Echo (ping) for ECU reachability monitoring and Destination Unreachable for connection diagnostics.

```c
typedef struct
{
    uint32_t targetIp;
    uint16_t sequenceNum;
    uint32_t lastRoundtripUs;
    uint32_t timeoutMs;
    boolean  isReachable;
    uint8_t  failCount;
    uint8_t  maxFailCount;
} IcmpMonitor_t;

Std_ReturnType Icmp_SendEchoRequest(IcmpMonitor_t *mon)
{
    IcmpEchoPacket_t pkt;
    pkt.type       = ICMP_ECHO_REQUEST;
    pkt.code       = 0U;
    pkt.identifier = ICMP_OWN_ID;
    pkt.sequence   = mon->sequenceNum++;
    pkt.checksum   = 0U;
    pkt.checksum   = Icmp_CalculateChecksum(&pkt, sizeof(pkt));

    return Eth_Transmit(mon->targetIp, IPPROTO_ICMP,
                        (const uint8_t *)&pkt, sizeof(pkt));
}

void Icmp_HandleEchoReply(IcmpMonitor_t *mon, uint32_t roundtripUs)
{
    mon->lastRoundtripUs = roundtripUs;
    mon->isReachable = TRUE;
    mon->failCount = 0U;
}

void Icmp_HandleTimeout(IcmpMonitor_t *mon)
{
    mon->failCount++;
    if (mon->failCount >= mon->maxFailCount)
    {
        mon->isReachable = FALSE;
        ReportDtc(DTC_ETH_PEER_UNREACHABLE);
    }
}
```

**Automotive use cases:**
- Gateway ECU monitoring connectivity to all Ethernet peers
- Pre-communication reachability check before DoIP sessions
- Network startup sequencing — wait for peer ECUs before sending data

### 6.12 VLAN and QoS Priority Mapping

**Impact: MEDIUM (traffic isolation and priority, IEEE 802.1Q)**

Use VLAN tagging to isolate traffic domains (safety, diagnostics, infotainment) and QoS priority mapping to ensure time-critical frames are prioritized by Ethernet switches.

```c
typedef struct
{
    uint16_t vlanId;
    uint8_t  priority;     /* PCP: 0 (best effort) to 7 (highest) */
    const char *domain;
} VlanConfig_t;

static const VlanConfig_t g_vlanConfig[] =
{
    { .vlanId = 10U, .priority = 7U, .domain = "Safety-Critical" },
    { .vlanId = 20U, .priority = 5U, .domain = "ADAS-Sensors"    },
    { .vlanId = 30U, .priority = 4U, .domain = "Diagnostics"     },
    { .vlanId = 40U, .priority = 2U, .domain = "OTA-Updates"     },
    { .vlanId = 50U, .priority = 0U, .domain = "Infotainment"    },
};

Std_ReturnType Eth_SetVlanTag(EthFrame_t *frame, uint16_t vlanId,
                               uint8_t priority)
{
    if (priority > 7U) { return E_NOT_OK; }

    frame->vlanTag.tpid = ETH_TPID_8021Q;  /* 0x8100 */
    frame->vlanTag.tci  = (uint16_t)((uint16_t)(priority << 13U) |
                                     (vlanId & 0x0FFFU));
    return E_OK;
}
```

**Key automotive Ethernet QoS considerations:**
- Map safety-critical traffic (brake, steering) to highest PCP priority (6-7)
- Map diagnostics to mid-priority (3-4)
- Map infotainment/OTA to best-effort priority (0-1)
- Ensure switch hardware supports strict priority queuing for safety traffic

Reference: IEEE 802.1Q, AUTOSAR SWS EthSwt

### 6.13 DHCP and AutoIP Address Configuration

**Impact: MEDIUM (IP address management)**

Implement DHCP client with AutoIP (link-local) fallback for ECU IP address assignment. In automotive networks, static IP is preferred for determinism, but DHCP is used for aftermarket/diagnostic devices.

```c
typedef enum
{
    IP_CONFIG_STATIC,
    IP_CONFIG_DHCP,
    IP_CONFIG_AUTOIP,
    IP_CONFIG_DHCP_WITH_AUTOIP_FALLBACK
} IpConfigMode_t;

typedef struct
{
    IpConfigMode_t mode;
    uint32_t ipAddr;
    uint32_t subnetMask;
    uint32_t gateway;
    uint32_t dhcpLeaseTimeS;
    uint32_t dhcpRetryCount;
    boolean  isConfigured;
} IpConfig_t;

Std_ReturnType IpConfig_Init(IpConfig_t *cfg)
{
    switch (cfg->mode)
    {
        case IP_CONFIG_STATIC:
            Eth_SetIpAddress(cfg->ipAddr, cfg->subnetMask, cfg->gateway);
            cfg->isConfigured = TRUE;
            return E_OK;

        case IP_CONFIG_DHCP:
            return Dhcp_StartDiscovery(cfg);

        case IP_CONFIG_DHCP_WITH_AUTOIP_FALLBACK:
            if (Dhcp_StartDiscovery(cfg) != E_OK)
            {
                return AutoIp_GenerateLinkLocal(cfg);
            }
            return E_OK;

        case IP_CONFIG_AUTOIP:
            return AutoIp_GenerateLinkLocal(cfg);

        default:
            return E_NOT_OK;
    }
}
```

Reference: RFC 2131 (DHCP), RFC 3927 (AutoIP / IPv4 Link-Local)

### 6.14 SOME/IP Serialization

**Impact: MEDIUM (Adaptive AUTOSAR service-oriented communication)**

Use correct SOME/IP serialization with proper byte order (big-endian for header, configurable for payload), length fields, and padding alignment.

```c
typedef struct __attribute__((packed))
{
    uint16_t serviceId;
    uint16_t methodId;
    uint32_t length;
    uint16_t clientId;
    uint16_t sessionId;
    uint8_t  protocolVersion;
    uint8_t  interfaceVersion;
    uint8_t  messageType;
    uint8_t  returnCode;
} SomeIp_Header_t;

Std_ReturnType SomeIp_SerializeRequest(uint8_t *buf, uint16_t bufSize,
                                        uint16_t serviceId, uint16_t methodId,
                                        const uint8_t *payload, uint16_t payloadLen,
                                        uint16_t *totalLen)
{
    uint16_t headerAndPayload = SOMEIP_HEADER_LEN + payloadLen;
    if (headerAndPayload > bufSize) { return E_NOT_OK; }

    SomeIp_Header_t *hdr = (SomeIp_Header_t *)buf;
    hdr->serviceId        = htons(serviceId);
    hdr->methodId         = htons(methodId);
    hdr->length           = htonl(payloadLen + 8U);
    hdr->clientId         = htons(g_clientId);
    hdr->sessionId        = htons(g_nextSessionId++);
    hdr->protocolVersion  = SOMEIP_PROTOCOL_VERSION;
    hdr->interfaceVersion = 1U;
    hdr->messageType      = SOMEIP_MSG_REQUEST;
    hdr->returnCode       = SOMEIP_RC_OK;

    (void)memcpy(&buf[SOMEIP_HEADER_LEN], payload, payloadLen);
    *totalLen = headerAndPayload;
    return E_OK;
}
```

Reference: AUTOSAR PRS_SOMEIP (SOME/IP Protocol Specification)

### 6.15 SOME/IP Service Discovery

**Impact: HIGH (dynamic service availability)**

SOME/IP-SD enables dynamic service offer/find/subscribe over UDP multicast (default 224.224.224.245:30490). Critical for Adaptive AUTOSAR service-oriented architecture.

```c
#define SOMEIP_SD_PORT           (30490U)
#define SOMEIP_SD_MULTICAST_ADDR "224.224.224.245"

typedef struct
{
    uint16_t serviceId;
    uint16_t instanceId;
    uint8_t  majorVersion;
    uint32_t minorVersion;
    uint32_t ttlSeconds;
    boolean  isAvailable;
    uint32_t lastOfferTimestamp;
} SomeIpSd_ServiceEntry_t;

void SomeIpSd_HandleOfferService(const SomeIpSd_ServiceEntry_t *entry)
{
    SomeIpSd_ServiceEntry_t *local = SomeIpSd_FindService(
        entry->serviceId, entry->instanceId);

    if (local == NULL)
    {
        SomeIpSd_RegisterService(entry);
    }
    else
    {
        local->isAvailable = TRUE;
        local->ttlSeconds  = entry->ttlSeconds;
        local->lastOfferTimestamp = Timer_GetCurrentMs();
    }

    SomeIpSd_NotifyConsumers(entry->serviceId, entry->instanceId,
                              SERVICE_AVAILABLE);
}

void SomeIpSd_HandleStopOfferService(uint16_t serviceId,
                                       uint16_t instanceId)
{
    SomeIpSd_ServiceEntry_t *local = SomeIpSd_FindService(
        serviceId, instanceId);

    if (local != NULL)
    {
        local->isAvailable = FALSE;
        SomeIpSd_NotifyConsumers(serviceId, instanceId,
                                  SERVICE_UNAVAILABLE);
    }
}
```

Reference: AUTOSAR PRS_SOMEIPSD (SOME/IP Service Discovery Protocol Specification)

### Diagnostics & Routing

### 6.16 UDS Diagnostic Service Handling

**Impact: HIGH (diagnostics compliance)**

Implement diagnostic services with proper NRC (Negative Response Code) handling per ISO 14229.

```c
void UDS_ReadDataByIdentifier(const uint8_t *request, uint16_t len,
                               uint8_t *response, uint16_t *respLen)
{
    uint16_t did;

    if (len < 3U)
    {
        SendNegativeResponse(SID_READ_DATA, NRC_INCORRECT_MSG_LENGTH);
        return;
    }

    did = ((uint16_t)request[1] << 8U) | request[2];

    if (!UDS_IsSessionAllowed(did, g_currentSession))
    {
        SendNegativeResponse(SID_READ_DATA, NRC_REQUEST_OUT_OF_RANGE);
        return;
    }

    if (!UDS_IsSecurityUnlocked(did))
    {
        SendNegativeResponse(SID_READ_DATA, NRC_SECURITY_ACCESS_DENIED);
        return;
    }

    Std_ReturnType result = DID_ReadData(did, &response[3], respLen);
    if (result != E_OK)
    {
        SendNegativeResponse(SID_READ_DATA, NRC_CONDITIONS_NOT_CORRECT);
        return;
    }

    response[0] = SID_READ_DATA + 0x40U;
    response[1] = request[1];
    response[2] = request[2];
    *respLen += 3U;
}
```

Reference: ISO 14229-1:2020 (UDS)

### 6.17 Gateway Message Routing

**Impact: MEDIUM (multi-bus ECU)**

Implement proper message routing in gateway ECUs with signal mapping, timing considerations, and bus-type translation (CAN-to-CAN, CAN-to-Ethernet, Ethernet-to-CAN).

```c
typedef struct
{
    uint32_t srcMsgId;
    uint8_t  srcBus;
    uint32_t dstMsgId;
    uint8_t  dstBus;
    uint8_t  routingMode;  /* DIRECT, SIGNAL_MAPPED, GATEWAY_TRANSLATED */
    uint16_t minIntervalMs;
} GatewayRoute_t;

static const GatewayRoute_t g_routeTable[] =
{
    { 0x100U, BUS_CAN0, 0x200U, BUS_CAN1, ROUTE_DIRECT,        10U },
    { 0x300U, BUS_CAN0, 0x400U, BUS_ETH0, ROUTE_SIGNAL_MAPPED, 20U },
};

void Gateway_RouteMessage(uint8_t srcBus, uint32_t msgId,
                           const uint8_t *data, uint8_t len)
{
    for (uint16_t i = 0U; i < ROUTE_TABLE_SIZE; i++)
    {
        if ((g_routeTable[i].srcBus == srcBus) &&
            (g_routeTable[i].srcMsgId == msgId))
        {
            if (Gateway_IsRateLimited(&g_routeTable[i]))
            {
                continue;
            }
            Gateway_TransmitRouted(&g_routeTable[i], data, len);
        }
    }
}
```

---

## 7. Concurrency & RTOS Patterns

**Impact: MEDIUM-HIGH**

RTOS task management, synchronization, and inter-task communication patterns critical for multi-tasking automotive ECUs.

### 7.1 Single Responsibility Task Design

**Impact: MEDIUM (maintainability, analyzability)**

Each RTOS task should have a single responsibility with a clear, analyzable execution path.

### 7.2 Minimize Critical Section Duration

**Impact: HIGH (reduces blocking time)**

**Incorrect: large critical section**

```c
void UpdateSharedData(void)
{
    OS_EnterCritical();
    ReadAllSensors();             /* Expensive I/O in critical section */
    CalculateOutputs();           /* CPU-intensive computation */
    g_sharedOutput = output;      /* Actual shared data access */
    OS_ExitCritical();
}
```

**Correct: minimal critical section**

```c
void UpdateSharedData(void)
{
    SensorData_t localSensors;
    float localOutput;

    ReadAllSensors(&localSensors);
    localOutput = CalculateOutputs(&localSensors);

    OS_EnterCritical();
    g_sharedOutput = localOutput;  /* Only protect the shared access */
    OS_ExitCritical();
}
```

### 7.3 Correct Mutex Usage

**Impact: HIGH (prevents deadlocks)**

Always acquire mutexes in the same global order. Never hold multiple mutexes simultaneously if possible.

### 7.4 Prefer Message Queues Over Shared Memory

**Impact: MEDIUM (decouples tasks, inherently safe)**

Message queues provide inherent synchronization and decouple sender from receiver timing.

### 7.5 Priority Inheritance Protocols

**Impact: HIGH (prevents priority inversion)**

Use RTOS features like priority inheritance mutexes for shared resources between tasks of different priorities.

### 7.6 Defer ISR Processing to Task Context

**Impact: HIGH (keeps ISRs short)**

ISRs should only capture data and signal a task. All processing happens in task context.

### 7.7 Task Stack Sizing

**Impact: HIGH (prevents stack overflow)**

Calculate task stack size based on call depth analysis plus safety margin (typically 20-30%).

```c
#define TASK_CONTROL_STACK_SIZE  (1024U + 256U)  /* Analyzed 1024 + 25% margin */

OS_CreateTask(TaskControl, TASK_CONTROL_STACK_SIZE, PRIORITY_HIGH);
```

---

## 8. CAPL Scripting Best Practices

**Impact: MEDIUM-HIGH**

CAPL is the scripting language for Vector CANoe/CANalyzer, widely used for ECU simulation, testing, and analysis in automotive development.

### 8.1 Message Handler Structure

**Impact: MEDIUM (readability, performance)**

Structure message handlers for clarity, with early returns for invalid conditions.

**Incorrect: monolithic handler**

```capl
on message EngineStatus
{
    /* 50+ lines of processing mixed together */
}
```

**Correct: structured handler**

```capl
on message EngineStatus
{
    if (this.dir != rx)
        return;

    UpdateEngineStatus(this);
    CheckEngineLimits(this);
    UpdatePanel();
}
```

### 8.2 Timer Patterns

**Impact: MEDIUM (correct timing behavior)**

Use timers correctly for cyclic and one-shot operations.

```capl
variables
{
    msTimer cyclicTimer;
    msTimer debounceTimer;
    int timerPeriodMs = 100;
}

on start
{
    setTimer(cyclicTimer, timerPeriodMs);
}

on timer cyclicTimer
{
    SendCyclicMessage();
    setTimer(cyclicTimer, timerPeriodMs);  /* Re-arm for cyclic */
}

on timer debounceTimer
{
    /* One-shot: no re-arm */
    ProcessDebouncedInput();
}
```

### 8.3 Test Case Structure

**Impact: HIGH (test reliability)**

Structure test cases with clear setup, stimulus, wait, and verification phases.

```capl
testcase TC_EngineStart()
{
    /* Setup */
    testStep("Preconditions", "Set initial conditions");
    setSignal(IgnitionSwitch, 0);
    testWaitForTimeout(500);

    /* Stimulus */
    testStep("Action", "Turn ignition ON");
    setSignal(IgnitionSwitch, 1);

    /* Wait for response */
    testStep("Wait", "Wait for engine response");
    if (testWaitForSignalMatch(EngineRunning, 1, 2000) != 0)
    {
        testStepFail("Verification", "Engine did not start within 2s");
        return;
    }

    /* Verification */
    testStepPass("Verification", "Engine started successfully");
}
```

### 8.4 Signal Access via Database Symbols

**Impact: MEDIUM (maintainability)**

Always access signals via database symbolic names, never via raw byte offsets.

**Incorrect: raw byte access**

```capl
on message 0x100
{
    byte engineRpm_h = this.byte(0);
    byte engineRpm_l = this.byte(1);
    int rpm = (engineRpm_h << 8) | engineRpm_l;
}
```

**Correct: symbolic signal access**

```capl
on message EngineData
{
    int rpm = this.EngineRPM.phys;
    write("Engine RPM: %d", rpm);
}
```

### 8.5 Error Frame Handling in Simulation

**Impact: MEDIUM (robust simulation)**

Handle error frames and bus-off conditions in simulation nodes.

### 8.6 Environment Variables for Panel Interaction

**Impact: LOW (panel integration)**

Use environment variables as the bridge between CAPL and CANoe panels.

```capl
on envVar EnvSetTargetSpeed
{
    int speed = getValue(this);
    if ((speed >= 0) && (speed <= 250))
    {
        g_targetSpeed = speed;
        UpdateSpeedDisplay();
    }
}
```

### 8.7 Diagnostic Testing Patterns

**Impact: HIGH (diagnostic validation)**

Implement diagnostic test patterns for UDS services with proper request/response validation.

```capl
testcase TC_ReadDID_F190()
{
    diagRequest ECU1.VehicleIdentification_Read req;
    diagResponse ECU1.VehicleIdentification_Read resp;

    testStep("Request", "Send ReadDataByIdentifier for VIN");
    diagSendRequest(req);

    if (testWaitForDiagResponse(resp, 2000) != 0)
    {
        testStepFail("Response", "No response received within 2s");
        return;
    }

    if (diagGetLastResponseCode(resp) != 0)  /* 0 = positive response */
    {
        long nrc = diagGetLastResponseCode(resp);
        testStepFail("Response", "Negative response: NRC 0x%02X", nrc);
        return;
    }

    testStepPass("Response", "VIN read successfully");
}
```

### 8.8 Node Simulation with State Machines

**Impact: MEDIUM (realistic simulation)**

Design simulation nodes using state machines for realistic ECU behavior.

```capl
variables
{
    enum SimState { INIT, RUNNING, FAULT, OFF };
    enum SimState currentState = INIT;
}

void TransitionTo(enum SimState newState)
{
    write("State transition: %d -> %d", currentState, newState);
    currentState = newState;
}

on timer cyclicTimer
{
    switch (currentState)
    {
        case INIT:
            SendInitMessages();
            TransitionTo(RUNNING);
            break;
        case RUNNING:
            SendNormalMessages();
            break;
        case FAULT:
            SendFaultMessages();
            break;
        case OFF:
            /* Send nothing */
            break;
    }
    setTimer(cyclicTimer, 10);
}
```

---

## 9. Code Organization & Architecture

**Impact: MEDIUM**

Proper software architecture ensures maintainability, portability, and testability of automotive embedded code.

### 9.1 Hardware Abstraction Layer

**Impact: HIGH (portability, testability)**

Isolate hardware dependencies behind abstract interfaces for portability across MCU families and unit test mockability.

```c
/* HAL interface — platform independent */
typedef struct
{
    Std_ReturnType (*Init)(const GpioConfig_t *config);
    Std_ReturnType (*WritePin)(uint8_t port, uint8_t pin, uint8_t level);
    uint8_t        (*ReadPin)(uint8_t port, uint8_t pin);
} Gpio_DriverApi_t;

/* Platform-specific implementation registered at startup */
extern const Gpio_DriverApi_t Gpio_Driver;
```

### 9.2 Clean Module Interfaces

**Impact: MEDIUM (information hiding)**

Each module exposes only its public API through a header. Internal state and helper functions are static.

### 9.3 Table-Driven State Machines

**Impact: MEDIUM (scalable, verifiable)**

```c
typedef Std_ReturnType (*StateHandler_t)(const Event_t *event,
                                          SystemState_t *nextState);

typedef struct
{
    SystemState_t currentState;
    EventType_t   event;
    StateHandler_t handler;
} StateTransition_t;

static const StateTransition_t g_transitions[] =
{
    { STATE_INIT,    EVT_POWER_ON,   Handler_InitToRun     },
    { STATE_RUNNING, EVT_FAULT,      Handler_RunToFault    },
    { STATE_RUNNING, EVT_SHUTDOWN,   Handler_RunToShutdown },
    { STATE_FAULT,   EVT_RECOVERY,   Handler_FaultToInit   },
};
```

### 9.4 Callback Patterns for Layer Decoupling

**Impact: MEDIUM (loose coupling between layers)**

Use function pointers / callbacks to decouple upper layers from lower layers.

### 9.5 Separate Configuration from Logic

**Impact: MEDIUM (calibratable parameters)**

Separate calibration/configuration data from application logic for ECU variant management.

```c
/* Configuration — separate file, calibratable */
const SensorConfig_t g_sensorConfig =
{
    .samplingRateMs  = 10U,
    .filterCoeff     = 0.85f,
    .minValid        = -40.0f,
    .maxValid        = 150.0f,
    .defaultValue    = 25.0f,
};

/* Logic — uses configuration */
void SensorTask_10ms(void)
{
    float raw = ReadSensorRaw();
    float filtered = ApplyFilter(raw, g_sensorConfig.filterCoeff);

    if (!IsInRange(filtered, g_sensorConfig.minValid,
                   g_sensorConfig.maxValid))
    {
        filtered = g_sensorConfig.defaultValue;
    }
}
```

### 9.6 Layered Architecture Pattern

**Impact: HIGH (AUTOSAR compliance, maintainability)**

Follow AUTOSAR layered architecture: MCAL → ECU Abstraction → Service Layer → RTE → Application SWC.

---

## 10. Performance Optimization

**Impact: MEDIUM**

Embedded microcontrollers have limited CPU, RAM, and Flash. Performance optimization is about doing more within tight resource budgets.

### 10.1 Loop Optimization for Embedded Targets

**Impact: MEDIUM (reduces CPU cycles)**

**Incorrect: recalculated on each iteration**

```c
for (uint16_t i = 0U; i < GetArraySize(arr); i++)
{
    arr[i] = arr[i] * GetScaleFactor();
}
```

**Correct: cached values**

```c
const uint16_t size = GetArraySize(arr);
const float scale = GetScaleFactor();
for (uint16_t i = 0U; i < size; i++)
{
    arr[i] = arr[i] * scale;
}
```

### 10.2 Lookup Tables Over Runtime Computation

**Impact: HIGH (trades Flash for CPU cycles)**

Pre-compute values and store in Flash ROM to avoid runtime computation.

```c
/* Pre-computed sine table (0-90 degrees, 1-degree resolution) */
static const int16_t g_sinTable[91] =
{
    0, 175, 349, 523, /* ... values scaled by 10000 */
};

int16_t FastSin(uint16_t angleDeg)
{
    angleDeg = angleDeg % 360U;
    if (angleDeg <= 90U)  { return g_sinTable[angleDeg]; }
    if (angleDeg <= 180U) { return g_sinTable[180U - angleDeg]; }
    if (angleDeg <= 270U) { return -g_sinTable[angleDeg - 180U]; }
    return -g_sinTable[360U - angleDeg];
}
```

### 10.3 Bitwise Operations for Flags and Registers

**Impact: MEDIUM (efficient register manipulation)**

Use bitwise operations for hardware register access and status flags.

```c
#define STATUS_FLAG_READY    ((uint8_t)0x01U)
#define STATUS_FLAG_ERROR    ((uint8_t)0x02U)
#define STATUS_FLAG_BUSY     ((uint8_t)0x04U)

static uint8_t g_statusFlags = 0U;

static inline void SetFlag(uint8_t flag)   { g_statusFlags |= flag; }
static inline void ClearFlag(uint8_t flag) { g_statusFlags &= (uint8_t)~flag; }
static inline boolean IsFlagSet(uint8_t flag)
{
    return ((g_statusFlags & flag) != 0U);
}
```

### 10.4 Cache-Friendly Data Organization

**Impact: MEDIUM (relevant for high-end automotive MCUs with cache)**

Organize frequently accessed data contiguously for cache line efficiency.

### 10.5 Inline Critical Functions

**Impact: LOW-MEDIUM (eliminates call overhead)**

Use `static inline` for small, frequently called functions.

### 10.6 Fixed-Point Arithmetic

**Impact: HIGH (avoids FPU dependency)**

Use fixed-point instead of floating-point on MCUs without hardware FPU.

```c
typedef int32_t fixed16_16_t;  /* Q16.16 format */

#define FIXED_ONE     ((fixed16_16_t)0x00010000)
#define FLOAT_TO_FIX(f)  ((fixed16_16_t)((f) * 65536.0f))
#define FIX_TO_FLOAT(x)  ((float)(x) / 65536.0f)

static inline fixed16_16_t FixMul(fixed16_16_t a, fixed16_16_t b)
{
    return (fixed16_16_t)(((int64_t)a * b) >> 16);
}
```

### 10.7 DMA for Bulk Data Transfers

**Impact: MEDIUM (offloads CPU)**

Use DMA for ADC sampling, SPI/UART transfers, and memory-to-memory copies.

---

## 11. Build, Compilation & Static Analysis

**Impact: MEDIUM**

Proper build configuration catches bugs at compile time and ensures code quality.

### 11.1 Compiler Warnings as Errors

**Impact: HIGH (catches bugs at compile time)**

```makefile
CFLAGS += -Wall -Wextra -Werror -Wpedantic
CFLAGS += -Wconversion -Wsign-conversion
CFLAGS += -Wcast-align -Wcast-qual
CFLAGS += -Wdouble-promotion -Wfloat-equal
CFLAGS += -Wshadow -Wstrict-prototypes
```

### 11.2 Static Analysis Integration

**Impact: HIGH (finds bugs without execution)**

Integrate static analysis tools into CI/CD: PC-lint, Polyspace, Coverity, cppcheck.

### 11.3 Safety-Optimized Compiler Flags

**Impact: MEDIUM (trade-offs between safety and performance)**

For safety-critical code, avoid aggressive optimization that may reorder or eliminate safety checks.

```makefile
# Safety-critical modules: limited optimization
CFLAGS_SAFETY += -O1 -fno-strict-aliasing -fno-delete-null-pointer-checks

# Non-safety modules: optimize for size
CFLAGS_NORMAL += -Os
```

### 11.4 Link-Time Optimization

**Impact: LOW-MEDIUM (cross-module optimization)**

Use LTO for non-safety-critical modules to reduce code size.

### 11.5 Reproducible Builds

**Impact: MEDIUM (traceability for ISO 26262)**

Ensure identical binary output from identical source for traceability and audit.

---

## 12. Testing & Verification

**Impact: MEDIUM**

Thorough testing is required by ISO 26262 with coverage targets varying by ASIL level.

### 12.1 Unit Test Patterns for Embedded

**Impact: HIGH (early defect detection)**

Structure unit tests for embedded C with proper hardware mocking.

```c
/* Using Unity test framework */
void test_PlausibilityCheck_RejectsOutOfRange(void)
{
    PlausibilityCheck_t check = {
        .currentValue = 25.0f,
        .previousValue = 25.0f,
        .maxDeltaPerCycle = 5.0f,
        .minValid = -40.0f,
        .maxValid = 150.0f
    };

    TEST_ASSERT_FALSE(IsPlausible(&check, 200.0f));  /* Above max */
    TEST_ASSERT_FALSE(IsPlausible(&check, -50.0f));  /* Below min */
    TEST_ASSERT_TRUE(IsPlausible(&check, 26.0f));    /* Valid */
}
```

### 12.2 Mock Hardware Dependencies

**Impact: HIGH (enables host-based testing)**

Use function pointers or link-time substitution to mock hardware.

### 12.3 Boundary Value Testing

**Impact: MEDIUM (systematic defect detection)**

Test at and around boundary values systematically: min, min-1, min+1, max, max-1, max+1, zero, typical.

### 12.4 Coverage Targets per ASIL Level

**Impact: HIGH (ISO 26262 compliance)**

| ASIL Level | Statement Coverage | Branch Coverage | MC/DC |
|------------|-------------------|-----------------|-------|
| QM         | Recommended       | —               | —     |
| ASIL A     | Required          | Recommended     | —     |
| ASIL B     | Required          | Required        | —     |
| ASIL C     | Required          | Required        | Recommended |
| ASIL D     | Required          | Required        | Required    |

### 12.5 Integration Testing Patterns

**Impact: MEDIUM (validates inter-module behavior)**

Test module interactions and communication paths.

### 12.6 HIL/SIL Test Patterns

**Impact: HIGH (validates real-time behavior)**

Structure Hardware-in-the-Loop and Software-in-the-Loop tests for timing and integration verification.

---

## 13. Security & Cybersecurity — ISO 21434

**Impact: HIGH**

ISO/SAE 21434 defines cybersecurity engineering requirements for road vehicles. As vehicles become more connected (Ethernet, OTA, V2X), security is no longer optional. These rules address secure coding patterns for automotive ECUs.

### 13.1 Secure Boot Chain Verification

**Impact: CRITICAL (prevents unauthorized firmware execution)**

Verify firmware integrity at every stage of the boot chain using cryptographic signatures.

```c
typedef struct
{
    uint32_t firmwareAddr;
    uint32_t firmwareSize;
    uint8_t  signature[256];
    uint16_t signatureLen;
    uint8_t  publicKeyHash[32];
} SecureBoot_Entry_t;

Std_ReturnType SecureBoot_VerifyStage(const SecureBoot_Entry_t *entry)
{
    uint8_t computedHash[32];
    Crypto_HashSha256((const uint8_t *)entry->firmwareAddr,
                      entry->firmwareSize, computedHash);

    if (Crypto_VerifySignatureRsa(computedHash, sizeof(computedHash),
                                   entry->signature, entry->signatureLen,
                                   entry->publicKeyHash) != CRYPTO_OK)
    {
        ReportDtc(DTC_SECURE_BOOT_FAILURE);
        EnterSafeState();
        return E_NOT_OK;
    }

    return E_OK;
}
```

### 13.2 Secure In-Vehicle Communication (TLS/DTLS)

**Impact: HIGH (protects Ethernet communication)**

Use TLS for TCP-based and DTLS for UDP-based in-vehicle Ethernet communication to prevent eavesdropping and tampering.

```c
typedef struct
{
    TlsContext_t *ctx;
    const char   *caCertPath;
    const char   *clientCertPath;
    const char   *clientKeyPath;
    uint16_t      minVersion;  /* TLS_VERSION_1_2 or TLS_VERSION_1_3 */
} SecureConnection_t;

Std_ReturnType Tls_InitClient(SecureConnection_t *conn)
{
    conn->ctx = Tls_CreateContext(TLS_CLIENT_MODE);
    if (conn->ctx == NULL) { return E_NOT_OK; }

    Tls_SetMinVersion(conn->ctx, conn->minVersion);
    Tls_LoadCaCert(conn->ctx, conn->caCertPath);
    Tls_LoadClientCert(conn->ctx, conn->clientCertPath, conn->clientKeyPath);

    /* Disable insecure cipher suites */
    Tls_SetCipherList(conn->ctx,
        "TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384");

    return E_OK;
}
```

### 13.3 Cryptographic Key Management

**Impact: CRITICAL (protects secret material)**

Store cryptographic keys in secure hardware (HSM/SHE) when available. Never store keys in plain text in Flash.

```c
Std_ReturnType Crypto_LoadKeyFromHsm(uint16_t keySlot,
                                       CryptoKey_t *key)
{
    if (Hsm_IsAvailable() != TRUE)
    {
        ReportDtc(DTC_HSM_NOT_AVAILABLE);
        return E_NOT_OK;
    }

    if (Hsm_ReadKey(keySlot, key->material, &key->length) != HSM_OK)
    {
        return E_NOT_OK;
    }

    key->isHsmBacked = TRUE;
    return E_OK;
}
```

### 13.4 Secure UDS Authentication

**Impact: HIGH (prevents unauthorized diagnostic access)**

Implement UDS Authentication service (0x29) for diagnostic tool verification before allowing security-sensitive operations.

### 13.5 Input Sanitization from External Interfaces

**Impact: HIGH (prevents injection and buffer attacks)**

Sanitize and validate all data received from external interfaces (CAN, Ethernet, USB, diagnostic).

```c
Std_ReturnType Sanitize_DiagPayload(const uint8_t *input, uint16_t inputLen,
                                      uint8_t *output, uint16_t outputSize,
                                      uint16_t *outputLen)
{
    if ((input == NULL) || (output == NULL) || (outputLen == NULL))
    {
        return E_NOT_OK;
    }

    if (inputLen > outputSize)
    {
        ReportDtc(DTC_INPUT_OVERFLOW_ATTEMPT);
        return E_NOT_OK;
    }

    if (inputLen > MAX_DIAG_PAYLOAD_SIZE)
    {
        return E_NOT_OK;
    }

    for (uint16_t i = 0U; i < inputLen; i++)
    {
        if (!IsValidByte(input[i]))
        {
            ReportDtc(DTC_INVALID_INPUT_DATA);
            return E_NOT_OK;
        }
    }

    (void)memcpy(output, input, inputLen);
    *outputLen = inputLen;
    return E_OK;
}
```

### 13.6 Secure OTA/Reflash with Signature Verification

**Impact: CRITICAL (prevents malicious firmware installation)**

All firmware updates must be cryptographically signed and verified before flashing.

### 13.7 Security Domain Access Control

**Impact: HIGH (freedom from interference)**

Enforce access control between security domains to prevent unauthorized cross-domain data flow.

### 13.8 Correct Cryptographic Primitive Usage

**Impact: CRITICAL (prevents weak crypto)**

Use approved algorithms (AES-128/256, SHA-256, HMAC-SHA256, CMAC) and never implement custom cryptography.

```c
/* Use standard AUTOSAR CSM (Crypto Service Manager) interface */
Std_ReturnType ComputeMessageMac(const uint8_t *data, uint32_t dataLen,
                                   uint8_t *mac, uint32_t *macLen)
{
    Csm_ReturnType ret;

    ret = Csm_MacGenerate(CSM_KEY_ID_MSG_AUTH,
                           CRYPTO_OPERATIONMODE_SINGLECALL,
                           data, dataLen,
                           mac, macLen);

    if (ret != CSM_E_OK)
    {
        ReportDtc(DTC_CRYPTO_MAC_FAILURE);
        return E_NOT_OK;
    }
    return E_OK;
}
```

Reference: ISO/SAE 21434:2021 — Road vehicles — Cybersecurity engineering, AUTOSAR SWS Crypto Service Manager

---

## 14. Tool Integration

**Impact: MEDIUM**

Automotive development relies on specialized tool chains with standardized exchange formats. Correct generation and maintenance of these artifacts is essential for tool interoperability.

### 14.1 A2L/ASAP2 Calibration Descriptions

**Impact: MEDIUM (enables measurement and calibration)**

Generate and maintain A2L files that accurately describe calibration parameters and measurement signals for tools like INCA, CANape.

```
/* A2L parameter definition — must match C source exactly */
/begin CHARACTERISTIC
    CalParam_BoostPressureTarget  /* Name matches C variable */
    "Target boost pressure"
    VALUE
    0x00080100                     /* Address from linker map */
    DAMOS_FW                       /* Deposit format */
    0.0                            /* Max diff */
    CalParam_Conv_Pressure         /* Conversion formula */
    0.0                            /* Lower limit */
    3.0                            /* Upper limit (bar) */
/end CHARACTERISTIC
```

### 14.2 ODX/PDX Diagnostic Descriptions

**Impact: MEDIUM (diagnostic tool compatibility)**

Structure ODX diagnostic data correctly for interoperability with diagnostic tools (EDIABAS, D-PDU API).

### 14.3 FIBEX Network Descriptions

**Impact: MEDIUM (Ethernet network description)**

Maintain FIBEX files for automotive Ethernet network topology and service endpoint descriptions.

### 14.4 DBC/ARXML and Code Synchronization

**Impact: HIGH (prevents signal mismatches)**

Keep DBC (CAN database) / ARXML definitions synchronized with source code signal structures to prevent runtime communication failures.

### 14.5 XCP Measurement and Calibration Protocol

**Impact: MEDIUM (real-time calibration)**

Implement XCP (Universal Measurement and Calibration Protocol) for real-time parameter tuning and signal measurement.

```c
typedef struct
{
    uint32_t address;
    uint8_t  addressExtension;
    uint8_t  dataSize;
} Xcp_DaqEntry_t;

Std_ReturnType Xcp_HandleShortUpload(const uint8_t *request,
                                       uint8_t *response, uint16_t *respLen)
{
    uint8_t numBytes = request[1];
    uint32_t addr    = ((uint32_t)request[4] << 24U) |
                       ((uint32_t)request[5] << 16U) |
                       ((uint32_t)request[6] << 8U)  |
                       (uint32_t)request[7];

    if (numBytes > XCP_MAX_CTO_SIZE - 1U)
    {
        Xcp_SendError(XCP_ERR_OUT_OF_RANGE);
        return E_NOT_OK;
    }

    if (!Xcp_IsAddressReadable(addr, numBytes))
    {
        Xcp_SendError(XCP_ERR_ACCESS_DENIED);
        return E_NOT_OK;
    }

    response[0] = XCP_PID_RESPONSE;
    (void)memcpy(&response[1], (const void *)addr, numBytes);
    *respLen = 1U + numBytes;
    return E_OK;
}
```

Reference: ASAM XCP Protocol Layer Specification

### 14.6 AUTOSAR ARXML Configuration

**Impact: HIGH (AUTOSAR toolchain integration)**

Generate and parse AUTOSAR ARXML configuration files correctly for BSW configuration, SWC descriptions, and ECU extracts.

---

## 15. MISRA Grouped Topics

**Impact: HIGH**

Beyond the core MISRA rules covered in Section 2, the MISRA C:2012 and C++:2023 standards define grouped rule families that address specific language aspects in depth. This section provides representative patterns from the most impactful groups — type system and deviation management — and references the full set of 12+ grouped rule files.

### 15.1 MISRA Type System — Conversions, Casting, Essential Type Model

**Impact: HIGH (prevents silent data corruption from implicit conversions)**

The MISRA essential type model (Rules 10.x) and cast rules (Rules 11.x) form the backbone of type-safe embedded C/C++. Implicit conversions, narrowing, and unchecked casts are the single largest source of field defects in automotive software.

**Rule 10.3 — Narrowing Assignment (most common violation)**

Incorrect (implicit narrowing):

```c
uint32_t raw_adc = ReadAdc();
uint16_t filtered = raw_adc * coeff;  /* narrowing: uint32_t → uint16_t */
int8_t temp = (raw_adc - 500) / 10;   /* signed/unsigned mix + narrowing */
```

Correct (explicit cast with range guard):

```c
uint32_t raw_adc = ReadAdc();
uint32_t calc = raw_adc * coeff;

if (calc <= UINT16_MAX) {
    uint16_t filtered = (uint16_t)calc;
} else {
    filtered = UINT16_MAX;
    Dem_ReportError(DEM_ADC_RANGE);
}
```

**Rule 11.3 — Object Pointer Cast (hardware register access)**

Incorrect (arbitrary pointer reinterpretation):

```c
uint8_t buffer[64];
uint32_t *word_ptr = (uint32_t *)buffer;  /* alignment not guaranteed */
```

Correct (use memcpy for type punning):

```c
uint8_t buffer[64];
uint32_t word_val;
(void)memcpy(&word_val, &buffer[0], sizeof(word_val));
```

Reference: MISRA C:2012 Rules 10.1–10.8, 11.1–11.9; MISRA C++:2023 Rules 7.0.x, 8.2.x  
See full details: `rules/misra-type-system.md`

### 15.2 MISRA Deviation Process

**Impact: MEDIUM (establishes auditable deviation workflow required for ISO 26262)**

MISRA compliance does not mean zero deviations — it means every deviation is documented, justified, risk-assessed, and approved. A well-managed deviation process is essential for ISO 26262 certification.

**Deviation Record Format:**

```
╔══════════════════════════════════════════════════════════════╗
║ MISRA DEVIATION RECORD                                       ║
╠══════════════════════════════════════════════════════════════╣
║ Deviation ID:     DEV-<CATEGORY>-<NNN>                       ║
║ MISRA Rule:       MISRA C:2012 Rule X.Y / Dir X.Y            ║
║ Rule Category:    Required / Advisory                        ║
║ Justification:    [technical reason]                         ║
║ Risk Assessment:  [risk + mitigation]                        ║
║ ASIL Impact:      [affected ASIL level]                      ║
║ Approved by:      [Safety Manager name + date]               ║
╚══════════════════════════════════════════════════════════════╝
```

**Static Analysis Tool Suppression Annotations:**

```c
/* Polyspace suppression */
/* polyspace<MISRA-C:11.3:Not a defect:Justify with DEV-TYPE-001> Hardware register cast */

/* PC-lint/FlexeLint suppression */
/*lint -e{9078} DEV-TYPE-001: hardware register cast */

/* PRQA/Helix QAC suppression */
/* PRQA S 0303 ++ DEV-TYPE-001: hardware register cast */
```

Reference: MISRA C:2012 Appendix A; MISRA Compliance:2020; ISO 26262-6:2018 clause 8  
See full details: `rules/misra-deviation-process.md`

### 15.3 Additional MISRA Rule Groups

The following rule files provide deep-dive guidance for specific MISRA rule families:

- `rules/misra-pointer-safety.md` — Pointer validity and null-check patterns
- `rules/misra-pointer-arithmetic.md` — Restricted pointer arithmetic
- `rules/misra-control-flow.md` — Control flow integrity and reachability
- `rules/misra-expressions.md` — Expression evaluation and operator usage
- `rules/misra-declarations.md` — Declaration and scope rules
- `rules/misra-functions.md` — Function parameter and return conventions
- `rules/misra-preprocessor.md` — Preprocessor directive safety
- `rules/misra-standard-library.md` — Standard library usage restrictions
- `rules/misra-initialization.md` — Variable initialization requirements
- `rules/misra-memory-model.md` — Memory model and object lifetime
- `rules/misra-concurrency.md` — Concurrency and data race prevention
- `rules/misra-boolean-expressions.md` — Boolean expression patterns
- `rules/misra-side-effects.md` — Side effect restrictions in expressions

---

## 16. AUTOSAR Classic BSW

**Impact: HIGH**

AUTOSAR Classic Platform defines standardized Basic Software (BSW) modules with specific API contracts, initialization sequences, and error handling patterns. Incorrect usage of BSW APIs causes failed startup, corrupted bus communication, missed wakeup events, and data loss. This section covers the most critical BSW modules with representative patterns.

### 16.1 EcuM — ECU State Manager

**Impact: HIGH (incorrect sequencing causes failed startup or uncontrolled shutdown)**

The ECU State Manager orchestrates the full ECU lifecycle: startup initialization, RUN state management, sleep entry, and wakeup validation.

**Incorrect (calling BSW modules before EcuM_Init completes):**

```c
void main(void)
{
    Com_Init(&ComConfig);      /* COM initialized too early — Dem/SchM not ready */
    EcuM_Init(&EcuM_Config);
}
```

**Correct (EcuM_Init drives the full startup sequence):**

```c
void main(void)
{
    EcuM_Init(&EcuM_Config);
    /* EcuM_Init internally calls:
     *   1. EcuM_AL_DriverInitZero()  — pre-OS drivers (Mcu, Port, Dio)
     *   2. Os_Init / StartOS()
     *   3. EcuM_StartupTwo():
     *        - EcuM_AL_DriverInitOne()  — post-OS drivers (SchM, BswM)
     *        - BswM_Init() / SchM_Init()
     */
}
```

**Correct wakeup validation (two-phase: set pending, then validate):**

```c
void EcuM_CheckWakeup(EcuM_WakeupSourceType wakeupSource)
{
    EcuM_SetWakeupEvent(wakeupSource);
}

void EcuM_ValidateWakeupEvent(EcuM_WakeupSourceType wakeupSource)
{
    /* Phase 2: Wakeup confirmed — EcuM transitions to RUN.
     * If validation timeout expires before this call, wakeup is rejected. */
}
```

Reference: AUTOSAR Classic R22-11 SWS_EcuM  
See full details: `rules/autosar-classic-ecum.md`

### 16.2 COM — Signal Packing and Transmission Modes

**Impact: HIGH (wrong signal packing causes corrupted bus data)**

The AUTOSAR COM module handles signal-level access for application SWCs: packing signals into I-PDUs, unpacking received I-PDUs, managing transmission modes, and monitoring reception deadlines.

**Incorrect (manual bit manipulation instead of COM API):**

```c
void App_SendEngineSpeed(uint16 rpm)
{
    uint8 pdu[8];
    pdu[0] = (uint8)(rpm & 0xFFu);
    pdu[1] = (uint8)((rpm >> 8u) & 0xFFu);
    CanIf_Transmit(PDU_ENGINE_STATUS, pdu);
}
```

**Correct (use Com_SendSignal for type-safe, config-driven packing):**

```c
void App_SendEngineSpeed(uint16 rpm)
{
    uint8 status = Com_SendSignal(ComConf_ComSignal_EngineSpeed, &rpm);
    if (status == COM_SERVICE_NOT_AVAILABLE)
    {
        /* I-PDU group not started — handle gracefully */
    }
}
```

**Reception Deadline Monitoring:**

```c
/* ARXML Configuration:
 *   ComRxDataTimeoutAction = SUBSTITUTE
 *   ComTimeout = 100ms
 *   ComTimeoutNotification = App_BrakeSigTimeout
 */
void App_BrakeSigTimeout(void)
{
    BrakeCtrl_EnterSafeState();
    Dem_ReportErrorStatus(DTC_BRAKE_SIG_TIMEOUT, DEM_EVENT_STATUS_FAILED);
}
```

Reference: AUTOSAR Classic R22-11 SWS_COM  
See full details: `rules/autosar-classic-com.md`

### 16.3 Additional Classic BSW Modules

The following rule files provide detailed patterns for additional BSW modules:

- `rules/autosar-classic-bswm.md` — BswM mode arbitration and action lists
- `rules/autosar-classic-nvm.md` — NvM block management and data persistence
- `rules/autosar-classic-dcm-dem.md` — DCM/DEM diagnostic event management
- `rules/autosar-classic-pdu-router.md` — PduR routing and gateway configuration
- `rules/autosar-classic-canif-cantp.md` — CAN interface and transport protocol
- `rules/autosar-classic-os.md` — AUTOSAR OS task/ISR configuration

---

## 17. AUTOSAR Adaptive Platform

**Impact: HIGH**

The AUTOSAR Adaptive Platform (R24-10) provides a POSIX-based runtime for high-performance automotive ECUs running Linux. It uses service-oriented architecture with ara::com for communication, ara::core for exception-free error handling, and specialized APIs for execution, diagnostics, logging, and health monitoring. All examples use C++14 per the Adaptive Platform specification.

### 17.1 ara::com — Service-Oriented Communication

**Impact: HIGH (incorrect usage causes service discovery failures or event data loss)**

ara::com implements the proxy/skeleton pattern: service providers implement skeletons, consumers use proxies. The binding layer (SOME/IP, DDS) is abstracted.

**Incorrect (hardcoding service endpoints):**

```cpp
auto proxy = RadarService::Proxy("192.168.1.10", 30509);
```

**Correct (dynamic service discovery via ara::com):**

```cpp
using namespace ara::com;

auto handles = RadarService::Proxy::FindService(
    InstanceIdentifier("RadarFrontLeft"));

if (!handles.empty())
{
    RadarService::Proxy proxy(handles[0]);
}
```

**Correct event subscription pattern:**

```cpp
void SetupEventSubscription(RadarService::Proxy& proxy)
{
    proxy.radarObjects.Subscribe(10);

    proxy.radarObjects.SetReceiveHandler([&proxy]() {
        proxy.radarObjects.GetNewSamples(
            [](ara::com::SamplePtr<const RadarObjectList> sample) {
                for (const auto& obj : sample->objects)
                {
                    ProcessRadarObject(obj);
                }
            },
            5);
    });
}
```

Reference: AUTOSAR Adaptive R24-10 SWS_CommunicationManagement  
See full details: `rules/autosar-adaptive-ara-com.md`

### 17.2 ara::core — Result, ErrorCode, Future/Promise

**Impact: HIGH (exceptions are prohibited in ASIL-rated automotive software)**

ara::core provides `Result<T, E>` for exception-free error handling — essential because C++ exceptions are prohibited in safety-critical automotive software.

**Incorrect (using C++ exceptions):**

```cpp
RadarData ReadRadar()
{
    if (!radarHw.IsReady())
        throw std::runtime_error("Radar not ready");  // Prohibited in ASIL code
    return radarHw.GetData();
}
```

**Correct (using ara::core::Result):**

```cpp
ara::core::Result<RadarData> ReadRadar()
{
    if (!radarHw.IsReady())
    {
        return ara::core::Result<RadarData>::FromError(
            MyErrors::MakeErrorCode(MyErrors::Errc::kHardwareNotReady));
    }
    return radarHw.GetData();
}

void ProcessRadar()
{
    auto result = ReadRadar();
    if (result.HasValue())
    {
        Process(result.Value());
    }
    else
    {
        HandleError(result.Error());
    }
}
```

Reference: AUTOSAR Adaptive R24-10 SWS_CoreTypes  
See full details: `rules/autosar-adaptive-ara-core.md`

### 17.3 Additional Adaptive Platform APIs

The following rule files provide detailed patterns for additional ara:: APIs:

- `rules/autosar-adaptive-ara-exec.md` — ara::exec execution management and process lifecycle
- `rules/autosar-adaptive-ara-diag.md` — ara::diag diagnostic services
- `rules/autosar-adaptive-ara-log.md` — ara::log structured logging
- `rules/autosar-adaptive-ara-per.md` — ara::per persistent data storage
- `rules/autosar-adaptive-ara-phm.md` — ara::phm platform health management

---

## 18. ECU Boot Sequence

**Impact: HIGH**

The ECU boot sequence is the foundation of system integrity. Incorrect startup ordering causes undefined behavior, stack corruption, or silent peripheral failures before any diagnostic output is available. This section covers bare-metal, AUTOSAR Classic, AUTOSAR Adaptive, and secure boot patterns.

### 18.1 Bare-Metal Boot Sequence

**Impact: HIGH (incorrect ordering causes undefined behavior before diagnostics are available)**

The sequence must follow: reset vector → stack pointer setup → vector table relocation → C runtime init (BSS clear, data copy) → `main()`.

**Incorrect (using global data before C runtime init):**

```c
volatile uint32_t g_bootCount = 0;  /* .data section — not yet copied from flash */

void Reset_Handler(void)
{
    g_bootCount++;  /* Reading garbage — .data not copied yet */
    main();
}
```

**Correct (proper startup with C runtime init):**

```c
extern uint32_t _sidata, _sdata, _edata, _sbss, _ebss;

__attribute__((naked, noreturn))
void Reset_Handler(void)
{
    __asm volatile("ldr sp, =_estack");

    /* Copy .data from flash to RAM */
    uint32_t *src = &_sidata;
    uint32_t *dst = &_sdata;
    while (dst < &_edata) { *dst++ = *src++; }

    /* Zero .bss */
    dst = &_sbss;
    while (dst < &_ebss) { *dst++ = 0U; }

    main();
    for (;;) {}
}
```

Reference: ARM Cortex-M Architecture Reference Manual  
See full details: `rules/boot-baremetal-startup.md`

### 18.2 Secure Boot Chain of Trust

**Impact: CRITICAL (broken chain allows execution of tampered firmware)**

Secure boot establishes a hardware-rooted chain of trust: ROM bootloader (immutable) → verify 1st-stage bootloader → verify 2nd-stage → verify application. Each stage cryptographically verifies the next before transferring control.

**Incorrect (jumping to app without verification):**

```c
void Bootloader_JumpToApp(void)
{
    EntryPoint_t app_entry = (EntryPoint_t)(*(volatile uint32_t *)(APP_START_ADDR + 4));
    app_entry();  /* No integrity check — attacker can run arbitrary code */
}
```

**Correct (HSM-verified secure boot):**

```c
static BootVerifyResult_t SecureBoot_VerifyImage(const BootImageHeader_t *header)
{
    /* Step 1: Anti-rollback check */
    if (header->version < minVersion) { return BOOT_VERIFY_ROLLBACK; }

    /* Step 2: Compute and verify hash */
    HSM_SHA256((const uint8_t *)header->loadAddr, header->size, computedHash);
    if (Crypto_SecureCompare(computedHash, header->hash, 32) != 0)
        return BOOT_VERIFY_HASH_FAIL;

    /* Step 3: Verify signature using HSM-stored public key */
    if (HSM_VerifySignature(HSM_KEY_SLOT_BOOT_PUB, HSM_ALG_ECDSA_P256_SHA256,
                            computedHash, 32, header->signature,
                            sizeof(header->signature)) != HSM_OK)
        return BOOT_VERIFY_SIG_FAIL;

    /* Step 4: Log measurement for boot attestation */
    SecureBoot_ExtendMeasurementLog(header->loadAddr, header->size, computedHash);
    return BOOT_VERIFY_OK;
}
```

Reference: UNECE WP.29 R155/R156; NIST SP 800-193  
See full details: `rules/boot-secure-boot-chain.md`

### 18.3 AUTOSAR Classic Boot Sequence

**Impact: HIGH (incorrect phase ordering causes BSW initialization failures)**

Classic AUTOSAR defines: `EcuM_Init()` → StartupOne (MCAL) → StartupTwo (BSW) → `BswM` → `SchM` → `Rte_Start()` → SWC init.

```c
void EcuM_Init(void)
{
    /* STARTUP ONE: MCAL */
    Mcu_Init(&Mcu_Config);
    Port_Init(&Port_Config);
    Dio_Init(&Dio_Config);

    /* STARTUP TWO: BSW — bottom up */
    Can_Init(&Can_Config);
    CanIf_Init(&CanIf_Config);
    PduR_Init(&PduR_Config);
    Com_Init(&Com_Config);
    Dcm_Init(&Dcm_Config);
    Dem_Init();

    BswM_Init(&BswM_Config);
    SchM_Init();
    Rte_Start();
    EcuM_SetState(ECUM_STATE_APP_RUN);
}
```

Reference: AUTOSAR SWS_EcuM; AUTOSAR SWS_BswM  
See full details: `rules/boot-autosar-classic-startup.md`

### 18.4 Additional Boot Topics

- `rules/boot-autosar-adaptive-startup.md` — Adaptive Platform startup with ara::exec
- `rules/boot-bootloader-reprogramming.md` — Bootloader reprogramming sequences

---

## 19. NVM Management

**Impact: HIGH**

Non-volatile memory management is critical for data persistence across power cycles. Misconfigured NvM blocks cause data loss on power failure, corrupted safety-critical calibration, or excessive startup time. This section covers AUTOSAR NvM configuration, flash wear leveling, and Fee/Ea abstraction.

### 19.1 AUTOSAR NvM Block Configuration

**Impact: HIGH (misconfigured blocks cause data loss on power failure)**

AUTOSAR NvM supports Native (single copy), Redundant (dual copy with failover), and Dataset (array) block types.

**Incorrect (safety data in native block, no CRC):**

```c
[NVM_BLOCK_SAFETY_PARAMS] = {
    .blockType    = NVM_BLOCK_NATIVE,    /* Single copy — no redundancy */
    .blockCrcType = NVM_CRC_NONE,        /* No integrity check */
    .immediateWrite = FALSE,             /* May be lost on power cut */
},
```

**Correct (redundant block with CRC for safety data):**

```c
[NVM_BLOCK_SAFETY_PARAMS] = {
    .blockType        = NVM_BLOCK_REDUNDANT,
    .blockCrcType     = NVM_CRC32,
    .ramBlockDataAddr = &SafetyParams_Ram,
    .romDefaultAddr   = &SafetyParams_Default,
    .resistantToChange = TRUE,
    .immediateWrite   = TRUE,
    .writeVerification = TRUE,
    .calcRamBlockCrc  = TRUE,
},
```

Reference: AUTOSAR SWS_NvM; ISO 26262 Part 6  
See full details: `rules/nvm-autosar-block-config.md`

### 19.2 Flash Wear Leveling Strategies

**Impact: MEDIUM (without wear leveling, flash blocks exceed endurance within vehicle lifetime)**

Flash memory has finite write endurance (10K–100K cycles). Automotive ECUs must survive 15+ years. Wear leveling distributes writes evenly.

**Incorrect (always writing to same flash location):**

```c
#define CONFIG_ADDR  0x08060000U
void Config_Save(const Config_t *cfg)
{
    Flash_EraseSector(CONFIG_ADDR);
    Flash_Write(CONFIG_ADDR, (const uint8_t *)cfg, sizeof(Config_t));
}
```

**Correct (dynamic wear leveling with block rotation):**

```c
WL_Status_t WL_Write(const Config_t *data)
{
    g_sequenceCounter++;

    if (g_currentOffset + WL_ENTRY_SIZE > WL_SECTOR_SIZE)
    {
        g_currentSector = (g_currentSector + 1U) % WL_NUM_SECTORS;
        g_currentOffset = 0U;
        Flash_EraseSector(WL_BASE_ADDR + (g_currentSector * WL_SECTOR_SIZE));
    }

    WL_Header_t hdr = {
        .sequence = g_sequenceCounter,
        .crc      = CRC16_Compute((const uint8_t *)data, sizeof(Config_t)),
        .valid    = 0xA5U,
    };
    /* Write header + data at current offset */
    g_currentOffset += WL_ENTRY_SIZE;
    return WL_OK;
}
```

Reference: JEDEC JESD218; AUTOSAR Fee  
See full details: `rules/nvm-wear-leveling.md`

### 19.3 Fee and Ea Abstraction Layers

**Impact: MEDIUM (incorrect Fee/Ea causes flash corruption or blocked NvM operations)**

Fee handles flash-specific concerns (sector erase, wear leveling, garbage collection). Ea handles external EEPROM. Both provide a uniform block-based interface to NvM.

```c
/* Cyclic task drives Fee and Fls state machines */
void Task_NvM_10ms(void)
{
    Fls_MainFunction();   /* Process pending flash operations */
    Fee_MainFunction();   /* Process Fee state machine (incl. GC) */
    NvM_MainFunction();   /* Process NvM queue */
}
```

Reference: AUTOSAR SWS_Fee; AUTOSAR SWS_Ea  
See full details: `rules/nvm-fee-ea-abstraction.md`

### 19.4 Additional NVM Topics

- `rules/nvm-baremetal-flash.md` — Bare-metal flash driver patterns

---

## 20. Power Management

**Impact: MEDIUM-HIGH**

Automotive power management governs ECU sleep/wakeup cycles, MCU low-power mode selection, and partial networking. Incorrect handling causes excessive current draw (draining the 12V battery), missed wakeup events, or corrupted state on resume.

### 20.1 EcuM Sleep and Wakeup Cycle

**Impact: HIGH (incorrect sleep/wakeup causes missed wakeup events or battery drain)**

The EcuM manages: STARTUP → RUN → SLEEP → WAKEUP → (back to RUN or SHUTDOWN). Wakeup sources must be configured before entering sleep and validated after wakeup.

**Correct (proper EcuM sleep entry):**

```c
void EcuM_GoSleep(void)
{
    BswM_EcuM_CurrentState(ECUM_STATE_PREP_SHUTDOWN);
    ComM_DeInit();
    NvM_WriteAll();
    EcuM_WaitForNvMWriteAll();

    EcuM_EnableWakeupSources(ECUM_WKSOURCE_CAN_RX |
                              ECUM_WKSOURCE_POWER_SWITCH |
                              ECUM_WKSOURCE_TIMER);

    Mcu_SetMode(MCU_MODE_SLEEP);
    /* CPU resumes on wakeup interrupt */
    EcuM_SetState(ECUM_STATE_WAKEUP_ONE);
}
```

**Wakeup validation with spurious wakeup rejection:**

```c
void EcuM_WakeupReaction(void)
{
    if (g_validatedWakeupEvents != 0U)
    {
        EcuM_SetState(ECUM_STATE_APP_RUN);
        EcuM_PerformStartup();
    }
    else
    {
        g_pendingWakeupEvents = 0U;
        EcuM_GoSleep();  /* Spurious wakeup — go back to sleep */
    }
}
```

Reference: AUTOSAR SWS_EcuM; OEM sleep current requirements (typically < 100μA)  
See full details: `rules/power-ecum-sleep-wakeup.md`

### 20.2 MCU Low-Power Mode Selection

**Impact: MEDIUM (wrong mode causes excessive current or loss of volatile state)**

Automotive MCUs provide multiple low-power modes with different trade-offs:

| Mode | Current | Wakeup Latency | RAM | Wakeup Sources |
|------|---------|----------------|-----|----------------|
| SLEEP | ~5 mA | < 1 μs | Yes | Any interrupt |
| STOP | ~10 μA | ~5 μs | Yes | EXTI, RTC |
| STANDBY | ~1 μA | ~1 ms (reset) | Partial | Wakeup pins, RTC |
| SHUTDOWN | ~100 nA | ~5 ms (reset) | No | Wakeup pins only |

**Correct (STOP mode with state preservation):**

```c
void LowPower_EnterStop(void)
{
    uint32_t savedRCC = RCC->CFGR;
    EXTI->IMR |= EXTI_IMR_IM11;
    PWR->CR1 |= PWR_CR1_LPMS_STOP1;
    SCB->SCR |= SCB_SCR_SLEEPDEEP_Msk;
    __DSB();
    __WFI();

    /* Wakeup: restore system clock */
    SCB->SCR &= ~SCB_SCR_SLEEPDEEP_Msk;
    SystemClock_Restore(savedRCC);
}
```

Reference: MCU Reference Manual; AUTOSAR SWS_Mcu  
See full details: `rules/power-low-power-modes.md`

### 20.3 Partial Networking and Selective Wakeup

**Impact: MEDIUM (incorrect PN causes unnecessary wakeups draining the battery)**

Partial networking allows ECUs to remain in low-power mode while the CAN bus is active, waking only on specific Wake-Up Frames (WUF).

**Correct (selective wakeup with WUF and PNC configuration):**

```c
const CanTrcv_PnConfig_t CanTrcv_PnConfig = {
    .wufId       = 0x510U,
    .wufIdMask   = 0x7FFU,
    .wufDlc      = 8U,
    .wufDataMask    = {0xFF, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00},
    .wufDataPattern = {0x01, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00},
};
```

Reference: AUTOSAR SWS_ComM; AUTOSAR SWS_CanTrcv  
See full details: `rules/power-partial-networking.md`

### 20.4 Additional Power Management Topics

- `rules/power-bswm-shutdown.md` — BswM-driven shutdown sequencing
- `rules/power-clock-peripheral.md` — Clock and peripheral power gating

---

## 21. Automotive Ethernet Deep-Dive

**Impact: HIGH**

Beyond the basic Ethernet patterns in Section 6, this section covers advanced automotive Ethernet topics: Time-Sensitive Networking (TSN) for deterministic real-time communication, traffic shaping with gate control lists, and Ethernet switch configuration for VLAN isolation and security.

### 21.1 TSN Time Synchronization (IEEE 802.1AS / gPTP)

**Impact: HIGH (clock errors > 1μs cause safety-critical traffic to miss gate windows)**

IEEE 802.1AS (gPTP) synchronizes all devices to a common time base. Automotive requires < 1μs accuracy.

**Correct (peer delay measurement + clock servo):**

```c
void gPTP_HandlePdelayRespFollowUp(const gPTP_PdelayRespFollowUpMsg_t *fup)
{
    int64_t round_trip = Timestamp_DiffNs(&g_portData.pdelayRespRxTime,
                                           &g_portData.pdelayReqTxTime);
    int64_t remote_processing = Timestamp_DiffNs(&g_portData.pdelayRespTxTime,
                                                  &g_portData.pdelayReqRxTime);
    g_portData.peerDelay = (round_trip - remote_processing) / 2;
}

void gPTP_HandleFollowUp(const gPTP_FollowUpMsg_t *fup)
{
    int64_t rawOffset = Timestamp_DiffNs(&g_portData.syncRxTime,
                                          &g_portData.syncTxTime);
    int64_t correctedOffset = rawOffset - g_portData.peerDelay
                              - fup->header.correctionField;
    gPTP_ClockServo(correctedOffset, g_portData.neighborRateRatio);
}
```

Reference: IEEE 802.1AS-2020; AUTOSAR SWS_EthTSyn  
See full details: `rules/eth-tsn-time-sync.md`

### 21.2 TSN Traffic Shaping (IEEE 802.1Qbv)

**Impact: HIGH (incorrect gate control lists block safety-critical frames)**

IEEE 802.1Qbv defines time-aware traffic shaping with gate control lists (GCLs) that guarantee bounded latency for safety-critical traffic by reserving exclusive time windows.

**Correct (dedicated time windows with guard bands):**

```c
const GCL_Config_t g_gclConfig = {
    .cycleTime_ns = 1000000U,  /* 1ms cycle */
    .guardBand_ns = GUARD_BAND_100M_NS,
    .entryCount = 4,
    .entries = {
        { .gateState = GATE_Q7 | GATE_Q6, .timeInterval_ns = 200000U },
        { .gateState = GATE_Q5 | GATE_Q4, .timeInterval_ns = 300000U },
        { .gateState = GATE_BE,            .timeInterval_ns = 375000U },
        { .gateState = 0x00U,              .timeInterval_ns = GUARD_BAND_100M_NS },
    },
};
```

Reference: IEEE 802.1Qbv-2015; AUTOSAR SWS_EthSwt  
See full details: `rules/eth-tsn-traffic-shaping.md`

### 21.3 Automotive Ethernet Switch Configuration

**Impact: MEDIUM (misconfigured switches cause traffic leakage between security domains)**

Covers VLAN isolation for domain separation, MAC address filtering, port mirroring for diagnostics, and STP/RSTP for loop prevention.

**Correct (VLAN configuration for domain isolation):**

```c
#define VLAN_SAFETY       10U
#define VLAN_BODY          20U
#define VLAN_INFOTAINMENT  30U
#define VLAN_DIAG          40U

const PortVlanConfig_t g_vlanConfig[] = {
    { .port = 0, .pvid = VLAN_SAFETY,
      .allowedVlans = (1U << VLAN_SAFETY) | (1U << VLAN_DIAG) },
    { .port = 3, .pvid = VLAN_INFOTAINMENT,
      .allowedVlans = (1U << VLAN_INFOTAINMENT) | (1U << VLAN_DIAG) },
    { .port = 4, .pvid = VLAN_MGMT,
      .allowedVlans = 0xFFFFU, .tagOnEgress = true },
};
```

Reference: IEEE 802.1Q; UNECE R155  
See full details: `rules/eth-switch-configuration.md`

### 21.4 Additional Ethernet Topics

- `rules/eth-tsn-stream-filtering.md` — TSN stream identification and filtering (802.1CB)
- `rules/eth-avb-streaming.md` — Audio/Video Bridging for infotainment streaming

---

## 22. Compiler & Static Analysis

**Impact: MEDIUM-HIGH**

Proper compiler configuration and static analysis tool integration catch defects at compile time before they reach safety-critical targets. This section covers GCC warning flags for automotive, Polyspace formal verification, and references to additional tool-specific rule files.

### 22.1 GCC Warning Flags for Automotive

**Impact: HIGH (catches undefined behavior and type errors at compile time)**

**Correct (comprehensive automotive warning configuration):**

```makefile
WARN_BASE = -Wall -Wextra -Werror -Wpedantic

WARN_MISRA = -Wconversion -Wsign-conversion -Wcast-align \
             -Wcast-qual -Wfloat-equal -Wshadow -Wstrict-prototypes

WARN_SAFETY = -Wundef -Wswitch-default -Wswitch-enum \
              -Wstack-usage=1024 -Wdouble-promotion -Wformat-security

CFLAGS = -std=c11 -O2 -mcpu=cortex-m4 -mthumb \
         $(WARN_BASE) $(WARN_MISRA) $(WARN_SAFETY)
```

Key flags: `-Wconversion` catches MISRA Rule 10.x narrowing, `-Wstack-usage=N` enforces per-function stack limits for RTOS tasks, `-Wdouble-promotion` catches float→double on cores without FPU.

Reference: MISRA C:2012 Directive 4.1; ISO 26262-6 Table 1  
See full details: `rules/build-gcc-warnings.md`

### 22.2 Polyspace Bug Finder and Code Prover

**Impact: HIGH (formal verification provides ISO 26262 certification evidence)**

Polyspace Code Prover produces green/orange/red classifications proving the presence or absence of runtime errors.

| Color | Meaning | ISO 26262 Action |
|-------|---------|------------------|
| **Green** | Proved safe | Certification evidence |
| **Red** | Proved defective | Must fix before release |
| **Orange** | Unresolvable | Review, justify, or fix |
| **Gray** | Dead code | Review: intentional or defect? |

**Correct (turning orange to green with defensive checks):**

```c
/* BEFORE: orange — Polyspace cannot prove divisor is non-zero */
int32_t Calculate_Average(const int32_t *data, uint32_t count)
{
    int32_t sum = 0;
    for (uint32_t i = 0U; i < count; i++) { sum += data[i]; }
    return sum / (int32_t)count;  /* ORANGE: potential division by zero */
}

/* AFTER: defensive check turns orange to green */
int32_t Calculate_Average(const int32_t *data, uint32_t count)
{
    if ((data == NULL) || (count == 0U)) { return 0; }
    int32_t sum = 0;
    for (uint32_t i = 0U; i < count; i++) { sum += data[i]; }
    return sum / (int32_t)count;  /* GREEN: count > 0 proved */
}
```

Reference: ISO 26262-6 §9 Table 7; ISO 26262-8 §11  
See full details: `rules/analysis-polyspace.md`

### 22.3 Additional Compiler & Analysis Tools

The following rule files provide tool-specific configuration and usage patterns:

- `rules/build-clang-analysis.md` — Clang-Tidy and Clang Static Analyzer
- `rules/build-compiler-flags.md` — Cross-compiler flag matrices
- `rules/build-static-analysis.md` — Static analysis integration patterns
- `rules/build-warnings-as-errors.md` — Warning promotion strategies
- `rules/build-link-time-optimization.md` — LTO for embedded targets
- `rules/build-reproducible-builds.md` — Reproducible build configuration
- `rules/build-greenhills-safety.md` — Green Hills MULTI safety compiler
- `rules/analysis-pclint-config.md` — PC-lint/FlexeLint configuration
- `rules/analysis-coverity.md` — Coverity Synopsys integration
- `rules/analysis-cppcheck.md` — Cppcheck for embedded C/C++
- `rules/analysis-parasoft.md` — Parasoft C/C++test configuration
- `rules/analysis-ldra.md` — LDRA TBvision and Testbed

---

## 23. vTESTstudio CAPL

**Impact: MEDIUM**

vTESTstudio provides a structured test framework for automotive ECU testing. Proper use of test units, fixtures, data-driven testing, and XML test modules ensures repeatable test execution, automated CI/CD integration, and standardized reporting for certification evidence.

### 23.1 Test Unit and Fixture Structure

**Impact: HIGH (proper fixtures ensure repeatable and maintainable test execution)**

**Correct (structured test unit with setup/teardown fixtures):**

```capl
/*@@testunit: TU_EngineControl */
/*@@testgroup: TG_EngineStartStop */

@setup
void Setup_EngineTests()
{
    setBusContext(CAN1);
    setSignal(IgnitionSwitch, IGNITION_OFF);
    setSignal(StartButton, 0);
    testWaitForTimeout(200);
}

@teardown
void Teardown_EngineTests()
{
    setSignal(IgnitionSwitch, IGNITION_OFF);
    testWaitForTimeout(500);
}

@testcase
void TC_EngineStart_NormalSequence()
{
    testCaseTitle("TC-ENG-001", "Normal engine start sequence");
    setSignal(IgnitionSwitch, IGNITION_ON);
    testWaitForTimeout(500);
    setSignal(BrakePedal, 1);
    setSignal(StartButton, 1);

    if (testWaitForSignalMatch(EngineRunning, ENGINE_RUNNING,
                                STARTUP_TIMEOUT_MS) == 0)
        testStepPass("Verify", "Engine started within timeout");
    else
        testStepFail("Verify", "Engine did not start");
}
```

Reference: Vector vTESTstudio User Manual  
See full details: `rules/capl-vtest-test-unit.md`

### 23.2 Data-Driven Testing

**Impact: MEDIUM (eliminates test code duplication for boundary value testing)**

Parameterized test cases bind to external CSV/Excel files. Each row becomes a test iteration.

```capl
/*@@parameter_file: test_data/speed_warning_params.csv */

@testcase
void TC_SpeedWarning_Threshold(
    char TestID[], int Speed_kph, int Expected_Warning, char Description[])
{
    testCaseTitle(TestID, "Speed warning at %d kph", Speed_kph);
    setSignal(VehicleSpeed, Speed_kph);
    testWaitForTimeout(500);

    int actual = getSignal(SpeedWarningLamp);
    if (actual == Expected_Warning)
        testStepPass("Result", "%s: correct", TestID);
    else
        testStepFail("Result", "%s: got %d expected %d", TestID, actual, Expected_Warning);
}
```

Reference: Vector vTESTstudio User Manual — Test Parameter Tables  
See full details: `rules/capl-vtest-data-driven.md`

### 23.3 XML Test Module Integration

**Impact: MEDIUM (enables command-line automation and standardized reporting)**

XML test modules map CAPL implementations to formal test specifications, enabling CI/CD automation and producing HTML/PDF reports for certification.

```bash
"C:/Vector/CANoe/Exec64/CANoe64.exe" \
    -config "configs/BrakingSystem.cfg" \
    -testmodule "test_specs/braking_system_tests.xml" \
    -autostart -autoexit \
    -reportdir "reports/" -reportformat html,pdf
```

Reference: Vector vTESTstudio User Manual — XML Test Modules  
See full details: `rules/capl-vtest-xml-module.md`

### 23.4 Additional vTESTstudio Topics

- `rules/capl-vtest-verdict-reporting.md` — Verdict reporting and test step patterns
- `rules/capl-vtest-stimulus-response.md` — Stimulus-response test patterns

---

## 24. CAPL Advanced Simulation

**Impact: HIGH**

Advanced CANoe simulation patterns for multi-bus ECU simulation, reactive rest bus simulation with NM/UDS support, and gateway routing with cross-protocol translation. These patterns enable realistic virtual integration testing.

### 24.1 Multi-Channel Bus Simulation

**Impact: HIGH (incorrect channel routing corrupts simulation fidelity)**

Automotive ECUs often bridge multiple networks. Explicit channel assignment prevents messages from routing to the wrong bus.

**Correct (explicit channel assignment with named constants):**

```capl
variables
{
    const int CH_POWERTRAIN = 1;
    const int CH_CHASSIS = 2;
    message EngineStatus msg_Engine;
    message BrakeStatus msg_Brake;
}

on start
{
    msg_Engine.MsgChannel = CH_POWERTRAIN;
    msg_Brake.MsgChannel = CH_CHASSIS;
}
```

**CAN-to-LIN gateway (correct handler types):**

```capl
on message WindowControl
{
    if (this.dir != rx) return;
    msg_LinWindowCmd.WindowTarget = this.TargetPos;
    output(msg_LinWindowCmd);
}

on linMessage LIN_WindowPos
{
    if (this.dir != rx) return;
    msg_BodyStatus.WindowPos = this.CurrentPos;
    output(msg_BodyStatus);
}
```

Reference: Vector CANoe CAPL Programming Guide  
See full details: `rules/capl-canoe-multi-channel.md`

### 24.2 Reactive Rest Bus Simulation

**Impact: HIGH (missing reactive responses cause NM timeouts and failed diagnostics)**

Reactive RBS responds dynamically to received traffic: NM keep-alive, UDS diagnostic responses, and state-dependent behavior.

**Correct (service-aware UDS responder):**

```capl
on message DiagRequest
{
    if (this.dir != rx) return;
    switch (this.byte(0))
    {
        case 0x10: HandleSessionControl(this); break;
        case 0x22: HandleReadDID(this);         break;
        case 0x2E: HandleWriteDID(this);        break;
        default:   SendNegativeResponse(this.byte(0), 0x11); break;
    }
}
```

**Correct (IL for cyclic messages, CAPL for reactive):**

```capl
on start
{
    ILEnable();
    /* IL handles: EngineStatus, TransGear (cyclic)
     * CAPL handles: DiagResponse, NM_Gateway (reactive) */
}

on sysvar sysvar::Simulation::TargetRPM
{
    ILSetSignal(EngineStatus::EngineRPM, @this);
}
```

Reference: Vector CANoe CAPL Programming Guide — Interaction Layer  
See full details: `rules/capl-canoe-rbs-reactive.md`

### 24.3 Gateway Simulation and Routing Patterns

**Impact: HIGH (incorrect routing silently corrupts data or drops messages)**

Gateway simulation requires signal-level routing with scaling conversion, cross-protocol translation (CAN↔Ethernet, CAN↔LIN), and conditional routing based on NM state.

**Correct (signal-based routing with physical value conversion):**

```capl
on message EngineStatus_CAN1
{
    if (this.dir != rx) return;
    double physRpm = this.EngineSpeed.phys;
    msg_EngineRPM_CAN2.EngineSpeedRouted.phys = physRpm;
    output(msg_EngineRPM_CAN2);
}
```

**Correct (CAN→Ethernet via SOME/IP):**

```capl
on message VehicleSpeed_CAN
{
    if (this.dir != rx) return;
    word speedRaw = (word)(this.VehicleSpeed.phys / 0.01);
    payload[0] = (speedRaw >> 8) & 0xFF;
    payload[1] = speedRaw & 0xFF;
    pktHandle = SomeIpCreateMessage(0x1234, 0x0001, 0x01);
    SomeIpSetPayload(pktHandle, payload, elCount(payload));
    SomeIpSend(pktHandle);
}
```

Reference: Vector CANoe Application Notes — Gateway Simulation  
See full details: `rules/capl-canoe-gateway-routing.md`

---

## 25. CAPL Fault Injection

**Impact: HIGH**

Systematic fault injection validates ECU robustness against bus errors, signal corruption, timing violations, and network failures. All faults are controlled via environment variables for test automation integration. Covers CAN/CAN FD, LIN, and Ethernet fault patterns.

### 25.1 CAN/CAN FD Fault Injection Patterns

**Impact: HIGH (untested error handling risks safety-critical field failures)**

Covers bus-level faults (error frames, bus-off, BRS errors), signal-level faults (stuck-at, out-of-range, alive counter freeze, CRC corruption), and timing-level faults (delay, jitter, burst, message suppression).

**Correct (stuck-at signal value injection):**

```capl
on timer cyclicTimer
{
    if (stuckAtActive)
        msg_EngineStatus.EngineSpeed.phys = stuckAtValue;
    else
        msg_EngineStatus.EngineSpeed.phys = simulatedRpm;
    output(msg_EngineStatus);
    setTimer(cyclicTimer, 10);
}
```

**Correct (CRC corruption via preTransmit):**

```capl
on preTransmit EngineStatus
{
    if (crcCorruptActive)
        this.CRC = this.CRC ^ 0xFF;
}
```

**Correct (unified fault controller with profiles):**

```capl
on envVar FaultInject_Profile
{
    int profile = (int)getValue(this);
    faultStuckAt = 0; faultDelay = 0; faultCrcCorrupt = 0;
    switch (profile)
    {
        case 1: faultStuckAt = 1; stuckValue = 0.0; break;
        case 2: faultDelay = 1; delayMs = 100; break;
        case 3: faultStuckAt = 1; faultDelay = 1; faultCrcCorrupt = 1; break;
    }
}
```

Reference: ISO 11898 CAN Error Handling; AUTOSAR E2E Protection  
See full details: `rules/capl-fault-can.md`

### 25.2 LIN Fault Injection Patterns

**Impact: MEDIUM (untested LIN faults leave slave error handling unvalidated)**

Covers checksum errors, no-response simulation, wrong data length, wrong NAD, PID parity errors, schedule table disruption, and response timing violations.

**Correct (checksum error injection):**

```capl
on linReceiveHeader SensorResponse
{
    linMsg_SensorResponse.Temperature.phys = simulatedTemp;
    if (checksumFaultActive)
        linSetChecksumError(linMsg_SensorResponse, 1);
    else
        linSetChecksumError(linMsg_SensorResponse, 0);
    output(linMsg_SensorResponse);
}
```

**Correct (no-response simulation):**

```capl
on linReceiveHeader SensorResponse
{
    if (noResponseActive)
        return;  /* No output — master detects response timeout */
    linMsg_SensorResponse.Temperature.phys = simulatedTemp;
    output(linMsg_SensorResponse);
}
```

Reference: LIN Specification 2.2A  
See full details: `rules/capl-fault-lin.md`

### 25.3 Ethernet Fault Injection Patterns

**Impact: MEDIUM (untested Ethernet faults leave network stack error handling unvalidated)**

Covers link-down simulation, probabilistic packet loss, latency injection, packet reordering, payload corruption, and VLAN stripping/mistagging.

**Correct (probabilistic packet loss):**

```capl
on ethernetPacket *
{
    if (!packetLossActive) return;
    totalCount++;
    if (random(100) < lossRatePercent)
    {
        dropCount++;
        return;  /* Consume packet */
    }
}
```

**Correct (configurable VLAN fault modes):**

```capl
switch (vlanFaultMode)
{
    case 1: EthRemoveVlanTag(this); break;         /* Strip VLAN */
    case 2: EthSetVlanId(this, faultVlanId); break; /* Mistag */
    case 3: EthSetVlanPriority(this, 0); break;     /* Priority downgrade */
}
```

Reference: IEEE 802.3; IEEE 802.1Q  
See full details: `rules/capl-fault-eth.md`

---

## 26. CAPL Signal Manipulation

**Impact: MEDIUM-HIGH**

Reusable CAPL library functions for generating signal patterns in simulation and test environments. All functions are self-contained, follow consistent naming (`Signal<Pattern>`), and provide configurable parameters and stop mechanisms.

### 26.1 Signal Manipulation Library Functions

**Available signal generators:**

| Function | Pattern | Parameters |
|----------|---------|------------|
| `SignalRamp` | Linear increase/decrease | start, end, duration, step interval |
| `SignalSine` | Sine wave | amplitude, frequency, offset, interval |
| `SignalSquare` | Square wave | amplitude, frequency, offset |
| `SignalTriangle` | Triangle wave | amplitude, frequency, offset, interval |
| `SignalNoise` | Random noise | base value, amplitude, interval |
| `SignalStep` | Immediate change | trigger delay, new value |
| `SignalSequence` | Array playback | values array, interval, loop flag |

**Correct (configurable ramp function):**

```capl
void SignalRamp(char signalName[], float startVal, float endVal,
                int durationMs, int stepIntervalMs)
{
    gRamp_currentValue = startVal;
    gRamp_endValue = endVal;
    gRamp_stepCount = durationMs / stepIntervalMs;
    gRamp_increment = (endVal - startVal) / gRamp_stepCount;
    setSignal(gRamp_signalName, gRamp_currentValue);
    setTimer(gRamp_timer, gRamp_intervalMs);
}
```

**Correct (parameterized sine generator):**

```capl
void SignalSine(char signalName[], float amplitude, float frequencyHz,
                float offset, int intervalMs)
{
    gSine_phaseStep = 2.0 * 3.14159265 * frequencyHz * (intervalMs / 1000.0);
    gSine_running = 1;
    setTimer(gSine_timer, gSine_intervalMs);
}

on timer gSine_timer
{
    float val = gSine_offset + gSine_amplitude * sin(gSine_phaseRad);
    setSignal(gSine_signalName, val);
    gSine_phaseRad += gSine_phaseStep;
    setTimer(gSine_timer, gSine_intervalMs);
}
```

**Correct (bounded noise injection):**

```capl
void SignalNoise(char signalName[], float baseValue,
                 float noiseAmplitude, int intervalMs)
{
    /* Output: baseValue + random([-noiseAmplitude, +noiseAmplitude]) */
}
```

Reference: Vector CANoe CAPL Programming Guide — Signal Access  
See full details: `rules/capl-signal-manipulation.md`

---

## 27. CAPL External Integration

**Impact: HIGH**

Extending CANoe/vTESTstudio with native C/C++ DLLs, COM/Python automation, and CI/CD pipeline integration for automated test execution, result parsing, and quality gate enforcement.

### 27.1 CAPL DLL Integration

**Impact: HIGH (incorrect DLL integration causes measurement crashes)**

CAPL DLLs extend CANoe with native C/C++ code. Correct integration requires matching the CAPL DLL API contract, safe data exchange, and non-blocking execution.

**Correct (proper CAPL DLL export with init/release):**

```c
#include "cdll.h"

void CAPLEXPORT far CAPLPASCAL caplDllInit(void)
{
    /* Initialize resources at measurement start */
}

void CAPLEXPORT far CAPLPASCAL caplDllRelease(void)
{
    /* Release resources at measurement stop */
}

long CAPLEXPORT far CAPLPASCAL dllAdd(long a, long b)
{
    return a + b;
}

CAPL_DLL_EXPORT_TABLE_BEGIN
    CAPL_DLL_EXPORT(dllAdd, "long", "long,long",
                    "Add two integers", 'L', 2, "LL", "a", "b")
CAPL_DLL_EXPORT_TABLE_END
```

**Thread safety — non-blocking DLL with background worker:**

```c
static CRITICAL_SECTION gDataLock;
static double gLatestValue = 0.0;

/* Background thread performs slow I/O independently */
static DWORD WINAPI workerThread(LPVOID param)
{
    while (gWorkerRunning)
    {
        double val = readHardwareSensor();
        EnterCriticalSection(&gDataLock);
        gLatestValue = val;
        LeaveCriticalSection(&gDataLock);
        Sleep(100);
    }
    return 0;
}

/* Called from CAPL — returns immediately */
double CAPLEXPORT far CAPLPASCAL dllGetSensorValue(void)
{
    double val;
    EnterCriticalSection(&gDataLock);
    val = gLatestValue;
    LeaveCriticalSection(&gDataLock);
    return val;
}
```

Reference: Vector CANoe Help — CAPL DLL Programming  
See full details: `rules/capl-ext-dll-integration.md`

### 27.2 CI/CD Integration for CANoe Test Automation

**Impact: HIGH (automated testing catches regressions early)**

Run CANoe tests headlessly in CI pipelines with proper license management, artifact collection, and result parsing.

**Jenkins pipeline:**

```yaml
pipeline {
    agent { label 'canoe-runner' }
    stages {
        stage('Run CANoe Tests') {
            steps {
                bat '''
                    "C:\\Program Files\\Vector CANoe 17\\Exec64\\CANoe64.exe" ^
                        -d "configs\\integration_test.cfg" ^
                        -r "reports\\canoe_results.xml"
                '''
            }
        }
        stage('Publish Results') {
            steps {
                junit 'reports/*.xml'
                archiveArtifacts artifacts: 'reports/**', fingerprint: true
            }
        }
    }
    post {
        always {
            bat 'taskkill /F /IM CANoe64.exe /T 2>nul || exit /b 0'
        }
    }
}
```

**Artifact collection checklist (every run):**
- Test report XML
- CANoe Write window log
- Bus trace recordings (`.blf`)
- Screenshots on failure

Reference: Vector CANoe Help — Command Line Interface  
See full details: `rules/capl-ext-ci-cd.md`

### 27.3 Additional External Integration Topics

- `rules/capl-ext-com-python.md` — Python COM automation for CANoe
- `rules/capl-ext-com-csharp.md` — C# COM automation for CANoe

---

## References

1. MISRA C:2012 — Guidelines for the use of the C language in critical systems
2. MISRA C++:2023 — Guidelines for the use of C++ in critical systems
3. AUTOSAR C++14 Coding Guidelines (Release 19-03)
4. ISO 26262:2018 — Road vehicles — Functional safety
5. AUTOSAR Classic Platform Specification
6. AUTOSAR Adaptive Platform Specification
7. Vector CANoe/CANalyzer CAPL Programming Guide
8. ARM Cortex-M Programming Guide
9. CERT C Coding Standard (SEI CERT)
10. IEC 61508 — Functional safety of E/E/PE safety-related systems
11. ISO/SAE 21434:2021 — Road vehicles — Cybersecurity engineering
12. ISO 13400-2:2019 — DoIP transport protocol and network layer services
13. ISO 14229-1:2020 — UDS (Unified Diagnostic Services)
14. ISO 11898-1:2015 — CAN / CAN FD data link layer
15. IEEE 802.1Q — VLAN and QoS
16. AUTOSAR PRS_SOMEIP — SOME/IP Protocol Specification
17. AUTOSAR PRS_SOMEIPSD — SOME/IP Service Discovery Specification
18. ASAM XCP — Universal Measurement and Calibration Protocol
19. ASAM MCD-2D (ODX) — Open Diagnostic Data Exchange
20. ASAM MCD-2MC (A2L/ASAP2) — Measurement and Calibration Data Exchange
21. IEEE 802.1AS-2020 — Timing and Synchronization for Time-Sensitive Applications (gPTP)
22. IEEE 802.1Qbv-2015 — Enhancements for Scheduled Traffic (Time-Aware Shaping)
23. IEEE 802.1CB — Frame Replication and Elimination for Reliability
24. UNECE WP.29 R155/R156 — Cybersecurity and Software Update Regulations
25. NIST SP 800-193 — Platform Firmware Resiliency Guidelines
26. JEDEC JESD218 — Solid-State Drive (SSD) Endurance Workloads
27. LIN Specification 2.2A — Local Interconnect Network Protocol
28. MISRA Compliance:2020 — Achieving Compliance with MISRA Coding Guidelines
29. Vector vTESTstudio User Manual — Test Architecture and Automation
30. Vector CANoe CAPL Programming Guide — Simulation and Testing
