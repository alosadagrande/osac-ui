# OSAC UI OpenShift deployment guide

Step-by-step guide for deploying the OSAC UI on an OpenShift cluster with a running fulfillment-service and Keycloak.

## Prerequisites

- OpenShift cluster with `oc` CLI access.
- **fulfillment-service** deployed and reachable from the cluster (the `fulfillment-internal-api` Service must exist).
- **Keycloak** deployed with an OIDC realm (e.g. `osac`).
- Podman (or Docker) for building the container image.
- A container registry you can push to (e.g. `quay.io`).

## 1. Build and push the container image

```bash
podman build -t quay.io/<your-org>/osac-ui:latest -f Containerfile .
podman login quay.io
podman push quay.io/<your-org>/osac-ui:latest
```

## 2. Configure Keycloak

### 2.1. Set the external hostname

Keycloak must advertise its **external route URL** in the OIDC discovery document; otherwise the browser is redirected to an internal `svc.cluster.local` address it cannot resolve.

For Keycloak on Quarkus (v17+), set the `KC_HOSTNAME` environment variable on the Keycloak deployment:

```bash
oc set env deployment/keycloak-service -n keycloak \
  KC_HOSTNAME=keycloak-keycloak.apps.<cluster-domain>
```

Verify by checking the `authorization_endpoint` in the discovery document:

```bash
curl -sk https://keycloak-keycloak.apps.<cluster-domain>/realms/osac/.well-known/openid-configuration | jq .authorization_endpoint
```

It must return the external route hostname, not `svc.cluster.local`.

### 2.2. Register the OIDC client

Create a public OIDC client in the **osac** realm (not in the master realm):

| Setting               | Value                                                          |
| --------------------- | -------------------------------------------------------------- |
| Client ID             | `osac-ui`                                                      |
| Client type           | OpenID Connect                                                 |
| Client authentication | Off (public client)                                            |
| Standard flow         | Enabled                                                        |
| Root URL              | `https://osac-<namespace>.apps.<cluster-domain>`               |
| Valid redirect URIs   | `https://osac-<namespace>.apps.<cluster-domain>/callback`      |
| Web origins           | `https://osac-<namespace>.apps.<cluster-domain>`               |

Notice that the Keycloak admin user and password can be obtained from the KEYCLOAK_ADMIN and KEYCLOAK_ADMIN_PASSWORD variables set in the keycloak deployment

```bash
oc get deployment -n keycloak -oyaml | grep -E " KEYCLOAK_ADMIN| KEYCLOAK_ADMIN_PASSWORD" -A
```

Via Keycloak admin API:

```bash
# Obtain an admin token
TOKEN=$(curl -sk -X POST \
  "https://keycloak-keycloak.apps.<cluster-domain>/realms/master/protocol/openid-connect/token" \
  -d "grant_type=password&client_id=admin-cli&username=admin&password=<admin-password>" \
  | jq -r .access_token)

# Create the client in realm osac
curl -sk -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "https://keycloak-keycloak.apps.<cluster-domain>/admin/realms/osac/clients" \
  -d '{
    "clientId": "osac-ui",
    "publicClient": true,
    "directAccessGrantsEnabled": false,
    "standardFlowEnabled": true,
    "rootUrl": "https://osac-<namespace>.apps.<cluster-domain>",
    "redirectUris": ["https://osac-<namespace>.apps.<cluster-domain>/callback"],
    "webOrigins": ["https://osac-<namespace>.apps.<cluster-domain>"],
    "protocol": "openid-connect",
    "enabled": true
  }'
```

### 2.3. Create a user

In the Keycloak admin console, select realm **osac** > **Users** > **Add user**:

1. Set a username and enable the account.
2. Go to the **Credentials** tab, set a password with **Temporary: Off**.

### 2.4. Configure the fulfillment trusted issuer

The fulfillment-service must advertise the **external** Keycloak URL as the trusted token issuer. Otherwise the UI proxy discovers an internal URL and the browser cannot reach it.

Find the `--grpc-authn-trusted-token-issuers` flag in the fulfillment gRPC server deployment and set it to the external route:

```
--grpc-authn-trusted-token-issuers=https://keycloak-keycloak.apps.<cluster-domain>/realms/osac
```

Restart the deployment and verify:

```bash
curl -sk https://fulfillment-internal-api-<namespace>.apps.<cluster-domain>/api/fulfillment/v1/capabilities | jq .
```

Expected output:

```json
{
  "authn": {
    "trusted_token_issuers": [
      "https://keycloak-keycloak.apps.<cluster-domain>/realms/osac"
    ]
  }
}
```

## 3. Configure the UI deployment

### 3.1. Identify the fulfillment internal Service

```bash
oc get svc -n <namespace> | grep fulfillment-internal-api
```

Note the port (e.g. `8001/TCP`).

### 3.2. Edit the ConfigMap

Edit `deploy/<env>/configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: osac-config
  namespace: <namespace>
data:
  PORT: '8080'
  HOST: '0.0.0.0'
  LOG_LEVEL: 'info'
  FULFILLMENT_API_URL: 'https://fulfillment-internal-api.<namespace>.svc.cluster.local:<port>'
  OIDC_CLIENT_ID: 'osac-ui'
  # Dev only — skip TLS verification for self-signed certs:
  FULFILLMENT_TLS_INSECURE: '1'
  OIDC_TLS_INSECURE: '1'
```

Key points:

- **`FULFILLMENT_API_URL`**: use the **internal Service URL** (not the external route) to avoid TLS hostname mismatches. Do **not** append `/api` — the proxy adds the path prefix automatically.
- **`FULFILLMENT_TLS_INSECURE`**: set to `1` when the fulfillment Service uses a certificate signed by a private CA.
- **`OIDC_TLS_INSECURE`**: set to `1` when Keycloak uses a self-signed certificate.

### 3.3. Update the Deployment image

Edit `deploy/<env>/deployment.yaml` to point to your pushed image:

```yaml
containers:
  - name: osac
    image: quay.io/<your-org>/osac-ui:latest
```

## 4. Deploy

```bash
oc new-project <namespace>   # if it doesn't exist the osac or osac-dev namespace
oc apply -f deploy/<env>/
oc rollout status deployment/osac -n <namespace>
```

Get the route URL:

```bash
oc get route osac -n <namespace> -o jsonpath='{.spec.host}'
```

## 5. Verify

### 5.1. Health check

```bash
curl -sk https://$(oc get route osac -n <namespace> -o jsonpath='{.spec.host}')/health | jq .
# {"status":"ok"}
```

### 5.2. OIDC discovery

```bash
curl -sk https://$(oc get route osac -n <namespace> -o jsonpath='{.spec.host}')/api/login?redirect_base=https://$(oc get route osac -n <namespace> -o jsonpath='{.spec.host}') | jq .url
```

The URL must contain the **external** Keycloak hostname, not `svc.cluster.local`.

### 5.3. Browser login

Open the route URL in a browser. You should be redirected to Keycloak, log in with the user created in step 2.3, and land on the OSAC dashboard.

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| `could not determine OIDC issuer` | Proxy cannot reach the fulfillment `/capabilities` endpoint. | Check `FULFILLMENT_API_URL`, verify connectivity with `curl` from inside the pod. Set `FULFILLMENT_TLS_INSECURE=1` if TLS fails. |
| `/api/api/fulfillment/v1/...` (duplicated path) | `FULFILLMENT_API_URL` ends with `/api`. | Remove the trailing `/api` from the URL. The proxy appends `/api/fulfillment/v1/...` automatically. |
| Browser redirects to `svc.cluster.local` | Keycloak's OIDC discovery returns internal hostnames. | Set `KC_HOSTNAME` on the Keycloak deployment to the external route hostname. |
| `Client not found` at Keycloak login | The `osac-ui` client does not exist in the correct realm. | Create the client in realm **osac**, not master. Verify with the admin API. |
| `x509: certificate signed by unknown authority` | Fulfillment or Keycloak use self-signed certs. | Set `FULFILLMENT_TLS_INSECURE=1` and `OIDC_TLS_INSECURE=1` in the ConfigMap. |
| `context deadline exceeded` on OIDC discovery | Pod cannot reach Keycloak on the configured hostname/port. | Verify the Keycloak Service port (`oc get svc -n keycloak`) and that NetworkPolicies allow cross-namespace traffic. |
| `unauthorized` on `podman push` | Registry token has read-only permissions. | Use `podman login` with your Quay user credentials or a robot account with Write access, not an OpenShift pull-secret. |
