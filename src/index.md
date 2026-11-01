# Task Management

This monorepo implements task management from persistence to user interface. A Spring Boot API stores tasks, reusable tags, and hierarchical nomenclatures in PostgreSQL, three reusable clients expose the same contract, and a React console provides day-to-day CRUD and task bulk data operations.

## What You Will Run

The tutorial starts a complete local stack and creates one task. Tasks have a server-generated `Long` identifier, unique immutable `code`, required `label`, optional `description`, server-owned audit fields, status (`TODO` by default), priority (`NORMAL` by default), optional due date and assignee, tag IDs (`[]` by default), and an optional nomenclature ID.

Tags provide reusable flat labels. Nomenclatures provide a reusable tree through nullable `parentId`; tasks select one classification and any number of tags. The API rejects missing references, hierarchy cycles, and deletion of referenced taxonomy entries.

## Modules

| Module | Purpose |
| --- | --- |
| `api` | Spring Boot CRUD API backed by JPA and PostgreSQL. |
| `java-client` | Maven client library using Java `HttpClient`. |
| `react-client` | TypeScript client library for React or browser apps. |
| `backend` | React + Vite management console. |
| `dart-client` | Dart package for Flutter or Dart services. |
| `docs` | MkDocs project documentation. |
| `presentation` | Marp overview of the solution and its architecture. |

See [Recommended Copilot Skills](copilot-skills.md) for the skills to use when extending or maintaining each part of the monorepo.

## Supported Stack Versions

Use these maintained baseline versions for local development and CI. Newer compatible patch releases are supported unless a module pins a version explicitly.

| Technology | Version |
| --- | --- |
| Node.js | 24 |
| pnpm | 11 |
| Java | 25 |
| Maven | 3.9.16 |
| Dart SDK | 3.12.2 |
| Spring Boot | 4.1.1 |
| Vite | 8.2.2 |
| Vitest | 4.1.11 |
| Vite React plugin | 6.1.1 |
| Redux Toolkit / RTK Query | 2.12 |

## Tutorial: Run the Solution

### 1. Start the stack

From the repository root:

Apply the [task-domain database migration](api.md#persistence-auditing-and-required-migration) before starting the API, including for a new database.

```bash
make docker-up
```

Compose publishes the documentation on `8000`, the API on `8989`, and the console on `5173`.

### 2. Create a Task

Log in first (local defaults are `admin` / `admin`) and use the returned token:

```bash
TOKEN=$(curl -s -X POST http://localhost:8989/api/v1/auth/login \
	-H "Content-Type: application/json" \
	-d '{"username":"admin","password":"admin"}' | jq -r .token)

curl -X POST http://localhost:8989/api/v1/task \
	-H "$(printf 'Authorization: %s %s' Bearer "$TOKEN")" \
	-H "Content-Type: application/json" \
	-d '{"code":"RELEASE-001","label":"Prepare release","description":"Review the release checklist"}'
```

The response resembles:

```json
{
	"id": 1,
	"code": "RELEASE-001",
	"label": "Prepare release",
	"description": "Review the release checklist",
	"status": "TODO",
	"priority": "NORMAL",
	"dueDate": null,
	"assignedTo": null,
	"tagIds": [],
	"nomenclatureId": null,
	"createdDate": "2026-08-30T17:00:00Z",
	"lastModifiedDate": "2026-08-30T17:00:00Z",
	"createdBy": "admin",
	"lastModifiedBy": "admin"
}
```

The exact identifier and timestamps are assigned by the server.

### 3. Use the console

Open `http://localhost:5173`, select **Tasks**, and open the new task by its ID. From this page you can create, inspect, update, delete, seed, import, export, and print tasks. Use **Tags** and **Nomenclatures** to maintain reusable labels and classifications before assigning them to tasks.

Swagger UI is available at `http://localhost:8989/swagger-ui.html`.

### 4. Stop the stack

```bash
make docker-down
```

The PostgreSQL volume is retained. Follow [Reset the demo database](deployment.md#reset-the-demo-database) when you need a clean database.

## Develop and Verify

```bash
make setup
make test
make build
```

Generate Java, React, and Dart coverage reports:

```bash
make coverage
```

Run only the API locally against PostgreSQL on `localhost:5432`:

```bash
make java-run
```

Then open `http://localhost:8080/swagger-ui.html`.

Continue with the [API reference](api.md), [architecture explanation](architecture.md), [web console guide](web-console.md), or [deployment guide](deployment.md).