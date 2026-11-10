# Deployment

The application is split across several repositories. Each one publishes its own
artifact; the [`template-docker`](https://github.com/tiogars/template-docker)
repository assembles them in a single `docker-compose.yml` deployed by Dokploy.

| Repository                    | Published artifact                                 | Workflow                           |
| ----------------------------- | -------------------------------------------------- | ---------------------------------- |
| `template-java-api`           | `ghcr.io/tiogars/template-java-api` image          | `Docker Publish`                   |
| `template-react-backend`      | `ghcr.io/tiogars/template-react-backend` image     | `Build and publish Docker image`   |
| `template-flutter`            | GitHub Release with Android, Linux, Windows builds | `Flutter releases` (on `v*` tags)  |
| `template-docs-mkdocs`        | Documentation on GitHub Pages                      | `Build and deploy docs`            |
| `template-presentation`       | Slides on GitHub Pages                             | `Publish presentation`             |

Images are tagged `latest` (default branch), `sha-<commit>` and, for `v*` tags,
with the version.

## Services

`docker-compose.yml` declares three services and publishes no host port:

- `postgres`: PostgreSQL 17 with the persistent `postgres-data` volume. On the
  first start of an empty volume, it runs `postgres/initdb/01-task-domains.sql`,
  a copy of `template-java-api/src/main/resources/db/manual/task-domains.sql`.
- `api`: Spring Boot API on container port `8080`; healthy once
  `/actuator/health` answers `UP`.
- `backend`: web console served by Nginx on container port `8080`. It starts
  after the API is healthy.

## Deploy with Dokploy

1. Create a **Docker Compose** service with the GitHub provider, repository
   `tiogars/template-docker`, branch `main` and compose path `./docker-compose.yml`.
2. In **Environment**, set the variables listed below (see `.env.example`).
3. In **Domains**, assign one domain to `backend` (port `8080`) and one to `api`
   (port `8080`), with HTTPS.
4. Deploy. Dokploy pulls the images from GHCR; they must be public, otherwise
   register `ghcr.io` credentials (token with `read:packages`) in Dokploy.

To deploy a new image, rerun the deployment (or enable auto-deploy). Pin
`API_IMAGE_TAG` and `BACKEND_IMAGE_TAG` to `sha-<commit>` or a version for
reproducible deployments.

| Variable                     | Required | Purpose                                                              |
| ---------------------------- | -------- | -------------------------------------------------------------------- |
| `POSTGRES_PASSWORD`          | Yes      | Database password.                                                   |
| `POSTGRES_DB`, `POSTGRES_USER` | No     | Database name and user; default `template`.                          |
| `APP_ADMIN_USERNAME`         | No       | Forced administrator username; defaults to `admin`.                  |
| `APP_ADMIN_PASSWORD`         | Yes      | Forced administrator password, stored only as a BCrypt hash.         |
| `APP_JWT_SECRET`             | Yes      | JWT HMAC signing secret, at least 32 characters.                     |
| `APP_JWT_EXPIRATION_SECONDS` | No       | Access token lifetime; defaults to 3600 seconds.                     |
| `APP_CORS_ALLOWED_ORIGINS`   | Yes      | Comma-separated console origins, e.g. `https://console.example.com`. |
| `API_BASE_URL`               | Yes      | Public API URL used by the console, e.g. `https://api.example.com`.  |
| `DOCS_BASE_URL`              | No       | Documentation URL shown in the console.                              |
| `GITHUB_BASE_URL`            | No       | Repository URL shown in the console.                                 |
| `API_IMAGE_TAG`, `BACKEND_IMAGE_TAG` | No | Image tags; default `latest`.                                      |

Compose refuses to start when a required variable is missing.

### Console runtime configuration

The console image serves `/config.js`, generated at container start from
`API_BASE_URL`, `DOCS_BASE_URL` and `GITHUB_BASE_URL`. The same image can
therefore target any environment without being rebuilt. An empty value falls
back to the `VITE_*` value compiled into the bundle.

```bash
docker run --rm -p 5173:8080 -e API_BASE_URL=https://api.example.com \
  ghcr.io/tiogars/template-react-backend:latest
curl http://localhost:5173/config.js
# window.__APP_CONFIG__ = { apiBaseUrl: "https://api.example.com", docsBaseUrl: "", githubBaseUrl: "" };
```

## Local Test of the Stack

```bash
cd template-docker
cp .env.example .env
docker compose -f docker-compose.yml -f compose.local.yml up -d --wait
```

`compose.local.yml` publishes the API on `http://localhost:8989` and the console
on `http://localhost:5173`, and sets the matching URLs. Dokploy only uses
`docker-compose.yml`.

## Persistence Behavior

The API persists tasks, reusable tags, and hierarchical nomenclatures. The Compose volume is named from the project and the declared `postgres-data` volume; Docker Compose resolves the final volume name automatically.

The API also persists local authentication data in `auth_roles`, `auth_users`, and `auth_user_roles`. At startup it seeds the `ANONYMOUS`, `ADMIN`, and `USER` roles, then forces the administrator account from `APP_ADMIN_USERNAME` and `APP_ADMIN_PASSWORD`. If that username already exists, the account is enabled, its password hash is replaced from the environment value, and the `ADMIN` role is guaranteed.

Hibernate uses `ddl-auto: update`, but automatic schema updates do not replace
the task-domain migration, migrate legacy business data, or remove obsolete
schema objects. The init script only runs on an empty volume: apply
`task-domains.sql` manually on an existing database before upgrading. Follow
the [migration instructions](api.md#persistence-auditing-and-required-migration).

## Reset the Demo Database

Reset the volume when you want to remove all demo data and obsolete tables. This
operation is destructive: it removes all task, taxonomy, authentication, and
settings data; it is not an upgrade procedure.

```bash
docker compose down --volumes
docker compose up -d --wait
```

The init script recreates the task-domain schema on the new volume. In Dokploy,
stop the service and delete the `postgres-data` volume before redeploying.

## Publish the Flutter Applications

The console download page (`/downloads`) lists the files served from
`/releases/`, described by `/releases/manifest.json` (version, size, SHA-256,
default API URL). Missing platforms are shown as unavailable.

1. In `template-flutter`, set the `API_BASE_URL` Actions variable to the public
   HTTPS API URL compiled into the applications (users can still change it).
2. Push a `v*` tag. The `Flutter releases` workflow builds Android and Linux on
   Ubuntu and Windows on a Windows runner, then publishes a GitHub Release with
   the packages and `build-info-<platform>.json` files.
3. Rerun the `template-react-backend` image workflow. It downloads the latest
   `template-flutter` release, generates the manifest, and embeds the files in
   the image. Without any release, the image is published without downloads.
4. Redeploy in Dokploy.

Android release builds block clear-text HTTP traffic: use an HTTPS API URL for
the APK. The APK is signed with the debug key until a release signing
configuration is added to `android/app/build.gradle.kts`.

## Build Documentation and Slides

The documentation (`template-docs` sources, `template-docs-mkdocs` configuration)
and the presentation (`template-presentation`) are published on GitHub Pages by
their own workflows; they are not part of the Compose stack. Set `DOCS_BASE_URL`
to the published documentation URL.

## Production Considerations

Keep the API, database, console, and documentation as independently deployable
units. Replace `ddl-auto: update` with versioned migrations, provide credentials
through the Dokploy environment, terminate TLS at the platform edge, and back up
the `postgres-data` volume.
