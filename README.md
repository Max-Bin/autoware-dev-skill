# autoware-dev-skill

**Claude Code skill for Autoware autonomous driving development and debugging.**

Covers the full stack — architecture, package creation, node patterns, component interfaces, and issue diagnosis across perception, planning, control, and localization.

## Install

```bash
git clone git@github.com:Max-Bin/autoware-dev-skill.git ~/.claude/skills/autoware-dev
```

Or just ask your agent:

> Install the Autoware dev skill from https://github.com/Max-Bin/autoware-dev-skill

## What's Inside

| File | What it covers |
|------|---------------|
| `SKILL.md` | Quick reference tables and routing to the right guide |
| `references/architecture.md` | Data flow pipeline, three-tier structure, component interfaces, Agnocast |
| `references/development-guide.md` | CMake, package.xml, node, launch, config, and test templates |
| `references/debugging-guide.md` | Build failures, runtime errors, and module-specific issues |

## What It Helps With

- Creating Autoware ROS 2 packages with correct boilerplate
- Understanding the architecture (autoware → autoware_core → autoware_universe)
- Using standardized component interfaces and QoS settings
- Debugging build failures (CUDA/TensorRT, dependency conflicts, distro issues)
- Debugging runtime issues (QoS mismatch, DDS limits, empty data guards)
- Writing tests with `autoware_test_utils`

## License

Apache-2.0

---

*Built for AI agents. Made for autonomous driving developers.*
