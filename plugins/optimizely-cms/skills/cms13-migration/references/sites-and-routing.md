# Sites, Applications, and Routing Breaking Changes (CMS 12 → CMS 13)

## `SiteDefinition` → `Application` model

`SiteDefinition` is replaced by `Application`. `ISiteDefinitionResolver` is replaced by `IApplicationResolver`. A shim layer supports gradual migration.

```csharp
// Before
private readonly ISiteDefinitionResolver _siteResolver;
var site = _siteResolver.GetByContent(contentRef, false);

// After
private readonly IApplicationResolver _appResolver;
var app = _appResolver.GetByContent(contentRef, false);
```

**Obsoleted types:**
- `EPiServer.Web.SiteDefinition` → `Application`
- `EPiServer.Web.SiteDefinitionEventArgs` → `ApplicationEvent`
- `EPiServer.Web.SiteDefinitionUpdatedEventArgs` → `ApplicationUpdatedEvent`
- `EPiServer.Web.ISiteDefinitionRepository` → `IApplicationRepository`
- `EPiServer.Web.ISiteDefinitionResolver` → `IApplicationResolver`
- `EPiServer.DataAbstraction.ISiteConfigurationRepository` — removed, no replacement

**Wildcard host:** The `*` wildcard host definition is replaced with `Application.IsDefault = true`. Check Admin > Config > Manage Applications after upgrade.

---

## `SiteDefinition.Current.StartPage` — static singleton removed

```csharp
// Before
var startPageRef = SiteDefinition.Current.StartPage;

// After — inject IApplicationResolver
private readonly IApplicationResolver _appResolver;
var startPageRef = (_appResolver.GetByContext() as IRoutableApplication)?.EntryPoint
    ?? ContentReference.RootPage;
```

| Before | After |
|---|---|
| `SiteDefinition.Current.StartPage` | `IApplicationResolver.GetByContext()` as `IRoutableApplication` → `.EntryPoint` |
| `SiteDefinition.Current.RootPage` | `ContentReference.RootPage` |

---

## `ContentSearchProviderBase` constructor change

```csharp
// Before
public MySearchProvider(ISiteDefinitionResolver siteDefinitionResolver, ...)

// After
public MySearchProvider(IApplicationResolver applicationResolver, ...)
```

---

## `UriSupport` → dedicated replacements

`EPiServer.UriSupport` is obsolete. Resolve `IUriSupport` from DI, or use the static replacement classes:

| Removed member | Replacement |
|---|---|
| `UriSupport.UIUrl` | `EPiServer.Web.UIPathResolver` (inject) |
| `UriSupport.UtilUrl` | `EPiServer.Web.UIPathResolver` |
| `UriSupport.InternalUIUrl` | `EPiServer.Web.UIOption` |
| `UriSupport.InternalUtilUrl` | `EPiServer.Web.UIOption` |
| `UriSupport.IsStringWellFormedUri`, `Combine`, `BuildQueryString`, `AddQueryString`, etc. | `EPiServer.Web.UriUtil` |
| `UriSupport.EscapeUriSegments` | `EPiServer.UrlEncode` |
| `UriSupport.SiteUrl` | `EPiServer.Web.SiteDefinition.SiteUrl` |

`UIPathResolver.Instance` is obsoleted — inject `UIPathResolver` instead:

```csharp
// Before
var uiUrl = EPiServer.UriSupport.UIUrl;

// After
private readonly UIPathResolver _uiPathResolver;
var uiUrl = _uiPathResolver.UIUrl;
```

`HttpRequestSupport.IsSystemDirectory` removed → use `UIPathResolver.IsSystemPath`.

---

## UI URL path change: `/EPiServer/` → `/Optimizely/`

The default UI segment changed. Update any bookmarks, integrations, or hard-coded references:

| CMS 12 path | CMS 13 path |
|---|---|
| `/EPiServer/EPiServer.Cms.UI.Admin/` | `/Optimizely/Settings/` |
| `/EPiServer/EPiServer.Cms.UI.Settings/` | `/Optimizely/Profile/` |
| `/EPiServer/EPiServer.Cms.UI.VisitorGroups/` | `/Optimizely/VisitorGroups/` |
| `/EPiServer/EPiServer.Cms.UI.ContentManager/` | `/Optimizely/ContentManager/` |
| `/EPiServer/EPiServer.Cms.Forms.UI/` | `/Optimizely/Forms/` |
| `/EPiServer/EPiServer.Cms.UI.Webhooks/` | `/Optimizely/Webhooks/` |
| `/EPiServer/EPiServer.Cms.UI.Credentials/` | `/Optimizely/Credentials/` |
| `/EPiServer/Optimizely.Graph.Cms/` | `/Optimizely/GraphQL/` |
| `/EPiServer/EPiServer.Cms.TinyMce/` | `/Optimizely/TinyMce/` |
| `/EPiServer/EPiServer.Cms.UI.Extensibility/` | `/Optimizely/Extensibility/` |

---

## Routing events: `IContentRouteEvents` removed

```csharp
// Before
private readonly IContentRouteEvents _routeEvents;
_routeEvents.CreatingVirtualPath += ...;
_routeEvents.RoutingContent += ...;

// After
private readonly IContentUrlGeneratorEvents _generatorEvents;
private readonly IContentUrlResolverEvents _resolverEvents;
_generatorEvents.GeneratingUrl += ...;
_resolverEvents.ResolvingUrl += ...;
```

| CMS 12 event | CMS 13 event |
|---|---|
| `IContentRouteEvents.CreatingVirtualPath` | `IContentUrlGeneratorEvents.GeneratingUrl` |
| `IContentRouteEvents.CreatedVirtualPath` | `IContentUrlGeneratorEvents.GeneratedUrl` |
| `IContentRouteEvents.RoutingContent` | `IContentUrlResolverEvents.ResolvingUrl` |
| `IContentRouteEvents.RoutedContent` | `IContentUrlResolverEvents.ResolvedUrl` |

---

## `UrlResolverContext` / `UrlGeneratorContext` changes

- `UrlResolverContext.HostDefinition` → `ApplicationHost`
- `UrlResolverContext.RemainingPath` → `RemainingSegments`
- `UrlResolverContextExtensions.GetNextRemainingSegment` → `GetNextSegment`
- `UrlGeneratorContext.Host` → `ApplicationHost`
- `UrlGeneratorContext.CurrentHost` → `CurrentApplicationHost`

---

## `RoutingOptions` changes

- `UsePrimaryHostForOutgoingUrls` obsoleted — system always prefers primary hosts
- `UseEditHostForOutgoingMediaUrls` obsoleted — configure with application host type `Media` instead
- `RoutingOptions.ConfigureForExternalTemplates` removed → use `TemplateOptions.ConfigureForExternalTemplates`

---

## `EditUrlResolver` overload changes

Methods accepting `SiteDefinition` obsoleted in favor of overloads accepting `Application`.

---

## `SimpleAddressResolveContext` changes

- `SiteId` property → `ApplicationName`

---

## Language fallback behavior change

When a page has two language versions and one uses the other as fallback: if the primary language version expires, CMS 13 shows the fallback language instead of a 404.

---

## `IUpdateCurrentLanguage` changes

Methods `UpdateLanguage` and `UpdateReplacementLanguage` removed → use `SetRoutedContent`.

---

## `Html.CreatePlatformNavigationMenu()` removed

```cshtml
{{!-- Before --}}
@Html.CreatePlatformNavigationMenu()

{{!-- After --}}
<platform-navigation />
```
