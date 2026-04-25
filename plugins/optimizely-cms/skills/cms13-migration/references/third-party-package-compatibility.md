# Third-Party Package Compatibility (CMS 12 → CMS 13)

Known incompatible packages identified during real-world CMS 13 upgrade testing. For the most current compatibility status, check [nuget.optimizely.com](https://nuget.optimizely.com/).

---

## Package compatibility summary

| Package | CMS 12 Version | CMS 13 Status | Action |
|---|---|---|---|
| `EPiServer.Find.Cms` | 16.x | Not compatible | Remove — no CMS 13 version |
| `EPiServer.Forms` | 5.10.x | Upgrade required | Upgrade to `EPiServer.Forms` 6.0.0 |
| `EPiServer.Cms.WelcomeIntegration.UI` | 2.1.x | Not compatible | Replace with `EPiServer.Cms.DamIntegration.UI` |
| `Advanced.CMS.AdvancedReviews` | 1.4.x | Not compatible | Remove — await updated version |
| `Geta.NotFoundHandler.Optimizely` | 6.0.x | Not compatible | Remove — await CMS 13 version |
| `Geta.Optimizely.Sitemaps` | 3.2.x | Not compatible | Remove — await CMS 13 version |
| `Geta.Optimizely.GenericLinks` | 2.0.x | Not compatible | Remove — await CMS 13 version |
| `Geta.Optimizely.ContentTypeIcons` | 2.1.x | Not compatible | Remove — await CMS 13 version |
| `Geta.Optimizely.Categories` | 1.1.x | Not compatible | Remove — leaves orphaned content types |
| `Gulla.Episerver.SqlStudio` | 3.0.1 | Upgrade required | Upgrade to 3.0.2+ |
| `Stott.Optimizely.RobotsHandler` | Various | Not compatible | Remove — await updated version |

---

## Package details and removal steps

### `EPiServer.Find.Cms` (Search & Navigation)

**Why it fails**: No CMS 13-compatible version exists.

**Root cause of companion package failures**: Removing Search & Navigation may leave a missing `Newtonsoft.Json` reference because it was used as a transitive source. Also, companion packages such as `Geta.Optimizely.Categories.Find` may need to be removed.

**Removal steps:**
1. Remove `EPiServer.Find.Cms` and any related Search & Navigation packages from `.csproj`.
2. Remove service registrations (e.g., `services.AddFind()`).
3. Remove or replace Search & Navigation dependent code (search implementations, indexing jobs).
4. Add explicit Newtonsoft.Json reference if your project still uses it:
   ```xml
   <PackageReference Include="Newtonsoft.Json" Version="13.0.3" />
   ```

---

### `EPiServer.Forms`

**Why it needs upgrading**: Version 5.10.x is not compatible with CMS 13.

**Fix**: Upgrade to `EPiServer.Forms` 6.0.0, which is CMS 13-compatible.

---

### `EPiServer.Cms.WelcomeIntegration.UI`

**Why it fails**: Version 2.1.x references geolocation APIs that were moved to `EPiServer.Cms.DamIntegration.UI` in CMS 13.

**Fix**:
1. Remove the package reference and disable `AddDAMUi()` service registration.
2. Install `EPiServer.Cms.DamIntegration.UI`.
3. See [Configure DAM Asset picker](https://docs.developers.optimizely.com/content-management-system/v13.0.0-CMS/docs/configure-dam-asset-picker-cms13) for configuration details.

---

### `Advanced.CMS.AdvancedReviews`

**Why it fails**: References `ServiceCollectionExtensions` types that were moved from `Microsoft.Extensions.DependencyInjection` to `EPiServer.DependencyInjection` in CMS 13.

**Fix**: Remove the package and its service registrations. Re-add when a compatible version is released.

---

### `Geta.NotFoundHandler.Optimizely`

**Why it fails**: Uses the `SortIndex` attribute property that was removed in CMS 13, causing a `TypeLoadException` at runtime.

**Fix**: Remove the package references and service registrations. Re-add when a CMS 13-compatible version is available.

---

### `Geta.Optimizely.Sitemaps`

**Why it fails**: Access modifier conflicts with CMS 13. The `PropertySEOSitemaps` property causes a `TypeLoadException`. Also, `SitemapOptions.EnableLanguageDropDownInAdmin` was removed.

**Fix**: Remove the package. Re-add when a CMS 13-compatible version is available.

---

### `Geta.Optimizely.GenericLinks`

**Why it fails**: `TypeLoadException` caused by `PropertyLinkDataCollection` access modifier conflicts with CMS 13's changes to `PropertyLongString`.

**Fix**: Remove the package. Revert custom `LinkDataCollection<CustomLinkData>` properties to standard `LinkItemCollection`. Re-add when a CMS 13-compatible version is available.

---

### `Geta.Optimizely.ContentTypeIcons`

**Why it fails**: Build errors against CMS 13 APIs.

**Fix**:
1. Remove the package reference.
2. Comment out service registrations.
3. Remove `[ContentTypeIcon]` attributes from content type files.
4. Re-add when a CMS 13-compatible version is available.

---

### `Geta.Optimizely.Categories`

**Why it fails**: `ContentReferenceListEditorDescriptor` constructor signatures changed in CMS 13.

**Fix**: Remove the package. This leaves orphaned content types (`CategoryRoot`, `CategoryData`) in the database — see [Orphaned content types](#orphaned-content-types) below.

> Note: If you also use `Geta.Optimizely.Categories.Find`, remove that package too.

---

### `Gulla.Episerver.SqlStudio`

**Why it fails**: Version 3.0.1 references `ISynchronizedObjectInstanceCache` in a way incompatible with CMS 13.

**Fix**: Upgrade to version 3.0.2 or later. Ensure the service registration `services.AddSqlStudio()` is present; if the package is referenced, it must be registered or removed entirely.

---

### `Stott.Optimizely.RobotsHandler`

**Why it fails**: References types incompatible with CMS 13.

**Fix**: Remove the package or await an updated version. If you keep it, ensure both `services.AddRobotsHandler()` and `app.UseRobotsHandler()` are present — missing service registrations cause authorization policy errors.

---

## Generic procedure for incompatible packages

When a third-party package is not compatible with CMS 13:

1. Remove the package reference from `.csproj`.
2. Remove or comment out service registrations in `Startup.cs` / `Program.cs` (e.g., `services.AddSomePackage()` and `app.UseSomePackage()`).
3. Remove `using` statements referencing the package's namespaces.
4. Remove attributes and decorators from content types that reference the package.
5. Check for transitive dependencies — removing one package may expose missing references (e.g., `Newtonsoft.Json` after Search & Navigation removal).
6. Document what was removed so you can re-enable packages when CMS 13-compatible versions are available.
7. Watch for orphaned database artifacts — see below.

---

## Orphaned database artifacts

### Orphaned scheduled jobs

Removing packages that registered scheduled jobs (Search & Navigation, Forms, NotFoundHandler, Sitemaps) may leave orphaned job registrations in the database. These produce startup warnings but are **not blocking**.

**Resolution**: Delete orphaned registrations in the admin UI under **Settings** > **Scheduled Jobs**.

### Orphaned content types

Removing packages that registered content types (e.g., `Geta.Optimizely.Categories` registers `CategoryRoot` and `CategoryData`) may cause "could not create instance" errors on pages that reference those content types.

**Resolution options**:
1. Add stub classes matching the orphaned content type names to prevent instantiation errors:
   ```csharp
   [ContentType(GUID = "...original-guid...")]
   public class CategoryRoot : PageData { }

   [ContentType(GUID = "...original-guid...")]
   public class CategoryData : PageData { }
   ```
2. Clean up the orphaned content types directly in the database.
3. Use the admin UI to manage orphaned types if accessible.
