# Zephyr Contribution Style Skill — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a new `zephyr-contribution-style` skill that teaches the LLM Zephyr upstream coding standards, and wire it into `zephyr-driver-development` as a dependency.

**Architecture:** Single SKILL.md file with 7 sections of rules + WRONG/RIGHT examples. One surgical edit to `zephyr-driver-development/SKILL.md` to add the dependency and remove redundant content.

**Tech Stack:** Markdown (skill files)

**Spec:** `docs/superpowers/specs/2026-03-24-zephyr-contribution-style-design.md`

**Sources:** The implementing LLM MUST consult these Zephyr docs for exact rule details and edge cases:
- Contributing overview: https://docs.zephyrproject.org/latest/contribute/index.html
- Contribution guidelines: https://docs.zephyrproject.org/latest/contribute/guidelines.html
- Coding guidelines: https://docs.zephyrproject.org/latest/contribute/coding_guidelines/index.html

---

### Task 1: Create the `zephyr-contribution-style` skill

**Files:**
- Create: `skills/zephyr-contribution-style/SKILL.md`

**Context:** This is a new skill directory and file. Follow the frontmatter convention used by other skills in the `skills/` directory (see `skills/zephyr-app-development/SKILL.md` for format). The skill description in frontmatter must match what would trigger it from the skill list.

- [ ] **Step 1: Fetch the Zephyr contribution docs**

Read the three source URLs listed above. Extract the exact rules needed for each section. The spec has a summary, but the original docs are authoritative — verify details against them.

- [ ] **Step 2: Create the skill directory**

```bash
mkdir -p skills/zephyr-contribution-style
```

- [ ] **Step 3: Write the SKILL.md**

Create `skills/zephyr-contribution-style/SKILL.md` with frontmatter and 7 sections as defined in the spec. Key requirements:

**Frontmatter:**
```yaml
---
name: zephyr-contribution-style
description: Use when writing or modifying code in the Zephyr source tree for upstream contribution - covers C code style (tabs, C89 comments, braces), MISRA-C subset, naming conventions, commit message format, Doxygen style, Kconfig style, and inclusive language rules
---
```

**Sections (see spec for full content of each):**

1. **C Code Style** — 5 rules with WRONG/RIGHT examples: tabs 8-char, 100-col lines, C89 comments only, braces always, no binary literals. IMPORTANT: all code examples in this section must use tab indentation (not spaces) to be self-consistent with rule #1.
2. **MISRA-C Subset** — 10 rules. Include WRONG/RIGHT examples for rules 4 (error returns), 6 (if-else if chains). For rule 3 (dynamic allocation), state: "Upstream Zephyr forbids stdlib `malloc`/`free` (MISRA Dir 4.12). Only use them if the project has `CONFIG_COMMON_LIBC_MALLOC=y` with `CONFIG_COMMON_LIBC_MALLOC_ARENA_SIZE != 0`. Note: `k_malloc`/`k_free` use a separate kernel heap controlled by `CONFIG_HEAP_MEM_POOL_SIZE`."
3. **Naming Conventions** — snake_case, subsystem prefix table, macro rules.
4. **Commit Messages** — Format template, area prefixes, Signed-off-by/Assisted-by rules. For `Assisted-by`, show a concrete example: `Assisted-by: Claude:claude-opus-4.6`.
5. **Doxygen** — `@` not `\`, parameter docs, file headers, with WRONG/RIGHT example.
6. **Kconfig Style** — Tab indent, help text format, symbol prefixes, with WRONG/RIGHT example.
7. **Inclusive Language** — Banned terms → replacements table.

**Filtering principle:** Only include rules where Zephyr's convention differs from LLM defaults, or where Zephyr-specific knowledge is required. Omit general best practices the LLM already knows.

**Size target:** ~300 lines.

- [ ] **Step 4: Review the skill for self-consistency**

Read through the completed SKILL.md and verify:
- All code examples use tab indentation (not spaces)
- No `//` comments in any C code example
- All `if` blocks have braces in examples
- All `switch` examples have `default`
- All `if-else if` chains end with `else`
- The skill follows its own rules

- [ ] **Step 5: Commit**

```bash
git add skills/zephyr-contribution-style/SKILL.md
git commit -m "skills: add zephyr-contribution-style for upstream coding standards"
```

---

### Task 2: Update `zephyr-driver-development` to reference the new skill

**Files:**
- Modify: `skills/zephyr-driver-development/SKILL.md:13-16` (REQUIRED BACKGROUND list)
- Modify: `skills/zephyr-driver-development/SKILL.md:281-289` (Prerequisites checklist)

- [ ] **Step 1: Add dependency to REQUIRED BACKGROUND**

In `skills/zephyr-driver-development/SKILL.md`, add `zephyr-contribution-style` to the REQUIRED BACKGROUND list. Current content (lines 12-16):

```
**REQUIRED BACKGROUND:**
- zephyr-devicetree (DT overlay and binding patterns)
- zephyr-kconfig (driver Kconfig patterns)
- embedded-tdd (driver testing approach)
- west-workspace-management (manifest for fork/upstream)
```

Add after the last item:
```
- zephyr-contribution-style (upstream coding standards, for in-tree contributions)
```

- [ ] **Step 2: Remove the Prerequisites checklist**

Remove lines 281-289 (the `### Prerequisites` heading and the checklist). These style/compliance checks are now covered by `zephyr-contribution-style`. Keep the `### Workflow` section that follows — that's process, not style.

Before (lines 279-291):
```
## Upstreaming to Zephyr

### Prerequisites

- [ ] Driver follows Zephyr coding style: ...
- [ ] DT binding in ...
- [ ] Vendor prefix in ...
- [ ] Tests in ...
- [ ] Tests pass on ...
- [ ] `MAINTAINERS.yml` entry
- [ ] Commit messages: ...

### Workflow
```

After:
```
## Upstreaming to Zephyr

### Workflow
```

- [ ] **Step 3: Verify the edit didn't break anything**

Read the modified file and confirm:
- The REQUIRED BACKGROUND list has 5 items (4 original + 1 new)
- The Upstreaming section flows from `## Upstreaming to Zephyr` directly to `### Workflow`
- No orphaned blank lines or broken formatting

- [ ] **Step 4: Commit**

```bash
git add skills/zephyr-driver-development/SKILL.md
git commit -m "zephyr-driver-development: reference zephyr-contribution-style, remove redundant checklist"
```
