# Monorepo configuration example

PR Sheriff can help small monorepos keep pull requests reviewable, even before
first-class package-level monorepo support is available. This guide shows a
copyable starting point for a repository with two packages and shared
infrastructure.

## Example layout

```text
.
├── packages/
│   ├── api/
│   │   ├── src/
│   │   └── tests/
│   └── web/
│       ├── src/
│       └── tests/
├── infra/
│   ├── terraform/
│   └── migrations/
├── docs/
├── scripts/
└── .github/workflows/
```

In this layout, `packages/api` and `packages/web` are separate application
areas, while `infra`, `scripts`, and GitHub workflows are shared infrastructure
that usually deserve closer review.

## Copyable `.pr-sheriff.json`

```json
{
  "max_changed_lines": 900,
  "max_changed_files": 35,
  "require_tests_after_lines": 120,
  "test_patterns": [
    "packages/*/tests/**",
    "packages/**/*_test.*",
    "packages/**/*.test.*",
    "packages/**/*.spec.*",
    "tests/**"
  ],
  "sensitive_patterns": [
    ".github/workflows/**",
    "infra/**",
    "scripts/deploy/**",
    "**/auth/**",
    "**/migrations/**",
    "**/Dockerfile",
    "package-lock.json",
    "poetry.lock"
  ],
  "ignore_patterns": [
    "**/*.md",
    "docs/**",
    "packages/*/README.md"
  ],
  "path_rules": [
    {
      "name": "api package",
      "patterns": ["packages/api/**"],
      "max_changed_lines": 350,
      "max_changed_files": 18,
      "require_tests_after_lines": 60
    },
    {
      "name": "web package",
      "patterns": ["packages/web/**"],
      "max_changed_lines": 450,
      "max_changed_files": 22,
      "require_tests_after_lines": 80
    },
    {
      "name": "shared infrastructure",
      "patterns": ["infra/**", ".github/workflows/**", "scripts/deploy/**"],
      "max_changed_lines": 150,
      "max_changed_files": 8,
      "require_tests_after_lines": 1
    }
  ]
}
```

## How to read this policy

- The global limits keep very large cross-repository pull requests visible.
- `test_patterns` count tests from both package-local test folders and a shared
  top-level `tests/` folder.
- `ignore_patterns` lets documentation-only changes avoid consuming the review
  budget.
- `sensitive_patterns` highlights shared infrastructure, deployment scripts,
  authentication code, migrations, Dockerfiles, workflows, and lockfiles.
- `path_rules` applies stricter thresholds to each package and to shared
  infrastructure.

## Current limitations

PR Sheriff does not yet calculate separate risk scores per package. If a pull
request touches both `packages/api` and `packages/web`, the global policy still
sees one pull request, and each matching path rule reports its own violations.
Use the path rule names to make review comments clearer, but do not treat them
as independent package-level risk reports.

PR Sheriff also does not understand package dependency graphs. For example, a
change under `packages/api` that affects `packages/web` through generated types
or shared contracts will not be inferred automatically. Keep shared contracts in
sensitive paths or add a dedicated path rule for them.

## Rollout pattern

Start in advisory mode while contributors learn the policy:

```bash
pr-sheriff check --base origin/main --advisory
```

After the team agrees that the limits are useful, remove advisory mode in CI so
policy violations fail the check. Keep the first thresholds generous, then lower
them as the repository establishes normal pull request sizes.
