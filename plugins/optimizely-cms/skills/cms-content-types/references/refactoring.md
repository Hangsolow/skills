# Refactoring Content Types and Properties

Covers safe renaming, property type changes, and deletion.

---

## Rename a content type or property — use `MigrationStep`

Renaming a class or property in code **without a `MigrationStep`** creates a brand-new content type/property in the database. The old one remains with all existing content and is marked *missing its code* in the admin view. Existing content pages are **not** automatically migrated.

To rename and carry over existing data, create a `MigrationStep`:

```csharp
using EPiServer.DataAbstraction.Migration;

public class RenameBicycleMigration : MigrationStep
{
    public override void AddChanges()
    {
        // Rename the content type
        ContentType("Bicycle")
            .UsedToBeNamed("Velocipede");

        // Rename a property on that content type
        ContentType("Bicycle")
            .Property("PneumaticTire")
            .UsedToBeNamed("WoodenTire");
    }
}
```

> `MigrationStep` classes are **auto-detected** — no registration needed.

The `ContentType(name)` argument is the **new** name. `UsedToBeNamed(name)` is the **old** name. The engine uses string matching, so if a class is moved to a different namespace, include only the class name (not the fully-qualified name).

---

## Move a content type to a new namespace

As long as the `GUID` in `[ContentType(GUID = "...")]` stays the same, the sync engine matches by GUID and updates the association. No `MigrationStep` required for a namespace-only move.

---

## Change a property's type

Before changing a property's .NET type in code, **all** of the following must be true:

1. Neither the old nor the new type is a **block** type.
2. If the new type is `string`, no existing value exceeds 255 characters.
3. The existing value can be **assigned to** the new type, OR it can be parsed by the new type's `ParseToSelf` method.
4. The new type's backing `PropertyData` implements `System.IConvertible`.
5. The new type's backing `PropertyData.Type` is one of the first seven values of `PropertyDataType`.

If conversion fails for any row, **an error is logged and nothing is committed**.

### Type change requiring data loss (acceptable)

If the above conditions cannot be met and data loss is acceptable:

1. **Remove the property** from the code temporarily — let sync clean it up.
2. In the CMS admin view, manually delete the property (if it still exists there with data).
3. **Re-add the property** with the new type.

---

## Delete a content type

When a `ContentTypeAttribute` class is removed from the codebase:

- **No instances exist** → the content type is deleted from the database automatically.
- **Instances exist** → the content type remains in the database, marked as *missing its code* in the admin view. It is **never automatically deleted** while data exists.

---

## Delete a property

When a property is removed from the class:

- **No values stored** → the property definition is deleted from the database.
- **Values exist** → the property definition remains in the database, marked as *missing its code*. Delete it manually from the admin view when you no longer need the data.

---

## Sync and admin view precedence

During initialization, the sync engine merges code-defined settings with settings saved through the admin view. **Database settings win** — if an editor changed a property display name in the admin view, that value overrides what you have in `[Display(Name = "...")]`.

Use the **Revert to Default** button in the admin view to clear database overrides and re-apply code settings.

To disable the commit phase of sync (and require manual sync per type), set:

```csharp
options.ContentModelOptions.EnableModelSyncCommit = false;
```

---

## Add a `MigrationStep` for a property moved between base classes

If a property is removed from a derived class and added to an abstract base class (or vice versa), it appears to sync as a delete + create. Use `MigrationStep` to carry the data:

```csharp
ContentType("ArticlePage")
    .Property("MetaDescription")
    .UsedToBeNamed("MetaDescription"); // same name, but declared on the new base class
```

Even if the name hasn't changed, adding the step ensures sync treats it as an update rather than a delete + create.
