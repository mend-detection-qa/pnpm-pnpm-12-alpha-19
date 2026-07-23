# pnpm12-alpha19-registry-workspace

Probe for pnpm 12 Alpha 19. Exercises two feature categories
introduced or changed in that release: `install_command` and
`registry_source`.

## Feature exercised

This probe verifies that Mend's pnpm resolver (`PnpmLockCollector`,
lockfileVersion 9.0) correctly handles:

1. **`install_command` — `--no-runtime` in non-frozen installs.**
   pnpm 12 Alpha 19 changed the behavior of `--no-runtime` when
   used without `--frozen-lockfile`. Specifically, a non-frozen
   install that passes `--no-runtime` still writes a complete
   lockfile (no entries are pruned). Mend must read that full
   lockfile. A regression would be Mend seeing a partially-written
   or pruned lockfile and reporting fewer packages.

2. **`registry_source` — `workspace:` preservation under `--latest`
   and GitHub Actions registry in `.npmrc`.**
   When `--latest` is passed, pnpm resolves registry packages to
   their latest version but preserves `workspace:` references as
   local links. Additionally, `.npmrc` sets
   `@my-org:registry=https://npm.pkg.github.com`. Mend must:
   - Report `@my-org/shared-utils` (an intra-repo workspace
     package) as `source: "local"`, NOT `source: "private-index"`,
     because the lockfile resolves it via `link:` — the `.npmrc`
     registry override applies only to packages fetched from the
     network, not workspace packages.
   - Report `hono` and `zod` as `source: "registry"` (public
     npm registry), unaffected by the `@my-org:` scope override.

## Project structure

```
.
├── .npmrc
├── .whitesource
├── apps/
│   └── api/
│       └── package.json      (@my-org/api, depends on workspace:* + hono)
├── packages/
│   └── shared-utils/
│       └── package.json      (@my-org/shared-utils, depends on zod)
├── package.json              (root workspace manifest)
├── pnpm-lock.yaml            (lockfileVersion 9.0)
├── pnpm-workspace.yaml
└── README.md
```

## Expected dependency tree

See `expected-tree.json`. Key expectations:

- Root importer has no direct dependencies (root package.json
  declares none; workspaces are importers, not root dependencies).
- `@my-org/api` workspace package has two direct dependencies:
  - `@my-org/shared-utils` — `source: "local"` (workspace link)
  - `hono` — `source: "registry"`, version `4.4.13`
- `@my-org/shared-utils` workspace package has one direct
  dependency:
  - `zod` — `source: "registry"`, version `3.23.8`
- `hono@4.4.13` has no transitive dependencies in this probe.
- `zod@3.23.8` has no transitive dependencies in this probe.

## Mend config

**Bucket A** — `js-pnpm` has no dynamic tool-version detection
from the manifest. This probe emits `.whitesource` with
`scanSettings.versioning` pinning:

- `pnpm: "12.0.0-alpha.19"` — exact alpha release under test
- `node: "20.11.1"` — LTS Node version

`configMode` is `"AUTO"` because no `whitesource.config` is
present in the probe root.

No additional Mend config dimensions apply (no branch scoping,
no project-token routing, no suppression rules).

## Resolver notes

- Mend uses `PnpmLockCollector` (lockfileVersion 9.0 →
  `PnpmParserV9Impl`).
- In v9 format, dependencies come from `snapshots`, not
  `packages`. The `importers` section provides workspace
  membership. A known Mend failure mode for v9 is reading
  deps from `packages` (which has none in v9) and reporting
  zero transitive dependencies.
- The `.npmrc` registry override scopes to `@my-org`. In
  this lockfile the only `@my-org` package is a workspace
  link (`link:../../packages/shared-utils`), so it never
  hits the registry. Mend must not misclassify it as
  `private-index`.
- `--no-runtime` (pnpm 12 Alpha 19) does not alter the
  lockfile structure visible to Mend. The resolver reads
  the same lockfile regardless of which install flags were
  used. This pattern probes whether Mend's pre-step (when
  `npm.runPreStep=true`) correctly handles the new flag
  without erroring out or producing a partial lockfile.

## Pattern source

Added from pnpm 12 Alpha 19 release notes (categories:
`install_command`, `registry_source`).
Status: untested
