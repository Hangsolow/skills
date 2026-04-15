# Content Management and Repository Breaking Changes (CMS 12 → CMS 13)

## `IContentRepository` new methods

Three new methods were added — update any custom implementations:

- `Copy(...)` — new overload, change previous `Copy` implementations to match
- `Publish(ContentReference, ILanguageSelector)` — new method
- `ConvertLanguageBranch(...)` — new method

`MoveToWastebasket`:
- The `deletedBy` parameter was removed — current user is now read from `IPrincipalAccessor`
- The method now takes an `AccessLevel` parameter

---

## `PageReference` → `ContentReference`

`PageReference` is obsolete throughout CMS 13. Replace all usages with `ContentReference`.

Key types affected:
- `EPiServer.Core.PageData` — `ContentLink`, `ParentLink`, `ArchiveLink` now return `ContentReference`
- `ContentProvider.WastebasketReference` — returns `ContentReference`
- `ContentProvider` — `StartPage`, `RootPage`, `WasteBasket` changed to `ContentReference`
- `IPageCriteriaQueryService`, `PropertyCriteriaCollection`, `PageDataCollection`, `PageTypeConverter` — parameters updated

```csharp
// Before
PageReference startPage = SiteDefinition.Current.StartPage;

// After
ContentReference startPage = IApplicationResolver.GetByContext()?.EntryPoint
    ?? ContentReference.RootPage;
```

---

## `ContentArea` breaking changes

`ContentArea` **no longer inherits `XhtmlString`**. All string-manipulation members are obsoleted.

```csharp
// Before — render directly
Html.PropertyFor(x => x.MainContentArea)

// After — use HTML helpers or Tag Helpers (unchanged for most view scenarios)
Html.PropertyFor(x => x.MainContentArea)
```

**Changed members:**

| Before | After |
|---|---|
| `ContentArea.FilteredItems` | `ContentArea.Items` (or use `IContentAreaItemsRenderingFilter`) |
| `ContentArea.Tag` | Obsoleted (no longer used) |
| `IContentAreaLoader.Get()` | `IContentAreaLoader.LoadContent()` |
| `ContentAreaItemExtensions.GetContent()` | `ContentAreaItem.LoadContent()` |

`ContentAreaItem.RenderSettings` changed to `IDictionary<string, string>` with an initializer (not a setter).
Empty `ContentAreaItem` instances are no longer saved.

---

## `XhtmlString` changes

- `XhtmlString.ToHtmlString(IPrincipal principal)` — **obsolete with compilation error**, use HTML Helpers / Tag Helpers instead, or filter via `IEnumerable<IStringFragmentRenderingFilter>`
- `XhtmlString.ToHtmlString()` now always filters away content fragments and personalized fragments
- `StringFragmentCollection.GetFilteredFragments()` — obsolete with compilation error

---

## `ContentFragment` / personalization changes

| Before | After |
|---|---|
| `IContentGroup` | `IPersonalizedGroup` |
| `ISecuredFragmentMarkupGenerator` | `IPersonalizedFragmentMarkupGenerator` |
| `ISecuredFragmentMarkupGeneratorFactory` | `IPersonalizedFragmentMarkupGeneratorFactory` |
| `IRoleSecurityDescriptor` (for roles) | `IEnumerable<string>` |

`ContentFragment` no longer implements `ISecurable` or `IRenderSettings`.
`PersonalizedContentFragment` constructor: `ISecuredFragmentMarkupGenerator` → `IPersonalizedFragmentMarkupGenerator`.

---

## `PropertyString` / `PropertyLongString` changes

| Before | After |
|---|---|
| `PropertyString.PublicString` | `PropertyString.String` (now public) |
| `PropertyLongString.PublicLongString` | `PropertyLongString.LongString` (now public) |
| `PropertyLongString.PageLink` | `PropertyLongString.Parent.OwnerLink` |
| `PropertyUrl.String` | `PropertyUrl.LongString` (base class changed to `PropertyLongString`) |

---

## Validation system: `IValidate<T>` → explicit registration

Implementations of `IValidate<T>` are **no longer auto-discovered and registered**. Register each one explicitly:

```csharp
// In Startup.cs / service registration
services.AddCmsValidator<MyPageValidator>();
services.AddCmsValidator<MyBlockValidator>();
```

`DataAnnotationsValidator` default constructor is obsoleted — use the constructor that takes service dependencies.

---

## `SaveAction` renames

| CMS 12 | CMS 13 |
|---|---|
| `SaveAction.None` | `SaveAction.Default` |
| `SaveAction.DelayedPublish` | `SaveAction.Schedule` |

---

## `ScriptParserOptions` default behavior changes (security)

New defaults for XhtmlString HTML parsing:

| Setting | Old default | New default |
|---|---|---|
| `LoadingMode` | (permissive) | `ScriptParserMode.Remove` — illegal URIs/HTML elements/attributes removed on load |
| `SavingMode` | (permissive) | `ScriptParserMode.ThrowException` — saving content with illegal elements throws `InvalidPropertyValueException` |

New: `MediaUploadMode` and `MediaExtensionsToParse` settings — `.svg`, `.svgz`, `.html`, `.htm` are parsed by default and exceptions thrown for illegal attributes.

`AttributeNames` property removed — use `ElementAttributes` instead (allows per-element attribute rules).

---

## `ValidationException` entries changed

Entries in `ValidationException.Data` for multi-error properties are now `IList<ValidationError>` (previously single error).

---

## `PrincipalInfo` changes

| Before | After |
|---|---|
| `PrincipalInfo.HasEditAccess()` | `user.IsInRole(Roles.CmsEditors)` |
| `PrincipalInfo.IsPermitted()` | Inject `PermissionService` and call `IsPermitted()` |

---

## `SoftLink.LinkStatus` renamed

```csharp
// Before
var status = softLink.LinkStatus;

// After
var status = softLink.HttpStatusCode;
```
