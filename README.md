# eShop-workspace

A **meta repo** that orchestrates the eShop child repositories. It uses
[`meta`](https://github.com/mateodelnorte/meta) to clone and run commands across
all child projects at once, while this repo itself tracks only the orchestration
config — never the child repo source.

## Layout

```
eShop-workspace/
├── .meta                       # manifest listing child projects
├── package.json                # pins the meta tooling
├── .gitignore                  # ignores cloned child repos + node_modules
├── eShop-workspace.code-workspace  # multi-root workspace for VS Code / Kiro
└── eShop/                      # cloned child repo (ignored, created by `meta git clone`)
```

The `.meta` file is the source of truth for which repos belong to the workspace:

```json
{
  "ignore": [],
  "projects": {
    "eShop": "git@github.com:dotnet/eShop.git"
  }
}
```

## Getting started

Install the meta tooling (locally pinned via `package.json`):

```bash
npm install
```

Clone every child repo declared in `.meta`:

```bash
npx meta git clone
```

> The `meta` CLI can also be installed globally with `npm i -g meta`, in which
> case you can drop the `npx` prefix.

## Common commands

| Task                                  | Command                              |
| ------------------------------------- | ------------------------------------ |
| Clone all child repos                 | `npx meta git clone`                 |
| Pull latest for all repos             | `npx meta git pull`                  |
| Update/bootstrap missing repos        | `npx meta git update`                |
| Status across all repos               | `npx meta git status`                |
| Run an arbitrary command in each repo | `npx meta exec "<command>"`          |

Example — build every child repo:

```bash
npx meta exec "dotnet build"
```

## Adding a child repo

1. Add an entry under `projects` in `.meta`.
2. Add the directory to `.gitignore` so the meta repo doesn't track its source.
3. Run `npx meta git update` to clone the newly added repo.

## Opening in your editor

Open `eShop-workspace.code-workspace` to load the meta repo and the child repos
as a multi-root workspace.

## Docs

Cross-project notes live in the meta repo (child repo source is never tracked
here), under `docs/`:

- `docs/runbooks/` — evergreen operational references (deploy / verify / tear
  down, plus hard-won gotchas).
  - [eShop on AWS ECS Fargate](docs/runbooks/eshop-aws-fargate.md)
- `docs/sessions/` — dated session journals: what was done, what broke, and why.
  - [2026-11-03 — AWS Fargate deployment](docs/sessions/2026-11-03-aws-fargate-deployment.md)
