# Contributing to Butler Observability Pipeline Reference

Thank you for your interest in contributing to the Butler observability pipeline reference. This document provides guidelines and instructions for contributing.

## Code of Conduct

This project adheres to the [Contributor Covenant Code of Conduct](https://www.contributor-covenant.org/version/2/1/code_of_conduct/). By participating, you are expected to uphold this code.

## Developer Certificate of Origin

By contributing to this project, you agree to the Developer Certificate of Origin (DCO). This document was created by the Linux Kernel community and is a simple statement that you, as a contributor, have the legal right to make the contribution.

Every commit must be signed off:

```bash
git commit -s -m "Your commit message"
```

## How to Contribute

**Bug reports and deployment issues** — open an issue describing the problem, the environment (Kubernetes version, Flux version, storage class), and the error output from `flux get all` or `kubectl describe`.

**Feature suggestions** — open an issue describing the use case and the change you'd like to see.

**Major changes** — for new optional sources, structural changes to the overlay pattern, or breaking changes to the values schema, open a discussion issue first. These affect every downstream fork and need maintainer alignment before implementation.

**Small improvements** — documentation fixes, comment clarifications, values typos — submit a PR directly.

## Development Workflow

1. Fork the repository
2. Clone your fork:
   ```bash
   git clone https://github.com/YOUR_USERNAME/butler-observability-pipeline-reference.git
   cd butler-observability-pipeline-reference
   ```
3. Add the upstream remote:
   ```bash
   git remote add upstream https://github.com/butlerdotdev/butler-observability-pipeline-reference.git
   ```
4. Create a feature branch:
   ```bash
   git checkout -b feat/your-change
   ```
5. Make your changes
6. Validate locally:
   ```bash
   # Both overlays must render without errors
   kustomize build apps/dev/
   kustomize build apps/prd/

   # Helm template must accept the values
   helm repo add vector https://helm.vector.dev
   helm pull vector/vector --version "0.52.0" --untar --untardir /tmp/vector-chart
   python3 -c "
   import yaml
   with open('apps/dev/vector-values.yaml') as f:
       print(yaml.dump(yaml.safe_load(f)['spec']['values']))
   " > /tmp/values-dev.yaml
   helm template vector /tmp/vector-chart/vector -f /tmp/values-dev.yaml --namespace vector
   ```
7. Commit and push:
   ```bash
   git add .
   git commit -s -m "feat(sinks): add example for S3-compatible sink"
   git push origin feat/your-change
   ```
8. Open a Pull Request against `main`

## Commit Messages

Follow conventional commits:

```
type(scope): description

[optional body]

Signed-off-by: Your Name <your.email@example.com>
```

Types:
- `feat`: New feature (new optional source, new sink example, new documentation section)
- `fix`: Bug fix (incorrect default, broken kustomize render, wrong port mapping)
- `docs`: Documentation changes
- `chore`: Maintenance (version bumps, comment updates)
- `refactor`: Structural changes that don't alter behavior

Scopes (optional): `sources`, `transforms`, `sinks`, `infra`, `docs`, `cluster`

## What This Repo Is and Isn't

This is a fork-and-deploy reference, not a library or framework. Contributions should improve the reference pattern itself — better defaults, clearer documentation, additional optional sources, improved validation guidance.

This repo is not the place for:
- Customer-specific configuration (belongs in the customer's fork)
- Backend deployment manifests (separate repositories)
- Butler agent configuration (belongs in butler-charts)

## Pull Request Process

1. Ensure both `kustomize build` commands succeed for dev and prd overlays
2. If changing values, verify `helm template` accepts the values without errors
3. If adding optional sources, follow the matched-group pattern (source + transform + sink + containerPort + service port, all commented out together)
4. If changing documentation, ensure cross-references between README, DEPLOYMENT.md, and CUSTOMIZATION.md remain consistent
5. Maintainers review and approve before merge

## License

By contributing, you agree that your contributions will be licensed under the Apache License 2.0.
