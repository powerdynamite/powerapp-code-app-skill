# Power Apps Code App — Development Skill

You are an expert in building **Power Apps Code Apps** (React + TypeScript + Vite) that connect to Microsoft Dataverse. This skill captures production-proven patterns and hard-won fixes so you never repeat the same debugging cycle twice.

When the user asks you to build, fix, or extend a Power Apps Code App, apply every relevant section below before writing any code.

---

## 1. What a Power Apps Code App Is

- A React + TypeScript + Vite SPA deployed into Power Apps via `pac code push`
- Auth is **100% handled by the Power Apps host** — zero auth code lives in the app
- All Dataverse calls go through the auto-generated `MicrosoftDataverseService` in `src/generated/services/`
- The connector routes calls through Power Apps' Dataverse connector — not raw HTTP
- Deploy: `npm run build && pac code push` — deploys `dist/` to Power Apps
- Test URL is printed by `pac code push` after each deploy

---

## 2. Project Scaffold

```
code-app/
  src/
    App.tsx                         # root state + view routing
    styles.css
    types.ts                        # shared interfaces
    services/
      myService.ts                  # your domain logic (calls MicrosoftDataverseService)
    components/
      MyComponent.tsx
    utils/
      format.ts                     # formatFieldName, formatDate helpers
    generated/
      services/
        MicrosoftDataverseService.ts  # AUTO-GENERATED — but may need targeted edits (see §4)
  .power/
    schemas/appschemas/
      dataSourcesInfo.ts            # AUTO-GENERATED — may need targeted edits (see §4)
  power.config.json
  .env
```

**.env**
```
VITE_DATAVERSE_URL=https://orgXXXXXXXX.crm.dynamics.com
```

**power.config.json** — `dataSets` key does NOT fix "Invalid organization URL 'null'". Use `WithOrganization` variants instead (see §3).

---

## 3. Always Use `WithOrganization` Variants

Non-`WithOrganization` variants (`CreateRecord`, `UpdateOnlyRecord`, `ListRecords`) rely on a connection-stored org URL that is typically null. They will fail with **"Invalid organization URL 'null' provided"**.

**Always use:**
- `ListRecordsWithOrganization`
- `CreateRecordWithOrganization`
- `UpdateOnlyRecordWithOrganization`

Pass `ORG_URL` explicitly as the first parameter:
```typescript
const ORG_URL = (import.meta.env.VITE_DATAVERSE_URL as string)?.replace(/\/$/, "")
             ?? "https://orgXXXXXXXX.crm.dynamics.com";
```

---

## 4. The SDK Body-Wrapping Bug (Critical Fix)

The connector SDK (`connectorDataOperationExecutor.js`) wraps the body param as `{ paramName: value }` **unless** the param is literally named `"body"` — in which case it sends the value directly.

The auto-generated files use `"name": "item"` for the body param, which causes Dataverse to receive `{ "item": {...} }` and reject it with:
> `"Invalid property 'item' was found in entity"`

**Fix in `dataSourcesInfo.ts`** (find by operation name, change `"name": "item"` → `"name": "body"`):
```typescript
// CreateRecordWithOrganization — body param
{ "name": "body", "in": "body", ... }   // was: "name": "item"

// UpdateOnlyRecordWithOrganization — body param  
{ "name": "body", "in": "body", ... }   // was: "name": "item"
```

**Fix in `MicrosoftDataverseService.ts`** (find the matching operations, change the params object):
```typescript
// CreateRecordWithOrganization
const params = { ..., body: item, ... };  // was: { ..., item, ... }

// UpdateOnlyRecordWithOrganization
const params = { ..., body: item, ... };  // was: { ..., item, ... }
```

---

## 5. Optional Boolean Headers Must Be Explicit `false`

Optional boolean connector headers (e.g. `suppressDuplicateDetection`, `noMetadata`) default to `undefined`. The SDK serialises `undefined` as an empty string, and the connector backend calls `Boolean.Parse("")` which throws:
> `"Expression evaluation failed. String was not recognized as a valid Boolean"`

**Fix:** Always declare these as explicit constants:
```typescript
const NO_META = false;
const NO_MIP  = false;
// Pass these everywhere, never pass undefined for boolean params
```

---

## 6. EntityDefinitions — `$top` Not Supported

The `EntityDefinitions` metadata endpoint does **not** support `$top` through the Power Apps connector. Using it returns:
> `"The query parameter '$top' is not supported"`

**Fix:** Remove `$top`. A `$filter=LogicalName eq 'tableName'` already returns exactly 1 row.

```typescript
// WRONG
ListRecordsWithOrganization(ORG_URL, "EntityDefinitions", ..., "$top=1", ...)

// CORRECT
ListRecordsWithOrganization(ORG_URL, "EntityDefinitions", ..., undefined, ...,
  "EntitySetName", "LogicalName eq 'tableName'")
```

---

## 7. GET Response Caching — Cache Buster Pattern

The Power Apps connector caches GET responses within a session. The same OData URL returns the same cached result even if Dataverse data has changed. This breaks Refresh buttons.

**Fix:** Add a semantically-valid but time-varying condition to the filter:
```typescript
const nowIso = new Date().toISOString();
// Add to filter: makes every request URL unique without changing the result set
filter = `_objectid_value eq ${recordId} and createdon le ${nowIso}`;
```

---

## 8. Lookup Fields — Three Separate Concepts

Lookup fields in Dataverse have three different representations:

| Context | Name | Example |
|---|---|---|
| **Read (SQL/FetchXML)** | `_fieldname_value` | `_hl_approver_value` |
| **Write field logical name** | `fieldname` | `hl_approver` |
| **OData navigation property** | `ReferencingEntityNavigationPropertyName` | `hl_approver_hl_bookableresources` |

For **PATCH / `@odata.bind`** you need the **OData navigation property name** — this is NOT the same as the field logical name.

### How to Resolve the Navigation Property Name

Query the entity-scoped relationship endpoint — NOT the global `RelationshipDefinitions` (which does not expose `ReferencingEntity` as a filterable property):

```typescript
const entityPath = `EntityDefinitions(LogicalName='${tableLogicalName}')/ManyToOneRelationships`;
const result = await MicrosoftDataverseService.ListRecordsWithOrganization(
  ORG_URL, entityPath, undefined, "application/json", NO_META, NO_MIP,
  "ReferencingEntityNavigationPropertyName,ReferencingAttribute",
  `ReferencingAttribute eq '${fieldLogicalName}'`
);
const navProp = listValue(result.data)[0]?.ReferencingEntityNavigationPropertyName as string;
// navProp might be "hl_approver_hl_bookableresources", not "hl_approver"
```

Cache this result — it never changes for a given field.

### Lookup Values in Audit `changedata`

Lookup values are stored as `"entitytype,guid"` — e.g. `"hl_bookableresources,47b1c62a-..."`. The `entitytype` part may be the entity set name (plural), not the logical name (singular). Resolve the entity set name via `EntityDefinitions` if needed.

---

## 9. Setting Lookup Fields on Create vs PATCH

### WRONG — `@odata.bind` in POST body is rejected
```typescript
// Dataverse throws 0x80048d19: "undeclared property 'hl_approver'"
body["hl_approver@odata.bind"] = "/hl_bookableresources(guid)";
await CreateRecordWithOrganization(..., body, ...);
```

### CORRECT — Two-step: POST scalars first, then PATCH each lookup

```typescript
// Step 1: Create record with scalar (non-lookup) fields only
const scalarBody = { ...scalarFields, [`${tableLogicalName}id`]: originalGuid };
await CreateRecordWithOrganization(..., entitySet, scalarBody, ...);

// Step 2: PATCH each lookup using the REAL navigation property name
const navProp = await getNavigationPropertyName(tableLogicalName, fieldLogicalName);
await UpdateOnlyRecordWithOrganization(
  "return=representation", "application/json", "*",
  ORG_URL, entitySet, recordId,
  { [`${navProp}@odata.bind`]: `/${entitySet}(${lookupGuid})` },
  NO_META
);
```

### Preserve Original GUID on Recreate

Include the primary key in the POST body — Dataverse honours it:
```typescript
body[`${tableLogicalName}id`] = originalRecordGuid;
```

---

## 10. Audit Table Patterns

**Entity name:** `audits`  
**Key fields:** `auditid`, `operation` (1=Create, 2=Update, 3=Delete, 4=Access), `createdon`, `_userid_value`, `changedata`, `_objectid_value`

**Expand user name:**
```typescript
ListRecordsWithOrganization(ORG_URL, "audits", ...,
  "auditid,operation,createdon,_userid_value,changedata",
  `_objectid_value eq ${recordId} and createdon le ${nowIso}`,
  "createdon desc",
  "userid($select=fullname)"   // expand
)
```

**`changedata` JSON format** (only present on Update events):
```json
{ "changedAttributes": [
    { "logicalName": "hl_brand", "oldValue": "Dell Inc", "newValue": "Dell Technologies",
      "oldName": null, "newName": null }
] }
```

**Deduplication** — Dataverse emits two audit rows per delete (entity + cascade). Dedupe by normalising to the second:
```typescript
const key = `${e.operation}|${e.createdOn.slice(0, 19)}`;
```

---

## 11. Field Name Display Helper

```typescript
// src/utils/format.ts
export function formatFieldName(logicalName: string): string {
  // Strip publisher prefix (e.g. "hl_", "cr7_", "new_")
  const stripped = logicalName.replace(/^[a-z]{2,4}_/, "");
  return stripped.split("_")
    .map(w => w.charAt(0).toUpperCase() + w.slice(1))
    .join(" ");
}

export function formatDate(iso: string): string {
  return new Date(iso).toLocaleString(undefined, {
    month: "short", day: "numeric", year: "numeric",
    hour: "2-digit", minute: "2-digit"
  });
}
```

---

## 12. IOperationResult Pattern

All `MicrosoftDataverseService` calls return `IOperationResult<T>`:
```typescript
interface IOperationResult<T> { success: boolean; data: T; error?: Error; }
```

Always check `result.success` and throw a descriptive error on failure:
```typescript
if (!result.success) throw new Error(`Operation failed: ${JSON.stringify(result.error)}`);
const rows = (result.data?.["value"] as Record<string, unknown>[]) ?? [];
```

---

## 13. Deployment Checklist

Before every `pac code push`:
1. `npm run build` — must pass TypeScript + Vite with zero errors
2. Verify `.env` has the correct `VITE_DATAVERSE_URL`
3. `pac auth list` — confirm correct account is active
4. `pac code push` — note the play URL for testing

**Never use `undefined` for boolean connector params** — always pass `false` explicitly.  
**Never add `$top` to `EntityDefinitions` queries.**  
**Always use `WithOrganization` variants.**

---

## 14. Common Error → Fix Reference

| Error | Cause | Fix |
|---|---|---|
| `"Invalid organization URL 'null'"` | Using non-WithOrg variant | Switch to `*WithOrganization` variant |
| `"String was not recognized as a valid Boolean"` | `undefined` boolean header | Pass explicit `false` constant |
| `"Invalid property 'item' was found in entity"` | SDK wraps body as `{item: body}` | Rename param to `"body"` in `dataSourcesInfo.ts` + service |
| `"$top is not supported"` | EntityDefinitions rejects $top | Remove `$top`, use `$filter` only |
| `"The property 'fieldname' does not exist on type"` | Using field logical name as nav property | Query `ManyToOneRelationships` for `ReferencingEntityNavigationPropertyName` |
| `"undeclared property 'field' ... only has annotations"` | `@odata.bind` in POST body | Use two-step: POST scalars → PATCH each lookup separately |
| Refresh button shows stale data | Connector caches GET responses | Add `createdon le ${new Date().toISOString()}` to filter |
| Duplicate audit events | Dataverse logs two rows per Delete | Dedupe by `operation\|createdOn.slice(0,19)` |

---

## 15. Power Apps Connector Limitations (Confirmed)

- `RelationshipDefinitions` (global): **not filterable** by `ReferencingEntity`
- `EntityDefinitions`: **no `$top`** support
- `@odata.bind` in **POST** body: **rejected** by Dataverse (use two-step)
- GET responses: **cached per session** (add timestamp to bust)
- Non-`WithOrganization` variants: **fail** when connection has no fixed org URL
- Boolean headers: `undefined` **serialises as empty string** → connector throws
