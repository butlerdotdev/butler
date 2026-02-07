# Butler Releases

This directory contains release coordination information for the Butler platform.

## Contents

| Document | Description |
|----------|-------------|
| [Compatibility Matrix](compatibility-matrix.md) | Version compatibility between components |
| [Upgrade Guide](upgrade-guide.md) | How to upgrade Butler |
| [Release Notes](notes/) | Aggregated release notes by version |

## Release Model

Butler follows a coordinated release model across components:

1. **Component Releases**: Individual components release independently
2. **Platform Release**: Periodically, we coordinate a "platform release" that includes compatible versions of all components
3. **Release Notes**: Aggregated in this directory

## Versioning

Butler components follow [Semantic Versioning](https://semver.org/):

- **MAJOR**: Breaking changes
- **MINOR**: New features, backward compatible
- **PATCH**: Bug fixes, backward compatible

### Pre-1.0

During alpha/beta (0.x.y):

- MINOR may include breaking changes
- PATCH is backward compatible

### Post-1.0

After 1.0:

- Full semver guarantees
- Deprecation policy for API changes

## Finding Versions

### Current Versions

See [Compatibility Matrix](compatibility-matrix.md) for current component versions.

### Component Releases

Each component has its own releases:

| Component | Releases |
|-----------|----------|
| butler-controller | [Releases](https://github.com/butlerdotdev/butler-controller/releases) |
| butler-bootstrap | [Releases](https://github.com/butlerdotdev/butler-bootstrap/releases) |
| butler-cli | [Releases](https://github.com/butlerdotdev/butler-cli/releases) |
| butler-console | [Releases](https://github.com/butlerdotdev/butler-console/releases) |
| butler-server | [Releases](https://github.com/butlerdotdev/butler-server/releases) |
| butler-charts | [Releases](https://github.com/butlerdotdev/butler-charts/releases) |

## Release Announcements

Release announcements are posted in:

- [GitHub Discussions](https://github.com/butlerdotdev/butler/discussions/categories/announcements)
- Release notes in this directory

## Support Policy

| Version | Support |
|---------|---------|
| Latest | Full support |
| Previous minor | Security fixes |
| Older | No support |
