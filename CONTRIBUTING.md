# Contributing to RPCS3 Helm Chart

First off, thank you for considering contributing to this project!

## How Can I Contribute?

### Reporting Bugs

Before creating bug reports, please check existing issues to avoid duplicates. When you create a bug report, include as many details as possible:

- **Use a clear and descriptive title**
- **Describe the exact steps to reproduce the problem**
- **Provide specific examples**
- **Describe the behavior you observed and what you expected**
- **Include your environment details** (Kubernetes version, Helm version, etc.)

### Suggesting Enhancements

Enhancement suggestions are tracked as GitHub issues. When creating an enhancement suggestion:

- **Use a clear and descriptive title**
- **Provide a detailed description of the suggested enhancement**
- **Explain why this enhancement would be useful**
- **List any alternative solutions you've considered**

### Pull Requests

1. Fork the repository
2. Create a new branch from `main`:
   ```bash
   git checkout -b feature/my-new-feature
   ```
3. Make your changes
4. Test your changes thoroughly
5. Update documentation if needed
6. Commit your changes with clear messages
7. Push to your fork and submit a pull request

## Development Guidelines

### Testing Changes

Before submitting a PR, test your changes:

```bash
# Lint the chart
helm lint chart/

# Template the chart to check for errors
helm template rpcs3 chart/

# Install in a test cluster
helm install rpcs3-test chart/ --dry-run --debug
```

### Chart Versioning

- Follow [Semantic Versioning](https://semver.org/)
- Do **not** manually bump `version` in `Chart.yaml` in your PR — releases are driven by git tags.
  To cut a release: merge to `main`, then create and push a tag matching `vX.Y.Z` from the tip of
  `main` (e.g. `git tag v0.2.0 && git push origin v0.2.0`). The `Release Chart` workflow reads the
  tag, sets `chart/Chart.yaml`'s `version` to match, packages/publishes the chart, and commits the
  version bump back to `main`.
- Update `appVersion` in `Chart.yaml` in your PR when pinning the chart to a specific RPCS3 image
  tag (this is independent from the chart's own `version`/tag).

### Code Style

- Use 2 spaces for indentation in YAML files
- Keep lines under 120 characters when possible
- Add comments for complex configurations
- Follow Helm best practices

### Commit Messages

- Use the present tense ("Add feature" not "Added feature")
- Use the imperative mood ("Move cursor to..." not "Moves cursor to...")
- Limit the first line to 72 characters or less
- Reference issues and pull requests when relevant

## Questions?

Feel free to open an issue with your question or reach out to the maintainers.

Thank you! 🎮
