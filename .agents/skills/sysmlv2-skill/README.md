# sysmlv2-skill

An Agent Skill for working with SysML v2 systems modeling language
accurately and effectively.

## Overview

This skill provides AI agents with authoritative guidance for creating
valid SysML v2 models. It includes syntax reference, validation
workflows, common patterns, and error troubleshooting based on OMG
specifications.

SysML v2 is a textual modeling language with formal syntax,
standardized by the Object Management Group (OMG). This skill helps
agents produce valid models that work with tools like SysON and
Eclipse SysML.

## Installation

### For Agent Users

Clone or download this skill into your agent's skills directory:

```sh
git clone https://github.com/DeciSym/sysmlv2-skill.git ~/.skills/sysmlv2-skill
```

Or add as a git submodule to your project:

```sh
git submodule add https://github.com/DeciSym/sysmlv2-skill.git .skills/sysmlv2-skill
```

### For Claude Code Users

Add the skill path to your Claude Code configuration.

## Usage

Once installed, compatible agents will automatically activate this
skill when:

- Working with `.sysml` files
- Validating SysML v2 models
- User mentions "sysml", "systems modeling", or "architecture model"
- Working with SysON or Eclipse SysML tools

### Key Capabilities

- **Syntax guidance**: Correct SysML v2 textual notation
- **Error resolution**: Common errors with solutions
- **Pattern library**: Reusable modeling patterns

## Project Structure

```
sysmlv2-skill/
├── SKILL.md              # Main skill instructions (required)
├── README.md             # This file
├── CLAUDE.md             # Claude Code specific guidance
└── references/
    ├── SYNTAX.md         # Complete syntax reference
    ├── PATTERNS.md       # Common modeling patterns
    └── ERRORS.md         # Error troubleshooting guide
```

This structure follows the [Agent Skills
Specification](https://agentskills.io/specification).

## Portability

This skill is designed for portability:

- No external runtime dependencies
- Works with any compatible agent
- POSIX-friendly documentation

## References

- [OMG SysML v2 Specification](https://www.omg.org/spec/SysML/2.0/)
- [SysML v2 Pilot Implementation](https://github.com/Systems-Modeling/SysML-v2-Pilot-Implementation)
- [Eclipse SysON Documentation](https://doc.mbse-syson.org/)
- [Agent Skills Specification](https://agentskills.io/specification)
