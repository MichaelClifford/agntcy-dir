# Installing the Directory Service on OpenShift

This guide walks through deploying the AGNTCY Directory Service on an
OpenShift cluster with SPIFFE/SPIRE mTLS authentication and OIDC-based
access control via the Envoy auth gateway.

## What Gets Deployed

The Helm chart installs the following components into your namespace:

| Component | Description |
|-----------|-------------|
| **apiserver** | gRPC API server for the Directory Service |
| **reconciler** | Async worker that syncs records from remote registries and indexes them |
| **Zot registry** | OCI-compliant container registry used as the storage backend |
| **PostgreSQL** | Database for record metadata and indexing |
| **Envoy gateway** | Reverse proxy that handles authentication and speaks SPIFFE mTLS to the apiserver |
| **Authz server** | Ext_authz service that validates tokens and enforces RBAC |

The Envoy gateway is exposed via an OpenShift Route with TLS passthrough.
External clients authenticate via OIDC tokens (Dex, GitHub Actions) or
GitHub PATs; internal communication between Envoy and the apiserver uses
SPIFFE X.509-SVIDs issued by SPIRE.

## Prerequisites

### Required Tools

| Tool | Version | Purpose |
|------|---------|---------|
| `oc` | 4.x | OpenShift CLI |
| `helm` | 3.x+ | Helm chart installation |
| `openssl` | any | Generate TLS certificates |
| `dirctl` | latest | Directory Service CLI (for testing) |

### Cluster Requirements

- OpenShift 4.x cluster with cluster-admin access
- **SPIRE** deployed on the cluster (server, agent, CSI driver, and
  controller manager). The Directory chart creates `ClusterSPIFFEID`
  resources but does not install SPIRE itself.
- A storage class available for PostgreSQL and Zot persistent volumes

### Verify SPIRE Is Running

Confirm that the SPIRE agent, server, and CSI driver pods are healthy:

```bash
# Find SPIRE pods (they may be in any namespace)
oc get pods --all-namespaces | grep spire

# Verify the SPIFFE CSI driver is registered
oc get csidrivers | grep csi.spiffe.io
```

You should see a `spire-server`, `spire-agent` (DaemonSet), and
`spire-spiffe-csi-driver` (DaemonSet) all in a Running state.

## Step 1: Create the Namespace

```bash
oc new-project agent-directory
```

Or if the namespace already exists:

```bash
oc project agent-directory
```

## Step 2: Gather SPIRE Configuration

You need four values from the existing SPIRE installation.

### 2.1 Trust Domain

```bash
# Replace <spire-namespace> with the namespace where SPIRE is deployed
oc get configmap -n <spire-namespace> spire-server -o yaml | grep trust_domain
```

Note this value as `<YOUR-TRUST-DOMAIN>`.

### 2.2 Controller Manager className

Check existing ClusterSPIFFEIDs on the cluster to find the className:

```bash
oc get clusterspiffeids -o yaml | grep className
```

Note this value as `<YOUR-SPIRE-CLASS-NAME>`.

If there is no className configured (classless mode), you can set
`className: "none"` in the values file — the chart will omit the field
entirely.

### 2.3 SPIRE Agent Socket Filename

Different SPIRE deployments use different socket filenames inside the CSI
mount. Deploy a debug pod to check:

```bash
oc run spire-debug --image=busybox -n agent-directory --restart=Never \
  --overrides='{
    "spec": {
      "containers": [{
        "name": "debug",
        "image": "busybox",
        "command": ["ls", "-la", "/run/spire/agent-sockets/"],
        "volumeMounts": [{
          "name": "spire-socket",
          "mountPath": "/run/spire/agent-sockets",
          "readOnly": true
        }]
      }],
      "volumes": [{
        "name": "spire-socket",
        "csi": {
          "driver": "csi.spiffe.io",
          "readOnly": true
        }
      }]
    }
  }'

# Wait a moment, then check
oc logs spire-debug -n agent-directory

# Clean up
oc delete pod spire-debug -n agent-directory
```

You should see a socket file like `api.sock` (upstream SPIRE) or
`spire-agent.sock` (managed SPIRE operators). Note the filename as
`<YOUR-SOCKET-FILENAME>`.

### 2.4 Namespace Selector Pattern

Some SPIRE controller managers require ClusterSPIFFEID resources to include
a `namespaceSelector` that scopes them to specific namespaces. Check if the
existing ClusterSPIFFEIDs use one:

```bash
oc get clusterspiffeids -o yaml | grep -A5 namespaceSelector
```

If you see `namespaceSelector` blocks, you will need one too. The
`values-openshift.yaml` template includes a namespaceSelector by default
that matches the deployment namespace.

## Step 3: Create the Envoy TLS Certificate

The Envoy gateway needs a TLS certificate because OpenShift's HAProxy router
uses TLS passthrough for gRPC (HTTP/2) traffic. Envoy terminates TLS directly.

### Option A: Self-Signed Certificate

Generate a self-signed certificate and create the OpenShift secret:

```bash
# Determine the route hostname.
# OpenShift auto-generates hostnames in the format:
#   <release-name>-envoy-authz-<namespace>.<cluster-domain>
# For example: dir-release-envoy-authz-agent-directory.apps.mycluster.example.com
ENVOY_HOSTNAME="dir-release-envoy-authz-agent-directory.<YOUR-CLUSTER-DOMAIN>"

# Generate the certificate.
# Use a short CN (max 64 chars) — the full hostname goes in the SAN.
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /tmp/envoy-tls.key \
  -out /tmp/envoy-tls.crt \
  -subj "/CN=envoy-gateway" \
  -extensions SAN \
  -config <(cat /etc/ssl/openssl.cnf; printf "\n[SAN]\nsubjectAltName=DNS:${ENVOY_HOSTNAME}")

# Create the OpenShift TLS secret
oc create secret tls envoy-tls-cert \
  -n agent-directory \
  --cert=/tmp/envoy-tls.crt \
  --key=/tmp/envoy-tls.key
```

**Note:** If the `-config` approach fails on your system, you can use a
simple CN-only certificate instead. The passthrough route does not inspect
the certificate, and clients will use `--tls-skip-verify` for self-signed
certs anyway:

> ```bash
> openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
>   -keyout /tmp/envoy-tls.key \
>   -out /tmp/envoy-tls.crt \
>   -subj "/CN=envoy-gateway"
> ```

### Option B: cert-manager (If Available)

If cert-manager is deployed on your cluster, you can create a `Certificate`
resource instead:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: envoy-tls-cert
  namespace: agent-directory
spec:
  secretName: envoy-tls-cert
  issuerRef:
    name: <your-cluster-issuer>
    kind: ClusterIssuer
  dnsNames:
    - dir-release-envoy-authz-agent-directory.<YOUR-CLUSTER-DOMAIN>
```

## Step 4: Configure the Values File

The deployment uses a two-file layering approach:

1. **`values-openshift.yaml`** — Generic OpenShift settings (committed to the
   repo, not modified)
2. **Your cluster-specific values file** — Fills in the placeholders for your
   cluster

Create your cluster-specific values file:

```bash
cat > install/charts/dir/my-values.yaml << 'EOF'
apiserver:
  config:
    authn:
      socket_path: "unix:///run/spire/agent-sockets/<YOUR-SOCKET-FILENAME>"
    database:
      postgres:
        host: dir-release-postgresql

  authz_policies_csv: |
    p,<YOUR-TRUST-DOMAIN>,*
    p,*,/agntcy.dir.store.v1.StoreService/Pull
    p,*,/agntcy.dir.store.v1.StoreService/Push
    p,*,/agntcy.dir.store.v1.StoreService/PullReferrer
    p,*,/agntcy.dir.store.v1.StoreService/Lookup
    p,*,/agntcy.dir.search.v1.SearchService/SearchCIDs
    p,*,/agntcy.dir.search.v1.SearchService/SearchRecords
    p,*,/agntcy.dir.sync.v1.SyncService/RequestRegistryCredentials

  spire:
    trustDomain: <YOUR-TRUST-DOMAIN>
    className: <YOUR-SPIRE-CLASS-NAME>
    namespaceSelector:
      matchExpressions:
      - key: kubernetes.io/metadata.name
        operator: In
        values:
        - agent-directory
    dnsNameTemplates:
      - dir-release-apiserver-agent-directory.<YOUR-CLUSTER-DOMAIN>

  reconciler:
    config:
      database:
        postgres:
          host: dir-release-postgresql

  envoy-authz:
    envoy:
      backend:
        address: dir-release-apiserver.agent-directory.svc.cluster.local
        port: 8888
      spiffe:
        trustDomain: <YOUR-TRUST-DOMAIN>
        className: <YOUR-SPIRE-CLASS-NAME>
        socketPath: "/run/spire/agent-sockets/<YOUR-SOCKET-FILENAME>"
        namespaceSelector:
          matchExpressions:
          - key: kubernetes.io/metadata.name
            operator: In
            values:
            - agent-directory

    authServer:
      oidc:
        roles:
          admin:
            allowedMethods:
              - "*"
            users:
              - "<YOUR-OIDC-USER>"
EOF
```

Replace the placeholders with the values gathered in steps above:

| Placeholder | Description | Example |
|-------------|-------------|---------|
| `<YOUR-TRUST-DOMAIN>` | SPIRE trust domain (Step 2.1) | `apps.mycluster.example.com` |
| `<YOUR-SPIRE-CLASS-NAME>` | SPIRE controller manager className (Step 2.2) | `my-spire-class` |
| `<YOUR-SOCKET-FILENAME>` | SPIRE agent socket filename (Step 2.3) | `spire-agent.sock` |
| `<YOUR-CLUSTER-DOMAIN>` | OpenShift cluster wildcard domain | `apps.mycluster.example.com` |
| `<YOUR-OIDC-USER>` | Admin user identifier for RBAC | `github:myuser` |

The admin user format depends on your authentication setup:

| Auth Method | User Format | Example |
|-------------|-------------|---------|
| GitHub PAT | `github:<username>` | `github:octocat` |
| Dex OIDC | `user:<issuer>:<email>` | `user:https://dex.example.com:admin@example.com` |
| GitHub Actions | `ghwf:repo:<org>/<repo>:workflow:<name>:ref:<ref>:env:<env>` | (see chart docs) |

## Step 5: Build Dependencies and Install

### Build Chart Dependencies

The Helm chart has local subchart dependencies that must be built first:

```bash
# Build the apiserver subchart dependencies (includes envoy-authz, zot, postgresql)
helm dependency build ./install/charts/dir/apiserver

# Build the top-level chart dependencies
helm dependency build ./install/charts/dir
```

### Install the Chart

```bash
helm install dir-release ./install/charts/dir \
  --namespace agent-directory \
  -f install/charts/dir/values-openshift.yaml \
  -f install/charts/dir/my-values.yaml \
  --timeout 20m
```

> **Note:** The first boot may take several minutes due to PostgreSQL schema
> migration, especially on network-attached storage (e.g., Ceph RBD). The
> `--timeout 20m` flag accounts for this. Subsequent restarts are faster.

## Step 6: Verify the Deployment

### Check Pod Status

```bash
oc get pods -n agent-directory
```

You should see pods for the apiserver, reconciler, Zot registry, PostgreSQL,
Envoy proxy, and authz server — all in a `Running` state.

### Check the OpenShift Route

```bash
oc get routes -n agent-directory
```

The Envoy gateway route should be listed with a hostname. This is the external
endpoint for the Directory Service.

### Verify ClusterSPIFFEID Resources

```bash
oc get clusterspiffeids | grep dir-release
```

You should see entries for the apiserver, reconciler, and envoy gateway.

### Test Connectivity

Use `dirctl` to verify end-to-end connectivity:

```bash
# Get the route hostname
ROUTE_HOST=$(oc get route -n agent-directory dir-release-envoy-authz -o jsonpath='{.spec.host}')

# Test with authentication (replace with your token)
dirctl search --server-addr "${ROUTE_HOST}:443" --tls-skip-verify --github-token "${GITHUB_TOKEN}"
```

A successful response of `No record CIDs found` confirms the full chain is
working: TLS termination, authentication, SPIFFE mTLS, and the apiserver.

You can also test with `grpcurl`:

```bash
# Unauthenticated — should return an auth error (proves the service is up)
grpcurl -insecure "${ROUTE_HOST}:443" list
```


## Upgrading

To upgrade an existing installation:

```bash
# Rebuild dependencies if chart templates changed
helm dependency build ./install/charts/dir/apiserver
helm dependency build ./install/charts/dir

# Upgrade
helm upgrade dir-release ./install/charts/dir \
  -n agent-directory \
  -f install/charts/dir/values-openshift.yaml \
  -f install/charts/dir/my-values.yaml \
  --timeout 20m \
  --wait
```

## Uninstalling

```bash
helm uninstall dir-release -n agent-directory

# Clean up cluster-scoped resources (not removed by helm uninstall)
oc delete clusterspiffeids -l app.kubernetes.io/instance=dir-release

# Clean up the TLS secret
oc delete secret envoy-tls-cert -n agent-directory

# Optionally delete the namespace
oc delete project agent-directory
```
