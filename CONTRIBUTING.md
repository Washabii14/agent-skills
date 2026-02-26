# Contributing to Agent Skills

Thank you for your interest in contributing! This document provides guidelines for adding or improving skills.

## How to Contribute

### Adding a New Skill

1. Create a new folder under `skills/` with a lowercase, hyphenated name:
   ```
   skills/
   └── your-skill-name/
       ├── SKILL.md      (required)
       ├── AGENTS.md     (optional, compiled rules)
       ├── README.md     (optional)
       ├── rules/        (optional, individual rule files)
       └── docs/         (optional)
   ```

2. **SKILL.md** is required. It must have YAML frontmatter:
   ```yaml
   ---
   name: your-skill-name
   description: What this skill does and when to use it. Use when...
   license: MIT
   metadata:
     author: your-name
     version: "1.0.0"
   ---
   ```

   The `name` field **must exactly match** the folder name.

3. Update the repo-level [README.md](README.md) to list your new skill.

### Improving Existing Skills

- Edit files in `skills/automotive-embedded-skills/` (or the relevant skill folder)
- Follow the existing structure: incorrect → correct code examples in rule files
- Keep AGENTS.md in sync if you add or change rules

### Rule File Format

Each rule file in `rules/` should follow this pattern:

```markdown
---
title: Rule Title
impact: CRITICAL|HIGH|MEDIUM|LOW
impactDescription: brief description
tags: category, tag1, tag2
---

## Rule Title

Brief explanation.

**Incorrect (what's wrong):**
\```c
// bad code
\```

**Correct (what's right):**
\```c
// good code
\```

Reference: Standard reference (e.g., MISRA C:2012 Rule X.Y)
```

### Submitting Changes

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/your-feature`
3. Make your changes
4. Commit with a clear message: `git commit -m "Add X"` or `git commit -m "Fix Y in automotive-embedded-skills"`
5. Push: `git push origin feature/your-feature`
6. Open a Pull Request with a description of your changes

### Code of Conduct

By participating, you agree to abide by our [Code of Conduct](CODE_OF_CONDUCT.md).

### Questions?

Open an [Issue](https://github.com/Washabii14/agent-skills/issues) for questions or discussions.
