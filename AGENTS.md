# AGENTS.md

This repository is the FINOS TraderX sample trading application. It is intentionally simple, non-production, and designed for local experimentation across a distributed set of services.

## Start here (project context)
- `README.md` for overall purpose and run modes.
- `docs/running.md` for default ports, startup sequences, and environment variables.
- `docs/README.md`, `docs/overview.md`, and `docs/flows.md` for architecture and sequence flows.
- `docs/c4/workspace.dsl` for the C4 diagram source (use Structurizr Lite to render).

## Repository map (service roots)
- `database/` (H2 database)
- `reference-data/` (Node/NestJS reference data)
- `trade-feed/` (Node/Socket.IO pub-sub)
- `people-service/` (.NET)
- `account-service/` (Java/Spring Boot)
- `position-service/` (Java/Spring Boot)
- `trade-service/` (Java/Spring Boot)
- `trade-processor/` (Java/Spring Boot)
- `web-front-end/` (Angular + React clients)
- `docs/` and `website/` (documentation site)
- `gitops/` and `ingress/` (K8s/Tilt and ingress config)

## Quick run options
- Docker Compose (full system): from repo root, `docker compose up` (see `README.md`).
- Kubernetes/Tilt: see `gitops/local/Tiltfile` and `docs/running.md` for `tilt up`.
- Manual run: see `docs/running.md` for the recommended startup sequence and port env vars.

## Service-level guidance
- Each service has a `README.md` with run details and prerequisites. Read that before editing.
- OpenAPI specs are generated at runtime (Swagger) and saved as `*/openapi.json` when you run `scripts/generate-openapi.sh`. Swagger UI is typically exposed at `/swagger-ui.html` (or service-specific Swagger routes).
- Java services use Gradle wrapper (`./gradlew`) from their service directory.
- Node services use their local `package.json` scripts.
- `web-front-end/` contains both Angular and React implementations; check each subfolder's README.

## When making changes
- Prefer small, targeted edits in the service you are touching; do not refactor cross-service behavior unless requested.
- If behavior or APIs change, refresh the OpenAPI specs via `scripts/generate-openapi.sh` and update any relevant docs in `docs/`.
- Keep the non-production, demo nature of the project in mind.

## Keeping diagrams in sync
When adding, removing, or changing services or their interactions, update **all** relevant diagrams:

1. **C4 diagram** (`docs/c4/workspace.dsl`) - The Structurizr DSL is the source of truth for architecture. PNG images are auto-rendered by GitHub Actions on push.
2. **Mermaid diagrams** in `docs/overview.md` (simplified architecture) and `docs/flows.md` (sequence diagrams).
3. **Component table** in `README.md` if adding/removing services.

The C4 DSL and Mermaid diagrams should stay consistent—if you update one, check if the other needs the same change.

## Documentation and website
The project has two content surfaces that may overlap:
1. **`docs/`** - Markdown files served by Docusaurus (architecture, flows, running instructions)
2. **`website/src/components/`** - React components for the landing page (feature highlights, intro text)

**Avoid duplication**: The landing page is marketing-style (logo, tagline, feature cards, CTAs). The `docs/` folder is the reference material. If you add content, decide which surface it belongs to—don't put the same text in both places.

**Project history**: When making **major changes** (new services, architectural shifts, significant features), add an entry to `docs/project-history.md`. Minor fixes and routine maintenance do not need to be recorded.

**Relative links in docs**: Links like `../account-service` work on GitHub. The Docusaurus remark plugin (`website/src/remark/transformRelativeLinks.js`) converts them to full GitHub URLs at build time. Keep using relative links in `docs/` markdown files.

## Docker image guidelines
- **Multi-stage builds are the default pattern** for all services. When modifying or creating a Dockerfile, use a separate builder stage and a minimal runtime stage. Do not leave single-stage builds that mix build tooling with the runtime image.
  - **Java services**: builder stage uses the full JDK image (e.g., `eclipse-temurin:21-jdk-jammy AS builder`) running `./gradlew build --no-daemon`; runtime stage uses a JRE-only image (e.g., `eclipse-temurin:21-jre-jammy`). The final image runs the built JAR directly with `java -jar`.
  - **Node.js services**: builder stage uses a full Node image (e.g., `node:23 AS builder`) running `npm install --only=production` and `npm run build` (if TypeScript); runtime stage uses Alpine (`alpine:3.21`) with only Node installed via `apk add --no-cache nodejs`.
- **Copy node_modules from the builder stage** — never re-run `npm install` in the runtime stage. Use `COPY --from=builder /usr/src/app/node_modules ./node_modules`.
- **Remove `base.Dockerfile` files** when refactoring a service to a multi-stage build — these are dev container scaffolding files that are no longer needed once production builds are self-contained.
- **`docker-compose.yml` must not contain a top-level `version` field** — Docker Compose V2 does not require it and treats it as deprecated. Remove it if present.
- **Angular frontend** should use a production Dockerfile (`Dockerfile.prod`) with an nginx-based final stage for serving the built static assets. Update `docker-compose.yml` to reference `dockerfile: Dockerfile.prod`.

## Security and dependency management
- **Dependency upgrades must be comprehensive** — when upgrading a dependency (e.g., Spring Boot), apply the change in ALL services that use it: `account-service`, `position-service`, `trade-processor`, `trade-service`. Partial upgrades that miss services are treated as failures.
- **logback-core and logback-classic must always be pinned together** — when pinning or upgrading `logback-core` in a Java service, also explicitly declare `logback-classic` at the same version in the same `build.gradle`. Spring Boot does not automatically pin both.
- **After upgrading Java dependencies, update `.github/gradle-cve-ignore-list.xml`** — update any jar filename references that embed the old version (e.g., `h2-2.2.224.jar` → `h2-2.3.232.jar`) and add new `<suppress>` blocks for any CVEs introduced by the new versions.
- **.NET CVE suppressions** go in `.github/dotnet-cve-ignore-list.xml` with a `<notes>` entry explaining the safe usage context.
- **springdoc-openapi must be kept in sync with Spring Boot** — when upgrading Spring Boot, check whether `org.springdoc:springdoc-openapi-starter-webmvc-ui` in `account-service` and `position-service` also needs a version bump.
- **socket.io-client in Java services** (`trade-processor`, `trade-service`) is pinned in `build.gradle` at `io.socket:socket.io-client`. When addressing socket.io CVEs, update this version directly rather than relying on transitive resolution.
- **NestJS v11 requires @nestjs/cli at Docker build time** — when upgrading `reference-data` from NestJS v10 to v11, add `RUN npm install @nestjs/cli` to the Dockerfile after the production install step.

## Kubernetes and service port configuration
- **Port environment variables in K8s manifests must match containerPort** — when editing `gitops/base/<service>/deployment.yaml`, verify that any `*_PORT` environment variable matches the `containerPort` in the same manifest. Known reference: `reference-data` listens on port `18085` (not `18095`).
- **GHCR image references**: K8s deployment manifests under `gitops/base/` should use `ghcr.io/finos/traderx/<service>` as the image path when pre-built images are available.

## Docusaurus remark plugin (transformRelativeLinks)
The plugin at `website/src/remark/transformRelativeLinks.js` rewrites relative links in `docs/` markdown to absolute GitHub URLs. When modifying or implementing this plugin:
- **Only rewrite single-level `../` links** — links starting with `../../` (or deeper) must be left untouched. Guard: `if (!url || !url.startsWith('../') || url.startsWith('../../')) return;`
- **Skip `.md` and `.mdx` cross-links** — Docusaurus handles internal doc navigation itself.
- **Preserve hash fragments and query strings** — split the URL on `#` and `?`, rewrite the path portion, then re-append both.
- **Use `/blob/` for files, `/tree/` for directories** based on whether the path has a known file extension.
- **Register the plugin** in `docusaurus.config.js` under the docs preset's `remarkPlugins` array, passing `repoUrl` and `branch` as options.

## Useful files by task
- Architecture/flows: `docs/overview.md`, `docs/flows.md`, `docs/c4/workspace.dsl`
- Local run and ports: `docs/running.md`
- Docker/K8s: `docker-compose.yml`, `gitops/local/Tiltfile`
- Front-end: `web-front-end/angular/README.md`, `web-front-end/react/README.md`
- Docs site: `website/README.md`, `docs/`
- CVE ignore lists: `.github/gradle-cve-ignore-list.xml`, `.github/dotnet-cve-ignore-list.xml`
- Security workflow: `.github/workflows/security.yml`
