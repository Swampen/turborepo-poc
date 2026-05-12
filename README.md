# Dependabot Security Update Reproduction

This repository is a sanitized public proof of concept for a Dependabot npm security update failure in a Turborepo workspace. It is intended to be readable by Dependabot maintainers, support engineers, and AI agents investigating the behavior.

The fixture uses only neutral folder and package names. The important behavior is the dependency graph and lockfile shape, not the application code.

## What This Reproduces

Dependabot is asked to create a security update for:

```text
@babel/plugin-transform-modules-systemjs
```

The package is present as a transitive dependency, locked at `7.29.0`. The advisory in `dependabot-job.yml` marks versions `>= 7.12.0 <= 7.29.3` as affected, with `7.29.4` being the first non-vulnerable release.

The expected Dependabot result is:

```yaml
error-type: security_update_not_possible
error-details:
  conflicting-dependencies:
    - dependency_name: '@babel/plugin-transform-modules-systemjs'
      explanation: No patched version available for @babel/plugin-transform-modules-systemjs
      fix_available: false
      fix_updates: []
      top_level_ancestors: []
  dependency-name: '@babel/plugin-transform-modules-systemjs'
  latest-resolvable-version: 7.29.0
  lowest-non-vulnerable-version: 7.29.4
```

The full captured CLI output is committed in `dependabot-output.yml`.

## Repository Shape

This is an npm workspace Turborepo with:

- `apps/app-a`
- `apps/app-b`
- `apps/app-c`
- `apps/app-d`
- `packages/config-eslint`
- `packages/config-typescript`
- `packages/shared-content`
- `packages/shared-i18n`
- `packages/shared-lib`
- `packages/shared-ui`

The workspace references use neutral local package names under `@dependabot-repro/*`.

The dependency path that matters for the repro is:

```text
sanity
  -> @sanity/cli
    -> @sanity/codegen
      -> @babel/preset-env
        -> @babel/plugin-transform-modules-systemjs@7.29.0
```

## How To Run

Run Dependabot CLI from the repository root:

```powershell
dependabot update -f .\dependabot-job.yml -o .\dependabot-output.yml
```

`dependabot-job.yml` is pinned to the commit that is known to reproduce the issue. If you want to test a newer commit, update `job.source.commit` first.

This public fixture does not require a token for the basic repro. If you do run with credentials, note that Dependabot CLI rejects GitHub API credentials that have write access.

## Lockfile Notes

The lockfile is npm lockfile version 3. The relevant package should stay at:

```text
node_modules/@babel/plugin-transform-modules-systemjs: 7.29.0
```

If the lockfile needs to be regenerated, use a registry timestamp before the patched release was published:

```powershell
npm install --package-lock-only --ignore-scripts --before 2026-05-05T09:42:58Z
```

Then verify:

```powershell
node -e "const lock=require('./package-lock.json'); console.log(lock.packages['node_modules/@babel/plugin-transform-modules-systemjs'].version)"
```

The expected output is:

```text
7.29.0
```

## Files Of Interest

- `dependabot-job.yml`: Dependabot job definition for the security update.
- `dependabot-output.yml`: Captured output showing the reproduced error.
- `package-lock.json`: Lockfile containing the transitive vulnerable package version.
- `package.json`: Root npm workspace definition.

## Maintainer Notes

The surprising part is that the patched package exists on npm and Dependabot identifies `7.29.4` as the lowest non-vulnerable version, but the update is still reported as not possible with no top-level ancestors or fix updates.

The fixture is intentionally small at the source-code level and larger at the dependency level so the resolver sees a realistic npm workspace graph.
