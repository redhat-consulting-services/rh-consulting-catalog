# rh-consulting-catalog

Shared [File-Based Catalog (FBC)](https://olm.operatorframework.io/docs/reference/file-based-catalogs/) for Red Hat Consulting Services operators. This repository builds a single OLM catalog image serving multiple operators from one `CatalogSource`.

**Catalog image:** `quay.io/redhat-consulting-services/rh-consulting-catalog`

## Repository Structure

```
rh-consulting-catalog/
├── .github/workflows/
│   └── build-catalog.yaml          # CI: render, validate, build, push
├── operators/
│   └── ocp-support-web-operator/   # One directory per operator
│       ├── package.yaml            # OLM package definition
│       ├── channel.yaml            # Channel + upgrade chain
│       └── operator.yaml           # Rendered bundle blobs (opm render output)
├── Containerfile                   # Builds the catalog image
└── README.md
```

Each operator lives in its own directory under `operators/`. The `Containerfile` copies all operator directories into the catalog image, so every operator is served from a single `CatalogSource` on the cluster.

## How It Works

### Automatic Flow (tag a release)

```
Operator repo                          Catalog repo
─────────────                          ────────────
1. Push tag v1.9.0
2. CI builds operator image ──────►
3. CI builds bundle image  ──────►
4. CI sends repository_dispatch ──────► 5. Receives operator-release event
                                        6. Pulls bundle image from Quay
                                        7. Runs opm render → updates operator.yaml
                                        8. Updates channel.yaml (adds entry + replaces)
                                        9. Runs opm validate on ALL operators
                                       10. Builds catalog image
                                       11. Pushes to quay.io (version tag + latest)
                                       12. Commits updated configs to repo
```

When you tag a release on any operator repo (e.g. `git tag v1.9.0 && git push origin v1.9.0`), the operator's CI pipeline:

1. Builds and pushes the operator image
2. Builds and pushes the OLM bundle image
3. Fires a `repository_dispatch` event to this repo with the operator name, bundle image, and version

This repo's workflow then renders the bundle, validates the full catalog, builds the catalog image, and pushes it to Quay.

### Manual Trigger

Go to **Actions → Build and Push Catalog → Run workflow** and provide:

- **operator**: directory name under `operators/` (e.g. `ocp-support-web-operator`)
- **bundle_image**: full image reference (e.g. `quay.io/redhat-consulting-services/ocp-support-web-operator-bundle:v1.8.2`)

## Adding a New Operator

### 1. Create the operator directory

```bash
mkdir -p operators/my-new-operator
```

### 2. Create `package.yaml`

```yaml
---
schema: olm.package
name: my-new-operator
defaultChannel: alpha
description: Short description of the operator.
```

### 3. Create `channel.yaml`

```yaml
---
schema: olm.channel
name: alpha
package: my-new-operator
entries:
  - name: my-new-operator.v1.0.0
```

### 4. Render the initial bundle

```bash
opm render quay.io/redhat-consulting-services/my-new-operator-bundle:v1.0.0 \
  -o yaml > operators/my-new-operator/operator.yaml
```

### 5. Validate

```bash
opm validate operators/
```

### 6. Commit and push

The next catalog build will include the new operator.

### 7. Add the dispatch trigger to the operator's CI

In the operator repo's GitHub Actions workflow, add a job after the bundle is pushed:

```yaml
trigger-catalog:
  runs-on: ubuntu-latest
  needs: build-bundle
  if: github.event_name == 'push' && startsWith(github.ref, 'refs/tags/v')
  steps:
    - name: Determine version
      id: version
      run: echo "tag=${GITHUB_REF#refs/tags/}" >> "$GITHUB_OUTPUT"

    - name: Trigger catalog rebuild
      uses: peter-evans/repository-dispatch@v3
      with:
        token: ${{ secrets.CATALOG_DISPATCH_TOKEN }}
        repository: redhat-consulting-services/rh-consulting-catalog
        event-type: operator-release
        client-payload: |
          {
            "operator": "my-new-operator",
            "bundle_image": "quay.io/redhat-consulting-services/my-new-operator-bundle:${{ steps.version.outputs.tag }}",
            "version": "${{ steps.version.outputs.tag }}"
          }
```

## Required Secrets

| Secret | Where | Purpose |
|--------|-------|---------|
| `QUAY_IO_USERNAME` | This repo | Push catalog image to Quay.io |
| `QUAY_IO_PASSWORD` | This repo | Push catalog image to Quay.io |
| `REDHAT_REGISTRY_USER` | This repo | Pull base image from registry.redhat.io |
| `REDHAT_REGISTRY_PASSWORD` | This repo | Pull base image from registry.redhat.io |
| `CATALOG_DISPATCH_TOKEN` | Each operator repo | GitHub PAT with `repo` scope to trigger `repository_dispatch` on this repo |

If these secrets are configured at the GitHub organization level, they are inherited automatically.

## Deploying to OpenShift

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: rh-consulting-catalog
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: quay.io/redhat-consulting-services/rh-consulting-catalog:latest
  displayName: RH Consulting Services Operators
  publisher: Red Hat Consulting Services
  updateStrategy:
    registryPoll:
      interval: 5m
```

This single `CatalogSource` provides all operators in this catalog. Each operator can then be installed via a `Subscription` referencing `source: rh-consulting-catalog`.

## Local Development

### Prerequisites

- [opm](https://github.com/operator-framework/operator-registry/releases) (operator registry CLI)
- [podman](https://podman.io/) or docker
- Access to Quay.io and registry.redhat.io

### Build locally

```bash
# Validate all operators
opm validate operators/

# Build the catalog image
podman build -t quay.io/redhat-consulting-services/rh-consulting-catalog:dev .

# Push (requires login)
podman push quay.io/redhat-consulting-services/rh-consulting-catalog:dev
```

### Render a new bundle locally

```bash
OPERATOR=ocp-support-web-operator
VERSION=1.9.0

# Render
opm render quay.io/redhat-consulting-services/${OPERATOR}-bundle:v${VERSION} \
  -o yaml >> operators/${OPERATOR}/operator.yaml

# Add channel entry
cat >> operators/${OPERATOR}/channel.yaml <<EOF
  - name: ${OPERATOR}.v${VERSION}
    replaces: ${OPERATOR}.v1.8.2
EOF

# Validate
opm validate operators/
```

## Operators in This Catalog

| Operator | Channel | Latest Version |
|----------|---------|----------------|
| ocp-support-web-operator | alpha | v1.8.2 |

---

Assisted-By: Claude
