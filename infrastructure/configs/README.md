# Infrastructure Configs

This directory holds standalone custom resources that depend on CRDs
installed by `infrastructure/controllers/`.

Examples: ClusterIssuers (cert-manager), ServiceMonitors not managed by
Helm charts, storage classes, or any cluster-scoped resource that must
exist before app workloads can reference it.

The Flux Kustomization CR for this tier should use:

```yaml
dependsOn:
  - name: infra-controllers
```

This ensures the CRD-providing controllers are healthy before these
resources are applied. The apps tier then depends on both.

Currently empty. Add resources here as needed.
