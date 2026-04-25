# Content Types and Property Definitions Breaking Changes (CMS 12 → CMS 13)

## Content type / property naming validation

CMS 13 enforces strict naming rules on `ContentType.Name`, `PropertyDefinition.Name`, `PropertyDefinitionType.Name`, and `TabDefinition.Name`:

- **Length**: 2–255 characters
- **Format**: `^[A-Za-z][_0-9A-Za-z]+` — must start with a letter, only letters/digits/underscores

**Auto-migration on upgrade:** Existing invalid names are migrated with a prefix and invalid characters replaced:

| Definition type | Prefix added | Invalid chars replaced with |
|---|---|---|
| Content type | `CT_` | `_` |
| Property definition | `PD_` | `_` |
| Property definition type | `PDT_` | `_` |
| Tab definition | `G_` | `_` |

**Action required:** Tab names with spaces or special characters (e.g., `"Meta Data / SEO"`) must be updated in code. Use `[Display(Name = "...")]` to keep the human-readable label:

```csharp
// Before (invalid in CMS 13)
[GroupDefinitions]
public static class GroupNames
{
    public const string MetaData = "Meta Data";
}

// After
[GroupDefinitions]
public static class GroupNames
{
    [Display(Name = "Meta Data")]
    public const string MetaData = "MetaData";
}
```

---

## Dynamic Properties removed entirely

All Dynamic Property APIs are deleted — no replacement. Remove all usages:

- `EPiServer.DataAbstraction.DynamicProperty`
- `EPiServer.DataAbstraction.DynamicPropertyCollection`
- `EPiServer.DataAbstraction.DynamicPropertyStatus`
- `EPiServer.Core.DynamicPropertyBag`
- `EPiServer.Core.DynamicPropertyCache`
- `EPiServer.Core.DynamicPropertyLookup`
- `EPiServer.Core.DynamicPropertyPage`
- `EPiServer.Core.IDynamicPropertyLookup`
- `EPiServer.Core.ContentOptions.EnableDynamicProperties` — now throws `NotSupportedException`
- `PropertyDefinition.IsDynamicProperty` — removed
- `PropertyGetHandler.PropertyHandlerWithDynamicProperties()` — removed

Associated database tables and stored procedures are also removed.

---

## `IContentTypeRepository` changes

The generic `IContentTypeRepository<T>` is removed. Use the non-generic `IContentTypeRepository`:

```csharp
// Before
private readonly IContentTypeRepository<PageType> _pageTypeRepository;
private readonly IContentTypeRepository<BlockType> _blockTypeRepository;

// After
private readonly IContentTypeRepository _contentTypeRepository;
```

**Behavior changes:**
- `Save(ContentType)` now also saves `PropertyDefinitions` on the content type
- `Save` default behavior: properties **not** in `contentType.PropertyDefinitions` are **deleted**. To keep undefined properties: pass `ContentTypeSaveOptions.KeepUndefinedPropertyDefinitions`
- `Load(string name)` is now case-insensitive
- Casing-only changes to `Name`, `EditCaption`, `HelpText`, `DefaultValue`, `EditorHint` are no longer ignored when saving
- `Required` property change on a definition is now a `Major` semantic change (was `Minor`)
- Saving with empty `ACL` no longer resets to default; only `null` resets to default
- Saving with an `ACL` entry whose access level is not `Create` now throws `ValidationException`
- `List()` now sorts by `SortOrder` first, then `Name` (previously non-deterministic for same `SortOrder`)
- `ContentType.DefaultWebFormTemplate` removed (WebForms no longer supported)

---

## `BlockTypeRepository` and `PageTypeRepository` removed

```csharp
// Before
private readonly BlockTypeRepository _blockTypeRepo;
private readonly PageTypeRepository _pageTypeRepo;

// After
private readonly IContentTypeRepository _contentTypeRepo;
// Then filter by type: _contentTypeRepo.List().OfType<BlockType>()
```

---

## `PropertyDefinitionRepository` changes

- `Save` and `Delete` made obsolete with **compilation error** — to add/update/remove a `PropertyDefinition`, load the `ContentType` and modify its `PropertyDefinitions` collection
- `Save` now throws `ValidationException` when:
  - A block property is marked `Required` or `LanguageSpecific`
  - A block property is in a list with a category set
- `CheckUsage()` and `GetUsage()`: `isDynamic` parameter removed
- `ListDynamic()` removed

Use `IPropertyDefinitionRepository` (not the concrete class `PropertyDefinitionRepository`).

---

## `PropertyDefinition` changes

- `Searchable` property obsolete → use `IndexingType`
- `IsDynamicProperty` removed

---

## `SearchableAttribute` → `IndexingTypeAttribute`

```csharp
// Before
[Searchable]
public virtual string MyProperty { get; set; }

// After
[IndexingType(IndexingType.IncludedByDefault)]
public virtual string MyProperty { get; set; }
```

The constructor accepting a `bool` is obsolete. Use `IndexingTypeAttribute` directly.

`SearchableAttribute` now inherits from `IndexingTypeAttribute`; it is still compiled but its `bool` constructor is marked obsolete.

---

## `PropertyDefinitionTypeRepository` events

Events moved to the `IPropertyDefinitionTypeEvents` interface. Inject that instead of subscribing via the repository.

---

## `IPropertyDefinitionTypeResolver` return type change

`IPropertyDefinitionTypeResolver.Resolve` now returns `PropertyDefinitionTypeResolution` (previously returned `Type`). The resolution result includes the property definition type plus metadata.

---

## `BlockPropertyDefinitionType` — single common type

CMS 13 uses a single common `BlockPropertyDefinitionType` for all blocks. The specific type is on `PropertyDefinition.ItemTypeReference`. Code that checked the concrete `BlockPropertyDefinitionType` subclass must be updated to use `ItemTypeReference` instead.

---

## `ContentCoreData` → `ContentNode`

- `EPiServer.DataAbstraction.ContentCoreData` removed → use `EPiServer.DataAbstraction.ContentNode`
- `EPiServer.DataAbstraction.IContentCoreDataLoader` removed → use `EPiServer.DataAbstraction.IContentNodeLoader`

Custom `ContentProvider` implementations overriding `CreateContentResolveResult(ContentCoreData)` must update the parameter type to `ContentNode`.

---

## `PageType` changes

Properties `FileName`, `FileNameForSite`, `ExportableFileName` removed (WebForms support dropped). `fileName` removed from constructor.

---

## `ContentDataInterceptor` / `ContentDataInterceptorHandler`

Both made internal. To register a custom accessor for property data, implement `IPropertyDataInterceptorRegistrator` instead.

---

## `ContentTypeModelRepository` (runtime model)

The `ContentTypeModelRepository` class is obsolete. Use the `IContentTypeModelRepository` service. Constructor changed to require dependencies.

---

## `TabDefinition` validation

`TabDefinition.Save` now throws `ValidationException` (not `DataAbstractionException`) when name is empty.

---

## `Activity` and content activity changes

- `Activity.RawData` property removed → use `Activity.ExtendedData` (`IDictionary<string, string>`)
- `ContentActivityFeed.Instance` static property made obsolete — inject `IContentActivityFeed` instead

---

## `ILanguageBranchRepository.Save` exception changes

`Save` now throws different exceptions than before:
- Throws `ValidationException` for duplicate language/URL segment, reserved URL segment, or missing language code
- Previously threw `DataAbstractionException` for these cases

---

## Category, Frame, ContentType, PageType, PropertyDefinitionType — localization service properties removed

The `LocalizationService` properties on these objects (`Category.LocalizationService`, `ContentType.LocalizationService`, etc.) are removed. These objects no longer hold a DI reference to `LocalizationService`; inject it directly where needed.

---

## `IContentSoftLinkRepository` — new `Load` overload

A new `Load(ContentReference, bool, CancellationToken)` overload was added. Implementations that are mock or custom must add this method.
