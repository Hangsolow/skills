# Property Attributes Reference (Optimizely CMS)

---

## `EPiServer.DataAnnotations` namespace

| Attribute | Effect | Default |
|---|---|---|
| `[CultureSpecific]` or `[CultureSpecific(true)]` | Property has a separate value per language branch | Not culture-specific |
| `[CultureSpecific(false)]` | Explicitly marks property as language-neutral | — |
| `[Searchable]` | Value is indexed for full-text search | `string` properties are searchable; all others are not |
| `[BackingType(typeof(T))]` | Overrides the auto-detected `PropertyData` backing type | Auto-detected (see property-types.md) |
| `[Ignore]` | CMS ignores the property entirely — no `PropertyData` backing | Not ignored |
| `[AllowedTypes(Include = ..., Restrict = ...)]` | Restricts content types that can be added to `ContentArea`, `ContentReference`, or `ContentReferenceList` properties | All types allowed |

> `[Searchable]` is obsolete in CMS 13. Use `[IndexingType(IndexingType.IncludedByDefault)]` from `EPiServer.DataAnnotations` instead. The `[Searchable(bool)]` constructor overload remains but produces a compiler warning.

---

## `System.ComponentModel.DataAnnotations` namespace

| Attribute | Effect | Default |
|---|---|---|
| `[Display(Name, Description, GroupName, Order, Prompt)]` | Sets `EditCaption`, `HelpText`, tab (`GroupName`), sort position, and placeholder text in the editor | Name = property name; others null |
| `[Required]` | Editor must supply a value before saving | Not required |
| `[StringLength(max)]` | Maximum character count for `string` properties. Not supported on `XhtmlString`. | No restriction |
| `[Range(min, max)]` | Validates range for numeric and `DateTime` properties | No restriction |
| `[RegularExpression(pattern)]` | Validates input format (typically `string` properties) | No validation |
| `[UIHint(hint)]` | Selects a specific editor or renderer. Use `EPiServer.Web.UIHint.*` constants for built-in hints. | Default editor for the type |
| `[ScaffoldColumn(false)]` | Hides the property from the edit view (not shown to editors) | Visible |
| `[Editable(false)]` | Marks property as read-only in the editor | Editable |

### `UIHint` constants (`EPiServer.Web.UIHint`)

| Constant | Selects |
|---|---|
| `UIHint.Image` | Image picker |
| `UIHint.Video` | Video picker |
| `UIHint.MediaFile` | Generic media picker |
| `UIHint.Textarea` | Multi-line text area |
| `UIHint.BlockFolder` | Block folder picker |
| `UIHint.MediaFolder` | Media folder picker |

---

## `EPiServer.Cms.Shell.UI.ObjectEditing` namespace

| Attribute | Effect | Default |
|---|---|---|
| `[ReloadOnChange]` | Reloads the entire editing context when this property's value changes. Use for properties that other properties depend on. Does not work in Quick Edit dialog or during content creation (nothing to reload yet). | No reload |

---

## `EPiServer.DataAnnotations` — list item validation

These apply to each item inside an `IList<T>` property:

| Attribute | Effect |
|---|---|
| `[ListItems(max)]` | Maximum number of items in the list |
| `[ItemRangeAttribute(min, max)]` | Range validation on each item |
| `[ItemStringLength(max)]` | String length limit on each item |
| `[ItemRegularExpression(pattern)]` | Regex validation on each item |
| `[ListItemHeaderProperty(nameof(Block.Prop))]` | Property to use as the collapsed header label for each list item |

---

## `ContentTypeAttribute` properties

The `[ContentType]` attribute itself accepts several properties that control how the type appears in the edit UI and admin view:

| Property | Effect | Default |
|---|---|---|
| `GUID` | Stable identifier — allows renaming the class without orphaning content. **Always set this.** | None (new GUID generated) |
| `DisplayName` | Friendly name shown in the UI | Class name |
| `Description` | Tooltip or help text for editors | None |
| `GroupName` | Content type group (for organizing in the create-page dialog) | None |
| `Order` | Sort position within the group | 0 |
| `AvailableInEditMode` | Set to `false` to hide this type from editors when creating shared blocks | `true` |

> **CMS 13 constraint:** The `Name` value of `ContentTypeAttribute` must match `^[A-Za-z][_0-9A-Za-z]+` and be 2–255 characters. Invalid names silently prevent the content type from being registered during initialization.

---

## `AvailableContentTypes` attribute

Restricts which content types editors can create as children of a page, or what types can be added to a `ContentArea`. Applied at the content type level (not the property level).

```csharp
[AvailableContentTypes(Include = new[] { typeof(ArticlePage), typeof(LandingPage) })]
public class SectionPage : PageData { }
```

---

## `Access` attribute

Restricts which users or roles can create instances of the content type in the edit UI.

```csharp
[Access(Roles = "CmsEditors,CmsAdmins")]
public class RestrictedPage : PageData { }
```
