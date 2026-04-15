# API Replacement Map (CMS 12 → CMS 13)

Quick-reference tables for migrating types, methods, events, namespaces, and packages.

---

## Types

| CMS 12 | CMS 13 | Notes |
|---|---|---|
| `PageReference` | `ContentReference` | Replace throughout all usages |
| `BlockTypeRepository` | `IContentTypeRepository` | Non-generic |
| `PageTypeRepository` | `IContentTypeRepository` | Non-generic |
| `IContentTypeRepository<T>` | `IContentTypeRepository` | Non-generic |
| `ContentCoreData` | `ContentNode` | Direct replacement |
| `IContentCoreDataLoader` | `IContentNodeLoader` | Direct replacement |
| `IContentGroup` | `IPersonalizedGroup` | Direct replacement |
| `ISecuredFragmentMarkupGenerator` | `IPersonalizedFragmentMarkupGenerator` | Direct replacement |
| `IRoleSecurityDescriptor` | `IEnumerable<string>` | For role identities |
| `DynamicProperty` | *(removed)* | Feature removed entirely |
| `PlugInAttribute` | *(removed)* | Plugin system removed |
| `ScheduledPlugInAttribute` | `ScheduledJobAttribute` | From `EPiServer.Scheduler` |
| `DojoWidgetAttribute` | `CriterionPropertyEditor` | Visitor group criteria |
| `ISiteDefinitionResolver` | `IApplicationResolver` | Application model |
| `ISiteDefinitionRepository` | `IApplicationRepository` | Application model |
| `SiteDefinition` | `Application` | Type name only; `SiteDefinition.Current` requires DI refactoring |
| `VirtualPathResolver` | `VirtualPathUtilityEx` | Public static class; same namespace `EPiServer.Web` |

---

## Methods and properties

| CMS 12 | CMS 13 | Notes |
|---|---|---|
| `ContentArea.FilteredItems` | `ContentArea.Items` | Or use `IContentAreaItemsRenderingFilter` |
| `IContentAreaLoader.Get()` | `IContentAreaLoader.LoadContent()` | Direct replacement |
| `ContentAreaItemExtensions.GetContent()` | `ContentAreaItem.LoadContent()` | Direct replacement |
| `XhtmlString.ToHtmlString(IPrincipal)` | `XhtmlString.ToHtmlString()` | Parameter removed |
| `PropertyString.PublicString` | `PropertyString.String` | Now public |
| `PropertyLongString.PublicLongString` | `PropertyLongString.LongString` | Now public |
| `PropertyUrl.String` | `PropertyUrl.LongString` | Base class changed |
| `PropertyDefinition.Searchable` | `PropertyDefinition.IndexingType` | Direct replacement |
| `PropertyData.Clear()` (override) | `PropertyData.ClearImplementation()` | Direct replacement |
| `PrincipalInfo.HasEditAccess()` | `user.IsInRole(Roles.CmsEditors)` | Direct replacement |
| `PrincipalInfo.IsPermitted()` | `PermissionService.IsPermitted()` | Direct replacement |
| `SoftLink.LinkStatus` | `SoftLink.HttpStatusCode` | Direct replacement |
| `SaveAction.None` | `SaveAction.Default` | Direct replacement |
| `SaveAction.DelayedPublish` | `SaveAction.Schedule` | Direct replacement |
| `RenderSettings.CustomTag` | `RenderSettings.CustomTagName` | Direct replacement |
| `RenderSettings.ChildrenCustomTag` | `RenderSettings.ChildrenCustomTagName` | Direct replacement |
| `VirtualPathResolver.Instance.Method()` | `VirtualPathUtilityEx.Method()` | Static class, no `.Instance` |
| `SiteDefinition.Current.StartPage` | `IApplicationResolver.GetByContext()?.EntryPoint` | Inject `IApplicationResolver` |
| `SiteDefinition.Current.RootPage` | `ContentReference.RootPage` | Direct replacement |
| `UIPathResolver.Instance` | Inject `UIPathResolver` | Use service container |
| `ConstructorParameterResolver.Instance` | Inject from service container | Use service container |
| `PropertyData.Locate` | Inject services via constructor | — |
| `Html.CreatePlatformNavigationMenu()` | `<platform-navigation />` | Tag helper |
| `PageController<T>.PageContext.Page` | `PageContext.Content` | In CMS 13, `PageContext` exposes `.Content` directly — no extra injection needed in controller base classes. Only inject `IContentRouteHelper` if accessing content outside a `PageController<T>` subclass. |
| `IPageRouteHelper.Page` | `IContentRouteHelper.Content` | Inject `IContentRouteHelper` |
| `IPageRouteHelper.PageLink` | `IContentRouteHelper.ContentLink` | Inject `IContentRouteHelper` |

---

## Events

| CMS 12 | CMS 13 |
|---|---|
| `IContentRouteEvents.CreatingVirtualPath` | `IContentUrlGeneratorEvents.GeneratingUrl` |
| `IContentRouteEvents.CreatedVirtualPath` | `IContentUrlGeneratorEvents.GeneratedUrl` |
| `IContentRouteEvents.RoutingContent` | `IContentUrlResolverEvents.ResolvingUrl` |
| `IContentRouteEvents.RoutedContent` | `IContentUrlResolverEvents.ResolvedUrl` |

---

## DI namespace for registration methods

| CMS 12 namespace | CMS 13 namespace | Affected methods |
|---|---|---|
| `Microsoft.Extensions.DependencyInjection` | `EPiServer.DependencyInjection` | `AddCmsCore()`, `AddCmsData()`, `AddCmsFramework()`, `AddTinyMce()`, `AddAdmin()`, `AddCmsUI()`, `AddCmsShell()`, `AddCmsShellUI()`, `AddCmsBlobs()`, `AddCmsCache()`, `AddCmsGeolocation()`, `AddCmsChangeNotification()`, `AddCmsLinkAnalyzer()` |

---

## Packages now requiring explicit references

These were transitive in CMS 12. Add `<PackageReference>` explicitly in CMS 13:

| Namespace in code | Required package | Registration |
|---|---|---|
| `EPiServer.Cms.UI.AspNetIdentity` | `EPiServer.CMS.UI.AspNetIdentity` | — |
| `EPiServer.Cms.UI.VisitorGroups` | `EPiServer.CMS.UI.VisitorGroups` | `.AddVisitorGroupsMvc()`, `.AddVisitorGroupsUI()` |
| `EPiServer.Personalization` (geolocation) | `EPiServer.Geolocation` | `.AddCmsGeolocation()` |
| `EPiServer.Events.ChangeNotification` | `EPiServer.Events.ChangeNotification` | `.AddCmsChangeNotification()` |
| `EPiServer.Framework.Blobs` | `EPiServer.Blobs` | `.AddCmsBlobs()` |
| `EPiServer.Framework.Cache` | `EPiServer.Cache` | `.AddCmsCache()` |
| `EPiServer.Logging` | `EPiServer.Logging` | — |
| `EPiServer.HtmlParsing` | `EPiServer.HtmlParsing` | — |

---

## Removed without replacement

| CMS 12 | Notes |
|---|---|
| `EPiServer.Shell.Telemetry` / `TelemetryOptions` | Entirely removed |
| `SearchProvidersManager` | Made internal |
| `SiteDefinition.Current.StartPage` (static) | Use `IApplicationResolver` |
| Dynamic Properties (all) | Feature removed |
| Plugin system (`PlugInAttribute`, `ScheduledPlugInAttribute`) | Use `ScheduledJobAttribute` for jobs |
| `Castle.Windsor` dependency | Add explicit package if needed |
| `Newtonsoft.Json` dependency | Add explicit package if needed |
