# Web Console

The `backend` webapp is a React, Vite, and MUI console for tasks, reusable tags, and hierarchical nomenclatures. It identifies resources by server-generated `id` and presents business fields directly instead of editing a generic JSON payload.

## Run the Console

Start the API and the webapp from the repository root:

```bash
make java-run
make react-run
```

Open `http://localhost:5173`. The webapp reads the API base URL from `VITE_API_BASE_URL`; the Docker stack builds it with `http://localhost:8989`, while direct local development defaults to `http://localhost:8080`.

## Routes

| Route | Purpose |
| --- | --- |
| `/` | Dashboard with task metrics and recent tasks. |
| `/tasks` | List, import, export, print, and delete tasks. |
| `/tasks/new` | Create a task with a structured form. |
| `/tasks/{id}` | View and print one task by ID. |
| `/tasks/{id}/edit` | Update its mutable fields by ID. |
| `/tags` | Manage reusable task tags. |
| `/nomenclatures` | Manage reusable hierarchical classifications. |
| `/settings` | Manage your user settings; administrators also manage the `ANONYMOUS` role settings. |
| `/admin/users/{id}/edit` | Edit a user in the **Account** tab and its user settings in the **Settings** tab (administrators only). |
| `/about` | Show API, documentation, and Swagger UI links. |

## View Mode

Use the icons in the top bar or the `/settings` page to choose bright, dark, or system mode. The selected mode is saved in local storage and, when signed in, in the user settings through `/api/v1/settings/user/me`. Anonymous visitors start with the `ANONYMOUS` role settings.

The **Settings** tab of the user edit page is only available for users with the `USER` role.

## List Tasks

The Tasks page presents task identity, workflow status, priority, and business fields. Desktop and tablet layouts use MUI DataGrid. Mobile layouts provide the same view, edit, and delete actions in a compact list.

## Create or Update a Task

For creation, enter a required unique `code`, a required `label`, and an optional `description`. Choose status (`TODO`, `IN_PROGRESS`, `DONE`, or `CANCELLED`) and priority (`LOW`, `NORMAL`, `HIGH`, or `URGENT`); defaults are `TODO` and `NORMAL`. Due date is an optional `YYYY-MM-DD` date, and assignee is optional free text, not a user foreign key.

Select existing tags and an optional nomenclature. Tasks store their IDs as `tagIds` (default `[]`) and nullable `nomenclatureId`; create these reusable entries first. The server assigns ID, audit timestamps, and audit actors. Create calls `POST /api/v1/task`.

For update, the console loads `/tasks/{id}` and disables `code` because it is immutable. Only mutable fields are sent to `PUT /api/v1/task/{id}`; `code` is excluded.

## Manage Tags and Nomenclatures

Tags are flat, reusable task labels. Nomenclatures are reusable classifications
with an optional parent, for example **Engineering → Release**. Both have
`id`, `code`, `label`, `description`, `createdDate`, `lastModifiedDate`,
`createdBy`, and `lastModifiedBy`; nomenclatures additionally have nullable
`parentId`. Creation accepts code, label, description, and optional parent;
updates never send code.

The API requires references to exist and rejects self-parenting and hierarchy
cycles. A tag or nomenclature referenced by a task cannot be deleted, nor can a
nomenclature with children. Remove or change references first.

## Delete a Task

The delete action asks for confirmation, then calls `DELETE /api/v1/task/{id}`.
Task deletion archives the record; archived tasks disappear from normal views.

## View and Print

The detail page shows task fields and audit metadata. Print actions request PDF
output from the API and open it in a new browser tab. Use the PDF viewer's Print
control to choose a printer. The list also supports printing all or selected
tasks; see [PDF printing](api.md#pdf-printing).

## Import and Export

The Tasks page can seed sample tasks and import or export JSON and CSV. Seed and import upsert by `code`: existing tasks are updated and missing ones are created. Imported IDs, timestamps, and audit actors never replace server-owned values.

JSON import accepts an array of records:

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
    "lastModifiedDate": "2026-08-30T17:00:00Z"
  }
]
```

CSV import uses this header:

```csv
id,code,label,description,createdDate,lastModifiedDate,status,priority,dueDate,assignedTo,tagIds,nomenclatureId
42,RELEASE-001,Prepare release,Review the release checklist,2026-08-30T17:00:00Z,2026-08-30T17:00:00Z,TODO,HIGH,2026-11-01,alice,,
```

Tag IDs use semicolons within their CSV cell (for example `3;7`), not a JSON
array. Empty tag IDs mean `[]`; empty due date, assignee, and nomenclature mean
`null`. Blank status and priority use their defaults. The legacy six-column
header is accepted for import; exports use the extended header.

Both formats round-trip exports. Import uses code, label, description, and task
workflow/assignment/taxonomy fields; server-owned identity and audit values are
ignored. Referenced tags and nomenclatures must already exist in the same
database with matching IDs; taxonomy IDs are not automatically remapped between
environments. See the
[API format reference](api.md#import-and-export-formats) for CSV encoding.

## Coverage

Generate coverage reports for the monorepo:

```bash
make coverage
```

Reports are written to:

| Stack | Report location |
| --- | --- |
| Java API | `java/api/target/site/jacoco/index.html` |
| Java client | `java/java-client/target/site/jacoco/index.html` |
| React backend | `react/backend/coverage/index.html` |
| React client | `react/react-client/coverage/index.html` |
| Dart client | `flutter/dart-client/coverage/lcov.info` |