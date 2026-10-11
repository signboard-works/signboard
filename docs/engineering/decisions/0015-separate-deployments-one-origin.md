# 0015. Separate deployments for UI and API, served from one origin

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-10 |
| **Deciders** | Stuart Meeks |

## Context

The front end is a single-page application that talks only to the public API ([ADR 0009](0009-frontend-react.md), [ADR 0010](0010-api-first.md)). Signboard is a multi-tenant service ([ADR 0004](0004-multi-tenant.md)), so the UI and the API have very different loads: the UI is static files, while the API does the processing and holds the database connections for every business.

Two earlier decisions constrain where things are served from:

- The web app signs in with a secure, HttpOnly session cookie ([ADR 0010](0010-api-first.md)). That is simplest and safest when the app and the API share an origin.
- Each business is served at `<business>.signboard.works`, and custom domains for businesses may follow ([ADR 0004](0004-multi-tenant.md)). If the API lived on a different domain, its cookie would be a third-party cookie on a business's custom domain, and browsers block those.

## Decision

The UI and the API are built, deployed and scaled separately, but every business's address serves both from one origin.

- **UI:** the Vite build output is deployed as static files to object storage behind a CDN.
- **API:** the ASP.NET Core application is deployed as its own container image, scaled independently.
- **One origin:** an edge router (a CDN or reverse proxy) sends `/api/*` on each business's address to the API, and every other path to the static files. Unknown paths fall back to `index.html`, so deep links work.
- **Deploy order:** the API deploys first, then the UI. Changes within an API version are additive only ([ADR 0010](0010-api-first.md)), so a UI that is one release behind keeps working.
- **Stale tabs:** the app detects when a newer UI version has been deployed and asks the user to reload, rather than failing in unexpected ways.
- **Self-hosting:** the same layout at small scale, as a Docker Compose file with a small reverse proxy in front of the API container and the static files. There is no separate all-in-one image.

## Options considered

1. **One host serves both**: simplest; one deployment and one version. Static file traffic consumes API capacity, every UI change restarts the API, and the two cannot be scaled or rolled back separately.
2. **Separate deployments on separate domains** (for example `api.signboard.works`): independent, but cookie sign-in needs cross-origin configuration, and custom domains for businesses would break cookie sign-in completely.
3. **Separate deployments, one origin via an edge router**: independent deployment, scaling and rollback; static files served cheaply from a CDN; cookies, CSRF protection and custom domains work as if it were one site. **Chosen.**

## Consequences

- More infrastructure: object storage, a CDN and edge routing rules, all defined as code in the [deployment design](../design/deployment.md).
- UI and API versions can differ for a short time, so additive API changes and the reload prompt are requirements, not niceties.
- A UI fix ships in seconds without touching the API, and a bad UI release rolls back on its own.
- The static host holds no secrets and runs no code, so the API is the only attack surface.
- Self-hosters need Docker Compose rather than a single container.
