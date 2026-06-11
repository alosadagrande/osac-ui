# OSAC runbook

## Local development

1. `pnpm install`
2. Start the Go proxy (requires a reachable fulfillment API):

   ```bash
   FULFILLMENT_API_URL=https://fulfillment.your-env.example.com pnpm dev:proxy
   # listens on http://localhost:8080
   ```

3. Start the SPA:

   ```bash
   pnpm dev:frontend
   # Vite on http://localhost:5173 — /api/* proxied to :8080
   ```

4. Open [http://localhost:5173](http://localhost:5173). Unauthenticated visitors are redirected through OIDC sign-in via the Go proxy.

There is **no mock mode**. The proxy always forwards to `FULFILLMENT_API_URL`.

### Proxy environment variables

| Variable                         | Default      | Description                                                                        |
| -------------------------------- | ------------ | ---------------------------------------------------------------------------------- |
| `FULFILLMENT_API_URL`            | _(required)_ | Base URL of the upstream fulfillment API (e.g. `https://fulfillment.example.com`). |
| `PORT`                           | `8080`       | Proxy listen port.                                                                 |
| `OIDC_CLIENT_ID`                 | `osac-ui`    | OAuth client id registered in the IdP.                                               |
| `FULFILLMENT_TLS_CA_FILE`        | _(unset)_    | PEM bundle for proxy → fulfillment TLS (private PKI).                              |
| `FULFILLMENT_TLS_INSECURE`       | _(unset)_    | Set to `1` to skip upstream TLS verification (dev only).                           |
| `TEMP_FULFILLMENT_STATIC_BEARER` | _(unset)_    | TEMP: inject `Authorization: Bearer …` when the client sends no non-empty Bearer.  |

**SPA (Vite dev):** optional `VITE_DEV_BEARER_TOKEN` — see `apps/app-frontend/.env.example`.

### Proxied path prefixes

| Prefix                  | Destination                            |
| ----------------------- | -------------------------------------- |
| `/api/fulfillment/v1/*` | `$FULFILLMENT_API_URL` + original path |
| `/api/events/v1/*`      | `$FULFILLMENT_API_URL` + original path |
| `/api/osac/public/v1/*` | `$FULFILLMENT_API_URL` + original path |

`/health` and `/ready` are handled locally by the proxy.

### TEMP bearer (dev only)

- **Proxy:** `TEMP_FULFILLMENT_STATIC_BEARER` — upstream receives `Authorization: Bearer …` only when the browser request has no real Bearer token. **Exception:** it is **not** sent on `GET /api/fulfillment/v1/capabilities` so OIDC discovery matches unauthenticated `curl`.
- **SPA (dev):** `VITE_DEV_BEARER_TOKEN` in `.env` — used only when session storage has no token.

### CORS / origins

Default local flow is **same-origin**: the SPA calls `/api` on the Vite dev server, which proxies to the Go proxy; the proxy calls the cluster. The browser does not need CORS against the cluster for that path.

### Public API contract (Buf)

The API contract and generated SDKs live at [buf.build/osac-project/public-api](https://buf.build/osac-project/public-api). **This repo uses REST over the Go proxy** until the live gateway is confirmed to expose Connect/gRPC-Web end-to-end.

## Storybook

- Run: `pnpm storybook`
- Build static: `pnpm build-storybook`

## Validation

- Lint: `pnpm lint`
- Test: `pnpm test`
- Build: `pnpm build`
- E2E: `pnpm e2e:ci`

### Real fulfillment proxy (manual smoke)

1. Start the proxy with valid `FULFILLMENT_API_URL` and either `FULFILLMENT_TLS_CA_FILE` or `FULFILLMENT_TLS_INSECURE=1` (local dev only).
2. `curl -sS http://localhost:8080/api/fulfillment/v1/capabilities` (with `-H "Authorization: Bearer …"` if the gateway requires it, or set `TEMP_FULFILLMENT_STATIC_BEARER` and omit the header).
3. Confirm the JSON is from the gateway (not a local fixture).
4. Start the SPA with Vite; confirm the browser calls same-origin `/api/...` only (no direct cluster origin in the Network tab).

## Container

- `podman build -t osac:latest -f Containerfile .`
- `podman run --rm -p 8080:8080 osac:latest`

## OpenShift

For a full step-by-step guide — Keycloak hostname, OIDC client registration, fulfillment trusted issuer, and troubleshooting — see [`deployment-openshift-guide.md`](deployment-openshift-guide.md).

Manifests live in `deploy/dev/` and `deploy/integration/`. Configure Keycloak and fulfillment-service before applying; then:

```bash
oc apply -f deploy/dev/    # or deploy/integration/
```
