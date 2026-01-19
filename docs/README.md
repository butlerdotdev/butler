# Butler Documentation

Welcome to the Butler documentation. This guide covers everything from getting started to advanced operations.

## Documentation Structure

```
docs/
├── overview/           # What is Butler, core concepts
├── architecture/       # System design and internals
├── getting-started/    # Installation and first cluster
├── operations/         # Day-2 operations
├── providers/          # Infrastructure provider guides
├── contributing/       # Development and contribution
└── roadmap/            # Future plans
```

## Quick Links

| I want to... | Go to... |
|--------------|----------|
| Understand what Butler is | [Overview](overview/) |
| Install Butler | [Getting Started](getting-started/) |
| Create my first cluster | [Getting Started](getting-started/#create-your-first-cluster) |
| Learn the architecture | [Architecture](architecture/) |
| Set up Harvester | [Harvester Guide](providers/harvester.md) |
| Set up Nutanix | [Nutanix Guide](providers/nutanix.md) |
| Upgrade Butler | [Operations](operations/#upgrading-butler) |
| Contribute | [Contributing](contributing/) |
| See what's planned | [Roadmap](roadmap/) |

## Documentation by Role

### Platform Operators

1. [Getting Started](getting-started/) - Bootstrap your first management cluster
2. [Architecture](architecture/) - Understand how Butler works
3. [Operations](operations/) - Day-2 operations and maintenance
4. [Provider Guides](providers/) - Infrastructure-specific setup

### Platform Users (Developers)

1. [Overview](overview/) - Butler concepts
2. [Getting Started](getting-started/#create-your-first-cluster) - Create your first cluster
3. [Architecture: Addons](architecture/addon-system.md) - Available addons

### Contributors

1. [Contributing Guide](contributing/) - How to contribute
2. [Development Setup](contributing/development-setup.md) - Set up dev environment
3. [Architecture](architecture/) - Understand the codebase

## External Resources

| Resource | Link |
|----------|------|
| GitHub Repository | [butlerdotdev/butler](https://github.com/butlerdotdev/butler) |
| Component Registry | [COMPONENTS.md](../COMPONENTS.md) |
| Compatibility Matrix | [releases/compatibility-matrix.md](../releases/compatibility-matrix.md) |
| Design Decisions | [design/adr/](../design/adr/) |

## Feedback

Found an issue or want to improve the docs?

- [Open a documentation issue](https://github.com/butlerdotdev/butler/issues/new?labels=documentation)
- [Submit a PR](https://github.com/butlerdotdev/butler/pulls)
