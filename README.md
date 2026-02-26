# Agent Skills — Automotive Embedded

A collection of AI agent skills for automotive embedded software development.

## Available Skills

### automotive-embedded-skills

Comprehensive C/C++/CAPL coding guidelines for automotive embedded systems. Contains 180+ rules across 23 categories aligned with MISRA C:2012, AUTOSAR C++14, ISO 26262, and ISO 21434.

Use when:
- Writing or reviewing embedded C/C++ code for automotive ECUs
- Implementing CAN/LIN/Ethernet communication stacks
- Writing CAPL scripts for CANoe/vTESTstudio simulation and testing
- Ensuring MISRA, AUTOSAR, or ISO 26262 compliance
- Working with diagnostics (UDS, DoIP), cybersecurity, or safety-critical code

Categories covered:
- Memory Safety & Management (Critical)
- MISRA C/C++ Compliance (Critical)
- AUTOSAR Classic & Adaptive (Critical)
- Safety / ISO 26262 (High)
- Real-Time & Timing (High)
- Communication Protocols — CAN/LIN/Ethernet/UDS (High)
- Concurrency & RTOS Patterns (Medium-High)
- CAPL Scripting — CANoe & vTESTstudio (Medium-High)
- Fault Injection — CAN/LIN/Ethernet (High)
- Security / ISO 21434 (High)
- ECU Boot, NVM, Power Management (High)
- Automotive Ethernet — TSN/AVB (High)
- Compiler & Static Analysis Tools (Medium)
- Testing & Verification (Medium)
- Tool Integration — A2L/ODX/XCP/ARXML (Medium)

## Installation

```shell
npx skills add washabii14/agent-skills --skill automotive-embedded-skills
```

Or install all skills:

```shell
npx skills add washabii14/agent-skills
```

## Usage

Skills are automatically available once installed. Your AI agent will use them when relevant automotive embedded tasks are detected.

Examples:
```
Review this C code for MISRA compliance
```
```
Write a CAN message handler following AUTOSAR COM patterns
```
```
Create a CAPL fault injection script for CAN bus-off testing
```
```
Implement UDS ReadDataByIdentifier with proper NRC handling
```

## Skill Structure

Each skill contains:
- `SKILL.md` — Instructions and rule index for the agent
- `AGENTS.md` — Full compiled rules document (4700+ lines)
- `rules/` — Individual rule files (184 files)
- `docs/` — Analysis and design documents

## License

MIT
