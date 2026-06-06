# Contributing Upstream

MeshCore is an open-source project maintained at
[github.com/meshcore-dev/MeshCore](https://github.com/meshcore-dev/MeshCore).
Contributions of all sizes are welcome — from one-line typo fixes to new board
variants and major features.

The authoritative contribution guide is
[CONTRIBUTING.md](https://github.com/meshcore-dev/MeshCore/blob/main/CONTRIBUTING.md)
in the repo root. This page summarises the workflow and adds practical context
for the kinds of contributions most common in MeshCore.

## What to contribute

| Contribution type | Process |
|---|---|
| Typo / comment / small doc fix | Open a PR directly — no issue needed |
| Bug fix | Open a PR directly; reference any related issue |
| New example or board variant | Open an issue first to discuss; get a rough ✓ from maintainers; then PR |
| New feature or protocol change | Open an issue first; discuss; agree on approach; then PR |
| Number allocation (new data-type) | Add a row to `docs/number_allocations.md` and PR — must demonstrate a working app |

## The branch workflow

All work branches off **`dev`**, not `main`. `main` tracks the most recent
release; new work lands on `dev` first.

```bash
# Fork the repo on GitHub, then:
git clone https://github.com/YOUR_USERNAME/MeshCore.git
cd MeshCore
git remote add upstream https://github.com/meshcore-dev/MeshCore.git
git fetch upstream
git checkout -b feature/my-feature upstream/dev
```

Branch naming conventions:

| Prefix | Use for |
|---|---|
| `fix/` | Bug fixes |
| `feature/` | New features |
| `docs/` | Documentation changes |
| `board/` | New board variants (unofficial convention — aligns with the `feature/` class) |

## Making your changes

1. **One feature / fix = one pull request.** Smaller PRs review faster and land sooner.
2. **Update or add examples** when you add or change a public API.
3. **Add/update code comments** — especially for non-obvious decisions.
4. **If you change a public API**, update `README.md` and `library.properties`.
5. **Include an example sketch** in `examples/` for new features where practical.

## Coding style

MeshCore follows the `.clang-format` at the repo root. Key rules:

- **2 spaces** indentation — no tabs.
- **`camelCase`** for functions and variables.
- **`UpperCamelCase`** (PascalCase) for class names.
- **`ALL_CAPS`** for `#define` constants.
- Lines ≤ ~100 characters where reasonable.
- Consistency with existing code takes priority over strict rule-following.

Run clang-format before committing:

```bash
# Format a single file
clang-format -i src/helpers/MyHelper.cpp

# Format all changed files (requires git and clang-format on PATH)
git diff --name-only | grep -E '\.(cpp|h)$' | xargs clang-format -i
```

## Submitting the pull request

Push your branch and open a PR against `meshcore-dev/MeshCore`'s **`dev`** branch:

```bash
git push -u origin feature/my-feature
gh pr create --base dev --title "feature: add myboard variant" \
  --body "Adds support for the MyBoard (ESP32-S3, SX1262). Tested with repeater and sensor firmwares."
```

In the PR description:
- Reference any related issue with `Fixes #123` or `Closes #89`.
- Describe what hardware you tested on and what firmwares you built.
- If the PR adds a new board, note any known limitations.

## Descriptive commit messages

Good commit messages help reviewers and make the git history useful:

```
# Good
Fix I2C timeout handling on ESP32 when SDA held low after reset
Add RAK19007 base board variant with SX1262 and OLED
Update SensorManager interface to support multi-channel GPS

# Bad
fix
update
wip
```

## Automated agent PRs

If you are an automated agent building a PR, append `🤖🤖` to the PR title:

```
feature: add myboard variant 🤖🤖
```

This opts in to a fast-track merge process maintained by the project.

## After your PR is merged

1. Delete your feature branch.
2. Sync your fork with upstream `dev`:
   ```bash
   git fetch upstream
   git checkout dev
   git merge upstream/dev
   git push origin dev
   ```
3. If you added a new board variant, consider opening a PR to
   [meshcore-guide](https://github.com/jagoda/meshcore-guide) adding the board
   to the [What You Need](../getting-started/what-you-need.md) hardware list.

## Reporting bugs

Use the GitHub Issues tracker. Include:

- Your board name and firmware version (shown in the node's status reply).
- Exact steps to reproduce.
- IDE/toolchain version if it's a build issue.
- A minimal code snippet or config if applicable.

Prefix enhancement requests with `[Feature request]` in the title.

---

## The arc is complete

You have now followed the full learning path: from flashing your first device
and understanding the mesh, through operating infrastructure and reading the
protocol, to building your own sensor, application, or board variant and
contributing it back to the community.

If you built something that lives at the host layer — an application that speaks
to MeshCore via the Companion API — see [Building Applications →](../applications/index.md)
for design patterns, worked examples, and a testing guide.

For quick lookups, the [Reference →](../reference/index.md) section has the
glossary, spec index, QR/link formats, and regional frequency tables.
