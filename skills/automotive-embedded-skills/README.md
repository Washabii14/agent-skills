# Automotive Embedded C/C++ Agent Skill

**Version:** 1.0.0 | **License:** MIT

A comprehensive agent skill for AI-assisted development of safety-critical automotive embedded software. Provides coding rules, architectural patterns, and tool configurations aligned with MISRA C:2012, AUTOSAR C++14, ISO 26262, and ISO 21434.

## Folder Structure

```
Clangs_skills/
├── SKILL.md              # Skill entry point — read by AI agents on activation
├── AGENTS.md             # Detailed agent instructions (coding rules, patterns, workflow)
├── README.md             # This file — project overview and contributing guide
├── rules/                # Individual rule files (one rule per file)
│   ├── _template.md      # Template for creating new rules
│   ├── arch-*.md         # Architecture and design patterns
│   ├── misra-*.md        # MISRA C:2012 specific rules
│   ├── autosar-*.md      # AUTOSAR C++14 guidelines
│   ├── safety-*.md       # Functional safety (ISO 26262)
│   ├── security-*.md     # Cybersecurity (ISO 21434)
│   ├── rtos-*.md         # RTOS patterns and task management
│   ├── realtime-*.md     # Real-time and determinism rules
│   ├── memory-*.md       # Memory management for embedded
│   ├── perf-*.md         # Performance optimization
│   ├── comm-*.md         # Communication protocols (CAN, LIN, Ethernet)
│   ├── test-*.md         # Testing patterns (unit, integration, HIL/SIL)
│   ├── build-*.md        # Build system and compiler configuration
│   ├── analysis-*.md     # Static analysis tool configuration
│   ├── capl-canoe-*.md   # CANoe CAPL scripting rules
│   ├── capl-vtest-*.md   # vTESTstudio CAPL test rules
│   └── integration-*.md  # Tool integration (DBC, ARXML, ODX, A2L, etc.)
└── docs/                 # Documentation and analysis
    └── 001-*.md          # Numbered analysis and design documents
```

## Rule Naming Convention

Each rule file follows the pattern: **`prefix-description.md`**

| Prefix | Domain |
|--------|--------|
| `arch-` | Software architecture and design patterns |
| `misra-` | MISRA C:2012 coding rules |
| `autosar-` | AUTOSAR C++14 guidelines |
| `safety-` | Functional safety (ISO 26262) |
| `security-` | Cybersecurity (ISO 21434) |
| `rtos-` | RTOS and task management |
| `realtime-` | Real-time constraints and determinism |
| `memory-` | Memory layout, alignment, and management |
| `perf-` | Performance and optimization |
| `comm-` | Communication protocols |
| `test-` | Testing methodology |
| `build-` | Build system, compilers, and linker configuration |
| `analysis-` | Static analysis tools (PC-lint, Polyspace, Coverity, etc.) |
| `capl-canoe-` | CANoe CAPL scripting |
| `capl-vtest-` | vTESTstudio test automation |
| `integration-` | Tool chain integration (ARXML, DBC, ODX, A2L, etc.) |

## How to Use This Skill

### For AI Agents

1. Read `SKILL.md` to understand the skill scope and activation triggers
2. Follow `AGENTS.md` for detailed coding rules and patterns during code generation
3. Reference individual `rules/*.md` files for specific rule details and code examples

### For Human Developers

1. Browse `rules/` to find rules relevant to your current task
2. Each rule file contains:
   - **Frontmatter** with impact level and tags for quick filtering
   - **Incorrect** code example showing the anti-pattern
   - **Correct** code example showing the recommended approach
   - **Reference** to the relevant standard clause
3. Use `docs/` for architectural analysis and design rationale

## Standards Covered

| Standard | Scope | Coverage |
|----------|-------|----------|
| **MISRA C:2012** | C coding guidelines for critical systems | Mandatory, Required, and Advisory rules |
| **AUTOSAR C++14** | C++ coding guidelines for adaptive platform | Core rules and architectural patterns |
| **ISO 26262** | Functional safety for road vehicles | Parts 4 (system), 6 (software), 8 (processes) |
| **ISO 21434** | Cybersecurity engineering for road vehicles | Secure coding, key management, secure boot |
| **CERT C** | Secure coding standard | Referenced via static analysis tool rules |
| **DO-178C** | Airborne software safety (cross-referenced) | Tool qualification parallels with ISO 26262 |

## Contributing

### Adding a New Rule

1. Copy `rules/_template.md` to `rules/prefix-description.md`
2. Fill in the frontmatter:
   ```yaml
   ---
   title: Descriptive Rule Title
   impact: HIGH | MEDIUM | LOW
   impactDescription: One sentence explaining why this rule matters
   tags: prefix, keyword1, keyword2, relevant-standard
   ---
   ```
3. Write the rule explanation, keeping it concise
4. Include **Incorrect** and **Correct** code examples (both are mandatory)
5. Add a `Reference:` line citing the relevant standard clause
6. Use the appropriate filename prefix from the naming convention table

### Frontmatter Requirements

- **title**: Human-readable rule name (matches the `##` heading)
- **impact**: `HIGH` (safety/correctness), `MEDIUM` (quality/maintainability), `LOW` (style/preference)
- **impactDescription**: One sentence; focuses on the consequence of violating the rule
- **tags**: Comma-separated; first tag should match the filename prefix

### Code Example Guidelines

- Use C11 for C examples, C++14 for C++ examples, CAPL for Vector tool examples
- Include enough context to compile (headers, types, function signatures)
- Comment the *why*, not the *what*
- Show realistic automotive use cases (CAN messages, sensor data, ECU states)

## License

MIT License. See individual standard references for their respective terms.
