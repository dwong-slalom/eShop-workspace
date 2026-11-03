# Project Structure

## Workspace layout

```
eShop-workspace/                    # meta repo root (orchestration only)
├── .meta                           # manifest: which child repos belong here
├── package.json                    # pins the meta tooling
├── .gitignore                      # ignores cloned child repos + node_modules
├── eShop-workspace.code-workspace  # multi-root workspace for VS Code / Kiro
├── .kiro/steering/                 # workspace-wide steering (this directory)
└── eShop/                          # child repo — cloned by meta, own git history, NOT tracked here
```

## Meta repo conventions

- `.meta` is the source of truth for child repos. Its `projects` map lists each
  repo name → clone URL.
- Child repo directories are **git-ignored** at the meta root so the meta repo
  never tracks their source. Each child has its own `.git`.
- Commit meta-level config (`.meta`, `package.json`, `.gitignore`,
  `.kiro/`, the `.code-workspace`) to the meta repo. Commit application changes
  inside the relevant child repo.

### Adding a child repo

1. Add an entry under `projects` in `.meta`.
2. Add its directory to `.gitignore`.
3. Run `npx meta git update` to clone it.

## eShop child repo structure

The `eShop/` repo is a .NET Aspire solution. Key entry points:

- `src/eShop.AppHost/eShop.AppHost.csproj` — the Aspire AppHost (startup project)
- `aspire.config.json` — selects the AppHost for `aspire run`
- `eShop.slnx` / `eShop.Web.slnf` — solution / web solution filter
- `global.json` — pins the required .NET SDK
- `src/` — service projects; `tests/` — test projects; `e2e/` — Playwright journeys

## Where work happens

- **Cross-repo orchestration, tooling, steering** → edit at the workspace root.
- **Application code, service logic, app tests** → edit inside `eShop/` (or the
  relevant child repo), and commit there.
