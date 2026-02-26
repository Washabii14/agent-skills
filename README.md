# Agent Skills — Automotive Embedded

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT) [![Skills](https://img.shields.io/badge/Skills-AI%20Agents-blue)](https://skills.sh)

**A collection of AI agent skills for automotive embedded software development.**

Coding rules aligned with MISRA C:2012, AUTOSAR C++14, ISO 26262, and ISO 21434 — ready to install via `npx skills add`.

---

## 🌱 Getting Started

Install the skill with one command. Your AI agent will automatically use it when working on automotive embedded code.

```shell
npx skills add Washabii14/agent-skills --skill automotive-embedded-skills
```

Or browse and install all skills in this collection:

```shell
npx skills add Washabii14/agent-skills
```

After installation, the skill is available to your AI agent (Cursor, Codex, GitHub Copilot, Claude Code, etc.). No configuration needed.

---

## 📚 Available Skills

| Skill | Description | Rules | Use When |
|-------|-------------|-------|----------|
| **automotive-embedded-skills** | C/C++/CAPL guidelines for automotive ECUs | 184 rules, 23 categories | Writing embedded code, MISRA/AUTOSAR compliance, CAN/LIN/Ethernet, CAPL simulation, diagnostics, safety-critical systems |

---

## 🛠️ What's Covered

### automotive-embedded-skills

| Category | Impact | Rules |
|----------|--------|-------|
| Memory Safety & Management | Critical | 9 |
| MISRA C/C++ Compliance | Critical | 21 |
| AUTOSAR Classic & Adaptive | Critical | 23 |
| Safety / ISO 26262 | High | 8 |
| Real-Time & Timing | High | 7 |
| Communication (CAN/LIN/Ethernet/UDS) | High | 22 |
| CAPL — CANoe & vTESTstudio | Medium-High | 20 |
| Fault Injection (CAN/LIN/Ethernet) | High | 3 |
| Security / ISO 21434 | High | 8 |
| ECU Boot, NVM, Power Management | High | 14 |
| Compiler & Static Analysis | Medium | 15 |
| Testing & Verification | Medium | 6 |
| Tool Integration (A2L/ODX/XCP) | Medium | 6 |

**Standards:** MISRA C:2012, MISRA C++:2023, AUTOSAR C++14, ISO 26262, ISO 21434, ISO 13400 (DoIP), ISO 14229 (UDS)

---

## 💬 Example Prompts

Once installed, your AI agent will use these rules when you ask:

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

```
Refactor this function to follow ISO 26262 defensive programming
```

---

## 📂 Skill Structure

Each skill in this collection contains:

| File | Purpose |
|------|---------|
| `SKILL.md` | Agent instructions and rule index |
| `AGENTS.md` | Full compiled rules (4700+ lines) |
| `rules/` | Individual rule files with examples |
| `docs/` | Design documents and analysis |
| `README.md` | Skill-specific documentation |

---

## 🙏 Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/amazing-skill`)
3. Add or update skills under `skills/`
4. Ensure each skill has a valid `SKILL.md` with `name` matching the folder
5. Commit your changes (`git commit -m 'Add amazing skill'`)
6. Push to the branch (`git push origin feature/amazing-skill`)
7. Open a Pull Request

See [CONTRIBUTING.md](CONTRIBUTING.md) for detailed guidelines.

---

## 📄 License

This project is licensed under the MIT License — see [LICENSE](LICENSE) for details.

---

## 🔗 Links

- [Skills CLI Documentation](https://skills.sh/docs)
- [Browse Skills Leaderboard](https://skills.sh)
- [Agent Skills Format](https://agentskills.io)
