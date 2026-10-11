# CPF-GitOps

Desired state of every Common Provenance Framework environment. FluxCD reconciles
each cluster from an OCI artifact published to Harbor by `.github/workflows/publish.yml`.

## Layout

| Path | Contents |
| --- | --- |
| `credentials/` | Job generating the `postgres-superuser`, `nro-db` and `neo4j` secrets when missing |
| `infrastructure/` | postgres-16, neo4j, nro-db-bootstrap Job |
| `apps/core/` | nro-service, cpf-storage |
| `e2e/` | Job starting the E2E suite, re-created whenever a service chart changes |
| `clusters/<env>/bootstrap.yaml` | per-environment entrypoint, applied by hand once, reconciled by `cpf-<env>-sync` afterwards |
| `clusters/<env>/cpf-<org>.yaml` | organization namespace running its own databases, nro-service and cpf-storage |
| `clusters/<env>/e2e.yaml` | E2E stage, depending on the core stage of every organization namespace |
| `clusters/_template/bootstrap.yaml` | starting point for a new environment |
| `clusters/_template/org.yaml` | starting point for a new organization namespace |

Manifests under `infrastructure/` and `apps/` are environment-agnostic. Everything
that differs between environments lives in the `cpf-<env>-settings` ConfigMap and is
injected through Flux `postBuild` substitution. `NAMESPACE` is set on the `apps/core`
Kustomization itself and makes the service URLs fully qualified, e.g.
`http://cpf-store.<namespace>.svc.cluster.local:8080`.

Service chart versions are not pinned: each service `OCIRepository` selects the
highest published semver, so a chart release reaches every cluster on its own.
These `OCIRepository` objects live in `cozy-fluxcd`, defined in `bootstrap.yaml`,
and are shared by all organization namespaces, so every organization picks up a
chart release at the same moment.
The postgres and neo4j chart versions stay pinned in the settings ConfigMap.

## Environments

| Environment | Cluster | Namespace |
| --- | --- | --- |
| qa | kubernetes-qa | cpf, cpf-org-a, cpf-org-b, cpf-org-c |

## Adding an organization namespace

Create the namespace and its `nro-certs` secret by hand first, then:

```
sed 's/ENVIRONMENT_NAME/qa/g; s/ORG_NAME/org-d/g' clusters/_template/org.yaml > clusters/qa/cpf-org-d.yaml
```

Add `- name: cpf-qa-org-d-core` to the `dependsOn` list in `clusters/qa/e2e.yaml`.
Commit both; `cpf-qa-sync` applies them, no manual apply needed. The services are then
reachable as `nro-service.cpf-org-d.svc.cluster.local:8020` and
`cpf-store.cpf-org-d.svc.cluster.local:8080`.

## Adding an environment

```
cp clusters/_template/bootstrap.yaml clusters/dev/bootstrap.yaml
sed -i 's/ENVIRONMENT_NAME/dev/g; s/ENVIRONMENT_NAMESPACE/cpf-dev/g' clusters/dev/bootstrap.yaml
```

Adjust the settings ConfigMap, commit, then apply it once against the target cluster:

```
kubectl apply -f clusters/dev/bootstrap.yaml
```

Nothing under `infrastructure/` or `apps/` changes. Later edits to
`clusters/dev/bootstrap.yaml` need no second apply; `cpf-dev-sync` reconciles them
from the published artifact.

## Reconciliation order

```
environment namespace:   sync  ->  credentials  ->  infrastructure/postgres
organization namespace:  sync  ->  credentials  ->  infrastructure  ->  apps/core
all organizations:       apps/core of every organization  ->  e2e
```

The e2e stage shares the organization stages' source, so Flux starts it only once
every organization's core stage has applied the same release revision and its
HelmReleases are ready. Its Job then calls the orchestrator's `/api/v1/suite/start`.

`apps/core` additionally orders `cpf-storage` after `nro-service` through the
HelmRelease `dependsOn` field.

Organization stages read from the `cpf-<env>-release` ExternalArtifact, which the
`cpf-<env>-release` ArtifactGenerator builds from the GitOps artifact and both service
charts. A chart release is therefore a new revision for every organization stage.

## Prerequisites per cluster

- FluxCD addon enabled on the Cozystack tenant cluster.
- Secret `nro-certs` present in every organization namespace.
- Service Helm charts published to `oci://cerit.io/commonprovenanceframework`.
