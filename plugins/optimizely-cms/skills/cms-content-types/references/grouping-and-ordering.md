# Grouping and Ordering Content Types and Properties

---

## Content type groups

Use `GroupName` and `Order` on `[ContentType]` to organize page types (and other content types) in the *create page* dialog and admin view.

```csharp
[ContentType(GroupName = "Basic pages", Order = 1,
    DisplayName = "Standard Page",
    GUID = "abad391c-5563-4069-b4db-1bd94f7a1eea")]
public class StandardPage : PageData { }

[ContentType(GroupName = "Campaigns", Order = 2,
    DisplayName = "Landing Page",
    GUID = "b8fe8485-587d-4880-b485-a52430ea55de")]
public class LandingPage : PageData { }
```

Content type groups are separate from property groups — they do not appear as tabs in the All Properties view.

---

## Property groups (tabs)

Use `[Display(GroupName = "...")]` to assign a property to a named tab in the **All Properties** editing view. Use `Order` to control the position of properties within the tab.

```csharp
[Display(Name = "Author", GroupName = "Details", Order = 10)]
public virtual string Author { get; set; }

[Display(Name = "Category", GroupName = "Details", Order = 20)]
public virtual string Category { get; set; }
```

Tabs with no properties are not shown in the UI.

---

## Built-in tab names (`SystemTabNames`)

Use the constants from `EPiServer.DataAbstraction.SystemTabNames` to place properties on the built-in CMS tabs:

| Constant | Actual group name stored | Sort index | Notes |
|---|---|---|---|
| `SystemTabNames.Content` | `"Information"` | 10 | Default tab; used when no `GroupName` is set |
| `SystemTabNames.Settings` | `"Advanced"` | 30 | Built-in advanced settings tab |
| `SystemTabNames.PageHeader` | `"EPiServerCMS_SettingsPanel"` | N/A | Above-tabs header area in the On-Page edit view |

> **Gotcha:** `Scheduling`, `Shortcut`, and `Categories` are obsoleted group names. Do not use them.

---

## Defining custom groups in code

Define group names as constants and decorate the class with `[GroupDefinitions]`:

```csharp
[GroupDefinitions]
public static class GroupNames
{
    [Display(Order = 10)]
    public const string Content = SystemTabNames.Content;

    [Display(Order = 20)]
    public const string Marketing = "Marketing";

    [Display(Order = 30)]
    public const string SEO = "SEO";
}
```

Use these constants in your `[Display(GroupName = GroupNames.Marketing)]` attributes to avoid magic strings.

Multiple classes can carry `[GroupDefinitions]`, but each group name may only be defined once across all classes. Groups defined in code **cannot be edited in the admin view**.

Groups without a defined `Order` get `Order = -1` and are shown first.

---

## Access control on property groups

Use `[RequiredAccess(AccessLevel)]` on a group constant to restrict visibility:

```csharp
[GroupDefinitions]
public static class GroupNames
{
    [Display(Order = 10)]
    public const string Content = SystemTabNames.Content;

    [Display(Order = 40)]
    [RequiredAccess(AccessLevel.Publish)]
    public const string Publishing = "Publishing";
}
```

Editors without the required access level will not see properties in that tab.

---

## Access control on content type groups

Apply `[RequiredAccess]` to the group constant when defining content type groups, or directly on the `[ContentType]` using the `[Access]` attribute:

```csharp
[Access(Roles = "CmsAdmins")]
[ContentType(GroupName = "System", GUID = "...")]
public class SystemPage : PageData { }
```

---

## Sort order rules

- Within a group, content types and properties are sorted by their `Order` value (ascending).
- Items with the same `Order` value are sorted by display name.
- Groups with no `Order` defined are displayed first (`Order = -1`).
- Content type `Order` controls position within its `GroupName`; property `Order` controls position on its tab.
