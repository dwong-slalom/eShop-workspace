# Product

This is **eShop-workspace**, a meta repo that orchestrates the
[`dotnet/eShop`](https://github.com/dotnet/eShop) reference application and any
related child repositories.

## What eShop is

eShop ("AdventureWorks") is a reference .NET application implementing an
e-commerce website using a **services-based architecture** built on
[.NET Aspire](https://aspire.dev/). It is a sample/reference app, not a
production system — the sample catalog data, product images, and brand names are
fictional and AI-generated.

## Why this is a meta repo

The workspace root is orchestration-only. Child repositories (currently
`eShop`) are cloned in via the [`meta`](https://github.com/mateodelnorte/meta)
tool and are each independently versioned with their own git history. The meta
repo tracks the manifest (`.meta`) and shared tooling/config, not child source.

## Audience

Developers exploring, running, or extending the eShop reference app locally, and
anyone evaluating the services-based Aspire architecture.
