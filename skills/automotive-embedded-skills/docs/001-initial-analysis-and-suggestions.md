# Initial Analysis & Skill Design Suggestions

**Date:** 2026-02-26  
**Session:** Architecture Design — Automotive Embedded C/C++/CAPL AI Skill  
**Status:** Draft v1 — Open for Discussion

---

## 1. Analysis of Reference Skill (vercel-react-best-practices)

### 1.1 Structure Overview

The reference skill follows a well-organized pattern:

| Component | Purpose |
|-----------|---------|
| `SKILL.md` | Skill metadata (name, description, triggers), quick reference, and category index |
| `AGENTS.md` | Full compiled document with all rules expanded — the main artifact consumed by AI agents |
| `rules/` | Individual rule files (one per rule) with frontmatter metadata |
| `README.md` | Human-readable project overview with build/contribution instructions |
| `src/` | Build scripts to compile individual rules into AGENTS.md |

### 1.2 Rule File Structure (per rule)

Each rule file has:
- **Frontmatter:** title, impact level, impact description, tags
- **Body:** Brief explanation → Incorrect example → Correct example → Additional context → Reference links

### 1.3 Category Design

The reference uses 8 categories with prefixed naming (`async-`, `bundle-`, `server-`, etc.), sorted by impact priority from CRITICAL to LOW. Each rule has:
- Clear impact level (CRITICAL, HIGH, MEDIUM, LOW)
- Impact description (quantified where possible)
- Before/after code comparison

### 1.4 What Works Well (to adopt)

1. **Priority-ordered categories** — Helps AI agents focus on highest-impact improvements first
2. **Prefix-based naming** — Makes rules discoverable and categorizable
3. **Incorrect → Correct pattern** — The most effective format for AI code generation
4. **Impact quantification** — "2-10× improvement" helps prioritize refactoring
5. **Standard references** — Links to React docs, Vercel blog — we'll link to MISRA, AUTOSAR, ISO 26262
6. **Tags for cross-cutting concerns** — Enables multi-dimensional rule lookup

### 1.5 What Needs Adaptation for Embedded

| React Skill Aspect | Embedded Adaptation |
|--------------------|--------------------|
| Performance as primary concern | **Safety** as primary concern (performance is secondary) |
| JavaScript/TypeScript examples | **C, C++, and CAPL** examples |
| Web-specific patterns (SSR, hydration) | **MCU-specific patterns** (ISR, DMA, watchdog) |
| Bundle size optimization | **Flash/RAM optimization** |
| Re-render optimization | **Real-time deadline compliance** |
| Client-server architecture | **AUTOSAR layered architecture** |
| npm/pnpm build system | **Make/CMake + cross-compiler** toolchain |

---

## 2. Proposed Skill Design for C/C++/CAPL

### 2.1 Scope and Target Audience

**Domain:** Automotive embedded software development  
**Languages:** C (ISO C99/C11), C++ (C++14 per AUTOSAR), CAPL (Vector CANoe/CANalyzer)  
**Target platforms:** Automotive ECUs (Infineon AURIX, NXP S32, Renesas RH850, ARM Cortex-M/R)  
**RTOS:** AUTOSAR OS, FreeRTOS, OSEK/VDX compliant systems  
**Standards:** MISRA C:2012, MISRA C++:2023, AUTOSAR C++14, ISO 26262, CERT C

### 2.2 Category Design (12 Categories)

Categories ordered by **safety impact** (not just performance):

| # | Category | Priority | Prefix | Rules | Rationale |
|---|----------|----------|--------|-------|-----------|
| 1 | Memory Safety & Management | CRITICAL | `memory-` | 9 | #1 cause of embedded failures |
| 2 | MISRA C/C++ Compliance | CRITICAL | `misra-` | 9+ (→160) | Industry-mandated standard (expanding to full coverage per Decision #1) |
| 3 | AUTOSAR C++14 Guidelines | CRITICAL | `autosar-` | 8+ (→16) | Both Classic & Adaptive (per Decision #2) |
| 4 | Safety & Functional Safety | HIGH | `safety-` | 8 | ISO 26262 compliance |
| 5 | Real-Time & Timing | HIGH | `realtime-` | 7 | Deadline compliance |
| 6 | Communication Protocols | HIGH | `comm-` | 16 | CAN/CAN FD/LIN/TCP/UDP/DoIP/ARP/ICMP/SOME-IP/UDS |
| 7 | Concurrency & RTOS | MEDIUM-HIGH | `rtos-` | 7 | Multi-task safety |
| 8 | CAPL Scripting (CANoe) | MEDIUM-HIGH | `capl-canoe-` | 8 | Test automation/simulation in CANoe |
| 9 | CAPL Scripting (vTESTstudio) | MEDIUM-HIGH | `capl-vtest-` | 8 | Test automation in vTESTstudio (per Decision #4) |
| 10 | Code Architecture | MEDIUM | `arch-` | 6 | Maintainability |
| 11 | Performance Optimization | MEDIUM | `perf-` | 7 | Resource-constrained MCUs |
| 12 | Build & Static Analysis | MEDIUM | `build-` | 5+ | Compile-time safety + GCC/Clang specifics (per Decision #3) |
| 13 | Testing & Verification | MEDIUM | `test-` | 6 | ISO 26262 verification |
| 14 | **Security (ISO 21434)** | **HIGH** | `security-` | **8-10** | **Cybersecurity (per Decision #7)** |
| 15 | **Tool Integration** | **MEDIUM** | `integration-` | **6-8** | **A2L/ASAP2, FIBEX, ODX (per Decision #6)** |
| | **Total** | | | **~150-200** | |

### 2.3 Key Design Decisions

#### Why Safety > Performance as Top Priority?
In React/web development, performance is the primary concern — a slow page is a bad page. In automotive embedded, **a memory corruption or timing violation can endanger human lives**. Our priority ordering reflects this:

1. **Memory Safety** — Buffer overflows in ECUs don't crash a browser; they corrupt control signals
2. **Standards Compliance** — MISRA/AUTOSAR compliance is legally required, not optional
3. **Functional Safety** — ISO 26262 patterns are audited by external assessors
4. **Real-Time** — Missing a 10ms deadline in a brake controller has immediate consequences

#### Why Include CAPL?
CAPL is unique to automotive development. It's used for:
- ECU simulation during integration testing
- CAN/LIN bus analysis and monitoring
- Automated regression test execution in CANoe
- Diagnostic protocol validation (UDS/OBD-II)

No existing AI skill covers CAPL best practices, making this a high-value addition.

#### C vs C++ Separation
We don't separate C and C++ into different skills because:
- Many automotive projects mix C and C++ in the same codebase
- MCAL/BSW layers are typically C; application SWCs may be C++
- Rules like memory management, error handling, and safety patterns apply to both
- Where a rule is language-specific, we clearly indicate (C-only, C++-only)

---

## 3. Detailed Rule Inventory

### 3.1 Memory Safety & Management (9 rules)

| Rule | Impact | Description |
|------|--------|-------------|
| `memory-stack-over-heap` | CRITICAL | Prefer stack allocation in embedded context |
| `memory-static-allocation` | CRITICAL | Use static allocation for deterministic memory |
| `memory-buffer-bounds` | CRITICAL | Validate buffer boundaries before every access |
| `memory-pool-pattern` | HIGH | Memory pools for dynamic-like allocation |
| `memory-no-malloc-in-rt` | CRITICAL | No malloc/free in real-time paths |
| `memory-raii-cpp` | HIGH | RAII for C++ resource management |
| `memory-volatile-correctness` | CRITICAL | Correct volatile usage for HW registers |
| `memory-alignment` | MEDIUM | Proper struct alignment for target arch |
| `memory-zero-init` | HIGH | Always initialize variables |

### 3.2 MISRA C/C++ Compliance (9 rules)

| Rule | Impact | MISRA Ref |
|------|--------|-----------|
| `misra-no-implicit-conversions` | HIGH | Rules 10.1–10.8 |
| `misra-single-exit-point` | MEDIUM | Rule 15.5 |
| `misra-no-dynamic-memory` | CRITICAL | Rule 21.3 |
| `misra-no-recursion` | CRITICAL | Rule 17.2 |
| `misra-switch-default` | MEDIUM | Rule 16.4 |
| `misra-no-goto` | LOW-MEDIUM | Rule 15.1 |
| `misra-boolean-expressions` | LOW | Rule 14.4 |
| `misra-pointer-arithmetic` | MEDIUM | Rules 18.1–18.4 |
| `misra-side-effects` | MEDIUM | Rule 13.5 |

### 3.3 AUTOSAR C++14 Guidelines (8 rules)

| Rule | Impact | AUTOSAR Ref |
|------|--------|-------------|
| `autosar-smart-pointers` | HIGH | A18-1-1 |
| `autosar-no-exceptions-rt` | HIGH | A15-0-1 |
| `autosar-const-correctness` | MEDIUM | A7-1-1 |
| `autosar-override-final` | MEDIUM | A10-3-1 |
| `autosar-enum-class` | MEDIUM | A7-2-2 |
| `autosar-no-unions` | MEDIUM | A9-5-1 |
| `autosar-braces-init` | LOW-MEDIUM | A8-5-2 |
| `autosar-nodiscard` | MEDIUM | A0-1-2 |

### 3.4 Safety & Functional Safety (8 rules)

| Rule | Impact | ISO 26262 Ref |
|------|--------|---------------|
| `safety-defensive-programming` | HIGH | Part 6, Table 1 |
| `safety-error-detection` | HIGH | Part 6, Table 3 |
| `safety-redundant-checks` | CRITICAL | Part 6, Table 5 |
| `safety-watchdog-pattern` | HIGH | Part 6, Table 4 |
| `safety-state-machine-integrity` | HIGH | Part 6, §8.4.3 |
| `safety-crc-validation` | HIGH | Part 6, Table 6 |
| `safety-safe-state` | CRITICAL | Part 3, §7 |
| `safety-asil-decomposition` | HIGH | Part 9 |

### 3.5 Real-Time & Timing (7 rules)

| Rule | Impact |
|------|--------|
| `realtime-deterministic-execution` | CRITICAL |
| `realtime-wcet-awareness` | HIGH |
| `realtime-no-blocking-isr` | CRITICAL |
| `realtime-priority-inversion` | HIGH |
| `realtime-cyclic-scheduling` | MEDIUM |
| `realtime-interrupt-latency` | HIGH |
| `realtime-deadline-monitoring` | HIGH |

### 3.6 Communication Protocols (16 rules)

#### 3.6.1 CAN/LIN Bus Protocols

| Rule | Impact | Protocol |
|------|--------|----------|
| `comm-can-message-layout` | MEDIUM | CAN / CAN FD |
| `comm-can-error-handling` | HIGH | CAN |
| `comm-can-fd-handling` | MEDIUM | CAN FD |
| `comm-lin-schedule-table` | MEDIUM | LIN |
| `comm-signal-timeout` | HIGH | All |
| `comm-network-management` | MEDIUM | AUTOSAR NM |

#### 3.6.2 Automotive Ethernet / IP Stack

| Rule | Impact | Protocol |
|------|--------|----------|
| `comm-tcp-socket-lifecycle` | HIGH | TCP |
| `comm-udp-datagram-handling` | HIGH | UDP |
| `comm-doip-implementation` | HIGH | DoIP (ISO 13400) |
| `comm-arp-table-management` | MEDIUM | ARP |
| `comm-icmp-handling` | MEDIUM | ICMP |
| `comm-vlan-qos-priority` | MEDIUM | VLAN / QoS (IEEE 802.1Q) |
| `comm-dhcp-autoip` | MEDIUM | DHCP / AutoIP |
| `comm-someip-serialization` | MEDIUM | SOME/IP |
| `comm-someip-sd` | HIGH | SOME/IP-SD |

#### 3.6.3 Diagnostics & Routing

| Rule | Impact | Protocol |
|------|--------|----------|
| `comm-uds-service-handler` | HIGH | UDS (ISO 14229) |
| `comm-gateway-routing` | MEDIUM | Multi-bus |

### 3.7 Concurrency & RTOS (7 rules)

| Rule | Impact |
|------|--------|
| `rtos-task-design` | MEDIUM |
| `rtos-critical-section` | HIGH |
| `rtos-mutex-pattern` | HIGH |
| `rtos-message-queue` | MEDIUM |
| `rtos-no-priority-inversion` | HIGH |
| `rtos-isr-to-task` | HIGH |
| `rtos-stack-sizing` | HIGH |

### 3.8 CAPL Scripting (8 rules)

| Rule | Impact |
|------|--------|
| `capl-message-handler` | MEDIUM |
| `capl-timer-pattern` | MEDIUM |
| `capl-test-structure` | HIGH |
| `capl-signal-access` | MEDIUM |
| `capl-error-frame-handling` | MEDIUM |
| `capl-environment-variables` | LOW |
| `capl-diagnostic-testing` | HIGH |
| `capl-node-simulation` | MEDIUM |

### 3.9 Code Architecture (6 rules)

| Rule | Impact |
|------|--------|
| `arch-hal-abstraction` | HIGH |
| `arch-module-interface` | MEDIUM |
| `arch-state-machine` | MEDIUM |
| `arch-callback-pattern` | MEDIUM |
| `arch-config-separation` | MEDIUM |
| `arch-layered-architecture` | HIGH |

### 3.10 Performance Optimization (7 rules)

| Rule | Impact |
|------|--------|
| `perf-loop-optimization` | MEDIUM |
| `perf-lookup-table` | HIGH |
| `perf-bitwise-operations` | MEDIUM |
| `perf-cache-friendly` | MEDIUM |
| `perf-inline-critical` | LOW-MEDIUM |
| `perf-fixed-point` | HIGH |
| `perf-dma-usage` | MEDIUM |

### 3.11 Build & Static Analysis (5 rules)

| Rule | Impact |
|------|--------|
| `build-warnings-as-errors` | HIGH |
| `build-static-analysis` | HIGH |
| `build-compiler-flags` | MEDIUM |
| `build-link-time-optimization` | LOW-MEDIUM |
| `build-reproducible-builds` | MEDIUM |

### 3.12 Testing & Verification (6 rules)

| Rule | Impact |
|------|--------|
| `test-unit-test-pattern` | HIGH |
| `test-mock-hardware` | HIGH |
| `test-boundary-values` | MEDIUM |
| `test-coverage-targets` | HIGH |
| `test-integration-testing` | MEDIUM |
| `test-hil-sil-pattern` | HIGH |

---

## 4. Suggested Next Steps

### Phase 1: Foundation (Current)
- [x] Create folder structure (Clangs_skills/)
- [x] Create SKILL.md with category index
- [x] Create AGENTS.md with initial compiled rules
- [x] Create this analysis document

### Phase 2: Individual Rule Files (Completed 2026-02-26)
- [x] Create individual `.md` files in `rules/` for each rule (111 files + 1 template = 112 total)
- [x] Each file follows the frontmatter + incorrect/correct example pattern
- [x] Code examples extracted from AGENTS.md with proper language tags (c, cpp, capl)
- [x] Standard references included (MISRA rule numbers, AUTOSAR rule IDs, ISO 26262 clauses, ISO 21434, ISO 13400)

### Phase 3: Domain-Specific Enhancements (Completed 2026-02-26)
- [x] CAPL file rename: `capl-*` → `capl-canoe-*` (8 files renamed)
- [x] MISRA expansion: 12 grouped topic files covering all ~160 rules with deviation patterns
- [x] AUTOSAR Classic BSW modules: 8 rule files (EcuM, BswM, COM, PduR, Dcm/Dem, NvM, OS, CanIf/CanTp)
- [x] AUTOSAR Adaptive ara:: APIs: 7 rule files (ara::com, core, exec, diag, log, phm, per)
- [x] ECU boot sequence: 5 rule files (bare-metal, Classic, Adaptive, bootloader, secure boot)
- [x] NVM management: 4 rule files (AUTOSAR NvM, Fee/Ea, bare-metal flash, wear leveling)
- [x] Power management: 5 rule files (EcuM sleep/wake, partial networking, BswM shutdown, clock gating, low-power modes)
- [x] Automotive Ethernet deep-dive: 5 rule files (TSN time-sync, traffic shaping, stream filtering, switch config, AVB)
- [x] Compiler-specific: 3 rule files (GCC warnings, Clang analysis, GreenHills safety)
- [x] Static analysis tools: 6 rule files (PC-lint, Polyspace, Coverity, cppcheck, Parasoft, LDRA)
- [x] vTESTstudio CAPL: 5 rule files (test units, data-driven, XML modules, verdicts, stimulus/response)
- [x] README.md created
- **Total Phase 3: 61 new files → 172 rule files total**

### Phase 4: CAPL Deep Dive (Completed 2026-02-26)
- [x] Multi-channel simulation: CAN+CAN, CAN+LIN, CAN+ETH, full vehicle (1 file)
- [x] Rest Bus Simulation: cyclic RBS + reactive RBS with Interaction Layer (2 files)
- [x] Gateway simulation: signal/PDU/cross-protocol routing with manipulation (1 file)
- [x] Fault injection: CAN, LIN, Ethernet — bus/signal/timing/network layers (3 files)
- [x] Signal manipulation: reusable CAPL library (ramp, sine, noise, step, sequence) (1 shared file)
- [x] DLL integration: full CAPL DLL API with thread safety and 32/64-bit (1 file)
- [x] External scripting: Python COM, C# COM, CI/CD (Jenkins/GitLab) (3 files)
- [x] SKILL.md updated with all Phase 3+4 rules (v2.0.0, 23 categories)
- [x] AGENTS.md updated with Phase 3+4 content (27 sections, ~4700 lines)
- **Total Phase 4: 12 new rule files → 184 rule files total**

### Phase 5: Validation & Tooling
- [x] Create a `_template.md` for rule contribution (done in Phase 2)
- [x] README.md created (done in Phase 3)
- [ ] ~~Build a compiler script~~ — Decision G1: manual maintenance
- [ ] ~~LLM test cases~~ — Decision G2: skipped
- [ ] ~~Sample project~~ — Decision G3: skipped

---

## 5. Open Questions — Decisions (Answered 2026-02-26)

| # | Question | Decision | Impact on Skill |
|---|----------|----------|-----------------|
| 1 | **Scope of MISRA rules:** Cover all ~160 or focus on ~30? | **All ~160 rules** | Expand `misra-` category significantly. Phase 2 will need sub-grouping (e.g., `misra-type-`, `misra-ctrl-`, `misra-ptr-`) |
| 2 | **AUTOSAR version:** Classic, Adaptive, or both? | **Both, separated rules** | Split `autosar-` into `autosar-classic-` and `autosar-adaptive-` prefixes. Add category or sub-categories |
| 3 | **Compiler-specific rules:** GCC/Clang-specific or pure standard? | **Yes, include GCC/Clang** | Add `build-gcc-*`, `build-clang-*` rules with compiler-specific attributes, pragmas, builtins |
| 4 | **CAPL scope:** CANoe vs vTESTstudio? | **Different rule sets** | Split into `capl-canoe-*` and `capl-vtest-*` prefixes, or create separate sub-categories |
| 5 | **Hardware platform:** Specific MCUs or generic? | **Generic** | Keep rules MCU-agnostic. Reference specific MCU behavior only in "Notes" sections as examples |
| 6 | **Integration tools:** A2L/ASAP2, FIBEX/ODX? | **Yes** | Add new category or sub-category: `integration-a2l-*`, `integration-odx-*`, `integration-fibex-*` |
| 7 | **Security (ISO 21434):** Separate or woven in? | **Separate category** | Add new category 13: `security-` with ISO 21434 cybersecurity rules |

### Decisions Impact Summary

These decisions expand the skill significantly:

- **MISRA expansion:** ~9 rules → ~160 rules (grouped by MISRA directive/rule categories)
- **AUTOSAR split:** 8 rules → ~16+ rules (Classic + Adaptive separation)
- **New category: Security (ISO 21434):** ~8-10 new rules
- **CAPL split:** 8 rules → ~16 rules (CANoe + vTESTstudio)
- **New category: Tool Integration:** A2L/ASAP2, FIBEX, ODX patterns
- **Compiler-specific rules:** GCC/Clang pragmas, attributes, sanitizers

**Revised total rule estimate: ~150-200 rules across 14+ categories**

---

## 6. Open Questions for Phase 3+ — Decisions (Answered 2026-02-26)

### Decisions Summary Table

| ID | Question | Decision | Phase |
|----|----------|----------|-------|
| A1 | MISRA rule granularity | **Group by topic** (~15-20 files, not 1-per-rule) | 3 |
| A2 | MISRA prioritization | **Most commonly violated first** (~30-40 rules = 90% of findings) | 3 |
| A3 | MISRA C vs C++ | **Combined where they overlap** | 3 |
| A4 | MISRA deviations | **Yes, include accepted deviation patterns** | 3 |
| B1 | Classic AUTOSAR modules | **All** (COM, PDU Router, CanIf, CanTp, LinIf, SoAd, Sd, EcuM, BswM, SchM, Dcm, Dem, FiM, NvM, Fee, Ea, MemIf, OS) | 3 |
| B2 | Adaptive AUTOSAR APIs | **All** (ara::com, ara::core, ara::diag, ara::exec, ara::log, ara::phm, ara::per) | 3 |
| B3 | AUTOSAR release version | **Newest** (Classic R22-11, Adaptive R24-10) | 3 |
| C1 | Compilers to cover | **GCC, Clang, GreenHills first** | 3 |
| C2 | Compiler-specific content scope | **Skip — deferred to future phases** | Later |
| C3 | Static analysis tools | **All** (PC-lint, Polyspace, Coverity, Parasoft, cppcheck, Clang-Tidy, LDRA) | 3 |
| D1 | CAPL rename | **Rename `capl-*` → `capl-canoe-*`, add `capl-vtest-*`** | 3 |
| D2 | vTESTstudio scope | **All** (test units, data-driven, XML, verdicts, timing, DOORS/Polarion) | 3 |
| D3 | Shared CAPL patterns | **Keep shared under `capl-*`, platform-specific separated** | 3 |
| E1 | ECU boot sequence | **All** (bare-metal, Classic AUTOSAR, Adaptive, bootloader, secure boot) | 3 |
| E2 | NVM management | **All** (AUTOSAR NvM, Fee/Ea, bare-metal, wear leveling) | 3 |
| E3 | Power management | **Separated for each** (EcuM, partial networking, BswM, clock gating, low-power) | 3 |
| E4 | Ethernet deep-dive | **Separated for each** (TSN, switch config, AVB, gPTP) | 3 |
| E5 | ECU type tagging | **Yes**, tag rules by applicable ECU type | 3 |
| F1 | Advanced CAPL simulation | **All** (multi-channel, RBS, gateway, fault injection, signal manipulation) | 4 |
| F2 | CAPL external integration | **All** (DLL, COM automation, Python/MATLAB) | 4 |
| G1 | Build script | **Manual maintenance** (no build script) | — |
| G2 | LLM test cases | **Skip** | — |
| G3 | Sample project | **Skip** | — |
| H1 | Multi-language examples | **Show both C and C++ variants** | 3+ |
| H2 | Deployment format | **Both** (.agents/skills + .cursor/rules), create README.md | 3 |
| H3 | Versioning | **Semantic versioning** in SKILL.md | 3 |

---

## 8. Open Questions for Phase 4 — Decisions (Answered 2026-02-26)

### Phase 4 Decisions Summary Table

| ID | Question | Decision | Impact |
|----|----------|----------|--------|
| I1 | Multi-channel bus combos | **All** (CAN+CAN, CAN+LIN, CAN+ETH, full vehicle) | Multi-channel simulation file |
| I2 | RBS depth | **All** (simple, reactive, IL, per-protocol) | 3 RBS files |
| I3 | Vector hardware | **Hardware-agnostic** | No HW-specific references |
| J1 | Gateway routing | **All** (signal, PDU, cross-protocol) | Gateway simulation files |
| J2 | Gateway manipulation | **Yes** (scaling, byte order, mux, conditional) | Included in gateway files |
| K1 | Fault injection granularity | **Option C: Grouped by protocol** (`capl-fault-can`, `-lin`, `-eth`) | 3 fault files |
| K2 | Fault types | **All** (bus, signal, timing, network level) | All layers in each protocol file |
| L1 | Signal manipulation | **All** + reusable CAPL library functions | Shared `capl-*` file |
| M1 | DLL build environment | **Build-tool-agnostic** (API contract only) | No VS/CMake specifics |
| M2 | DLL topics | **All** (API, data exchange, threads, 32/64-bit, errors) | Full DLL file |
| N1 | COM languages | **Most commonly used** (Python, C#) | Lightweight reference |
| N2 | Automation use cases | **All** (test exec, monitoring, reports, CI/CD) | In external integration files |
| O1 | File naming | **Option C:** simulation → `capl-canoe-*`, external → `capl-ext-*` | Mixed prefix |
| O2 | Shared patterns | **Shared `capl-*`** for cross-platform patterns | Signal manipulation as `capl-*` |
| P1 | Scope boundary | **Focus on CAPL**, external integration lightweight | DLL full depth, COM/Python brief |
| P2 | AGENTS.md sync | **Update AGENTS.md** with all Phase 3+4 content | Major AGENTS.md update |
| P3 | SKILL.md sync | **Update after each phase** | SKILL.md update after Phase 4 |

---

## 9. Notes

- This analysis document serves as a living discussion record. All decisions and rationale will be captured here as we iterate.
- Phase 1 completed 2026-02-26: Folder structure, SKILL.md, AGENTS.md, this document.
- Phase 2 completed 2026-02-26: 111 individual rule files + 1 template = 112 files in `rules/`.
- Phase 3 completed 2026-02-26: 61 new files (MISRA expansion, AUTOSAR Classic/Adaptive, boot/NVM/power/Ethernet, compilers, analysis tools, vTESTstudio) → 172 total rule files + README.md.
- Phase 4 completed 2026-02-26: 12 new files (CAPL simulation, fault injection, signal manipulation, DLL, COM, CI/CD) → 184 total rule files. SKILL.md v2.0.0 (23 categories). AGENTS.md expanded to 27 sections (~4700 lines).
- Rule impact levels may be adjusted as we validate against real projects.
