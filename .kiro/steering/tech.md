# Tech Stack & Commands

## Meta repo tooling

- **Orchestrator:** [`meta`](https://github.com/mateodelnorte/meta) (pinned in root `package.json`)
- **Node:** required only to run the `meta` CLI and the eShop e2e tests

## eShop child repo stack

- **.NET 10 SDK** — must satisfy `eShop/global.json`
- **.NET Aspire** — the app model / orchestration layer (requires the Aspire CLI)
- **Container runtime** — Docker Desktop (recommended) or Podman, must be running before starting eShop
- **Playwright + Node** — browser-based end-to-end tests
- **Optional:** .NET MAUI workload (client apps), Azure CLI + Microsoft Foundry (AI chatbot), Azure Container Apps (deploy)

## Commands

> Run `meta`/Node commands from the **workspace root**. Run `aspire`/`dotnet`
> commands from inside the **`eShop/`** child repo unless noted otherwise.

### Meta repo (workspace root)

```bash
npm install            # install the pinned meta tooling
npx meta git clone     # clone all child repos declared in .meta
npx meta git update    # clone any missing child repos
npx meta git pull      # pull latest for every child repo
npx meta git status    # git status across all repos
npx meta exec "<cmd>"  # run <cmd> in every child repo
```

### Run eShop (from `eShop/`)

```bash
aspire run             # run the AppHost; prints a dashboard URL. Ctrl+C to stop
aspire start           # run the AppHost in the background
aspire ps              # list background AppHost processes
aspire stop            # stop the background AppHost
```

> The container runtime must be running before starting eShop. The root
> `aspire.config.json` selects `src/eShop.AppHost/eShop.AppHost.csproj`.

### Test eShop (from `eShop/`)

```bash
dotnet test --solution eShop.Web.slnf   # server tests

npm ci                                  # e2e setup
npx playwright install chromium
npm run test:e2e                        # Playwright browser journeys (starts AppHost; needs a running container runtime)
```

### Deploy eShop to Azure Container Apps (from `eShop/`)

```bash
az login
aspire deploy --list-steps   # preview the deployment pipeline
aspire deploy                # deploy
aspire destroy               # tear down the configured resource group (destructive)
```

> `aspire destroy` deletes the entire configured resource group, including
> resources Aspire did not create. Review the target before confirming.
