# Property Types Reference (Optimizely CMS)

Quick lookup: what you want to model → .NET type → UIHint → editor experience.

---

## Text

| Goal | .NET type | UIHint | Editor |
|---|---|---|---|
| Short plain text (≤255 chars) | `string` | — | Single-line text input |
| Long plain text | `string` | `UIHint.Textarea` | Multi-line textarea |
| Rich text / HTML | `XhtmlString` | — | TinyMCE HTML editor |

Control length with `[StringLength(n)]` on `string` properties. `StringLength` cannot be used on `XhtmlString`.

---

## Numbers and dates

| Goal | .NET type | UIHint | Editor |
|---|---|---|---|
| Integer | `int` | — | Number slider |
| Decimal / float | `double` | — | Number input |
| Date and time | `DateTime` | — | Date-time picker |
| True / false | `bool` | — | Checkbox |

Use `[Range(min, max)]` to constrain integers, doubles, and dates.

---

## Content references and links

| Goal | .NET type | UIHint | Editor |
|---|---|---|---|
| Link to any page | `ContentReference` | — | Content picker with drag-drop |
| Link to any page (CMS 12 only — use ContentReference in CMS 13) | `PageReference` | — | Content picker |
| Link to any image | `ContentReference` | `UIHint.Image` | Image picker with drag-drop |
| Link to any video | `ContentReference` | `UIHint.Video` | Video picker with drag-drop |
| Link to any media file | `ContentReference` | `UIHint.MediaFile` | Media picker with drag-drop |
| Link to image (as URL, supports query params) | `Url` | `UIHint.Image` | Image picker with drag-drop |
| Link to media (as URL, supports query params) | `Url` | `UIHint.MediaFile` | Media picker with drag-drop |
| URL (internal or external) | `Url` | — | Link dialog with drag-drop |
| Single internal or external link with text | `LinkItem` | — | Link editor |
| Multiple links (internal and/or external) | `LinkItemCollection` | — | Link collection editor |
| Multiple content items (pages, blocks, media) | `ContentReferenceList` | — | Multi-content picker |

> `LinkItem` and `LinkItemCollection` do **not** support `[AllowedTypes]`. Use `ContentReferenceList` when you need to restrict types.

> **CMS 13:** `PageReference` is obsolete. Use `ContentReference` for all link properties.

---

## Content areas and drag-drop regions

| Goal | .NET type | UIHint | Editor |
|---|---|---|---|
| Drag-drop region for blocks, media, pages | `ContentArea` | — | Drag-drop content area |

Use `[AllowedTypes]` on `ContentArea` to restrict what editors can drop in.

---

## Selects and lists

| Goal | .NET type | Attribute | Editor |
|---|---|---|---|
| Single select from short list | `string` or `int` | `[SelectOne(SelectionFactoryType = typeof(...))]` | Dropdown |
| Multi-select from short list | `string` or `int` | `[SelectMany(SelectionFactoryType = typeof(...))]` | Checkbox list |
| Single select from long list | `string` or `int` | `[AutoSuggestion(...)]` | Auto-suggest input |

---

## List properties (`IList<T>`)

Any scalar or block type can be modeled as a list. Use `IList<T>`, `IEnumerable<T>`, or `ICollection<T>` — all are matched by the list descriptor.

```csharp
public virtual IList<string> Tags { get; set; }
public virtual IList<int> Ratings { get; set; }
public virtual IList<DateTime> ImportantDates { get; set; }
public virtual IList<XhtmlString> RichSections { get; set; }
public virtual IList<ContentReference> RelatedItems { get; set; }
```

For lists of ContentReference with a UIHint:

```csharp
[UIHint(UIHint.Image)]
public virtual IEnumerable<ContentReference> Gallery { get; set; }
```

For lists of a block type:

```csharp
public virtual IList<TeaserBlock> Teasers { get; set; }
```

> Block types used in lists should have `AvailableInEditMode = false` unless you also want them as standalone shared blocks.

Control list size with `[ListItems(max)]`. Validate items with `[ItemRangeAttribute]`, `[ItemStringLength]`, `[ItemRegularExpression]`.

Override the list item header with:

```csharp
[ListItemHeaderProperty(nameof(TeaserBlock.Heading))]
public virtual IList<TeaserBlock> Teasers { get; set; }
```

---

## Other types

| Goal | .NET type | Notes |
|---|---|---|
| Binary data (image blob) | `Blob` | Routed at `<url>/PropertyName` |
| Page type selector | `PageType` | Rarely used; selects a content type |

---

## Backing type table

The backing `PropertyData` type is auto-detected. Override with `[BackingType(typeof(...))]` only when needed.

| .NET type | Auto-detected backing type |
|---|---|
| `string` | `PropertyLongString` |
| `int` | `PropertyNumber` |
| `double` | `PropertyFloatNumber` |
| `bool` | `PropertyBoolean` |
| `DateTime` | `PropertyDate` |
| `Url` | `PropertyUrl` |
| `XhtmlString` | `PropertyXhtmlString` |
| `ContentArea` | `PropertyContentArea` |
| `ContentReference` | `PropertyContentReference` |
| `IList<ContentReference>` | `PropertyContentReferenceList` |
| `LinkItemCollection` | `PropertyLinkCollection` |
| `IList<T>` | `PropertyCollection<T>` |
| `CategoryList` | `PropertyCategory` |
| `Blob` | `PropertyBlob` |
| `PageType` | `PropertyPageType` |
