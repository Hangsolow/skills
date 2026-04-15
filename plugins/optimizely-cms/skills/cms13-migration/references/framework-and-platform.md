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
| `LanguageBranch` | `ValidateLanguageEdititingAccessRights` | `ValidateLanguageEditingAccessRights` |
| Class | `DefaultContentTypeAvailablilityService` | `DefaultContentTypeAvailabilityService` |
| Class | `VirutalPathResolverExtensions` | `VirtualPathResolverExtensions` |
| `VisitorGroupOptions` | `StatisticsPersistanceInterval` | `StatisticsPersistenceInterval` |
| `SearchWordCriterionOptions` | `SearchStringRegexpression` | `SearchStringRegex` |
| `PropertyLinkCollection` | `PrincipalAcccessor` | `PrincipalAccessor` |

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

- `Assemblies` property removed → use `AppDomain.CurrentDomain.GetAssemblies()` and filter dynamic assemblies
- `ScanAssemblies`, `BuildTypeScanner`, `GetDependencySortedModules`, `ConfigureModules` methods removed
- `Configure()` called multiple times now throws `InvalidOperationException` — use `Configure(complete: false)` instead
- `FrameworkInitialization` and `DataInitialization` no longer implement `IConfigurableModule` — services registered via `AddCmsFramework()` / `AddCmsData()`
