# autoware-dev-skill

Autoware development and debugging skill for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). Covers the full autonomous driving stack — architecture, package creation, node implementation patterns, component interfaces, and common issue diagnosis across perception, planning, control, and localization modules.

## What's Included

```
SKILL.md                        — Main skill: quick reference tables and routing
references/
  architecture.md               — System architecture, data flow, component interfaces, Agnocast
  development-guide.md          — Templates for CMake, package.xml, nodes, launch, config, tests
  debugging-guide.md            — Common issues and fixes for build, runtime, and module-specific problems
```

## Install

```bash
claude skill add --from Max-Bin/autoware-dev-skill
```

## What It Helps With

- Creating new Autoware ROS 2 packages with correct CMake, package.xml, and node boilerplate
- Understanding the three-tier architecture (autoware → autoware_core → autoware_universe)
- Navigating the data flow pipeline (Sensing → Perception → Planning → Control)
- Using standardized component interfaces and QoS settings
- Debugging build failures (CUDA/TensorRT, dependency conflicts, distro incompatibilities)
- Debugging runtime issues (QoS mismatch, DDS limits, missing parameters, empty data guards)
- Writing tests with `autoware_test_utils`

## License

Apache-2.0
