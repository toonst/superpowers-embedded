---
name: west-workspace-management
description: Use when managing west workspaces, west manifests, Zephyr version pinning, adding west modules, multi-repo Zephyr project setup, or troubleshooting west update and west init issues
---

# West Workspace Management

## Overview

Managing Zephyr workspaces with west — manifest files, module dependencies, Zephyr version pinning, and multi-repo project organization.

## When to Use

- Setting up a new Zephyr project workspace
- Adding or updating west modules
- Pinning or updating the Zephyr version
- Multi-repo project setup with custom modules
- `west update` failures or version conflicts

**When NOT to use:**
- Build/flash commands (use zephyr-app-development)
- Debugging sessions (use embedded-debugging)

## Workspace Layout (Application-Centric)

```
my-workspace/
├── my-app/                 # ← manifest repo (your code)
│   ├── west.yml
│   ├── CMakeLists.txt
│   ├── prj.conf
│   └── src/
├── zephyr/                 # Managed by west
├── modules/                # Managed by west
└── .west/
    └── config
```

### Init from scratch

```bash
mkdir my-workspace && cd my-workspace
git init my-app
# (create west.yml in my-app)
west init -l my-app
west update
```

## West Manifest (west.yml)

### Minimal manifest

```yaml
manifest:
  remotes:
    - name: zephyrproject-rtos
      url-base: https://github.com/zephyrproject-rtos

  projects:
    - name: zephyr
      remote: zephyrproject-rtos
      revision: v4.1.0
      import: true              # Pull in Zephyr's own dependencies

  self:
    path: my-app
```

### Adding a custom module

```yaml
manifest:
  remotes:
    - name: zephyrproject-rtos
      url-base: https://github.com/zephyrproject-rtos
    - name: my-org
      url-base: https://github.com/my-org

  projects:
    - name: zephyr
      remote: zephyrproject-rtos
      revision: v4.1.0
      import: true

    - name: my-driver-library
      remote: my-org
      revision: main
      path: modules/lib/my-driver-library

  self:
    path: my-app
```

### Overriding an imported project's revision

```yaml
  projects:
    - name: zephyr
      remote: zephyrproject-rtos
      revision: v4.1.0
      import: true

    # Overrides the revision imported by Zephyr
    - name: hal_stm32
      remote: zephyrproject-rtos
      revision: my-custom-branch
      path: modules/hal/stm32
```

Manifest repo declarations take priority over imported ones.

### Filtering imports (reduce clone time)

```yaml
    - name: zephyr
      import:
        name-allowlist:
          - cmsis
          - hal_stm32
          - hal_nordic
```

## Common West Operations

```bash
west update                          # Update all to manifest
west update zephyr                   # Update single project
west list                            # Show manifest state
west list zephyr --format "{abspath}" # Show project path
west diff                            # Manifest vs actual
west manifest --resolve              # Show import resolution
west manifest --validate             # Check for errors
```

## Updating Zephyr Version

1. Check current: `cat zephyr/VERSION && west list zephyr`
2. Read release notes / migration guide for target version
3. Update `revision` in `west.yml`
4. `west update`
5. Check for breaking changes: `west build -b <board> -p auto 2>&1 | grep -i "deprecated\|binding"`
6. Build and test on all target boards

### Version pinning strategy

| Strategy | When |
|----------|------|
| Release tag (`v4.1.0`) | Production — stable, tested |
| Branch (`main`) | Development only — tracking latest |
| Commit SHA | Maximum reproducibility — CI/CD |

## Making a Library a West Module

```
my-library/
├── zephyr/
│   └── module.yml          # ← Required
├── CMakeLists.txt
├── Kconfig
├── include/
└── src/
```

**`zephyr/module.yml`:**
```yaml
build:
  cmake: .
  kconfig: Kconfig
```

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `west update` fails | Revision not found | `git ls-remote <url> <revision>` |
| Module not found by build | Missing `zephyr/module.yml` | Create it with cmake/kconfig paths |
| Import cycle | Manifests import each other | Use explicit deps, not circular imports |
| `west: unknown command 'build'` | Zephyr not in workspace | `west update`, check `west list` |
| Stale build after update | CMake cache stale | `west build -p always` |
