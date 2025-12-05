# Metabase Customization Context

This document provides context for customizations made to this Metabase fork for future reference.

---

## Feature: Granular Native Query Permissions

### Summary
Implemented the ability to set "Query Builder and Native" permissions at the individual table level when using granular data source permissions. Previously, native query access was an all-or-nothing setting at the database level.

### Problem
When using granular permissions, administrators could only set tables to "Query Builder Only" or "No Access". There was no way to grant native SQL query access to specific tables while restricting others. Users with native query access to one table could effectively query any table in the database.

### Solution
Extended the permissions system to support `query-builder-and-native` at the table level with proper enforcement.

### Background

Metabase has a permissions system that controls data access at multiple levels:
- **Database level**: Full access to all tables
- **Schema level**: Access to all tables in a schema
- **Table level**: Granular access to specific tables

For query creation, there are two permission levels:
- **Query Builder Only** (`query-builder`): User can use the visual query builder
- **Query Builder and Native** (`query-builder-and-native`): User can use both query builder AND write raw SQL

### Original Limitation

Previously, native query access was effectively database-wide. When granular table permissions were enabled:
1. The UI only showed "Query Builder Only" option at table level
2. If a user had native access to ANY table, they could write SQL against ANY table
3. No enforcement existed to validate which tables were referenced in native SQL queries

### Architecture Overview

#### Permission Storage
- Permissions stored in `data_permissions` table
- `perm_type`: e.g., `"perms/create-queries"`, `"perms/view-data"`
- `perm_value`: e.g., `"query-builder-and-native"`, `"query-builder"`, `"no"`
- `table_id`: NULL for database-level, populated for table-level permissions

#### Key Files

**Frontend (Permission UI):**
```
frontend/src/metabase/admin/permissions/
├── constants/data-permissions.tsx    # Permission option definitions
├── selectors/data-permissions/
│   ├── schemas.ts                    # Database/schema level permissions
│   ├── tables.ts                     # Table level permissions
│   └── fields.ts                     # Field level permissions
└── types.ts                          # TypeScript types for permissions
```

**Backend (Permission Enforcement):**
```
src/metabase/
├── permissions/
│   ├── api/permission_graph.clj      # API schema validation (Malli)
│   └── models/data_permissions.clj   # Permission model & queries
└── query_permissions/impl.clj        # Query execution permission checks
```

### Implementation Details

#### 1. UI Changes (tables.ts, fields.ts)

**Before:**
```typescript
options: _.compact([
  dbValue === DataPermissionValue.QUERY_BUILDER_AND_NATIVE &&
    DATA_PERMISSION_OPTIONS.queryBuilderAndNative,  // Only shown if DB has native
  DATA_PERMISSION_OPTIONS.queryBuilder,
  ...
]),
```

**After:**
```typescript
options: _.compact([
  DATA_PERMISSION_OPTIONS.queryBuilderAndNative,  // Always available
  DATA_PERMISSION_OPTIONS.queryBuilder,
  ...
]),
```

#### 2. Schema Validation (permission_graph.clj)

Added `query-builder-and-native` to `Perms` enum and added decoder:
```clojure
(def ^:private Perms
  [:enum {:decode/perm-graph keyword}
   :all :segmented :none ... :query-builder :query-builder-and-native :no :blocked])
```

#### 3. Permission Checking (impl.clj)

**Key Functions:**

```clojure
;; Get tables where user has native query permission
(defn- get-user-native-query-table-ids [db-id]
  ;; Returns table IDs with perm_value = "query-builder-and-native"
  )

;; Check if user has ANY native-enabled table (to show native editor)
(defn- user-has-native-query-table-perms? [perm-type db-id]
  ;; Returns true if at least one table has query-builder-and-native
  )

;; Validate SQL query references only allowed tables
(defn- validate-native-query-tables [query db-id]
  ;; 1. Extract table names from SQL using regex
  ;; 2. Map table names to table IDs
  ;; 3. Check all referenced tables are in user's allowed set
  ;; 4. Return {:valid? true/false :unauthorized-tables #{...}}
  )
```

**Permission Check Flow:**
```
Native Query Execution
        │
        ▼
has-perm-for-db? ──────────────────────────────────┐
        │                                          │
        ▼                                          │
Has full DB native perms? ──Yes──► Allow           │
        │                                          │
        No                                         │
        │                                          │
        ▼                                          │
Has any table with native perms? ──No──► Deny      │
        │                                          │
        Yes                                        │
        │                                          │
        ▼                                          │
validate-native-query-tables()                     │
        │                                          │
        ▼                                          │
All tables in SQL have native perms? ──No──► Deny  │
        │                                          │
        Yes                                        │
        │                                          │
        ▼                                          │
      Allow ◄──────────────────────────────────────┘
```

### Important Technical Notes

1. **Qualified Names**: Permission types are stored with namespace (`"perms/create-queries"`). Must use `(u/qualified-name :perms/create-queries)` not `(name :perms/create-queries)` which only returns `"create-queries"`.

2. **SQL Parsing Limitations**: Table extraction uses regex patterns:
   - Handles `FROM table`, `JOIN schema.table`, `table AS alias`
   - May not catch all edge cases (CTEs, subqueries, dynamic SQL)
   - Fails closed (denies access) on parsing errors

3. **Malli Decoders**: JSON values come as strings. Added `{:decode/perm-graph keyword}` to convert `"query-builder-and-native"` → `:query-builder-and-native`.

### Testing Scenarios

| Scenario | Table A (native) | Table B (query-builder) | Expected |
|----------|------------------|------------------------|----------|
| `SELECT * FROM A` | ✅ | - | ✅ Allow |
| `SELECT * FROM B` | - | ✅ | ❌ Deny |
| `SELECT * FROM A JOIN B` | ✅ | ✅ | ❌ Deny |
| `SELECT * FROM A, B` | ✅ | ✅ | ❌ Deny |
| Open native editor | ✅ | - | ✅ Allow |

### Commits
- `92560e4` - Add granular table-level native query permissions
- `b793d7c` - Add query-builder-and-native to table-level permission schema
- `3bc0f03` - Add decoder to permission enums for proper string-to-keyword conversion
- `5b834af` - Fix perm_type query to use qualified-name instead of name

### Future Considerations

1. **SQL Parsing**: Current regex-based parsing is basic. Consider using actual SQL parser for better accuracy.

2. **View/Function Support**: Currently only validates table references. Views, functions, and stored procedures may bypass checks.

3. **Cross-Database Queries**: Logic assumes single database context. May need extension for cross-database scenarios.

4. **Audit Logging**: Consider logging when queries are blocked due to table-level restrictions.

5. **Error Messages**: Could provide more specific error messages indicating which tables are unauthorized.

---

## Feature: Audit Log for Ad-hoc Queries

### Summary
Added audit logging for ad-hoc (unsaved) queries to track query execution.

### Commit
- `8017a53` - Added audit log for adhoc queries

---

*Last Updated: December 2024*
