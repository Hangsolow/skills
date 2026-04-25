---
name: cms-content-types
description: "Use this skill when creating or updating page types and block types in Optimizely CMS 12 or CMS 13, adding or changing properties on content types, controlling language-specificity of properties, grouping properties under tabs, restricting available child page types or content area types, using blocks as inline properties, or safely renaming/refactoring content types and properties without losing data. USE FOR: creating PageData subclasses, creating BlockData subclasses, adding CMS properties (XhtmlString, ContentArea, ContentReference, ContentReferenceList, IList<Block>, etc.), applying property attributes (CultureSpecific, Required, UIHint, Display, AllowedTypes), grouping properties with GroupDefinitions, defining property tabs, renaming content types with MigrationStep, changing property types, understanding sync behavior. DO NOT USE FOR: CMS 12→13 upgrade migrations (use cms13-migration skill), Optimizely Commerce catalog types, custom property editors."
compatibility: "Optimizely CMS 12 and CMS 13. CMS 13-specific differences are noted inline."
---

# Optimizely CMS — Creating and Updating Page and Block Types

## Gotchas

Silent failures and hard-to-spot mistakes — read these before writing any code:

- **Properties must be `virtual`.** CMS generates a proxy class that overrides properties to read/write from the underlying property collection. Non-virtual properties will not store or load data — no error at startup, just missing data at runtime.
- **Always assign a GUID.** Without a `GUID` in `[ContentType]`, renaming the class creates a brand-new content type and orphans all existing content under the old (now code-less) type. Generate one before you start: run `scripts/new-guid.ps1` (Windows) or `scripts/new-guid.sh` (macOS/Linux).
- **Admin view settings override code settings.** After synchronization, settings changed through the CMS admin view take precedence over attributes in code. If a property display name or group was changed in the admin view, your code change won't take effect. Use **Revert to Default** in the admin view to clear overrides. See [`references/refactoring.md`](references/refactoring.md).
- **Renaming a property in code creates a new property — existing data is NOT transferred.** The sync engine sees a new property name as a new property and drops the old one (if no data). Use `MigrationStep` to carry values over. See [`references/refactoring.md`](references/refactoring.md).
- **CMS 13: ContentType `Name` must match `^[A-Za-z][_0-9A-Za-z]+`, 2–255 chars.** Invalid names silently prevent the type from registering during initialization — no exception, the type just won't appear.
- **`AvailableInEditMode = false` on blocks only used as properties.** Without it, the block appears in the shared-block creation dialog alongside shared content blocks. Set it on any block type that exists only to group properties on pages.
- **Sync does not delete types or properties with existing data.** Removing a class from code leaves the content type in the database (marked *missing its code*). This is safe — existing content is preserved — but the type will keep appearing in the admin view until manually deleted after data is cleared.
- **`[Ignore]` computed properties.** Any public property with a getter and setter that is not meant to be stored in CMS must be decorated with `[Ignore]`. Without it, CMS will try to find or create a backing `PropertyData` type and may fail at startup.
- **`IList<BlockType>` requires `virtual`.** Just like scalar properties, list-of-block properties must be `virtual`. The list property itself also needs `AvailableInEditMode = false` on the block type unless you want it as a shared block.
- **CMS 13: `PageReference` is obsolete.** Use `ContentReference` for all link and reference properties. `PageReference` still compiles in CMS 13 but produces warnings and is a breaking change candidate in future versions.

---

## Creating a page type

### Minimal example

```csharp
using EPiServer.Core;
using EPiServer.DataAnnotations;

[ContentType(
    DisplayName = "Article Page",
    GUID = "b8fe8485-587d-4880-b485-a52430ea55de",
    Description = "For articles and news posts.")]
public class ArticlePage : PageData
{
    public virtual XhtmlString Body { get; set; }
}
```

### Recommended full example

```csharp
using EPiServer.Core;
using EPiServer.DataAbstraction;
using EPiServer.DataAnnotations;
using EPiServer.SpecializedProperties;
using System.ComponentModel.DataAnnotations;

[ContentType(
    DisplayName = "Article Page",
    GUID = "b8fe8485-587d-4880-b485-a52430ea55de",
    Description = "For articles and news posts.",
    GroupName = "Editorial",
    Order = 10)]
public class ArticlePage : PageData
{
    [CultureSpecific]
    [Display(Name = "Heading", GroupName = SystemTabNames.Content, Order = 10)]
    public virtual string Heading { get; set; }

    [CultureSpecific]
    [Display(Name = "Body text", GroupName = SystemTabNames.Content, Order = 20)]
    public virtual XhtmlString Body { get; set; }

    [Display(Name = "Hero image", GroupName = SystemTabNames.Content, Order = 30)]
    [UIHint(UIHint.Image)]
    public virtual ContentReference HeroImage { get; set; }
}
```

---

## Creating a block type

### Shared block (available in asset panel)

```csharp
using EPiServer.Core;
using EPiServer.DataAbstraction;
using EPiServer.DataAnnotations;
using System.ComponentModel.DataAnnotations;

[ContentType(
    DisplayName = "Teaser Block",
    GUID = "38d57768-e09e-4da9-90df-54c73c61b270",
    Description = "Heading with image.")]
public class TeaserBlock : BlockData
{
    [CultureSpecific]
    [Display(Name = "Heading", GroupName = SystemTabNames.Content, Order = 10)]
    public virtual string Heading { get; set; }

    [Display(Name = "Image", GroupName = SystemTabNames.Content, Order = 20)]
    [UIHint(UIHint.Image)]
    public virtual ContentReference Image { get; set; }
}
```

### Property-only block (not shown in asset panel)

```csharp
[ContentType(
    GUID = "11d57768-e09e-4da9-90df-54c73c61b271",
    AvailableInEditMode = false)]
public class AddressBlock : BlockData
{
    public virtual string Street { get; set; }
    public virtual string City { get; set; }
}
```

---

## Choosing the right property type

Quick guide — if the type you need is not here, read [`references/property-types.md`](references/property-types.md) for the full lookup table including UIHint constants, `IList<T>` patterns, and backing types.

| Scenario | Use |
|---|---|
| Short plain text | `string` |
| Long plain text | `string` + `[UIHint(UIHint.Textarea)]` |
| Rich text / HTML | `XhtmlString` |
| Link to a page | `ContentReference` |
| Link to an image | `ContentReference` + `[UIHint(UIHint.Image)]` |
| Drag-drop content area | `ContentArea` |
| Multiple links | `ContentReferenceList` |
| Dropdown (single) | `string` + `[SelectOne(...)]` |
| List of blocks | `IList<TeaserBlock>` |

---

## Making properties culture-specific

Decorate with `[CultureSpecific]` for properties that should have a different value per language:

```csharp
[CultureSpecific]
public virtual string Heading { get; set; }       // different per language
public virtual ContentReference HeroImage { get; set; } // shared across all languages
```

Properties that are NOT `[CultureSpecific]` are stored once and shared across all language versions of the content. Changes in one language affect all languages.

When you need attributes beyond `[CultureSpecific]`, `[Display]`, `[UIHint]`, or `[Required]` — such as `[BackingType]`, `[Ignore]`, `[AllowedTypes]`, `[Access]`, `[Searchable]`/`[IndexingType]`, or list-item validators — read [`references/property-attributes.md`](references/property-attributes.md).

---

## Grouping properties under tabs

Use `[Display(GroupName = "...", Order = ...)]` to assign properties to tabs:

```csharp
[Display(Name = "Author", GroupName = "Details", Order = 10)]
public virtual string Author { get; set; }

[Display(Name = "Published", GroupName = "Details", Order = 20)]
public virtual DateTime PublishedDate { get; set; }
```

Use `SystemTabNames.Content` and `SystemTabNames.Settings` for the built-in CMS tabs. Define custom group constants in a `[GroupDefinitions]` class for reuse.

When you need to define custom tabs, control tab sort order, apply access restrictions on tabs, or look up `SystemTabNames` actual stored string values, read [`references/grouping-and-ordering.md`](references/grouping-and-ordering.md).

---

## Restricting available content types

Restrict which page types editors can create as children of a page type:

```csharp
[AvailableContentTypes(Include = new[] { typeof(ArticlePage), typeof(NewsPage) })]
public class SectionPage : PageData { }
```

Restrict what can be dropped into a `ContentArea`:

```csharp
[AllowedTypes(typeof(TeaserBlock), typeof(CallToActionBlock))]
public virtual ContentArea MainArea { get; set; }
```

Restrict a `ContentReference` or `ContentReferenceList`:

```csharp
[AllowedTypes(typeof(ImageFile))]
public virtual ContentReference HeroImage { get; set; }
```

---

## Using a block as a property

Add any `BlockData` subclass as a property on a page or another block:

```csharp
public class StandardPage : PageData
{
    [Display(Order = 10, GroupName = SystemTabNames.Content)]
    public virtual TeaserBlock Teaser { get; set; }
}
```

The block's properties render as inline fields within the page editor. The block data is stored, versioned, and published together with the page.

Set `AvailableInEditMode = false` on `TeaserBlock` if it should only be used this way and not as a standalone shared block.

---

## List of blocks

```csharp
[ContentType(GUID = "38d57768-e09e-4da9-90df-54c73c61b272", AvailableInEditMode = false)]
public class ContactBlock : BlockData
{
    public virtual string Name { get; set; }
    public virtual string Email { get; set; }
}

public class ContactPage : PageData
{
    [ListItemHeaderProperty(nameof(ContactBlock.Name))]
    public virtual IList<ContactBlock> Contacts { get; set; }
}
```

---

## Renaming a content type or property

Always use `MigrationStep` — otherwise existing content is orphaned.

**Rename checklist:**
1. Write the `MigrationStep` class (see example below) **before** renaming in code
2. Rename the class or property in code
3. Build and start the application — sync runs automatically and carries over existing data

```csharp
public class RenameMigration : MigrationStep
{
    public override void AddChanges()
    {
        ContentType("ArticlePage").UsedToBeNamed("BlogPost");
        ContentType("ArticlePage").Property("BodyText").UsedToBeNamed("Body");
    }
}
```

When moving a property to a different base class (same name, different declaring type) or needing to understand the full `MigrationStep` API, read [`references/refactoring.md`](references/refactoring.md).

---

## Changing a property's type safely

Before changing a property's .NET type, verify **all** of the following:

- [ ] Neither the old nor the new type is a **block** type
- [ ] If the new type is `string`, no existing value exceeds 255 characters
- [ ] Existing values can be assigned to the new type, OR the new type's `ParseToSelf` method can parse them
- [ ] The new type's backing `PropertyData` implements `System.IConvertible`
- [ ] The new type's `PropertyData.Type` is one of the first seven `PropertyDataType` values

If any condition fails → create a new property with a different name and the desired type, then migrate data manually (or accept data loss by removing the old property, deleting it in the admin view, and re-adding with the new name and type).

For the full procedure and the data-loss step sequence, read [`references/refactoring.md`](references/refactoring.md).

---

## Generating a GUID for new content types

Every new `[ContentType]` needs a unique GUID. Use the included scripts:

- **Windows (PowerShell):** `.\scripts\new-guid.ps1`
- **macOS / Linux:** `sh scripts/new-guid.sh`

Copy the output directly into `[ContentType(GUID = "paste-here")]`.

---

## Common errors and fixes

| Problem | Fix |
|---|---|
| Property always reads `null` or default | Check the property is `virtual` |
| Renaming class → type shows as *missing its code* in admin | Add a `MigrationStep` with `UsedToBeNamed` |
| Content type not appearing at startup (CMS 13) | Check `Name` matches `^[A-Za-z][_0-9A-Za-z]+`, 2–255 chars |
| Code change to display name / group not reflected | Admin view override; click **Revert to Default** on that property |
| Block showing in asset panel when it should be property-only | Add `AvailableInEditMode = false` to `[ContentType]` |
| `[AllowedTypes]` on `LinkItemCollection` has no effect | Use `ContentReferenceList` instead — `LinkItemCollection` doesn't support `AllowedTypes` |
| Property removed from code, still in admin view | Has stored data; manually delete from admin view after clearing data |
