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

## `SaveAction` renames and new skip flags

| CMS 12 | CMS 13 |
|---|---|
| `SaveAction.None` | `SaveAction.Default` |
| `SaveAction.DelayedPublish` | `SaveAction.Schedule` |

**New skip flags (CMS 13 only):**

| Flag | Effect |
|---|---|
| `SaveAction.SkipReferenceValidation` | Skip validation of content references |
| `SaveAction.SkipApprovalValidation` | Skip approval workflow enforcement (requires Administer rights) |
| `SaveAction.SkipDataValidation` | Skip data validators |
| `SaveAction.SkipValidation` | Composite: all three Skip flags combined |

> **Note:** `SaveAction.Patch` now uses `SaveAction.ForceCurrentVersion | SaveAction.SkipValidation`.

References are now validated on every save. Use `SkipReferenceValidation` only as a migration tool; leave it off in production code.

---

## `IContentSaveValidate<T>` changes

`IContentSaveValidate<T>` now requires a `SkipOption` property. Implement `ContentValidatorBase<TContent>` for the default behavior.

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

---

## `ContentProvider` obsolete

`ContentProvider` base class is now obsolete. New content provider implementations should use the replacement API (see `IContentProviderFacade`).

---

## `ContentAreaItem` changes

- Constructor no longer accepts `ContentFragment` as first argument — use `ContentAreaItem(ContentReference)` or the property initializer
- Empty `ContentAreaItem` instances are no longer saved to the database
- `IContentAreaLoader.LoadContent` has a new overload that accepts a `ContextMode` parameter

---

## `ContentFragment` changes

| Before | After |
|---|---|
| `ContentFragment.IsInlineBlockAttributeName` | `ContentFragment.InlineBlockTypeIdAttributeName` |
| `ContentFragment` implements `IRenderSettings` | Removed |
| `ContentFragment` implements `ISecurable` | Removed |

---

## `PropertyBlock<T>` changes

- No longer implements `IPersonalizedRoles`; `GetRoles()` removed
- `ItemTypeReference` type changed to `Type` (was `TypeReference`)

`IPropertyBlock.BlockPropertyDefinitionTypeID` → `IPropertyBlock.BlockPropertyDefinitionID`

---

## `PropertyXhtmlString` changes

No longer implements `IPersonalizedRoles`; `GetRoles()` removed.

---

## `PropertyData` changes

- `PropertyData.Locate` — obsolete; resolve via DI instead
- `PropertyData.Clear()` is no longer virtual; override `ClearImplementation()` instead

---

## `PropertyDataCollection` changes

`PropertyDataCollection.LanguageBranch` — obsolete.

---

## `PropertyCriteriaCollection` changes

No longer inherits `CollectionBase`; now implements `IList<PropertyCriteria>`. Any code using `CollectionBase` members directly will break.

---

## `RawContent` / `RawNameAndXml` / `RawProperty` changes

- Public fields `RawContent.RawProperty`, `RawNameAndXml.RawProperty`, `RawProperty.X` → properties (not fields)
- `RawPage` removed → use `RawContent`
- `RawProperty.PageDefinitionID` → `RawProperty.PropertyDefinitionID`

---

## `IContentVersionRepository` changes

- `List(...)` **no longer auto-counts total results**. `totalCount` returns `-1` unless you explicitly set `VersionFilter.IncludeTotalCount = true`.
- New `Delete(ContentVersion)` overload
- New `LoadCommonDraft(ContentReference, string)` and `LoadPublished(ContentReference, string)` overloads
- New `CommonDraftAssigned` event
- `IContentVersionRepositoryEx` obsoleted; `ListObsolete` moved to `IContentVersionRepository`

---

## `IVersionable.Variation` new property

New property `Variation` added to `IVersionable`. Custom `IVersionable` implementations must implement this property.

---

## `LanguageLoaderOption` default change

`LanguageLoaderOption.EvaluatePublishDates` default changed from `false` to `true`.

---

## `LanguageSelector` changes

- `LanguageSelectionSource.Master` removed
- `LanguageSelector` constructor parameter `autoFallback` removed; use `new LanguageSelector(language)` then set `.EvaluatePublishDates` explicitly if needed

---

## `ContentLanguageSettingsHandler` — must use DI

`ContentLanguageSettingsHandler` is no longer accessible as a static instance. Inject `IContentLanguageSettingsHandler` from DI.

---

## `IContentEvents` new events

- `ConvertingContentLanguage` — fired before converting a language branch
- `ConvertedContentLanguage` — fired after converting a language branch

Custom `IContentEvents` implementations must add these.

---

## `ContentOptions.AllowModifiedMasterLanguageProperties`

Saving properties that are not culture-specific in a non-master language now throws `InvalidOperationException` by default. Previously it was silently ignored. Enable `AllowModifiedMasterLanguageProperties` in `ContentOptions` to restore the old behavior.

---

## `MetaDataProperties.PageFolderID` removed

No direct replacement; use the content repository to look up folder relationships.

---

## `ContentService.Copy` behavior change

`ContentService.Copy` no longer auto-publishes the copied content. The copy is created as a draft. Publish explicitly if needed.

---

## `IContentCopyHandler` changes

Old `Copy(IContent, ContentReference)` overload removed. New overload accepts `CopyContentOptions`:

```csharp
// Before
handler.Copy(content, destination);

// After
handler.Copy(content, destination, new CopyContentOptions());
```

---

## Transfer types renamed

| CMS 12 | CMS 13 |
|---|---|
| `ITransferPageData` | `ITransferContentData` |
| `TransferPageData` | `TransferContentData` |

---

## `PageData` changes

- `PageData.IsModified` only available via `IModifiedTrackable` interface cast
- `PageData.ACL` obsolete; set access rights via `IContentSecurityRepository`
- `PageData` constructor accepting `AccessControlList` removed

---

## Edit tab values removed

`EditTab.Category`, `EditTab.Link`, `EditTab.Scheduling` removed. Use custom tab grouping instead.

---

## `PropertyUrl` link editor type properties removed

`PropertyUrl.LinkEditorType`, `PropertyDocumentUrl.LinkEditorType`, `PropertyImageUrl.LinkEditorType` removed.

---

## `IPersonalizedRoles` removed

`IPersonalizedRoles` interface and `PageDataPersonalizationExtension` removed entirely. Personalization is handled via Visitor Groups / `IVisitorGroupRole`.

---

## Visitor Groups (Audiences) not enabled by default

Visitor Groups are not registered unless you explicitly add:

```csharp
services.AddVisitorGroupsMvc();
services.AddVisitorGroupsUI();
```

without this, `IVisitorGroupRepository`, `IVisitorGroupRoleRepository`, and Visitor Group criteria will not be available.

---

## `RequiredPropertyValueException` → `ValidationException`

Saving content with invalid required properties now throws `ValidationException` instead of `RequiredPropertyValueException`. Update any `catch` blocks accordingly.
