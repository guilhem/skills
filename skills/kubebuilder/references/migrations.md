# Migrations

Keep your Kubebuilder operator scaffold aligned with ecosystem changes across releases.

Migration may involve updating to a newer plugin version (e.g., `go.kubebuilder.io/v3` → `v4`)
or updating the scaffold within the same plugin across CLI releases (e.g., `v4.9.0` → `v4.10.1`).

See [Kubebuilder migrations](https://master.book.kubebuilder.io/migrations.html).

## The PROJECT File

From Kubebuilder v3.0.0 onwards, all inputs are tracked in the `PROJECT` file.
All automation tools (`alpha update`, `alpha generate`, AutoUpdate plugin) depend on it.

Always use the CLI to scaffold resources (`kubebuilder create api`, `kubebuilder create webhook`)
so they are tracked. Manually created APIs won't appear in `PROJECT` and won't be
handled by migration tools.

## Migration Methods

The command examples below target the released **Kubebuilder v4.16.0** CLI.
Check `kubebuilder version` and each command's `--help` before using another
release; alpha behavior and flags can change.

**Protect local data first.** In v4.16.0, in-place `alpha generate` removes every
entry except `.git` (it reads `PROJECT` into memory and recreates it).
`alpha update` also cleans its working directory while building merge branches.
Its `git status --porcelain` check does not include ignored files: a clean tree
does not protect local configuration, credentials, generated artifacts, or caches.
Keep the source checkout untouched and use fresh directories outside it.
Back up required ignored/untracked files outside the migration directory;
do not commit secrets to make the tree clean. A clone includes only committed
code, so capture intended source changes in the selected branch first. If migration
itself needs local files, check how the tool reads them before cleanup; use manual
integration if it cannot retain them. Restore needed inputs from the external
backup when validating the result, after cleanup.

See the pinned [generate implementation](https://github.com/kubernetes-sigs/kubebuilder/blob/v4.16.0/internal/cli/alpha/internal/generate.go),
[update cleanup](https://github.com/kubernetes-sigs/kubebuilder/blob/v4.16.0/internal/cli/alpha/internal/update/update.go),
and [Git validation](https://github.com/kubernetes-sigs/kubebuilder/blob/v4.16.0/internal/cli/alpha/internal/update/validate.go).

### 1. AutoUpdate Plugin (GitHub Actions) — Recommended

```bash
kubebuilder edit --plugins="autoupdate/v1-alpha"
```

Scaffolds a GitHub Action that monitors new Kubebuilder releases and opens an issue
with a PR compare link. Works like Dependabot for your operator scaffold.

- Requires GitHub repository
- Requires Kubebuilder v4.5.0+
- Branch protection rules recommended for safety
- Uses a 3-way merge of committed code; review conflicts and generated changes

See [AutoUpdate Plugin docs](https://master.book.kubebuilder.io/plugins/available/autoupdate-v1-alpha.html).

### 2. Local Update (`alpha update`)

```bash
# Run from the source project on a named branch containing the intended code.
set -eu
source_dir=$(pwd -P)
source_branch=$(git branch --show-current)
test -n "$source_branch"
migration_dir=$(mktemp -d /tmp/kubebuilder-update.XXXXXX)
git clone --no-local --branch "$source_branch" "$source_dir" "$migration_dir/project"
cd "$migration_dir/project"
kubebuilder alpha update \
  --from-version "<original-cli-version>" \
  --to-version v4.16.0 \
  --from-branch "$source_branch"
```

Replace `<original-cli-version>` with the release that scaffolded the source,
normally recorded in `PROJECT.cliVersion`. This is an independent clone, rather
than a linked worktree sharing refs with the source. The command creates local
commits/branches there and stops on conflicts by default; review the result and
run project validation before transferring the chosen changes back.
Do not use `--force` to treat unresolved conflicts as a successful migration.
No push is needed for inspection.

The 3-way merge attempts to retain committed customizations, but cannot recover
ignored or uncommitted data and does not guarantee a conflict-free or correct result.

- Requires v4.5.0+
- For projects created before v4.6.0: supply the known original `--from-version`;
  if unknown or unsupported, use separate generation and manual integration below
- For projects created with v4.6.0+: includes `cliVersion` in `PROJECT` for automatic
  version detection

See [alpha update reference](https://master.book.kubebuilder.io/reference/commands/alpha_update.html).

### 3. Regenerate + Manual Merge (`alpha generate`)

```bash
# Run from the source project; the output is a fresh directory outside it.
set -eu
source_dir=$(pwd -P)
migration_dir=$(mktemp -d /tmp/kubebuilder-generate.XXXXXX)
kubebuilder alpha generate \
  --input-dir "$source_dir" \
  --output-dir "$migration_dir/scaffold"
```

Re-scaffolds the resources recorded in `PROJECT` into a separate directory.
This is a new scaffold, not a copy of custom API/controller code or local data.
Compare the relevant files with the source and integrate selected changes
manually in an isolated checkout/copy; do not replace the source directory with
the generated tree. Keep both directories until review and tests are complete.

- Needs a supported `PROJECT` file (project configuration v3+); this example uses
  the v4.16.0 CLI, not an alpha command assumed to exist in the original CLI
- Only re-scaffolds resources created via CLI and tracked in `PROJECT`
- Useful as a one-time step to reach a version that supports automated updates

See [alpha generate reference](https://master.book.kubebuilder.io/reference/commands/alpha_generate.html).

### 4. Fully Manual Migration

Create a new project with the latest Kubebuilder and port code over. Use when:

- Project was created with Kubebuilder before v3.0.0 (no `PROJECT` file)
- Scaffold is heavily customized beyond standard patterns
- APIs/controllers/webhooks were created manually (not tracked by CLI)

[AI migration helpers](https://master.book.kubebuilder.io/migration/ai-helpers.html) can
automate repetitive tasks in this process.

See [manual migration guide](https://master.book.kubebuilder.io/migration/manual-process.html).

## Comparison

| Method | Requirements | Best for |
|--------|-------------|----------|
| AutoUpdate plugin | GitHub repo, v4.5.0+ | Ongoing automated upgrades |
| `alpha update` | Supported original CLI v4.5.0+, `PROJECT`; isolated clone | Local one-off upgrades |
| `alpha generate` | Supported `PROJECT` (v3+); separate output | Scaffold comparison and manual integration |
| Manual migration | Any version | Pre-v3 projects or non-standard layouts |

## Pitfalls

### Operator not at git root (monorepo)

`kubebuilder alpha update`, `alpha generate`, and the AutoUpdate GitHub Action all
expect the `PROJECT` file at the working directory root.

When the operator lives in a subdirectory:

- **CLI**: for generation, pass the operator directory as `--input-dir` and a
  fresh external `--output-dir`. v4.16.0 has no `--project-dir` flag for these alpha
  commands. For updates, the cleanup assumes the operator is also the Git root;
  use separate generation and manual integration for a monorepo subdirectory,
  rather than running cleanup from the original repository
- **GitHub Action**: set `working-directory` in the workflow YAML
- **Makefile**: relative paths (e.g., `config/crd/bases/`) break silently if `make`
  is invoked from the wrong directory. Always run `make` from the operator root.

### Scaffold files moved or renamed

Automated migration tools rely on the standard Kubebuilder layout. Moving
scaffolded files breaks `alpha update` and `alpha generate`.

Keep the layout intact. Add custom packages alongside, not instead of the scaffold.

### Manually created APIs not tracked

Resources not scaffolded via `kubebuilder create api` or `kubebuilder create webhook`
won't appear in the `PROJECT` file and won't be regenerated by migration tools.

Prefer always using the CLI to scaffold, even for external types. The CLI has been
improved to handle various options including controllers and webhooks for external types.

### cliVersion missing in older projects

Projects created before v4.6.0 don't have `cliVersion` in `PROJECT`.
Pass a verified original release via `--from-version` in the isolated update
clone. If it is unknown or too old for update, generate into a separate output
and port custom code manually. Regeneration with the new CLI records the new
version; it cannot establish which release originally produced your custom code.
