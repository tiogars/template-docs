# Deployment

The local deployment uses six services:

- `postgres`: PostgreSQL 17 with a persistent named volume;
- `api`: Spring Boot application available on port `8989` locally. The base Compose file does not publish the API port; `make docker-up` adds the local override that publishes it;
- `backend`: static Vite build served by Nginx on port `5173`.
- `docs`: MkDocs site published on port `8000`.
- `java-site`: Maven Site with aggregated Javadocs published on port `8083`.
- `presentation`: Marp presentation published on port `8082`.

## Start the Stack

Before starting the API, apply
`java/api/src/main/resources/db/manual/task-domains.sql` to the target database,
including for a new database. Follow the
[migration instructions](api.md#persistence-auditing-and-required-migration).

```bash
make docker-up
```

After startup:

- Documentation: `http://localhost:8000`
- Java Maven Site: `http://localhost:8083`
- Java API documentation: `http://localhost:8083/apidocs/`
- API health: `http://localhost:8989/actuator/health`
- Swagger UI: `http://localhost:8989/swagger-ui.html`
- Management console: `http://localhost:5173`

The local Compose file supplies a development administrator account and JWT secret. The default console login is `admin` / `admin`.

Deployments on platforms such as Dokploy should use `docker/compose.yml` alone. The API remains available to other Compose services on port `8080`, with no host port published. The local development override is `docker/compose.local.yml`.

Inspect service logs with:

```bash
make docker-logs
```

Stop containers while retaining PostgreSQL data:

```bash
make docker-down
```

## Persistence Behavior

The API persists tasks, reusable tags, and hierarchical nomenclatures. The Compose volume is named from the project and the declared `postgres-data` volume; Docker Compose resolves the final volume name automatically.

The API also persists local authentication data in `auth_roles`, `auth_users`, and `auth_user_roles`. At startup it seeds the `ANONYMOUS`, `ADMIN`, and `USER` roles, then forces the administrator account from `APP_ADMIN_USERNAME` and `APP_ADMIN_PASSWORD`. If that username already exists, the account is enabled, its password hash is replaced from the environment value, and the `ADMIN` role is guaranteed.

Hibernate uses `ddl-auto: update`, but automatic schema updates do not replace
the task-domain migration, migrate legacy business data, or remove obsolete
schema objects. A volume created by an earlier version may retain
`domain_templates` or `domain_records`. The older
`domain-template-auditing-soft-delete.sql` migration remains legacy documentation
for the former schema; use `task-domains.sql` for this application version.

## Reset the Demo Database

Reset the volume when you want to remove all demo data and obsolete tables. This operation is destructive.

```bash
docker compose -f docker/compose.yml -f docker/compose.local.yml down --volumes
docker compose -f docker/compose.yml -f docker/compose.local.yml up -d postgres
```

After PostgreSQL is ready, apply the task-domain migration using the database
connection settings, then run `make docker-up` to start the full stack. A reset
removes all task, taxonomy, authentication, and settings data; it is not an
upgrade procedure.

## Build Documentation and Slides

```bash
make docs-build
make presentation-build
```

The host commands `make docs-build` and `make docs-serve` use the lightweight
`docs/mkdocs.yml` configuration with MkDocs Material. Both read `docs/src` and
write generated output to `site_output`.

The Docker documentation service uses
`ghcr.io/tiogars/mkdocs-docker-image:latest` with the richer
`docs/mkdocs/mkdocs.yml` configuration, including PDF generation. The full `docs`
directory is mounted read-only at `/server/docs` so the configuration, hooks,
assets, overrides, templates, and Markdown sources are all available.
`site_output` is mounted separately at `/server/site_output` for generated files.
The server watches both the sources and the configuration directory.

Restart only the documentation service after changing its Compose configuration:

```bash
make docker-up-docs
```

To build HTML and PDF output with the image's bundled plugins:

```bash
docker compose -f docker/compose.yml run --rm --no-deps docs build --config-file docs/mkdocs/mkdocs.yml --strict
```

`MKDOCS_CONFIG` overrides the server configuration path inside the container.
Use a path under the mounted tree, such as `docs/mkdocs/mkdocs.yml`, rather than
a host path or `../docs/...`.

## Publish the Flutter Applications

The `backend` image also serves the Flutter application builds and a public
download page at `http://localhost:5173/downloads` (linked from the login page
and the console menu). Files are served from `/releases/`, described by
`/releases/manifest.json` (version, size, SHA-256, default API URL).

Docker Compose builds Android and Linux packages automatically and passes them
to the backend image through a named build context. No Flutter SDK or prior
artifact export is required on the host. This requires Docker Compose 2.17+
with a BuildKit builder supporting additional contexts. The first build downloads
the Flutter and Android SDKs and takes longer; subsequent builds reuse Docker's
cache.

The `flutter-artifacts` service has zero replicas: it is only a build dependency,
not a running application. Local packages in `docker/flutter-artifacts` are also
included (one sub-directory per platform); freshly built Android/Linux packages
take precedence over local copies.

```bash
# Build and start the stack, including Android APK + Linux archive
API_BASE_URL=https://api.example.com make docker-up

# Or rebuild just the backend and its Flutter build dependency
API_BASE_URL=https://api.example.com make docker-up-backend
```

Windows cannot be cross-compiled from Linux. To include it, first package it
on a Windows machine and place the resulting `windows` directory under
`docker/flutter-artifacts` on the deployment host:

```bash
make flutter-docker-build-windows API_BASE_URL=https://api.example.com  # Docker in Windows containers mode
make flutter-package-windows API_BASE_URL=https://api.example.com       # or on a Windows host with Flutter

API_BASE_URL=https://api.example.com make docker-up-backend
```

For standalone artifact exports without deploying the stack,
`make flutter-docker-build API_BASE_URL=https://api.example.com` remains available.

`API_BASE_URL` is compiled into the applications with
`--dart-define=API_BASE_URL=...` and becomes the default server shown in the
application settings; users can still change it. The `flutter-deploy-*`
targets accept the same variable. Missing platforms are shown as unavailable
on the download page.

Android release builds block clear-text HTTP traffic: use an HTTPS API URL for
the APK. The APK is signed with the debug key until a release signing
configuration is added to `flutter/application/android/app/build.gradle.kts`.

### GitHub Actions and the Published Backend Image

The `Flutter releases and backend image` workflow builds Android and Linux in
the existing Flutter Docker builder, and Windows on a native Windows runner.
It runs for pull requests, pushes to `main`, `v*` tags, and manual dispatches.
Each run uploads the packages and their `build-info.json` files as GitHub
Actions artifacts, retained for 14 days.

Configure these repository **Actions variables** before publishing:

- `API_BASE_URL`: public HTTPS API origin compiled into all three applications
  and the console. The default `https://api.example.com` is a placeholder, not a
  working API. A manual run can override it with the `api_base_url` input.
- `DOCS_BASE_URL`: public documentation URL for the console (defaults to
  `http://localhost:8000`).

After both platform jobs succeed, the workflow downloads artifacts from that
same run and supplies them as the backend Docker build's `flutter-artifacts`
named context. The image generates its release manifest and embeds all three
packages: no GitHub token, external artifact service, or mounted release
directory is needed at runtime.

On the default branch or a `v*` tag, the workflow publishes
`ghcr.io/tiogars/template-java-solution-backend` using the built-in
`GITHUB_TOKEN` with `packages: write` permission. Images receive a
commit reference (`sha-<full-commit-sha>`); tag builds also receive the Git tag
(for example, `v1.0.0`), and default-branch builds update `latest`.
Pull requests and manual runs on other branches build without publishing.
Ensure the repository permits Actions to publish packages; authenticate to GHCR
when pulling a private package, or make the package public.

Deploy the published image instead of rebuilding the backend from an empty
local artifact directory:

```bash
docker run -d --name tasks-backend -p 5173:8080 \
  ghcr.io/tiogars/template-java-solution-backend:latest
```

Use a commit tag for reproducible deployments. In Dokploy, configure the backend
service to use this registry image and container port `8080`. The console at
`/downloads` and files at `/releases/` then come directly from that image.
The API remains a separate service; the compiled API URL must point to it.
The local `make docker-up-backend` path builds Android/Linux automatically and
includes any locally supplied Windows package.
Application versions continue to come from `flutter/application/pubspec.yaml`;
update its version before publishing a new application version. Android signing
and HTTPS requirements above also apply to CI builds.

## Production Considerations

Keep the API, database, console, and documentation as independently deployable units. Replace `ddl-auto: update` with versioned migrations, provide managed PostgreSQL credentials through secrets, terminate TLS at the platform edge, and set the console API base URL to the public API origin.

Provide these security variables through the deployment secret store:

| Variable                     | Required | Purpose                                                      |
| ---------------------------- | -------- | ------------------------------------------------------------ |
| `APP_ADMIN_USERNAME`         | Yes      | Forced administrator username.                               |
| `APP_ADMIN_PASSWORD`         | Yes      | Forced administrator password, stored only as a BCrypt hash. |
| `APP_JWT_SECRET`             | Yes      | JWT HMAC signing secret, at least 32 characters.             |
| `APP_JWT_EXPIRATION_SECONDS` | No       | Access token lifetime; defaults to 3600 seconds.             |
