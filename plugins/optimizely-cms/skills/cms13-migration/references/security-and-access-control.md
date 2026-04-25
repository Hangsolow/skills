# Security and Access Control Breaking Changes (CMS 12 → CMS 13)

## `PrincipalInfo` changes

| Before | After |
|---|---|
| `PrincipalInfo.HasEditAccess()` | `user.IsInRole(Roles.CmsEditors)` |
| `PrincipalInfo.IsPermitted()` | Inject `PermissionService` and call `IsPermitted()` |

---

## `AccessControlEntry` and `AccessControlList`

Both now override `object.Equals` and `object.GetHashCode`. If you compare instances by reference equality, update to use `.Equals()`.

`AccessControlEntry` fields were changed to public properties. The `SID` property was removed because it is no longer used.

`AccessControlEntry` values `RecursiveReplace`, `RecursiveModify`, and `Modify` were removed because they are no longer used.

`ContentAccessControlList.WebServiceAccess` was removed.

---

## Preview tokens

Preview tokens are simplified in CMS 13. Tokens are no longer issued for specific content — the `ContentReference` argument has been removed throughout.

```csharp
// Before
var token = previewTokenService.TryGetPreviewToken(contentRef, principal);

// After — no ContentReference parameter
var token = previewTokenService.TryGetPreviewToken(principal);
```

Specific changes:
- `IPreviewTokenService.TryGetPreviewToken` — `ContentReference` parameter removed; now requires `IPrincipal` to determine the user.
- `PreviewTokenService` — constructor arguments `IPrincipalAccessor`, `IContentAccessEvaluator`, and `IContentLoader` removed.
- `PreviewToken.ContentReference` property removed.
- `PreviewToken.User` is now `IPrincipal` instead of `IIdentity`.
- `PreviewTokenContentReferenceValidation` enum removed.
- `PreviewTokenOptions.ContentReferenceValidation` property removed.

---

## `IContentSecurityRepository` events

Events were moved from `IContentSecurityRepository` to the `IContentSecurityEvents` interface. Inject `IContentSecurityEvents` to subscribe:

| Before (on `IContentSecurityRepository`) | After (on `IContentSecurityEvents`) |
|---|---|
| `ContentSecuritySaved` | `ContentSecuritySaved` |
| `ContentSecuritySaving` | `ContentSecuritySaving` |
| `ContentSecurityDeleted` | `ContentSecurityDeleted` |

---

## Approval engine enforcement

**This is a behavioral breaking change that may cause silent failures in integration tests and background jobs.**

In CMS 13, content that is under an approval definition **must go through the approval workflow before publishing**. Calling `IContentRepository.Save` with `SaveAction.Publish` on such content throws a `ValidationException`.

To bypass this validation you must:
1. Have **Administer** access rights, AND
2. Use `SaveAction.SkipApprovalValidation` (or the composite `SaveAction.SkipValidation`).

Alternatively, pass `AccessLevel.NoAccess` to `IContentRepository.Save` to skip the access level check entirely — but this bypasses all security.

`IApprovalEngine` also throws `ArgumentOutOfRangeException` (previously `IndexOutOfRangeException`) if `stepIndex` is outside the valid range.

`EPiServer.Approvals.ApprovalStepEventHandler` delegate was removed in favour of `EPiServer.Approvals.IApprovalEngineEvents`.

`IApprovalTypeRegistry` has a new `Unregister` method.

---

## `PermissionRepository` — sync methods removed

```csharp
// Before
_permissionRepository.GetPermissions(subject);
_permissionRepository.SavePermissions(subject, permissions);
_permissionRepository.DeletePermissions(subject);

// After — use async equivalents
await _permissionRepository.GetPermissionsAsync(subject);
await _permissionRepository.SavePermissionsAsync(subject, permissions);
await _permissionRepository.DeletePermissionsAsync(subject);
```
