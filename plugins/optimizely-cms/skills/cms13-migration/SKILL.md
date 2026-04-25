---
name: cms13-migration
description: "Use this skill when upgrading an Optimizely CMS 12 project to CMS 13, fixing compile errors after a CMS upgrade, or replacing any removed CMS 12 APIs. USE FOR: upgrading from CMS 12 to CMS 13, fixing compile errors after CMS upgrade, understanding CMS 13 breaking changes, replacing removed APIs (SiteDefinition, IApplicationResolver, EPiServer.Applications namespace, PageReference, DynamicProperty, ContentArea.FilteredItems, XhtmlString.ToHtmlString, IContentRouteEvents, PlugInAttribute, IServiceLocator, ServiceLocator.GetInstance), migrating Newtonsoft.Json to System.Text.Json in CMS context, updating NuGet packages for CMS 13. DO NOT USE FOR: CMS 11 migrations, Optimizely Commerce upgrades, non-Optimizely .NET projects."
compatibility: "Optimizely CMS 12 projects (.NET 8). Requires NuGet source https://api.nuget.optimizely.com/v3/index.json."
---

# Optimizely CMS 12 → CMS 13 Migration

## Gotchas

These are silent failures that won't produce obvious errors at the right moment — watch for them throughout the migration:

- **`IValidate<T>` validators are silently dropped at runtime.** Implementations are no longer auto-discovered. Every validator needs `services.AddCmsValidator<T>()` or it simply won't run — no warning, no error.
- **Tab/group names with spaces compile fine but break the DB.** The database auto-migrates invalid names on upgrade (e.g., `"Meta Data"` → `"MetaData"` with prefix `G_`), but your code constants still hold the old value. You'll get a mismatch silently — the group tab won't render the property.
- **`ScriptParserOptions` defaults changed.** `SavingMode` is now `ThrowException` — content that previously saved with inline scripts or unsafe HTML now throws `InvalidPropertyValueException`. Test content editing flows after upgrade.
- **`SiteDefinition.Current.StartPage` doesn't throw — it returns an empty reference.** Code that checks `!ContentReference.IsNullOrEmpty(...)` on it may silently fall through with no site settings loaded.
- **`IApplicationResolver` and `IRoutableApplication` live in `EPiServer.Applications`, not `EPiServer.Web`.** Every C# file using these types needs `using EPiServer.Applications;`. In Razor views, add both `@using EPiServer.Applications` and the fully-qualified inject directive (see Step 5). Missing this namespace causes a flood of "type or namespace not found" errors after migrating `SiteDefinition.Current`.
- **`ContentArea.FilteredItems` is obsolete but still compiles.** It won't respect rendering filters in CMS 13. Use `ContentArea.Items` or the filter will be ignored with no error.
- **`context.Locate.Advanced` inside `IConfigurableModule.Initialize()` fails.** `InitializationEngine.Locate` returns `IServiceLocator`, which is removed. Any call like `context.Locate.Advanced.GetInstance<T>()` will not compile. Replace with `context.Services.GetInstance<T>()`.
- **`ServiceLocator.Current.GetInstance<T>()` no longer compiles anywhere.** The `GetInstance<T>()` extension came from `ServiceLocatorExtensions`, which is removed. `ServiceLocator.Current` itself still works in CMS 13 (now returns `IServiceProvider`), but you must switch to `.GetRequiredService<T>()` (from `Microsoft.Extensions.DependencyInjection`). The fix varies by context — see Step 4.
- **The DI namespace change is a silent build failure.** If you have `using Microsoft.Extensions.DependencyInjection;` and call `AddCmsCore()`, the old extension method is gone — you get a confusing "no overload" error. Change to `using EPiServer.DependencyInjection;`.
- **`IApprovalEngine` now enforces the approval workflow before publishing.** Content under an approval definition will throw `ValidationException` on `SaveAction.Publish`. To bypass (e.g., in programmatic migrations), the current user must have Administer rights and you must pass `SaveAction.SkipApprovalValidation`. When you need the full skip-flag table or access-level bypass patterns, read [`references/security-and-access-control.md`](references/security-and-access-control.md).
- **`IContentVersionRepository.List` silently returns `totalCount = -1`.** Unless you explicitly set `VersionFilter.IncludeTotalCount = true`, the count is not computed. Code that reads `totalCount` after calling `List` will receive `-1` with no error.
- **`ClientEditorAttribute.EditorConfiguration` JSON must use double-quoted property names.** CMS 13 parses this with `System.Text.Json`. Single-quoted or unquoted JSON property keys (common in C# string literals) will silently fail to deserialize, producing an empty editor configuration.
- **`/lang` folder localization files are no longer auto-registered.** The `XmlLocalizationProvider` that scanned the `/lang` folder on startup is removed. If you have XML language files there, they must be switched to embedded resources in the assembly instead. See [`references/framework-and-platform.md`](references/framework-and-platform.md) for localization changes.
- **Visitor Groups are not enabled by default.** Unless you call `services.AddVisitorGroupsMvc()` and `services.AddVisitorGroupsUI()`, visitor group criteria and personalization will be unavailable with no compile-time error. See Step 3 and [`references/content-management-and-repository.md`](references/content-management-and-repository.md).

## Migration workflow

Work through each area below in order. Steps 1–4 must complete before the project will compile again.

**Phase 1 — Foundation** (steps 1–4): .NET target framework, incompatible packages, NuGet update, DI namespace  
**Phase 2 — API migration** (steps 5–16): service locator, SiteDefinition, PageReference, content types, ContentArea, routing, validators, SaveAction  
**Phase 3 — Cleanup** (steps 17–20): UI URLs, Newtonsoft.Json, security review, build warnings

### Step 1 — Update .NET target framework

In every `.csproj` in the solution:

```xml
<TargetFramework>net10.0</TargetFramework>
```

Update Docker base images and CI/CD pipelines to use .NET 10 images.

---

### Step 2 — Remove or upgrade incompatible third-party packages

Several popular Optimizely ecosystem packages are incompatible with CMS 13. Check the list and act on each:

| Package | Action |
|---|---|
| `EPiServer.Find.Cms` | Remove — no CMS 13 release. Also loses `Newtonsoft.Json` transitively (see Step 17). |
| `EPiServer.Forms` | Upgrade to 6.0.0+ |
| `EPiServer.Cms.WelcomeIntegration.UI` | Replace with `EPiServer.Cms.DamIntegration.UI` (6.0.0+) |
| `Advanced.CMS.AdvancedReviews` | Remove — no CMS 13 release |
| `Geta.NotFoundHandler.Optimizely` | Remove — incompatible; await updated release |
| `Geta.Optimizely.Sitemaps` | Remove — incompatible; await updated release |
| `Geta.Optimizely.GenericLinks` | Remove — incompatible; await updated release |
| `Geta.Optimizely.ContentTypeIcons` | Remove — incompatible; await updated release |
| `Geta.Optimizely.Categories` | Remove — incompatible. **Warning:** leaves orphaned `CategoryRoot`/`CategoryData` content types in DB causing "could not create instance" errors at startup. |
| `Gulla.Episerver.SqlStudio` | Upgrade to 3.0.2+ |
| `Stott.Optimizely.RobotsHandler` | Remove — incompatible; await updated release |

See [`references/third-party-package-compatibility.md`](references/third-party-package-compatibility.md) for per-package removal steps, the generic removal procedure, and how to handle orphaned content types and scheduled jobs left by removed packages.

ALWAYS check the latest version of each package before starting your migration — some may have released CMS 13-compatible versions since this skill was written.

ALWAYS ask the user if they want to remove incompatible packages or wait for an update, if they want to wait then stop the migration. If they want to remove, offer to help find replacement packages or write custom code for missing features, and warn about the consequences of removal (e.g., orphaned content types, lost features).

---

### Step 3 — Update NuGet packages

Bump `EPiServer.CMS` (and related packages) to version 13.x. After updating, add explicit `<PackageReference>` entries for any previously-transitive packages you use:

```xml
<!-- Add if you use these namespaces -->
<PackageReference Include="EPiServer.CMS.UI.AspNetIdentity" Version="13.x.x" />
<PackageReference Include="EPiServer.Geolocation" Version="13.x.x" />       <!-- EPiServer.Personalization -->
<PackageReference Include="EPiServer.Blobs" Version="13.x.x" />             <!-- EPiServer.Framework.Blobs -->
<PackageReference Include="EPiServer.Cache" Version="13.x.x" />             <!-- EPiServer.Framework.Cache -->
<PackageReference Include="EPiServer.Events.ChangeNotification" Version="13.x.x" />
<PackageReference Include="EPiServer.Logging" Version="13.x.x" />
<PackageReference Include="EPiServer.HtmlParsing" Version="13.x.x" />
<!-- Add if you removed Episerver.Find and still need it -->
<PackageReference Include="Newtonsoft.Json" Version="..." />
```

Call the corresponding registration methods in `Startup.cs` / `Program.cs`:
```csharp
services.AddCmsBlobs();
services.AddCmsCache();
services.AddCmsGeolocation();
services.AddCmsChangeNotification();
```

When using namespaces from packages that were previously transitive (e.g. `EPiServer.Framework.Blobs`, `EPiServer.Framework.Cache`, `EPiServer.Logging`), read [`references/framework-and-platform.md`](references/framework-and-platform.md) for the full table of extracted packages and registration methods.

---

### Step 4 — Fix DI namespace for CMS registration methods

```csharp
// Before
using Microsoft.Extensions.DependencyInjection;

// After
using EPiServer.DependencyInjection;
```

Affected: `AddCmsCore()`, `AddCmsData()`, `AddCmsFramework()`, `AddTinyMce()`, `AddAdmin()`, `AddCmsUI()`, `AddCmsShell()`, `AddCmsShellUI()`, and the new extracted-package methods above.

---

### Step 5 — Remove service locator usages

Replace constructor injection of removed types (`IServiceLocator`, `ServiceLocationHelper`, etc.) with standard constructor injection from `Microsoft.Extensions.DependencyInjection`. When you encounter types not listed above (`IInterceptorRegister`, `IRegisteredService`, `IServiceConfigurationProvider`, etc.), read [`references/framework-and-platform.md`](references/framework-and-platform.md).

The `GetInstance<T>()` extension method is removed (it came from `ServiceLocatorExtensions`). `ServiceLocator.Current` still exists in CMS 13 and now returns `IServiceProvider`. The replacement depends on the context:

#### Constructor-injectable classes

Use constructor injection — the preferred pattern for any class resolved from DI:

```csharp
// Before
public class MyService
{
    public MyService() { _repo = ServiceLocator.Current.GetInstance<IContentRepository>(); }
}

// After
public class MyService
{
    public MyService(IContentRepository repo) { _repo = repo; }
}
```

#### `IConfigurableModule.Initialize` — `context.Locate.Advanced`

```csharp
// Before — context.Locate returns IServiceLocator (removed)
var events = context.Locate.Advanced.GetInstance<ITemplateResolverEvents>();

// After — static shim is still available in CMS 13
var events = context.Services.GetInstance<ITemplateResolverEvents>();
```

#### Razor page base classes

Razor page base classes (inheriting from `RazorPage<TModel>`) don't support constructor injection. The old pattern used `ServiceLocator.Current.GetInstance<T>()` in the parameterless constructor. Replace with `Context.RequestServices.GetRequiredService<T>()`, using the `HttpContext Context` property available on `RazorPage<TModel>`:

```csharp
// Before — ServiceLocator in constructor
public class MyPageBase : RazorPage<TModel>
{
    private readonly MyService _svc;
    public MyPageBase() { _svc = ServiceLocator.Current.GetInstance<MyService>(); }
    protected void UseService() => _svc.DoSomething();
}

// After — resolve on demand from the request service provider
public class MyPageBase : RazorPage<TModel>
{
    protected void UseService()
        => Context.RequestServices.GetRequiredService<MyService>().DoSomething();
}
```

#### Static extension methods with `IHtmlHelper`

Static helpers that receive `IHtmlHelper` have access to `HttpContext` via the helper — use it instead of `ServiceLocator`:

```csharp
// Before
var contentLoader = ServiceLocator.Current.GetInstance<IContentLoader>();

// After
var contentLoader = helper.ViewContext.HttpContext.RequestServices.GetRequiredService<IContentLoader>();
// Also add: using Microsoft.Extensions.DependencyInjection;
```

#### `ContentData` overrides (e.g. `SetDefaultValues`)

`ContentData` subclasses (page types, block types) are not resolved from DI — constructor injection isn't available. Use `ServiceLocator.Current.GetRequiredService<T>()` with `using Microsoft.Extensions.DependencyInjection`:

```csharp
// Before
NewsList.Heading = ServiceLocator.Current.GetInstance<LocalizationService>().GetString("/key");

// After
using Microsoft.Extensions.DependencyInjection;
NewsList.Heading = ServiceLocator.Current.GetRequiredService<LocalizationService>().GetString("/key");
```

---

### Step 6 — Migrate `SiteDefinition` → `Application`

```csharp
// Before
private readonly ISiteDefinitionResolver _siteResolver;
var startPage = SiteDefinition.Current.StartPage;

// After — IApplicationResolver and IRoutableApplication are in EPiServer.Applications
using EPiServer.Applications;

private readonly IApplicationResolver _appResolver;
var startPage = (_appResolver.GetByContext() as IRoutableApplication)?.EntryPoint
    ?? ContentReference.RootPage;
```

Replace `ISiteDefinitionResolver` → `IApplicationResolver`, `ISiteDefinitionRepository` → `IApplicationRepository`.

#### Razor views

When `SiteDefinition.Current.StartPage` appears in `.cshtml` files, replace it with `@inject` and a local variable:

```cshtml
@using EPiServer.Applications
@inject EPiServer.Applications.IApplicationResolver AppResolver

@{
    var startPageRef = (AppResolver.GetByContext() as IRoutableApplication)?.EntryPoint
        ?? ContentReference.RootPage;
}

@* Use startPageRef wherever SiteDefinition.Current.StartPage was used *@
@Html.MenuList(startPageRef, ItemTemplate)
```

Note: both `@using EPiServer.Applications` **and** the fully-qualified type in `@inject` are required because Razor view compilation resolves namespaces differently than C# files.

See [`references/sites-and-routing.md`](references/sites-and-routing.md) for the full list.

---

### Step 7 — Replace `PageReference` with `ContentReference`

`PageReference` is obsolete. Global find-and-replace in most cases:
On pages and blocks, replace `PageReference` → `ContentReference`. Check properties with `[AllowedTypes]` — if they only allow page types, add `[AllowedTypes(typeof(PageData))]` to preserve validation.
```csharp
// Before
public virtual PageReference ContactPageLink { get; set; }

// After
[AllowedTypes(typeof(PageData))]
public virtual ContentReference ContactPageLink { get; set; }
```

everywhere else, replace `PageReference` → `ContentReference` without adding `[AllowedTypes]`.
```csharp
// Before
PageReference pageRef = page.ContentLink as PageReference;

// After
ContentReference pageRef = page.ContentLink;
```

Check `PageData` properties (`ContentLink`, `ParentLink`, `ArchiveLink`) — they now return `ContentReference`.

Also replace **instance usages** of `page.PageLink` — the property is obsolete and will produce warnings:

```csharp
// Before — in C# code
new SelectItem { Value = page.PageLink }

// After
new SelectItem { Value = page.ContentLink }
```

```cshtml
{{!-- Before — in .cshtml views --}}
<a href="@Url.ContentUrl(item.Page.PageLink)">

{{!-- After --}}
<a href="@Url.ContentUrl(item.Page.ContentLink)">
```

---

### Step 8 — Fix content type / tab naming

Check all `[GroupDefinitions]` classes and `[ContentType]` / `[TabDefinition]` attributes for names containing spaces or special characters. CMS 13 auto-migrates the database, but code must match:

```csharp
// Before
[Display(Name = "Meta Data")]
public const string MetaData = "Meta Data";  // space — invalid

// After
[Display(Name = "Meta Data")]
public const string MetaData = "MetaData";   // no space
```

When you need the full naming regex, the prefix table for all definition types, or the complete Dynamic Property removal list, read [`references/content-types-and-properties.md`](references/content-types-and-properties.md).

---

### Step 9 — Remove Dynamic Properties

Delete all usages of `DynamicProperty`, `DynamicPropertyCollection`, `DynamicPropertyBag`, and related types. No replacement exists — migrate data to regular content properties.

Read [`references/content-types-and-properties.md`](references/content-types-and-properties.md) if you need the complete list of removed types.

---

### Step 10 — Fix `ContentArea` and `XhtmlString` usages

```csharp
// Before
var items = contentArea.FilteredItems;
var html = xhtmlString.ToHtmlString(User);

// After
var items = contentArea.Items; // or use IContentAreaItemsRenderingFilter
// For XhtmlString: use @Html.PropertyFor() / Tag Helpers in views
```

Replace `IContentAreaLoader.Get()` → `.LoadContent()`.
Replace `ContentAreaItemExtensions.GetContent()` → `.LoadContent()`.

Read [`references/content-management-and-repository.md`](references/content-management-and-repository.md) if you encounter other `ContentArea`, `XhtmlString`, `PropertyString`, or `PropertyLongString` members that don't compile.

---

### Step 11 — Fix `IContentTypeRepository` generic usages

```csharp
// Before
private readonly IContentTypeRepository<PageType> _repo;

// After
private readonly IContentTypeRepository _repo;
```

For generic usages of `IContentTypeRepository<T>`, remove the generic type parameter and cast the result of `Load()` to the expected type.
remember that the ContentType from `Load()` can be null if the type isn't found and the cast will also return null if the type does not match, so consider making the method return type nullable and adding null checks where appropriate.  

```csharp
// New way
var contentPageType = contentTypeRepository.Load(pageType) as PageType;
```
---

### Step 12 — Fix `PropertyString` / `PropertyLongString` accessors

```csharp
// Before: prop.PublicString, prop.PublicLongString
// After:  prop.String,       prop.LongString
```

---

### Step 13 — Register validators explicitly

```csharp
// Implementations of IValidate<T> are no longer auto-discovered
services.AddCmsValidator<MyContentValidator>();
```

---

### Step 14 — Fix routing events

```csharp
// Before
private readonly IContentRouteEvents _routeEvents;
_routeEvents.CreatingVirtualPath += OnCreatingVirtualPath;

// After
private readonly IContentUrlGeneratorEvents _generatorEvents;
_generatorEvents.GeneratingUrl += OnGeneratingUrl;
```

When you need the full event member mapping (`UrlResolverContext`, `UrlGeneratorContext` property renames) or other routing context changes, read [`references/sites-and-routing.md`](references/sites-and-routing.md).

| Before | After |
|---|---|
| `SaveAction.None` | `SaveAction.Default` |
| `SaveAction.DelayedPublish` | `SaveAction.Schedule` |

---

### Step 16 — Replace `PlugInAttribute` / scheduled jobs

```csharp
// Before
[ScheduledPlugIn(DisplayName = "My Job")]
public class MyJob : JobBase { }

// After
[ScheduledJob(DisplayName = "My Job")]
public class MyJob : ScheduledJobBase { }
```

---

### Step 17 — Update UI URL references

The CMS UI base path changed from `/EPiServer/` to `/Optimizely/`. Update bookmarks, scripts, webhooks, and any hard-coded URL references. See [`references/sites-and-routing.md`](references/sites-and-routing.md) for the full mapping.

---

### Step 18 — Verify Newtonsoft.Json

If your project (or removed packages like `Episerver.Find`) used Newtonsoft.Json transitively, add an explicit reference:

```xml
<PackageReference Include="Newtonsoft.Json" Version="13.x.x" />
```

---

### Step 19 — Review security and access control changes

Review the access control and approval workflow changes:

- `IApprovalEngine` now enforces approval definitions on publish — programmatic saves may need `SaveAction.SkipApprovalValidation` with Administer rights
- `PermissionRepository` sync methods removed — use async equivalents
- `IContentSecurityRepository` events moved to `IContentSecurityEvents` interface

When implementing programmatic publish flows, customizing `AccessControlEntry`, or using preview tokens, read [`references/security-and-access-control.md`](references/security-and-access-control.md).

---

### Step 20 — Look at build warnings fix obsolete API usages

build the solution and look for warnings about obsolete APIs. Many of these have direct replacements that need to be updated in code. consult the complete CMS 12 → 13 mapping in [`references/api-replacement-map.md`](references/api-replacement-map.md) for the full list of API replacements.

---

## Common compile-error patterns after upgrade

When the build fails after upgrading and the step above doesn't address it, read [`references/api-replacement-map.md`](references/api-replacement-map.md) — it contains a quick-lookup table of common compile errors and their fixes, plus the complete CMS 12 → 13 type, method, event, and namespace mapping.
