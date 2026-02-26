# `turbo prune --docker` generates broken pnpm lockfile

Minimal reproduction for a `turbo prune` bug where the pruned `pnpm-lock.yaml` is internally inconsistent, causing `pnpm install --frozen-lockfile` to fail.

## Bug

When a workspace package has a module listed as both a `devDependency` and a `peerDependency`, and the pruned app depends on that workspace package but does **not** directly depend on the module, `turbo prune --docker` keeps the importer entry referencing the module but omits the package resolution from the pruned lockfile.

```
ERR_PNPM_LOCKFILE_MISSING_DEPENDENCY  Broken lockfile: no entry for 'uuid@11.1.0' in pnpm-lock.yaml
```

## Setup

- `packages/pkg-a` — has `uuid` as a `devDependency` and a `peerDependency`
- `apps/app-a` — depends on `pkg-a` (does **not** depend on `uuid`)
- `apps/app-b` — depends on `pkg-a` **and** `uuid` directly

Pruning for `app-a` includes `pkg-a` in the pruned output. The pruned lockfile retains `pkg-a`'s importer section (which references `uuid@11.1.0` as a devDependency) but drops the `uuid@11.1.0` package resolution and snapshot entries.

## Reproduce

```bash
pnpm install
npx turbo prune app-a --docker
cd out/json
pnpm install --frozen-lockfile
```

Expected: install succeeds.
Actual: `ERR_PNPM_LOCKFILE_MISSING_DEPENDENCY`.

## Workaround

Add `uuid` (the missing dependency) to `app-a`'s `devDependencies` so that turbo prune includes the package resolution in the pruned lockfile.
