# Superpowers — Embedded Edition

A fork of [Superpowers](https://github.com/obra/superpowers) extended with **Zephyr RTOS embedded systems development** skills. Same workflow engine, same discipline — now with domain-specific knowledge for firmware development.

Superpowers is a complete software development workflow for your coding agents, built on top of a set of composable "skills" and some initial instructions that make sure your agent uses them.

## How it works

It starts from the moment you fire up your coding agent. As soon as it sees that you're building something, it *doesn't* just jump into trying to write code. Instead, it steps back and asks you what you're really trying to do. 

Once it's teased a spec out of the conversation, it shows it to you in chunks short enough to actually read and digest. 

After you've signed off on the design, your agent puts together an implementation plan that's clear enough for an enthusiastic junior engineer with poor taste, no judgement, no project context, and an aversion to testing to follow. It emphasizes true red/green TDD, YAGNI (You Aren't Gonna Need It), and DRY. 

Next up, once you say "go", it launches a *subagent-driven-development* process, having agents work through each engineering task, inspecting and reviewing their work, and continuing forward. It's not uncommon for Claude to be able to work autonomously for a couple hours at a time without deviating from the plan you put together.

There's a bunch more to it, but that's the core of the system. And because the skills trigger automatically, you don't need to do anything special. Your coding agent just has Superpowers.


## Sponsorship

If Superpowers has helped you do stuff that makes money and you are so inclined, I'd greatly appreciate it if you'd consider [sponsoring my opensource work](https://github.com/sponsors/obra).

Thanks! 

- Jesse


## Installation

This is a fork with embedded systems skills. Install from this repo, not the upstream marketplace.

### Claude Code (from GitHub)

```bash
/install-github-plugin toonst/superpowers-embedded
```

### Codex

Tell Codex:

```
Fetch and follow instructions from https://raw.githubusercontent.com/toonst/superpowers-embedded/refs/heads/main/.codex/INSTALL.md
```

**Detailed docs:** [docs/README.codex.md](docs/README.codex.md)

### OpenCode

Tell OpenCode:

```
Fetch and follow instructions from https://raw.githubusercontent.com/toonst/superpowers-embedded/refs/heads/main/.opencode/INSTALL.md
```

**Detailed docs:** [docs/README.opencode.md](docs/README.opencode.md)

### Gemini CLI

```bash
gemini extensions install https://github.com/toonst/superpowers-embedded
```

To update:

```bash
gemini extensions update superpowers-embedded
```

### Verify Installation

Start a new session and ask for something that should trigger an embedded skill (for example, "help me set up a Zephyr project" or "I'm getting a HardFault"). The agent should automatically invoke the relevant skill.

## The Basic Workflow

1. **brainstorming** - Activates before writing code. Refines rough ideas through questions, explores alternatives, presents design in sections for validation. Saves design document.

2. **using-git-worktrees** - Activates after design approval. Creates isolated workspace on new branch, runs project setup, verifies clean test baseline.

3. **writing-plans** - Activates with approved design. Breaks work into bite-sized tasks (2-5 minutes each). Every task has exact file paths, complete code, verification steps.

4. **subagent-driven-development** or **executing-plans** - Activates with plan. Dispatches fresh subagent per task with two-stage review (spec compliance, then code quality), or executes in batches with human checkpoints.

5. **test-driven-development** - Activates during implementation. Enforces RED-GREEN-REFACTOR: write failing test, watch it fail, write minimal code, watch it pass, commit. Deletes code written before tests.

6. **requesting-code-review** - Activates between tasks. Reviews against plan, reports issues by severity. Critical issues block progress.

7. **finishing-a-development-branch** - Activates when tasks complete. Verifies tests, presents options (merge/PR/keep/discard), cleans up worktree.

**The agent checks for relevant skills before any task.** Mandatory workflows, not suggestions.

## What's Inside

### Embedded Systems Skills (New)

**Zephyr App Development**
- **zephyr-app-development** - Project structure, build/flash, Zephyr API patterns (GPIO, SPI, I2C, logging, threads), adapting samples
- **zephyr-kconfig** - Kconfig reference: prj.conf patterns, custom symbols, merge order, debugging symbol resolution
- **zephyr-devicetree** - Devicetree reference: overlay syntax, DT macros (DT_ALIAS, DT_NODELABEL, DT_PROP), property access in C

**Embedded Debugging**
- **embedded-debugging** - cortex-debug VSCode setup (J-Link + OpenOCD), GDB commands, stack overflow detection, memory corruption diagnosis
- **cortex-m-fault-diagnosis** - ARM Cortex-M fault register decode (CFSR/HFSR), common fault scenarios, diagnosis flowchart

**Build & Workspace**
- **west-workspace-management** - West manifests, module management, Zephyr version pinning, multi-repo project setup

**Embedded Testing**
- **embedded-tdd** - TDD for firmware: ztest/native_sim unit tests, pytest HIL over serial, mocking hardware dependencies

**Driver Development**
- **zephyr-driver-development** - Zephyr device model, DT bindings, DT_INST macros, twister testing, upstream PR workflow, fork management

### Core Skills (from upstream Superpowers)

**Testing**
- **test-driven-development** - RED-GREEN-REFACTOR cycle (includes testing anti-patterns reference)

**Debugging**
- **systematic-debugging** - 4-phase root cause process (includes root-cause-tracing, defense-in-depth, condition-based-waiting techniques)
- **verification-before-completion** - Ensure it's actually fixed

**Collaboration**
- **brainstorming** - Socratic design refinement
- **writing-plans** - Detailed implementation plans
- **executing-plans** - Batch execution with checkpoints
- **dispatching-parallel-agents** - Concurrent subagent workflows
- **requesting-code-review** - Pre-review checklist
- **receiving-code-review** - Responding to feedback
- **using-git-worktrees** - Parallel development branches
- **finishing-a-development-branch** - Merge/PR decision workflow
- **subagent-driven-development** - Fast iteration with two-stage review (spec compliance, then code quality)

**Meta**
- **writing-skills** - Create new skills following best practices (includes testing methodology)
- **using-superpowers** - Introduction to the skills system

## Embedded Workflow

The embedded skills integrate naturally with the core Superpowers workflow:

1. **brainstorming** → Design your firmware feature
2. **writing-plans** → Plan references `zephyr-app-development`, `zephyr-kconfig`, `zephyr-devicetree` for Zephyr-specific guidance
3. **embedded-tdd** → RED-GREEN-REFACTOR with ztest on native_sim, then promote to pytest HIL
4. **embedded-debugging** / **cortex-m-fault-diagnosis** → When things crash
5. **west-workspace-management** → Manage your Zephyr workspace, modules, and versions
6. **zephyr-driver-development** → Write drivers, test with twister, upstream to Zephyr

**Target hardware:** Any board supported by Zephyr — STM32, Nordic nRF, NXP, Raspberry Pi Pico, and more. Board is typically fixed per project.

**Toolchain:** west + CMake + Kconfig, command-line workflow.

## Philosophy

- **Test-Driven Development** - Write tests first, always — even for firmware
- **Systematic over ad-hoc** - Process over guessing
- **Complexity reduction** - Simplicity as primary goal
- **Evidence over claims** - Verify before declaring success

Read more: [Superpowers for Claude Code](https://blog.fsck.com/2025/10/09/superpowers/)

## Contributing

Skills live directly in this repository. To contribute:

1. Fork the repository
2. Create a branch for your skill
3. Follow the `writing-skills` skill for creating and testing new skills
4. Submit a PR

See `skills/writing-skills/SKILL.md` for the complete guide.

## Updating

Skills update automatically when you update the plugin:

```bash
/plugin update superpowers
```

## License

MIT License - see LICENSE file for details

## Community

Superpowers is built by [Jesse Vincent](https://blog.fsck.com) and the rest of the folks at [Prime Radiant](https://primeradiant.com).

For community support, questions, and sharing what you're building with Superpowers, join us on [Discord](https://discord.gg/Jd8Vphy9jq).

## Support

- **Issues**: https://github.com/toonst/superpowers-embedded/issues
- **Upstream Superpowers**: https://github.com/obra/superpowers
- **Upstream Discord**: [Join us on Discord](https://discord.gg/Jd8Vphy9jq)
