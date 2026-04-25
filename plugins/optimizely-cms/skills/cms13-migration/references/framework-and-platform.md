# Framework and Platform Breaking Changes (CMS 12 → CMS 13)

## .NET version

CMS 13 requires **.NET 10**. Update every `.csproj`:

```xml
<!-- Before -->
<TargetFramework>net8.0</TargetFramework>

<!-- After -->
<TargetFramework>net10.0</TargetFramework>
```

Also update Docker base images and CI/CD pipeline configurations accordingly.

---

## NuGet package restructuring

### Packages extracted from `EPiServer.CMS.Core`

Several namespaces are now in standalone packages. Add explicit `<PackageReference>` entries for each namespace you use:

| Namespace in code | Required package | Registration method |
|---|---|---|
| `EPiServer.Logging` | `EPiServer.Logging` | N/A |
| `EPiServer.HtmlParsing` | `EPiServer.HtmlParsing` | N/A |
| `EPiServer.Personalization` (geolocation) | `EPiServer.Geolocation` | `.AddCmsGeolocation()` |
| `EPiServer.Events.ChangeNotification` | `EPiServer.Events.ChangeNotification` | `.AddCmsChangeNotification()` |
| `EPiServer.Framework.Blobs` | `EPiServer.Blobs` | `.AddCmsBlobs()` |
| `EPiServer.Framework.Cache` | `EPiServer.Cache` | `.AddCmsCache()` |

### Packages that were transitive in CMS 12 but now need explicit references

| Usage | Package |
|---|---|
| `EPiServer.Cms.UI.AspNetIdentity` | `EPiServer.CMS.UI.AspNetIdentity` |
| `EPiServer.Cms.UI.VisitorGroups` | `EPiServer.CMS.UI.VisitorGroups` (also call `.AddVisitorGroupsMvc()` and `.AddVisitorGroupsUI()`) |

### Other package changes

- `EPiServer.Cms.HealthCheck` removed. SQL health check consolidated into `EPiServer.Data`.
- `EPiServer.Data.Cache` integrated into `EPiServer.Data` — remove explicit reference.

---

## Dependency injection namespace change

Service registration extension methods moved from `Microsoft.Extensions.DependencyInjection` to **`EPiServer.DependencyInjection`**.

```csharp
// Before
using Microsoft.Extensions.DependencyInjection;
services.AddCmsCore();

// After
using EPiServer.DependencyInjection;
services.AddCmsCore();
```

Affected methods: `AddCmsCore()`, `AddCmsData()`, `AddCmsFramework()`, `AddTinyMce()`, `AddAdmin()`, `AddCmsUI()`, `AddCmsShell()`, `AddCmsShellUI()`, `AddCmsBlobs()`, `AddCmsCache()`, `AddCmsGeolocation()`, `AddCmsChangeNotification()`, `AddCmsLinkAnalyzer()`.

Bear in mind that `ApplicationUser` is still in the `EPiServer.Cms.UI.AspNetIdentity` namespace so that must not be removed if `ApplicationUser` is used by `AddCmsAspNetIdentity()`

### Removed service locator types

The following obsolete types are removed — replace with constructor injection:

- `IInterceptorRegister`, `IRegisteredService`, `IServiceConfigurationProvider`
- `IServiceLocator`, `ServiceDescriptor`, `ServiceLocationHelper`
- `ServiceLocatorExtensions`, `ServiceLocatorImplBase`
- `AutoDiscovery.IServiceLocatorFactory`, `ServiceLocatorFactoryAttribute`
- `IServiceCollection` extension methods that duplicate `Microsoft.Extensions.DependencyInjection` methods

---

## Newtonsoft.Json → System.Text.Json

CMS 13 migrated entirely to `System.Text.Json`. If your project still uses Newtonsoft, add an explicit `<PackageReference Include="Newtonsoft.Json" />`.

**Removed APIs:**
- `EPiServer.Framework.Serialization.Json.Internal.JsonObjectSerializer` → use `SystemTextJsonObjectSerializer`
- `NewtonsoftJsonSerializerSettingsOptions` removed
- Extension method `UseNewtonsoftJson` removed
- `FormatterType` enum removed
- `ExtendedNewtonsoftJsonOutputFormatter` removed
- `NewtonsoftFormatterExtensions` removed
- Shell classes `JsonIdentityConverter`, `TypeArrayConverter`, `TypeConverter` removed

**Rewritten converters** (now use System.Text.Json internally — no API change needed):
- `EPiServer.Framework.Serialization.Json.NotANumberConverter`
- `EPiServer.Framework.Serialization.Json.NameValueCollectionConverter`
- `EPiServer.Framework.Serialization.Json.DateTimeConverter`

---

## Castle.Windsor removal

CMS no longer depends on Castle.Windsor. If you use it, add an explicit `<PackageReference Include="Castle.Windsor" />`.

---

## Serialization constructors removed

`protected SomeClass(SerializationInfo info, StreamingContext context)` constructors made obsolete in .NET 8 are now removed. Remove any overrides of these constructors.

---

## Validation throws `ArgumentException` instead of `ArgumentNullException`

- Empty strings → `ArgumentException`
- Null properties on argument objects → `ArgumentException`
- Empty `ContentReference` arguments → `ArgumentException`

Update any `catch (ArgumentNullException)` blocks that handle these cases.

---

## Nullable reference type annotations

New nullable annotations on these libraries may produce compile-time warnings:

`EPiServer.ImageLibrary`, `EPiServer.Data.Cache`, `EPiServer.Hosting`, `EPiServer.Logging`, `EPiServer.Personalization`, `EPiServer.HtmlParsing`, `EPiServer.Events`

Add `#nullable enable` per-file or enable project-wide and address warnings.

---

## API spelling corrections (binary breaking changes)

| Type | Old name | New name |
|---|---|---|
| `IContentLanguageSettingsHandler.GetFallbackLanguages` | `conentLink` param | `contentLink` |
| `IContentRepository.GetReferencesToContent` | `includeDecendents` param | `includeDescendants` |
| `ContentProvider.GetReferencesToLocalContent` | `includeDecendents` param | `includeDescendants` |
| `IPropertyDefinitionRepository.List` | `pageTypeID` param | `contentTypeID` |
| `PropertyDefinitionRepository.List` | `pageTypeID` param | `contentTypeID` |
| `LanguageBranch` | `ValidateLanguageEdititingAccessRights` | `ValidateLanguageEditingAccessRights` |
| Class | `DefaultContentTypeAvailablilityService` | `DefaultContentTypeAvailabilityService` |
| Class | `VirutalPathResolverExtensions` | `VirtualPathResolverExtensions` |
| `VisitorGroupOptions` | `StatisticsPersistanceInterval` | `StatisticsPersistenceInterval` |
| `SearchWordCriterionOptions` | `SearchStringRegexpression` | `SearchStringRegex` |
| `PropertyLinkCollection` | `PrincipalAcccessor` | `PrincipalAccessor` |
| `PlugInSummaryAttribute` constructor | `moreInfourl` param | `moreInfoUrl` |
| `ContentScannerExtension.AssignAvailableTypes` | `contentypeModel` param | `contentTypeModel` |
| `IFileTransferObject.CheckIn` | `commment` param | `comment` |

---

## Blob changes

- `EPiServer.Framework.Blobs.Blob` is now **abstract**. Methods `OpenRead()`, `OpenWrite()`, `Write(Stream)`, `AsFileInfoAsync()` changed from virtual to abstract. New abstract methods: `Exists()`, `ExistsAsync()`, `OpenReadAsync()`, `OpenWriteAsync()`, `WriteAsync()`.
- New `BlobNotFoundException` thrown by `Blob.OpenRead()` / `Blob.OpenReadAsync()` when blob doesn't exist.
- `FileBlobProvider`: default constructor and constructors with `IPhysicalPathResolver` removed. Use the constructor with `FileBlobProviderOptions`.

---

## Validation service registration

`IValidate<T>` implementations are **no longer auto-registered**. Register explicitly:

```csharp
services.AddCmsValidator<MyValidator>();
```

---

## InitializationEngine changes

- `IInitializationEngine` interface removed (was not used in practice)
- `Assemblies` property removed → use `AppDomain.CurrentDomain.GetAssemblies()` and filter dynamic assemblies
- `Set` method removed from `Modules` property
- `Set` method removed from `Locate` property — use `context.Services` instead
- `ScanAssemblies`, `BuildTypeScanner`, `GetDependencySortedModules`, `ConfigureModules` methods removed
- `Configure()` called multiple times now throws `InvalidOperationException` — use `Configure(complete: false)` instead
- `FrameworkInitialization` and `DataInitialization` no longer implement `IConfigurableModule` — services registered via `AddCmsFramework()` / `AddCmsData()`
- `HostType.LegacyMirroringAppDomain` removed (mirroring no longer supported)
- `AssemblyList` obsolete class removed
- `InitializableModuleAttribute.UninitializeOnShutdown` removed — uninitialize is always called on shutdown

---

## Removed framework APIs

- `EPiServer.Framework.EnvironmentOptions.BasePath` — removed
- `EPiServer.Framework.SmtpOptions.SenderEmailAddress` and `SenderDisplayName` — removed (never used; see `NotificationOptions` for notification sender config)
- `EPiServer.Framework.Timers.ITimer` — obsolete; no implementation is registered by the Framework
- `EPiServer.Web.IWebHostingEnvironment` / `EPiServer.Web.WebHostingEnvironment` — obsolete (`WebRootPath` available from `Microsoft.AspNetCore.Hosting.IWebHostEnvironment`)
- `EPiServer.Web.VirtualPathResolver` — obsolete
- `EPiServer.Framework.Web.Resources.ClientResources.Render` and `RenderRequiredResources` — removed
- `EPiServer.Framework.Security.ValidateAntiForgeryReleaseToken` — removed; use `Microsoft.AspNetCore.Mvc.ValidateAntiForgeryTokenAttribute`
- `AspNetAntiforgery` and `AspNetAntiforgeryOptions` — removed
- `EPiServer.Framework.Security.ISiteSecretManager` — made obsolete (existing secrets can be read but not created)
- `EPiServer.ServiceLocation.ServiceCollectionExtensions.AddServiceAccessor` non-generic overload removed — use `AddServiceAccessor<T>()` instead

### `EPiServer.Framework.FileSystem` — removed classes

The following obsolete FileSystem classes were removed:
- `IDirectory`, `IFile`, `IFileSystemWatcher`, `PhysicalDirectory`, `PhysicalFile`

### `EPiServer.Security.SecurityEntityProvider` — sync methods removed

Use the async equivalents:

| Removed | Use instead |
|---|---|
| `GetRolesForUser` | `GetUsersInRoleAsync` |
| `FindUsersInRole` | `FindUsersInRoleAsync` |
| `Search` | `SearchRolesAsync` / `SearchUsersByNameAsync` / `SearchUsersByEmailAsync` |

Extension methods `SearchRoles`, `SearchUsersByName`, `SearchUsersByEmail`, `GetUsersInRole`, `FindUsersInRole` from `SecurityEntityProviderExtensions` are also removed.

---

## Caching changes

`EPiServer.CacheManager` is obsolete. Use the `EPiServer.Framework.Cache.ISynchronizedObjectInstanceCache` service directly instead.

`ISynchronizedObjectInstanceCache.SynchronizationFailedStrategy` setting is no longer applied to the cache.

---

## Event system changes

- `EPiServer.Events.EventMessage.VerificationData` and `SiteId` were removed. The event provider is now responsible for message security and integrity.
- `IEventMessageFactory.Create` no longer takes a parameter to add a checksum.
- `EPiServer.Events.Providers.EventProvider.ValidateMessageIntegrity` setting is no longer used.
- `EPiServer.Events.EventsServiceKnownTypeAttribute` replaced by `services.TryAddCmsEventsParameterType<T>()`.
- `EPiServer.Events.EventsServiceKnownTypesLookup` replaced by `EPiServer.Events.EventProviderOptions.ParameterTypes`.
