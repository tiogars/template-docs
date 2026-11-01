# Task and Taxonomy API Reference

The API manages tasks, reusable tags, and hierarchical nomenclatures at the
singular paths `/api/v1/task`, `/api/v1/tag`, and `/api/v1/nomenclature`.
Swagger UI is published at `/swagger-ui.html`.

All business endpoints require JWT authentication. Obtain a token with
`POST /api/v1/auth/login`, then send it as `Authorization: Bearer <token>`. The
forced administrator account is configured with `APP_ADMIN_USERNAME` and
`APP_ADMIN_PASSWORD` at API startup.

## Resource Schemas

### Shared Fields

Tasks, tags, and nomenclatures share these identity, business, and audit fields.

| Field              | JSON type        | Required      | Rules                                                |
| ------------------ | ---------------- | ------------- | ---------------------------------------------------- |
| `id`               | number           | Response only | Server-generated Java `Long`.                        |
| `code`             | string           | Yes           | Unique among active rows and immutable.              |
| `label`            | string           | Yes           | Non-blank and mutable.                               |
| `description`      | string or `null` | No            | Mutable.                                             |
| `createdDate`      | string           | Response only | Server-owned `Instant` in ISO-8601 format.           |
| `lastModifiedDate` | string           | Response only | Server-owned `Instant` in ISO-8601 format.           |
| `createdBy`        | string           | Response only | Authenticated username at creation.                  |
| `lastModifiedBy`   | string           | Response only | Authenticated username at the last persisted change. |

### Task Fields

| Field | JSON type | Rules |
| --- | --- | --- |
| `status` | string | `TODO`, `IN_PROGRESS`, `DONE`, or `CANCELLED`; default `TODO`. |
| `priority` | string | `LOW`, `NORMAL`, `HIGH`, or `URGENT`; default `NORMAL`. |
| `dueDate` | string or `null` | Nullable calendar date in `YYYY-MM-DD` format, without a timezone. |
| `assignedTo` | string or `null` | Nullable free-text assignee; not a user foreign key. |
| `tagIds` | number array | Existing tag IDs (`Long[]`); default `[]`. |
| `nomenclatureId` | number or `null` | Nullable existing nomenclature ID. |

### Reusable Taxonomy Domains

`Tag` provides flat labels that can be attached to multiple tasks, such as
`RELEASE` or `DOCUMENTATION`. It has only the shared fields.

`Nomenclature` provides a reusable classification hierarchy, such as
**Engineering → Release → Documentation**. It has the shared fields plus
nullable `parentId` (a server-generated `Long` referencing another nomenclature).
`null` denotes a root. Self-parenting and longer cycles are rejected.

Task tag/nomenclature references and nomenclature parent references must exist.
Deleting a tag used by a task, a nomenclature used by a task, or a nomenclature
with children is rejected rather than silently removing references.
Invalid references and hierarchy cycles return `400 Bad Request`; deleting a
referenced taxonomy entry returns `409 Conflict`.

Task and taxonomy reads require `ADMIN` or `USER`; writes require `ADMIN`.

## Endpoints

### Authentication and Administration

| Method   | Path                                 | Description                                                                             |
| -------- | ------------------------------------ | --------------------------------------------------------------------------------------- |
| `POST`   | `/api/v1/auth/login`                 | Exchange username/password for a JWT.                                                   |
| `GET`    | `/api/v1/auth/me`                    | Read the authenticated user.                                                            |
| `GET`    | `/api/v1/auth/users`                 | List users. Requires `ADMIN`.                                                           |
| `POST`   | `/api/v1/auth/users`                 | Create a user with a BCrypt-stored password. Requires `ADMIN`.                          |
| `GET`    | `/api/v1/auth/users/{id}`            | Read a user. Requires `ADMIN`.                                                          |
| `PUT`    | `/api/v1/auth/users/{id}`            | Update username, enabled flag, roles, and optionally password. Requires `ADMIN`.        |
| `DELETE` | `/api/v1/auth/users/{id}`            | Delete a user. Requires `ADMIN`.                                                        |
| `POST`   | `/api/v1/auth/users/delete-selected` | Delete the users listed in `{"ids":[...]}`. Requires `ADMIN`.                           |
| `GET`    | `/api/v1/auth/roles`                 | List roles. Requires `ADMIN`.                                                           |
| `POST`   | `/api/v1/auth/roles`                 | Create a role. Requires `ADMIN`.                                                        |
| `GET`    | `/api/v1/auth/roles/{id}`            | Read a role. Requires `ADMIN`.                                                          |
| `PUT`    | `/api/v1/auth/roles/{id}`            | Update a role label. Requires `ADMIN`.                                                  |
| `DELETE` | `/api/v1/auth/roles/{id}`            | Delete a non-system role. Requires `ADMIN`.                                             |
| `POST`   | `/api/v1/auth/roles/delete-selected` | Delete the roles listed in `{"ids":[...]}`; system roles are skipped. Requires `ADMIN`. |

The `ANONYMOUS`, `ADMIN`, and `USER` roles are seeded at first startup and
cannot be deleted. Bulk delete endpoints respond `200 OK` with
`{"deletedIds":[...],"skippedIds":[...]}`; unknown IDs and protected rows are
reported as skipped. User responses never include passwords or password hashes.

### User Settings

User settings are implemented in the `fr.tiogars.domaintemplate.settings.user`
package. The first setting is `viewMode` (`light`, `dark`, or `system`; default
`system`).

| Method | Path                                   | Description                                                                                          |
| ------ | -------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `GET`  | `/api/v1/settings/user/me`             | Read the authenticated user's settings. Requires `USER` or `ADMIN`.                                  |
| `PUT`  | `/api/v1/settings/user/me`             | Update the authenticated user's settings. Requires `USER` or `ADMIN`.                                |
| `GET`  | `/api/v1/settings/user/users/{userId}` | Read the settings of a user with the `USER` role. Requires `ADMIN`; returns `404` for other users.   |
| `PUT`  | `/api/v1/settings/user/users/{userId}` | Update the settings of a user with the `USER` role. Requires `ADMIN`; returns `404` for other users. |
| `GET`  | `/api/v1/settings/user/anonymous`      | Read the `ANONYMOUS` role settings. Public.                                                          |
| `PUT`  | `/api/v1/settings/user/anonymous`      | Update the `ANONYMOUS` role settings. Requires `ADMIN`.                                              |

Settings that were never saved are returned with default values; they are stored
on first update and deleted with their user.

### Tasks

| Method   | Path                                        | Description                                                                 |
| -------- | ------------------------------------------- | --------------------------------------------------------------------------- |
| `GET`    | `/api/v1/task`                    | List all tasks, ordered by code.                                          |
| `GET`    | `/api/v1/task/{id}`               | Read one task by server ID.                                               |
| `POST`   | `/api/v1/task`                    | Create a task.                                                           |
| `PUT`    | `/api/v1/task/{id}`               | Update its mutable fields.                                                  |
| `DELETE` | `/api/v1/task/{id}`               | Archive one task without removing its database row.                       |
| `POST`   | `/api/v1/task/delete-selected`    | Archive the tasks listed in `{"ids":[...]}`. Requires `ADMIN`.            |
| `POST`   | `/api/v1/task/seed`               | Upsert an array by code.                                                    |
| `POST`   | `/api/v1/task/import.json`        | Import and upsert a JSON array by code.                                     |
| `POST`   | `/api/v1/task/import.csv`         | Import and upsert CSV rows by code.                                         |
| `GET`    | `/api/v1/task/export.json`        | Export all tasks as JSON.                                                 |
| `GET`    | `/api/v1/task/export.csv`         | Export all tasks as CSV.                                                  |
| `GET`    | `/api/v1/task/{id}/print.pdf`     | Print one active task as PDF.                                             |
| `GET`    | `/api/v1/task/print.pdf`          | Print all active tasks as PDF, ordered by code.                           |
| `POST`   | `/api/v1/task/print-selected.pdf` | Print the active tasks listed in `{"ids":[...]}` as PDF, ordered by code. |

### Tags and Nomenclatures

Both taxonomy resources expose the following CRUD operations, where `{domain}`
is `tag` or `nomenclature`:

| Method | Path | Description |
| --- | --- | --- |
| `GET` | `/api/v1/{domain}` | List entries. |
| `GET` | `/api/v1/{domain}/{id}` | Read an entry. |
| `POST` | `/api/v1/{domain}` | Create with `code`, `label`, optional `description`, and optional `parentId` for nomenclatures. |
| `PUT` | `/api/v1/{domain}/{id}` | Update `label`, `description`, and optional `parentId` for nomenclatures; exclude `code`. |
| `DELETE` | `/api/v1/{domain}/{id}` | Delete an unreferenced entry. |

Create taxonomy entries before tasks that reference them. For example, create a
tag with `{"code":"RELEASE","label":"Release","description":null}` and a root
nomenclature with
`{"code":"ENGINEERING","label":"Engineering","description":null,"parentId":null}`.
Use the returned IDs, not codes, in task references and child `parentId` values.

## PDF Printing

PDF printing uses Apache PDFBox 3.0.7 and three dedicated services for one, all,
and selected tasks. All three endpoints require the same JWT
authentication and `ADMIN` or `USER` role as reading tasks and return
`application/pdf`, inline `Content-Disposition`, and `Cache-Control: no-store`.

Each record starts on a new A4 page and includes its ID, code, label,
description, and audit fields. Long fields wrap and continue onto additional
numbered pages. The embedded Liberation Sans font supplied by PDFBox supports
accented Latin, Greek, and Cyrillic text; unsupported characters return
`422 Unprocessable Content` instead of silently changing the content. Printing
an empty collection returns a PDF stating that there are no tasks.

The selected request accepts 1 to 1000 positive IDs. Duplicate IDs print only
once. Invalid selections return `400 Bad Request`; an unknown or archived ID
returns `404 Not Found` for the entire request, so a PDF never silently omits
selected records. Archived records are excluded from all PDF queries.

In the React backend console, **Print all** prints every active record,
independently of grid pagination or filtering. **Print selected** prints only
the checkbox-selected grid rows, including selections across pages, and is
disabled when nothing is selected. **Print** on the detail page prints the
current saved record. Each action opens the generated PDF in a new browser tab;
use the PDF viewer's native Print control to choose a printer. Popups must be
allowed for the site. Failed requests close the blank viewer and display an
error in the console.

The React client's `print(id)`, `printAll()`, and `printSelected(ids)` methods
return `Promise<Blob>`. Binary requests use the configured API URL, fetcher, and
token provider directly, outside the JSON-oriented RTK Query cache, avoiding
non-serializable PDF objects in Redux state.

## Create

`POST /api/v1/task` accepts only business fields:

```json
{
    "code": "RELEASE-001",
    "label": "Prepare release",
    "description": "Review the release checklist",
    "status": "TODO",
    "priority": "HIGH",
    "dueDate": "2026-11-01",
    "assignedTo": "alice",
    "tagIds": [],
    "nomenclatureId": null
}
```

The response is `201 Created`, includes the complete resource, and sets
`Location` to `/api/v1/task/{id}`. A duplicate code returns
`409 Conflict`. Blank required fields return `400 Bad Request`.

## Read

`GET /api/v1/task` returns a JSON array.
`GET /api/v1/task/{id}` returns one task or `404 Not Found`.

## Update

`PUT /api/v1/task/{id}` accepts the mutable fields only; `code` is excluded:

```json
{
    "label": "Prepare release",
    "description": null,
    "status": "IN_PROGRESS",
    "priority": "HIGH",
    "dueDate": "2026-11-01",
    "assignedTo": "alice",
    "tagIds": [],
    "nomenclatureId": null
}
```

The server preserves `id`, `code`, and `createdDate`, and replaces
`lastModifiedDate`. The response is `200 OK`, or `404 Not Found` for an unknown
ID.

## Delete

`DELETE /api/v1/task/{id}` returns `204 No Content`, or
`404 Not Found` for an unknown or already archived ID.

`POST /api/v1/task/delete-selected` archives several tasks in one
transaction. The body is `{"ids":[1,2,3]}` and the response is
`{"deletedIds":[1,2],"skippedIds":[3]}`, where skipped IDs were unknown or
already archived. The admin console uses it for the grid's "Delete selected"
action.

Deletion sets `deleted_date` and `deleted_by`; the original ID, code, business
fields, and creation metadata remain in PostgreSQL. Archived rows are excluded
from all normal JPA queries, including read, update, list, export, duplicate
checks, and seed/import lookups. They cannot be updated or restored through the
API.

After deletion, creating the same code produces a new row with a new ID.
Seed/import also creates a new row rather than reviving the archived row.
Multiple archived rows may share a code, but a PostgreSQL partial unique index
permits at most one active row per code. Optimistic versioning prevents stale
writers from overwriting a deletion or another update.

Concurrent writes that lose an optimistic-lock race, or violate active-code
uniqueness, return `409 Conflict`; other database integrity errors are not
treated as duplicate-code conflicts. Refresh the resource before retrying.

## Persistence Auditing and Required Migration

Tasks, tags, and nomenclatures have server-owned auditing. Task deletion archives
the task; taxonomy deletion is subject to reference checks. Users, roles,
and user settings retain their existing behavior. Spring Data JPA supplies
creation and modification timestamps, truncated to PostgreSQL microsecond
precision, and records the authenticated username:

| Database column    | Meaning                                                    |
| ------------------ | ---------------------------------------------------------- |
| `created_by`       | Username at creation; never changed.                       |
| `last_modified_by` | Username at the last persisted change, including deletion. |
| `deleted_by`       | Username that archived the row, or `NULL` while active.    |
| `deleted_date`     | Archival timestamp, or `NULL` while active.                |
| `version`          | Optimistic concurrency token managed by JPA.               |

Actors are username snapshots, not foreign keys: changing or deleting a user
does not erase the recorded name. Writes without an authenticated security
context fail; scheduled jobs must establish an explicit service identity. JSON
responses (including JSON exports) expose `createdBy` and `lastModifiedBy` as
read-only fields, displayed alongside their timestamps in the backend console
detail view. Historical actors backfilled by the migration appear as `unknown`.
Deletion metadata and the optimistic-lock version remain persistence-only; the
CSV fields are described below. Spring Data auditing records the creator and
most recent editor, not a full revision history.

**Before deploying this API version, stop API writers, back up the database, and
run
[the manual PostgreSQL task-domain migration](https://github.com/tiogars/template-java-solution/blob/main/java/api/src/main/resources/db/manual/task-domains.sql).**
Run it for both existing and new databases, using the same schema/search path as
the API. For example, with PostgreSQL connection variables configured:

```powershell
psql -v ON_ERROR_STOP=1 -f java\api\src\main\resources\db\manual\task-domains.sql
```

The task-domain migration defines the task and taxonomy persistence schema.
Hibernate `ddl-auto: update` is not a replacement for this migration and does
not remove obsolete tables or migrate legacy business data automatically.
Review the script against a database backup before upgrading.

This script replaces the former auditing migration for the
legacy `domain_templates` schema. Archived task history is persistence-only and is not exposed by normal
API endpoints.

The PostgreSQL lifecycle tests require Docker and run with the normal API test
suite:

```powershell
mvn -B -f java\pom.xml -pl api test
```

## Seed and Import

Seed and both import formats use upsert semantics keyed by `code`: a missing
code creates a task and an existing code updates all mutable task fields.
Client-supplied `id`, `createdDate`, `lastModifiedDate`, `createdBy`, and
`lastModifiedBy` values are ignored. The server keeps ownership of identifiers,
timestamps, and audit actors.

`POST /api/v1/task/seed` and `POST /api/v1/task/import.json`
accept an array. A minimal document is:

```json
[
    {
        "code": "RELEASE-001",
        "label": "Prepare release",
        "description": null
    }
]
```

Both return `201 Created` with the stored resources.

Omitted workflow fields use the task defaults; optional date, assignee, and
nomenclature values may be `null`. Imports may include all mutable fields,
including `tagIds`; taxonomy references must already exist in the same database.
Exported numeric taxonomy IDs are not portable to another database unless the
same referenced entries exist there with matching IDs.

## Import and Export Formats

JSON export returns the complete resource schema. The same exported JSON can be
imported: server-owned fields are accepted only as transport data and ignored
during the upsert.

```json
[
    {
        "id": 42,
        "code": "RELEASE-001",
        "label": "Prepare release",
        "description": "Review the release checklist",
        "status": "TODO",
        "priority": "HIGH",
        "dueDate": "2026-11-01",
        "assignedTo": "alice",
        "tagIds": [],
        "nomenclatureId": null,
        "createdDate": "2026-08-30T17:00:00Z",
        "lastModifiedDate": "2026-08-30T17:00:00Z",
        "createdBy": "admin",
        "lastModifiedBy": "admin"
    }
]
```

CSV import and export use this header:

```csv
id,code,label,description,createdDate,lastModifiedDate,status,priority,dueDate,assignedTo,tagIds,nomenclatureId
```

The 12-column header and column order above round-trip all task business fields.
`tagIds` is a semicolon-separated list of positive IDs (for example `3;7`);
an empty cell means `[]`. Empty `description`, `dueDate`, `assignedTo`, and
`nomenclatureId` cells become `null`. Blank status and priority use `TODO` and
`NORMAL`. Date values use `YYYY-MM-DD`; referenced IDs must already exist.
On import, `id`, `createdDate`, and `lastModifiedDate` are ignored. Audit actors
are not included in CSV.

```csv
id,code,label,description,createdDate,lastModifiedDate,status,priority,dueDate,assignedTo,tagIds,nomenclatureId
42,RELEASE-001,Prepare release,Review the release checklist,2026-08-30T17:00:00Z,2026-08-30T17:00:00Z,TODO,HIGH,2026-11-01,alice,,
```

For backward-compatible import, the exact legacy header
`id,code,label,description,createdDate,lastModifiedDate` is also accepted. Its
rows use the new task-field defaults; export always writes the extended header.

## Response

```json
{
    "id": 42,
    "code": "RELEASE-001",
    "label": "Prepare release",
    "description": "Review the release checklist",
    "status": "TODO",
    "priority": "HIGH",
    "dueDate": "2026-11-01",
    "assignedTo": "alice",
    "tagIds": [],
    "nomenclatureId": null,
    "createdDate": "2026-08-30T17:00:00Z",
    "lastModifiedDate": "2026-08-30T17:05:00Z",
    "createdBy": "admin",
    "lastModifiedBy": "editor"
}
```

## Clients

The `java-client`, `react-client`, and `dart-client` modules provide typed Java,
TypeScript, and Dart access to CRUD, seed, import, and export operations.
Construct each client with the service origin; the client appends
`/api/v1/task`. Taxonomy clients use `/api/v1/tag` and
`/api/v1/nomenclature`. Application package names and artifact coordinates remain
unchanged.

The `react-client` also generates RTK Query endpoints and exports an
`AuthClient` for login, current-user lookup, and user and role administration
under `/api/v1/auth`, and a `UserSettingsClient` for user settings under
`/api/v1/settings/user`.
