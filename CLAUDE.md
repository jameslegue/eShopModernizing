# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Rules
- `eShopLegacyMVCSolution/` is **read-only**. Never modify anything under it (including `eShopPorted/` and `eShopLegacy.Utilities/`). It is the reference behavior for the port.
- All new code goes in `/modern`. All tests go in `/tests`.
- Work in small, single-feature changes.
- A change is not done until the golden master tests pass.
- Read `ARCHITECTURE.md` for system context (routes, data model, validation rules, porting risks) before changing behavior.

## What this repo is
Microsoft's eShopModernizing sample: hypothetical legacy .NET Framework back-office apps (product catalog CRUD over SQL Server) and their "lift and shift" Windows-container versions. Independent solutions, no shared code between them:
- `eShopLegacyMVCSolution/` — ASP.NET MVC 5 + Web API 2, .NET Framework 4.7.2 (current porting focus; see ARCHITECTURE.md)
- `eShopLegacyWebFormsSolution/`, `eShopLegacyNTier/` (WCF + WinForms) — legacy siblings
- `eShopModernized*` — containerized versions of the above; `docker-compose*.yml`, `ACI/`, `Kubernetes*/`, `ServiceFabric/`, `VM/` are deployment assets for these only
- `eShopLegacyMVCSolution/eShopPorted/` — an older partial ASP.NET Core 2.2 / EF Core 2.2 port (net461) that sits in the legacy MVC .sln; not the legacy app itself

## Build / run
Everything targets .NET Framework and needs Windows + Visual Studio/MSBuild; nothing builds or runs on macOS/Linux.
- Legacy MVC: `nuget restore eShopLegacyMVCSolution\eShopLegacyMVC.sln` then `msbuild eShopLegacyMVCSolution\src\eShopLegacyMVC\eShopLegacyMVC.csproj`; run under IIS Express (http://localhost:52429/). Needs SQL Server LocalDB unless mock mode is on.
- Modernized apps + Docker images: `build.cmd` from a VS Developer Command Prompt, then `docker-compose up` (MVC :5115, WebForms :5114, WCF :5113).
- There are no test projects and no lint configuration.

## Legacy MVC app — big picture
- Composition root is `Global.asax.cs`: Autofac container (`Modules/ApplicationModule.cs`) shared by MVC and Web API resolvers, then routes/bundles, then EF6 initializer.
- `UseMockData` (Web.config appSettings) swaps `ICatalogService` between `CatalogService` (EF6, per-request) and `CatalogServiceMock` (in-memory singleton seeded from `Models/Infrastructure/PreconfiguredData.cs`). Use mock mode to run without a database.
- DB is created on first DbContext use by `CatalogDBInitializer` (`CreateDatabaseIfNotExists`, no migrations): runs the `Models/Infrastructure/*.sql` SEQUENCE scripts, then seeds. Item IDs come from the `catalog_hilo` sequence via the singleton `CatalogItemHiLoGenerator` (blocks of 10) — IDs are never DB-generated.
- `UseCustomizationData=true` seeds from `Setup/*.csv` and wipes/re-extracts `Pics/` from `Setup/CatalogItems.zip`.
- Images are served by `PicController` at `/items/{id}/pic` from `Pics/`; views get `PictureUri` set by `CatalogController`, not stored in the DB.
- `/api/files` returns BinaryFormatter output via the `eShopLegacy.Utilities` project.
- Logging: log4net configured by an assembly attribute in `Properties/AssemblyInfo.cs` → `log4Net.xml` → `logFiles\myapp.log`.
- Package versions: the csproj `PackageReference`s are authoritative; `packages.config` is stale.
