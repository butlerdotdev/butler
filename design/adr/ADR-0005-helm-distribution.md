# ADR-0005: Helm Chart Distribution via OCI

## Status

Accepted

## Context

Butler needs to distribute Helm charts for all components. Options considered:

1. **Chart Repository (HTTP)**: Traditional Helm repository hosted on web server
2. **GitHub Pages**: Host chart repository on GitHub Pages
3. **OCI Registry**: Publish charts as OCI artifacts to container registry
4. **Artifact Hub**: Submit to central Helm hub

Considerations:

- Ease of publishing (CI/CD integration)
- User experience (installation commands)
- Authentication (private registries)
- Ecosystem alignment

## Decision

We distribute Helm charts as **OCI artifacts** via GitHub Container Registry (GHCR).

```
oci://ghcr.io/butlerdotdev/charts/butler-controller
oci://ghcr.io/butlerdotdev/charts/butler-bootstrap
oci://ghcr.io/butlerdotdev/charts/butler-console
...
```

### Publishing

Charts are published via GitHub Actions on release:

```yaml
- name: Push Chart to GHCR
  run: |
    helm push butler-controller-*.tgz oci://ghcr.io/butlerdotdev/charts
```

### Installation

Users install charts directly from OCI:

```bash
helm install butler-controller \
  oci://ghcr.io/butlerdotdev/charts/butler-controller \
  --version 0.1.0
```

## Consequences

### Positive

- **Single registry**: Container images and charts in same place (GHCR)
- **Native CI/CD**: GitHub Actions has built-in GHCR authentication
- **Version immutability**: OCI tags are immutable
- **Modern approach**: Aligned with Helm 3+ direction
- **No additional infrastructure**: No web server to maintain

### Negative

- **OCI support required**: Helm 3.8+ required (released Jan 2022)
- **Discovery**: No browsable index like traditional repos
- **Helm search**: `helm search repo` doesn't work directly

### Neutral

- Same authentication as container images
- Versioning follows OCI tag conventions

## Alternatives Considered

### Alternative 1: Traditional Chart Repository

Host index.yaml and chart tarballs on web server or GitHub Pages.

**Why rejected:**

- Additional infrastructure to maintain
- Separate authentication from container registry
- Index file updates are error-prone

### Alternative 2: Artifact Hub

Submit charts to central Artifact Hub.

**Why rejected:**

- Additional registration and maintenance
- Not suitable for alpha/development phase
- May add later for discoverability

## Implementation Notes

### Chart Structure (butler-charts)

```
butler-charts/
├── charts/
│   ├── butler-crds/
│   ├── butler-controller/
│   ├── butler-bootstrap/
│   ├── butler-console/
│   ├── butler-addons/
│   ├── butler-provider-harvester/
│   └── butler-provider-nutanix/
├── .github/
│   └── workflows/
│       └── release.yaml
└── README.md
```

### CI/CD Pipeline

```yaml
name: Release Charts
on:
  push:
    tags:
      - 'v*'
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Login to GHCR
        run: echo "${{ secrets.GITHUB_TOKEN }}" | helm registry login ghcr.io -u ${{ github.actor }} --password-stdin
      - name: Package and Push
        run: |
          for chart in charts/*; do
            helm package $chart
            helm push $(basename $chart)-*.tgz oci://ghcr.io/butlerdotdev/charts
          done
```

### User Installation

```bash
# Install CRDs first
helm install butler-crds oci://ghcr.io/butlerdotdev/charts/butler-crds

# Install controller
helm install butler-controller oci://ghcr.io/butlerdotdev/charts/butler-controller \
  -n butler-system --create-namespace
```

## References

- [Helm OCI Support](https://helm.sh/docs/topics/registries/)
- [GitHub Container Registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
- [butler-charts repository](https://github.com/butlerdotdev/butler-charts)
