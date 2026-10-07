# eShopLegacyMVC — Architecture

This document describes the legacy ASP.NET MVC catalog app as it exists today. It is the reference for porting the app to ASP.NET Core on .NET 10.

- **In scope:** `eShopLegacyMVCSolution/src/eShopLegacyMVC` and the `eShopLegacyMVCSolution/eShopLegacy.Utilities` library it references.
- **Out of scope:** the WebForms, WCF/NTier, and `eShopModernized*` solutions.
- **Paths:** unless a path starts with `eShopLegacyMVCSolution/`, it is relative to `eShopLegacyMVCSolution/src/eShopLegacyMVC/`.
- **How this was produced:** by reading the source, config, views, and seed data. The app was **not run**, because it needs Windows, IIS Express, and SQL Server LocalDB. Anything that depends on runtime behavior is marked **⚠ Unverified** and listed again in [§11](#11-open-questions--unverified).

---

## 1. Overview

The app is called "Catalog manager (MVC)". It is an internal back-office CRUD app that lets staff list, view, create, edit, and delete products in an eShop catalog stored in SQL Server. It also exposes a few demo API endpoints.

What the app does **not** have:
- No authentication or authorization of any kind.
- No customer-facing storefront, basket, or orders.
- No image upload.

### Sibling project: `eShopPorted`

`eShopLegacyMVCSolution/eShopLegacyMVC.sln` also contains `eShopPorted/`. This is an earlier, partial port of the same app to ASP.NET Core 2.2 and EF Core 2.2. It still targets `net461`, so it is not a .NET Core app. This document does not analyze it. It may be useful as a reference for how someone approached the port before, but it is not the target.

---

## 2. Tech stack & versions

Versions come from `eShopLegacyMVC.csproj`. The csproj uses `PackageReference`, but a stale `packages.config` and some `HintPath` references are still present. Where the two disagree, the csproj wins, and the table notes the disagreement.

| Area | Component | Version | Notes |
|---|---|---|---|
| Runtime | .NET Framework | 4.7.2 | `Web.config` has `compilation targetFramework="4.7.2"` but `httpRuntime targetFramework="4.6.1"`. |
| Web | ASP.NET MVC | 5.2.7 | |
| Web | Razor / WebPages | 3.2.7 | |
| Web | ASP.NET Web API 2 | 5.2.7 | `WebApi`, `.Core`, `.WebHost`, `.Client` |
| Web | System.Web.Optimization (bundling) | 1.1.3 | WebGrease 1.6.0, Antlr 3.5.0.2 |
| Web | SessionStateModule (async) | 1.1.0 | Replaces the built-in session module in `Web.config`. |
| Data | Entity Framework | 6.2.0 | SqlClient provider. LocalDB connection factory. |
| DI | Autofac | 6.1.0 | `packages.config` says 4.9.1. |
| DI | Autofac.Mvc5 | 4.0.2 | |
| DI | Autofac.WebApi2 | 6.0.1 | Not listed in `packages.config`. |
| Logging | log4net | 2.0.10 | |
| Telemetry | Microsoft.ApplicationInsights (+ Web, DependencyCollector, PerfCounterCollector, WindowsServer, TelemetryChannel) | 2.9.1 | Agent.Intercept 2.4.0 |
| Telemetry | Microsoft.AspNet.TelemetryCorrelation | 1.0.5 | |
| Serialization | Newtonsoft.Json | 12.0.1 | |
| Front end | Bootstrap | 4.3.1 | The views use Bootstrap 3 class names. See [§10](#10-porting-risks-for-net-10--aspnet-core). |
| Front end | jQuery | 3.5.0 (package) | Only `Scripts/jquery-3.3.1*.js` is on disk, so the bundle serves 3.3.1. |
| Front end | jQuery.Validation / Unobtrusive | 1.19.4 / 3.2.11 | |
| Front end | Modernizr / Respond / popper.js | 2.8.3 / 1.4.2 / 1.14.3 | |
| Build | Microsoft.Net.Compilers / CodeDom.Providers.DotNetCompilerPlatform | 2.10.0 / 2.0.1 | Roslyn for runtime view compilation. |
| Library | `eShopLegacy.Utilities` (project reference) | targets net461 | Contains a single class, `Serializing`, which uses BinaryFormatter. |

Packages that are referenced but never used in the code: Pipelines.Sockets.Unofficial, System.IO.Pipelines, System.Threading.Channels, System.Memory, System.Buffers. These look like leftover transitive dependencies (for example, from an earlier Redis client). **⚠ Unverified.**

The solution root also contains `Convert-ToPackageReference.ps1` and `.xsl`, which are the tools used for the partial migration from `packages.config`.

---

## 3. Project structure

```
eShopLegacyMVCSolution/
├── eShopLegacyMVC.sln             # 3 projects: eShopLegacyMVC, eShopLegacy.Utilities, eShopPorted
├── eShopLegacy.Utilities/         # Serializing.cs (BinaryFormatter wrapper)
├── eShopPorted/                   # earlier partial ASP.NET Core 2.2 port (out of scope)
└── src/eShopLegacyMVC/
    ├── App_Start/                 # BundleConfig, FilterConfig, RouteConfig, WebApiConfig
    ├── Controllers/               # CatalogController, PicController (MVC)
    │   ├── Api/                   # CatalogController2 — MVC controller at /api (demo)
    │   └── WebApi/                # BrandsController, FilesController (Web API 2)
    ├── Models/                    # entities, CatalogDBContext, CatalogItemHiLoGenerator
    │   └── Infrastructure/        # CatalogDBInitializer, PreconfiguredData, *.Sequence.sql
    ├── Modules/                   # ApplicationModule (Autofac registrations)
    ├── Services/                  # ICatalogService, CatalogService (EF), CatalogServiceMock
    ├── ViewModel/                 # PaginatedItemsViewModel<T>
    ├── Views/                     # Catalog/*, Shared/_Layout + Error, Web.config
    ├── Setup/                     # CSVs + CatalogItems.zip for "customization" seeding
    ├── Pics/                      # product images 1.png–12.png, dummy.png (served by PicController)
    ├── Images/                    # site chrome (brand logos, banner, footer text)
    ├── Content/ Scripts/          # CSS / JS consumed by bundles
    ├── Global.asax(.cs)           # application + session lifecycle
    ├── Web.config (+ Debug/Release transforms), log4Net.xml, ApplicationInsights.config
```

### Startup and request lifecycle (`Global.asax.cs`)

`Application_Start` runs these steps in order:
1. `RegisterContainer()` builds the Autofac container:
   - Registers all MVC controllers and Web API controllers from the assembly.
   - Registers `ApplicationModule(UseMockData)`.
   - Sets both `DependencyResolver` (MVC) and `GlobalConfiguration.Configuration.DependencyResolver` (Web API) to the same container.
2. `GlobalConfiguration.Configure(WebApiConfig.Register)`.
3. `AreaRegistration.RegisterAllAreas()`. The app has no areas.
4. `FilterConfig`: a global `HandleErrorAttribute`.
5. `RouteConfig`: MVC attribute routes, then the default conventional route.
6. `BundleConfig`.
7. `ConfigDataBase()`: when `UseMockData` is false, calls `Database.SetInitializer(container.Resolve<CatalogDBInitializer>())`.

The other lifecycle handlers:
- `Session_Start` stores `Session["MachineName"] = Environment.MachineName` and `Session["SessionStartTime"] = DateTime.Now`. The layout footer displays both.
- `Application_BeginRequest` puts two lazily evaluated objects into log4net's `LogicalThreadContext.Properties`:
  - `activityid` (`ActivityIdHelper`), which creates or reads `Trace.CorrelationManager.ActivityId`.
  - `requestinfo` (`WebRequestInfo`), which returns the raw URL and the user agent.

  It then writes a debug log line.

---

## 4. Data model & database access

### Entities

The schema is defined by both data annotations (`Models/*.cs`) and fluent configuration (`CatalogDBContext.OnModelCreating`).

**`CatalogItem` → table `Catalog`**

| Property | Type | DB constraint (fluent) | Model annotation |
|---|---|---|---|
| `Id` | int | PK, `DatabaseGeneratedOption.None` | — |
| `Name` | string | required, max 50 | `[Required]` |
| `Description` | string | none (EF6 convention → `nvarchar(max)`) | — |
| `Price` | decimal | required (EF6 default `decimal(18,2)`) | regex, `[Range(0, 1000000)]`, `[DataType(Currency)]` |
| `PictureFileName` | string | required | `[Display(Name="Picture name")]`. The constructor defaults it to `"dummy.png"`. |
| `PictureUri` | string | **ignored** (not mapped) | Computed per request by the controller. |
| `CatalogTypeId` → `CatalogType` | int / nav | required FK | `[Display(Name="Type")]` |
| `CatalogBrandId` → `CatalogBrand` | int / nav | required FK | `[Display(Name="Brand")]` |
| `AvailableStock` | int | — | `[Range(0, 10000000)]`, "Stock" |
| `RestockThreshold` | int | — | `[Range(0, 10000000)]`, "Restock" |
| `MaxStockThreshold` | int | — | `[Range(0, 10000000)]`, "Max stock" |
| `OnReorder` | bool | — | — |

**`CatalogBrand` → table `CatalogBrand`**: `Id` (PK), `Brand` (required, max 100).
**`CatalogType` → table `CatalogType`**: `Id` (PK), `Type` (required, max 100).

The relationships from brand and type to item are one-to-many. The navigation property exists only on `CatalogItem` (`.WithMany()`).

### Connection

Connection string `CatalogDBContext` in `Web.config`:

```
Data Source=(localdb)\MSSQLLocalDB; Initial Catalog=Microsoft.eShopOnContainers.Services.CatalogDb;
Integrated Security=True; MultipleActiveResultSets=True;
```

`CatalogDBContext` uses `base("name=CatalogDBContext")`.

### Schema creation and seeding

The app has **no migrations**. `Models/Infrastructure/CatalogDBInitializer.cs` derives from `CreateDatabaseIfNotExists<CatalogDBContext>`. EF6 runs it lazily, the first time a `CatalogDBContext` is used, not at startup. If the database already exists, nothing is created or seeded.

`Seed` runs these steps:
1. Executes three SQL scripts. Each creates a `bigint` SEQUENCE with `START WITH 1 INCREMENT BY 10`.
   - The sequences are `catalog_hilo`, `catalog_brand_hilo`, and `catalog_type_hilo`.
   - The scripts are loaded from `AppDomain.CurrentDomain.BaseDirectory` using Windows-style relative paths (`Models\Infrastructure\…sql`).
   - Each script starts with a hardcoded `USE [Microsoft.eShopOnContainers.Services.CatalogDb]`.
2. Inserts types. It calls `NEXT VALUE FOR catalog_type_hilo` once and assigns consecutive Ids starting from that value.
3. Inserts brands the same way, using `catalog_brand_hilo`.
4. Inserts items, getting each Id from `CatalogItemHiLoGenerator`.
5. If customization data is on, refreshes the pictures (see below).

**Default seed data** comes from `Models/Infrastructure/PreconfiguredData.cs`:
- 4 types: Mug, T-Shirt, Sheet, USB Memory Stick.
- 5 brands: Azure, .NET, Visual Studio, SQL Server, Other.
- 12 items: `1.png`–`12.png`, `AvailableStock = 100`.

The hardcoded brand and type Ids (1–5, 1–4) in the default items line up with the sequence-assigned Ids only because the sequences start at 1.

**Customization seed** is used when `UseCustomizationData=true`:
- Types, brands, and items are read from `Setup/CatalogTypes.csv`, `Setup/CatalogBrands.csv`, and `Setup/CatalogItems.csv`. If a file is missing, that data falls back to `PreconfiguredData`.
- Headers are lowercased and validated:
  - Items require `catalogtypename`, `catalogbrandname`, `description`, `name`, `price`, and `picturefilename`.
  - Items optionally accept `availablestock`, `restockthreshold`, `maxstockthreshold`, and `onreorder`.
- Rows are split with the regex `,(?=(?:[^"]*"[^"]*")*[^"]*$)`. Values are trimmed and stripped of quotes.
- Price is parsed with `InvariantCulture`.
- Items find their brand and type by **name**.
- Any bad row throws, which aborts the seed.
- `AddCatalogItemPictures()` **deletes every file in `Pics/`** and then extracts `Setup/CatalogItems.zip` (13 PNGs, including `dummy.png`) into it.
- The CSVs as checked in contain test rows: `CatalogBrandTestOne`, `CatalogBrandTestTwo`, `CatalogTypeTestOne`, `CatalogTypeTestTwo`, and an item named "pepito".

### HiLo Id generation

`Models/CatalogItemHiLoGenerator.cs` is an Autofac **singleton**:
- It calls `SELECT NEXT VALUE FOR catalog_hilo` once to get the "hi" value.
- It then returns that value and the next 9 integers from memory, under a `lock`.
- The block size of 10 is hardcoded to match the sequence's `INCREMENT BY 10`.

Both the seed and `CatalogService.CreateCatalogItem` use it. Each app restart throws away the unused part of the current block, so gaps in Ids are expected. Only items use it at runtime; the brand and type sequences are used only during seeding.

### Service layer

`Services/ICatalogService.cs` (extends `IDisposable`):

```csharp
CatalogItem FindCatalogItem(int id);
IEnumerable<CatalogBrand> GetCatalogBrands();
PaginatedItemsViewModel<CatalogItem> GetCatalogItemsPaginated(int pageSize, int pageIndex);
IEnumerable<CatalogType> GetCatalogTypes();
void CreateCatalogItem(CatalogItem catalogItem);
void UpdateCatalogItem(CatalogItem catalogItem);
void RemoveCatalogItem(CatalogItem catalogItem);
```

`Services/CatalogService.cs` is the EF6 implementation:
- **Paginated:** runs `LongCount()` for the total, then `Include(Brand).Include(Type).OrderBy(Id).Skip(size*index).Take(size)`.
- **Find:** `Include` brand and type, then `FirstOrDefault`.
- **Brands/types:** returns the `DbSet` directly, unmaterialized.
- **Create:** gets the Id from HiLo, then `Add` and `SaveChanges`.
- **Update:** `Entry(item).State = EntityState.Modified`, then `SaveChanges`. This **overwrites every column** with whatever was posted.
- **Remove:** `Remove` and `SaveChanges`.

`ViewModel/PaginatedItemsViewModel<T>` holds `ActualPage`, `ItemsPerPage`, `TotalItems`, `TotalPages = ceil(count / pageSize)`, and `Data`.

### DI registrations (`Modules/ApplicationModule.cs`)

| Service | Implementation | Lifetime |
|---|---|---|
| `ICatalogService` | `CatalogServiceMock` (mock mode) | SingleInstance |
| `ICatalogService` | `CatalogService` (DB mode) | InstancePerLifetimeScope (per request) |
| `CatalogDBContext` | self | InstancePerLifetimeScope |
| `CatalogDBInitializer` | self | InstancePerLifetimeScope (resolved once at startup from the root container) |
| `CatalogItemHiLoGenerator` | self | SingleInstance |

`CatalogController.Dispose` also calls `service.Dispose()`, which disposes the request's DbContext. Autofac disposes it again at the end of the request scope. This is harmless.

---

## 5. Mock-data mode

Setting `UseMockData=true` in `Web.config` has three effects:
- It registers `Services/CatalogServiceMock.cs` as a **singleton** `ICatalogService`.
- It skips `Database.SetInitializer`.
- No database is touched. `CatalogDBContext` is still registered but nothing resolves it.

How the mock behaves:
- **Data:** an in-memory `List<CatalogItem>` copied from `PreconfiguredData` (12 items). Brands and types always come straight from `PreconfiguredData`.
- **Lifetime:** shared by every user and lost on restart.
- **Thread safety:** the list is mutated without locking, so it is not thread-safe.
- **Create:** `Id = max(Id) + 1`.
- **Update:** replaces the stored object with the posted object.
- **Remove:** removes the item by reference.
- **Paginated:** same ordering and paging as the EF version. It first fills in `CatalogBrand` and `CatalogType` on **every item in the shared list**, which mutates it.
- **Navigation properties:** `FindCatalogItem` returns the stored object as-is. Brand and type are only filled in after an Index request has run since the item was created or last edited. Create and Edit both redirect to Index, so in normal use this is hard to notice. A direct link to Details or Edit right after startup would show a blank brand and type. **⚠ Unverified.**
- **Pictures:** `PicController` still reads from `Pics/` on disk.

---

## 6. Pages & routes

### Controller inventory

The csproj compiles exactly five controllers:

| Controller | File | Kind |
|---|---|---|
| `CatalogController` | `Controllers/CatalogController.cs` | MVC |
| `PicController` | `Controllers/PicController.cs` | MVC (attribute-routed) |
| `CatalogController2` | `Controllers/Api/CatalogController.cs` (file name differs from class) | MVC (attribute-routed) |
| `BrandsController` | `Controllers/WebApi/BrandsController.cs` | Web API 2 `ApiController` |
| `FilesController` | `Controllers/WebApi/FilesController.cs` | Web API 2 `ApiController` |

There is **no `AccountController`**. A search found no account, login, `[Authorize]`, Identity, or OWIN code in the controllers, views, or config. Any login feature in the port would be new work.

### Routing configuration

- MVC (`App_Start/RouteConfig.cs`): `MapMvcAttributeRoutes()`, then ignore `{resource}.axd/{*pathInfo}`, then the default route `{controller}/{action}/{id}` with defaults `Catalog` / `Index` / optional `id`.
- Web API (`App_Start/WebApiConfig.cs`): `MapHttpAttributeRoutes()`, then `api/{controller}/{id}` with optional `id`. HTTP verbs are matched by method name (`Get`, `Delete`).

### Routes

| Method | URL | Action | Behavior | View |
|---|---|---|---|---|
| GET | `/`, `/Catalog`, `/Catalog/Index?pageSize=10&pageIndex=0` | `CatalogController.Index` | Paged list ordered by Id. Defaults are `pageSize=10` and `pageIndex=0`. Sets `PictureUri` on each item. | `Catalog/Index.cshtml` + partial `CatalogTable.cshtml` |
| GET | `/Catalog/Details/{id}` | `Details` | Returns 400 if `id` is missing and 404 if not found. | `Catalog/Details.cshtml` |
| GET | `/Catalog/Create` | `Create` | An empty `CatalogItem` (picture `dummy.png`). Brand and type dropdowns come from `ViewBag.CatalogBrandId` and `ViewBag.CatalogTypeId` `SelectList`s. | `Catalog/Create.cshtml` |
| POST | `/Catalog/Create` | `Create` | `[ValidateAntiForgeryToken]` and `[Bind(Include="Id,Name,Description,Price,PictureFileName,CatalogTypeId,CatalogBrandId,AvailableStock,RestockThreshold,MaxStockThreshold,OnReorder")]`. If valid, creates the item and redirects to Index. Otherwise re-renders the form with the dropdowns. | same |
| GET | `/Catalog/Edit/{id}` | `Edit` | Returns 400 or 404 as above. Shows the form with the picture, and `PictureFileName` is rendered `readonly`. | `Catalog/Edit.cshtml` |
| POST | `/Catalog/Edit/{id}` | `Edit` | Same antiforgery token and Bind list as Create. If valid, updates the item and redirects to Index. | same |
| GET | `/Catalog/Delete/{id}` | `Delete` | Returns 400 or 404 as above. Shows a confirmation page. | `Catalog/Delete.cshtml` |
| POST | `/Catalog/Delete/{id}` | `DeleteConfirmed` (`[ActionName("Delete")]`) | Antiforgery token required. Find, then remove, then redirect to Index. | — |
| GET | `/items/{catalogItemId:int}/pic` (route name `GetPicRouteTemplate`) | `PicController.Index` | Returns 400 if id ≤ 0 and 404 if the item is not found. Otherwise reads `~/Pics/{PictureFileName}` and returns the bytes with a MIME type based on the extension (png, gif, jpg/jpeg, bmp, tiff, wmf, jp2, svg; anything else is `application/octet-stream`). | — |
| GET | `/api` | `CatalogController2.Index` | Returns `Json(new { Message = "Hello World!" })` **without** `JsonRequestBehavior.AllowGet`, so MVC 5 most likely throws `InvalidOperationException` on GET. **⚠ Unverified.** | — |
| GET | `/api/brands` | `BrandsController.Get()` | All brands as JSON or XML, depending on content negotiation. | — |
| GET | `/api/brands/{id}` | `BrandsController.Get(id)` | 200 with the brand, or 404. | — |
| DELETE | `/api/brands/{id}` | `BrandsController.Delete` | Returns 404 if the brand is unknown. Otherwise returns **200 without deleting anything** (a code comment says "demo only"). | — |
| GET | `/api/files` | `FilesController.Get` | Projects brands to `[Serializable] BrandDTO { Id, Brand }` and returns a `List<BrandDTO>` **serialized with BinaryFormatter** (`eShopLegacy.Utilities.Serializing.SerializeBinary`). No content type is set. | — |

`PictureUri` is built in `CatalogController.AddUriPlaceHolder` with `Url.RouteUrl("GetPicRouteTemplate", { catalogItemId }, Request.Url.Scheme)`, which produces an **absolute** URL.

### Shared views

- **`Views/Shared/_Layout.cshtml`:**
  - Header: brand logo linking to Catalog/Index, and the hero title "Catalog manager (MVC)".
  - Footer: images, plus `"{Session[MachineName]}, {Session[SessionStartTime]}"`, read through `HttpContext.Current.Session`.
  - Renders the bundles `~/Content/css` and `~/bundles/modernizr` in the head, and `~/bundles/jquery` and `~/bundles/bootstrap` at the end of the body.
  - Has an optional `scripts` section. Create and Edit use it to add `~/bundles/jqueryval`.
- **`Views/Shared/Error.cshtml`:** a static "An error occurred" page used by the global `HandleErrorAttribute`. No `<customErrors>` element is configured, so ASP.NET's default (`RemoteOnly`) applies: remote users see this page, and local requests see the detailed error page.
- **Index pager:**
  - Shows "Previous" and "Next" links. On the first and last page they are still rendered but hidden with the CSS class `esh-pager-item--hidden`. On the first page, the hidden Previous link points to `pageIndex=-1`.
  - The text reads "Showing {ItemsPerPage} of {TotalItems} products - Page {n} - {TotalPages}".

---

## 7. Business rules & validation

### Enforced rules

These rules are enforced both server-side (`ModelState`) and client-side (jQuery unobtrusive validation; `ClientValidationEnabled` and `UnobtrusiveJavaScriptEnabled` are true).

| Field | Rule | Message |
|---|---|---|
| Name | `[Required]` | default |
| Price | `^\d+(\.\d{0,2})*$` | "The field Price must be a positive number with maximum two decimals." |
| Price | `[Range(0, 1000000)]` | default |
| Stock | `[Range(0, 10000000)]` | "The field Stock must be between 0 and 10 million." |
| Restock | `[Range(0, 10000000)]` | "The field Restock must be between 0 and 10 million." |
| Max stock | `[Range(0, 10000000)]` | "The field Max stock must be between 0 and 10 million." |
| Brand / Type | non-nullable `int`, so implicitly required | default |
| Form posts | `[ValidateAntiForgeryToken]` on Create, Edit, and Delete | — |

Other rules:
- Culture is pinned to `en-US` by `<globalization>` in `Web.config`. This affects how decimals are parsed and how `DataType.Currency` is displayed (`$`).
- **Pictures cannot be changed.**
  - Create shows "Uploading images not allowed for this version." The item keeps the default `dummy.png`, because `PictureFileName` is not posted and keeps its constructor value.
  - Edit renders `PictureFileName` as a readonly input. It is still posted and bound.

### Not enforced (gaps, not rules)

Don't port these as intended behavior without deciding first.
- Nothing checks that `RestockThreshold ≤ MaxStockThreshold` or that `AvailableStock ≤ MaxStockThreshold`.
- The 50-character limit on `Name` exists only in the EF fluent configuration. The model has no `[StringLength]`, so a longer name passes `ModelState` and should fail later in EF6's `SaveChanges` validation (`DbEntityValidationException`), which shows the error page. **⚠ Unverified.** In mock mode, any length is accepted.
- The `*` in the Price regex allows repeated groups like `1.2.3` to pass the regex. Model binding to `decimal` should still reject such a value.
- `pageSize` and `pageIndex` are not validated:
  - `pageSize=0` divides a decimal by zero in `PaginatedItemsViewModel` → `DivideByZeroException`.
  - A negative `pageIndex` passes a negative `Skip` to EF6 / LINQ. **⚠ Unverified** what happens.
- `DeleteConfirmed` does not null-check the item. Posting the delete form for an Id that no longer exists passes `null` to `Remove`.
- `PictureFileName` on Edit is readonly only in the browser. A tampered post can store any value, including `..\` paths. `PicController` then calls `Path.Combine(Server.MapPath("~/Pics"), PictureFileName)` and returns that file. See [§10](#10-porting-risks-for-net-10--aspnet-core).

### Quirks the port must decide whether to keep

- **Editing resets `OnReorder` to false.** `OnReorder` is in the `[Bind]` list but neither form has an input for it. It always binds as `false`, and `EntityState.Modified` writes every column. In mock mode, the whole object is replaced, with the same result.
- The pager's "Showing X of Y" shows the **page size**, not the number of items actually shown.
- When a CSV item references an unknown brand, the error message wrongly says "type=… does not exist in catalogTypes". The variable names in the int-parse error messages are also wrong.

---

## 8. Configuration

### `Web.config` `appSettings`

| Key | Value | Read by | Purpose |
|---|---|---|---|
| `UseMockData` | `false` | `Global.asax.cs` (`bool.Parse`, twice) | Chooses between the in-memory mock and EF/SQL. |
| `UseCustomizationData` | `false` | `CatalogDBInitializer` ctor (`bool.Parse`) | Chooses between CSV/zip seed data and the built-in seed. |
| `ClientValidationEnabled` | `true` | MVC | Client-side validation. |
| `UnobtrusiveJavaScriptEnabled` | `true` | MVC | Unobtrusive validation. |
| `webpages:Version` / `webpages:Enabled` | `3.0.0.0` / `false` | WebPages | Standard MVC settings. |

If either custom key is missing, `bool.Parse(null)` throws at startup.

### Other `Web.config` sections

- **`connectionStrings`:** `CatalogDBContext` (see [§4](#connection)), `providerName="System.Data.SqlClient"`.
- **`system.web`:**
  - `compilation debug="true"`. `Web.Release.config` removes `debug`.
  - `sessionState mode="InProc"`.
  - `globalization culture/uiCulture="en-US"`.
  - `httpModules`: TelemetryCorrelation and ApplicationInsights.
- **`system.webServer`:**
  - Modules: TelemetryCorrelation, ApplicationInsights, and `Session` replaced with `Microsoft.AspNet.SessionState.SessionStateModuleAsync`.
  - Extensionless URL handlers.
- **`entityFramework`:** `LocalDbConnectionFactory (mssqllocaldb)` and the `SqlProviderServices` provider.
- **`runtime/assemblyBinding`:** binding redirects for Newtonsoft.Json, Optimization, WebGrease, Autofac, Antlr, DiagnosticSource, Helpers, WebPages, Mvc, and Http.
- **`system.codedom`:** the Roslyn compilers used to compile views at runtime.
- **`Views/Web.config`:**
  - The Razor host factory references MVC **5.2.3.0**; the binding redirect maps it to 5.2.7.
  - Default view namespaces: `System.Web.Mvc*`, `System.Web.Optimization`, `System.Web.Routing`, `eShopLegacyMVC`.
  - A `BlockViewHandler` prevents serving `.cshtml` files directly.
- **Transforms:** `Web.Debug.config` contains only comments. `Web.Release.config` only removes `debug`.

### Logging (`log4Net.xml`)

- Loaded by `[assembly: log4net.Config.XmlConfigurator(ConfigFile = "log4net.xml")]` in `Properties/AssemblyInfo.cs`. The file on disk is named `log4Net.xml` with a capital N; this works on Windows because the file system is case-insensitive.
- The root level is `ALL`, with a `RollingFileAppender` writing to `logFiles\myapp.log` (size-based, 10 MB × 5 backups).
- The pattern is `%date [%thread] %property{activity} %level %logger - %property{requestinfo}%newline%message…`. The code sets `activityid`, but the pattern reads `activity`, so **the activity id is never written to the log.**
- Controllers log `Info` on every action, with the route and parameters.

### Application Insights (`ApplicationInsights.config`)

- Uses the standard 2.9 module and initializer set, with adaptive sampling at 5 items/sec.
- **No `InstrumentationKey`** is configured, so telemetry is probably not sent anywhere. **⚠ Unverified.**

### Dev hosting

IIS Express at `http://localhost:52429/` (csproj `IISUrl`), with development server port 53134.

---

## 9. External dependencies

| Dependency | How it is used | Required? |
|---|---|---|
| SQL Server / LocalDB | EF6 data store. Needs `CREATE SEQUENCE` support (SQL Server 2012+). Integrated Security. | Only when `UseMockData=false`. |
| Local file system: `Pics/` | Read on every image request. Deleted and rewritten during customization seeding. | Yes. |
| Local file system: `logFiles/` | log4net output. | Yes; the folder must be writable. |
| Local file system: `Models/Infrastructure/*.sql`, `Setup/*` | Read during seeding. | When seeding. |
| Application Insights | Telemetry via HTTP modules. | Optional; no key is configured. |
| Session state (in-process) | Footer machine name and session start time. | Yes, for the layout. |

No outbound HTTP calls, message queues, caches, email, or external auth providers were found.

---

## 10. Porting risks for .NET 10 / ASP.NET Core

The risks are ranked roughly by how much effort they take and how much they could break.

1. **BinaryFormatter (`/api/files`, `eShopLegacy.Utilities`).**
   - .NET 9 removed the BinaryFormatter implementation, and it throws at runtime. The endpoint cannot be ported as is.
   - The wire format has to change (for example to JSON), and that breaks any client that expects BinaryFormatter output.
   - Nothing in this repository calls the endpoint, so **the consumers are unknown**. Find out who calls it before choosing a replacement.

2. **EF6 → EF Core.**
   - `CreateDatabaseIfNotExists` and `Seed` have no direct equivalent. Use migrations plus `UseSeeding`/`HasData`, or an explicit startup seeder.
   - Existing databases already contain the tables and sequences, so the first migration needs a baseline (an empty migration applied against the existing schema).
   - The HiLo scheme maps to EF Core's built-in `UseHiLo("catalog_hilo")`. The increment of 10 and the existing sequence must stay. Remove the hand-written generator and the raw `Database.SqlQuery<Int64>` calls. Brand and type Ids come from the other two sequences only during seeding.
   - `EntityState.Modified` updates every column. Decide whether to keep that, including the `OnReorder` reset, or switch to field-level updates.
   - EF6 validates `MaxLength` on `SaveChanges`. EF Core does not, so a name over 50 characters would become a SQL truncation error instead. Add `[StringLength(50)]` on the model.
   - The fluent API needs translating: `EntityTypeConfiguration`, `HasRequired().WithMany().HasForeignKey()`, `HasDatabaseGeneratedOption`, and `Ignore`.
   - `GetCatalogBrands()`/`GetCatalogTypes()` return a live `DbSet` that is enumerated in the view. Materialize them in the port.

3. **System.Web coupling.**
   - `HttpContext.Current` is used in `_Layout.cshtml` (Session) and in `WebRequestInfo`.
   - `Server.MapPath("~/Pics")`, `HostingEnvironment.ApplicationPhysicalPath`, and `AppDomain.CurrentDomain.BaseDirectory` are used for file paths. Replace them with `IWebHostEnvironment` (WebRootPath and ContentRootPath).
   - `Request.Url.Scheme` is used for absolute picture URLs.
   - `Global.asax` events become middleware. **ASP.NET Core has no `Session_Start` event.** The footer's session start time needs a "first time this session is seen" check, or a decision to drop it.
   - Session requires `AddSession` plus a distributed cache.

4. **Two pipelines (MVC 5 + Web API 2) merge into one ASP.NET Core MVC.**
   - `ApiController`, `IHttpActionResult`, `HttpResponseMessage`, and `ResponseMessage(...)` become `ControllerBase`/`IActionResult`.
   - Web API picks the action by HTTP verb from the method name (`Get`, `Delete`). ASP.NET Core needs explicit `[HttpGet]`/`[HttpDelete]` and routes.
   - Web API 2 content negotiation can return XML. ASP.NET Core returns JSON only unless XML is turned on. Decide which `/api/brands` should return.
   - `[Bind(Include="…")]` becomes `[Bind("…")]`. `HttpStatusCodeResult`/`HttpNotFound()` become `StatusCode()`/`NotFound()`.
   - In ASP.NET Core, `Json()` allows GET, so the broken `/api` endpoint would start working. Decide whether to keep it.
   - The new route table needs checking for conflicts between `/api`, `/api/{controller}/{id}`, and the default route.

5. **Bundling and minification (`System.Web.Optimization`).**
   - There is no built-in equivalent. Serve static files from `wwwroot` (or use LibMan or a build step) and replace `@Styles.Render`/`@Scripts.Render`.
   - The `{version}` and `*` wildcards in `BundleConfig` currently resolve to jQuery 3.3.1, Modernizr 2.6.2 **and** 2.8.3 (both match `modernizr-*`), and every `jquery.validate*` file. **⚠ Unverified** which files actually get emitted.

6. **Razor views.**
   - `Html.Partial` should become `<partial>` or `PartialAsync`.
   - `@Html.DropDownList("CatalogBrandId", null, …)` relies on looking up a `SelectList` in ViewBag with the same name. ASP.NET Core supports this too, but check it. Using typed `asp-items` is cleaner.
   - The namespaces in `Views/Web.config` move to `_ViewImports.cshtml`. `@Html.AntiForgeryToken()` becomes automatic with form tag helpers.

7. **Web.config and configuration.** Each section needs a home:
   - `ConfigurationManager.AppSettings` + `bool.Parse` (in `Global.asax.cs` and `CatalogDBInitializer`) → `IConfiguration`/options (`UseMockData`, `UseCustomizationData`).
   - `connectionStrings` and EF `providerName` → `ConnectionStrings` in appsettings plus `UseSqlServer`.
   - `sessionState InProc` and SessionStateModuleAsync → `AddSession` + `AddDistributedMemoryCache`.
   - The AI and TelemetryCorrelation `httpModules` → SDK or OpenTelemetry registration.
   - `globalization en-US` → `RequestLocalizationOptions` with en-US as the default. Without it, Price parsing and `$` formatting depend on the server's culture.
   - The extensionless handlers and `Views/Web.config` → removed, or moved to `_ViewImports`.
   - `assemblyBinding` redirects and `system.codedom` → removed (no longer needed).
   - XML transforms (`Web.Debug/Release.config`) → `appsettings.{Environment}.json`.

8. **Dependency injection (Autofac).**
   - Either keep Autofac through `Autofac.Extensions.DependencyInjection`, or move the five registrations to the built-in container. The module is small, so either works.
   - The lifetimes must stay the same: the mock is a **singleton** (state shared across requests), and the EF service and DbContext are per request (`AddScoped`).
   - The HiLo generator is a singleton that holds a lock.
   - The initializer is resolved from the root container at startup.
   - The manual `service.Dispose()` in `CatalogController.Dispose` should go away, because the container owns disposal.

9. **Logging and telemetry.**
   - log4net's `LogicalThreadContext` properties and `Trace.CorrelationManager.ActivityId` → `ILogger` scopes and `System.Diagnostics.Activity`. The broken `activity` vs `activityid` pattern is a chance to fix the correlation id.
   - App Insights 2.x classic modules and `ApplicationInsights.config` → Azure Monitor OpenTelemetry or the AI ASP.NET Core SDK.
   - Decide whether log files are still needed, given that Info is logged on every action.

10. **Platform and case-sensitivity assumptions.** The app has only ever run on Windows. Things that break on Linux or in containers:
    - Backslash paths: `Models\Infrastructure\*.sql` and `logFiles\myapp.log`.
    - LocalDB (Windows-only) and Integrated Security.
    - The hardcoded `USE [Microsoft.eShopOnContainers.Services.CatalogDb]` in the sequence scripts, which breaks if the database name changes.
    - **Case mismatches that only work on case-insensitive file systems:**
      - The bundle lists `~/Content/site.css`, but the file is `Content/Site.css`.
      - The layout uses `~/images/brand.png`, but the folder is `Images/`.
      - `AssemblyInfo` references `log4net.xml`, but the file is `log4Net.xml`.
      - The URLs `/Catalog` vs `catalog` (`Url.Action("index", "catalog")`) are fine, because routing is case-insensitive.

11. **Security issues to fix rather than carry over.**
    - **Path traversal and arbitrary file read** through `PicController`. `PictureFileName` comes from the client on Edit and is passed unchecked to `Path.Combine` and `File.ReadAllBytes`. The antiforgery token doesn't help, because the app has no authentication, so anyone can get a token. This was found by reading the code; nobody tried to exploit it. Fix it in the port by not binding `PictureFileName` on Edit and by restricting reads to file names inside the pictures folder.
    - A missing picture file makes `File.ReadAllBytes` throw, which returns a 500 instead of a 404.
    - There is no authentication or authorization on any page or API, including the DELETE API.
    - BinaryFormatter is insecure if it is ever used to deserialize data (risk #1). The `DeserializeBinary` method exists, though this app never calls it.

12. **Front-end version mismatches.**
    - The markup uses **Bootstrap 3** classes (`navbar-static-top`, `hidden-xs`, `col-md-offset-2`, `dl-horizontal`, `control-label`) with the Bootstrap 4.3.1 package. `Content/custom.css` and `base.css` may make up for some of this. **⚠ Unverified** how the pages actually look.
    - `~/bundles/bootstrap` includes `bootstrap.js` but not `popper.js` (Bootstrap 4 needs popper for its dropdowns and tooltips). No current page uses those components.
    - The jQuery package version (3.5.0) and the file on disk (3.3.1) differ.
    - Decide whether to keep the current look exactly (which matters for golden-master HTML comparisons) or update it during the port.

---

## 11. Open questions / unverified

Nothing below was observed at runtime. The app needs Windows, IIS Express, and SQL Server LocalDB, and this analysis was done on macOS by reading the code.

1. Does `GET /api` (`CatalogController2`) throw because `JsonRequestBehavior.AllowGet` is missing? Does its controller-level `[Route("api")]` bind to `Index` as expected?
2. In mock mode, do Details, Edit, and Delete show a blank brand and type before any Index request has filled in the navigation properties?
3. Does a `Name` longer than 50 characters cause a `DbEntityValidationException` and the error page in DB mode?
4. What happens with a negative `pageIndex` (for example, the hidden Previous link on page 1, `pageIndex=-1`)?
5. Is Application Insights sending telemetry anywhere, given that no instrumentation key is configured?
6. Are Pipelines.Sockets.Unofficial, System.IO.Pipelines, System.Threading.Channels, and similar packages truly unused?
7. **Who calls `/api/files` and `/api/brands`?** This decides how the BinaryFormatter replacement and the XML/JSON content negotiation should behave.
8. Should `eShopPorted/` (the ASP.NET Core 2.2 attempt) inform the .NET 10 port, or be ignored?
9. Is `Catalog.Description` really `nvarchar(max)` in existing databases? This is the EF6 convention, but databases created another way may differ.
10. Which files do the `{version}` and `*` wildcards in the bundles actually emit (jQuery, Modernizr)?
11. How do the pages render with Bootstrap 4 CSS and Bootstrap 3 markup? This matters for HTML or visual golden-master baselines.
