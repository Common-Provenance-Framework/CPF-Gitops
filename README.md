# CPF-GitOps

Desired state of every Common Provenance Framework environment. FluxCD reconciles
each cluster from an OCI artifact published to Harbor by `.github/workflows/publish.yml`.

## Layout

| Path | Contents |
| --- | --- |
| `credentials/` | Job generating the `postgres-superuser`, `nro-db` and `neo4j` secrets when missing |
| `infrastructure/` | postgres-16, neo4j, nro-db-bootstrap Job |
| `apps/core/` | nro-service, cpf-storage |
| `apps/optional/` | pt-service, pa-service, reconciled without blocking on readiness |
| `clusters/<env>/bootstrap.yaml` | per-environment entrypoint, applied by hand once, reconciled by `cpf-<env>-sync` afterwards |
| `clusters/_template/bootstrap.yaml` | starting point for a new environment |

Manifests under `infrastructure/` and `apps/` are environment-agnostic. Everything
that differs between environments lives in the `cpf-<env>-settings` ConfigMap and is
injected through Flux `postBuild` substitution. `NAMESPACE` is set on the `apps/core`
Kustomization itself and makes the service URLs fully qualified, e.g.
`http://cpf-store.<namespace>.svc.cluster.local:8080`.

Service chart versions are not pinned: each service `OCIRepository` selects the
highest published semver, so a chart release reaches every cluster on its own.
The postgres and neo4j chart versions stay pinned in the settings ConfigMap.

## Environments

| Environment | Cluster | Namespace |
| --- | --- | --- |
| qa | kubernetes-qa | cpf |

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
sync  ->  credentials  ->  infrastructure  ->  apps/core  ->  apps/optional
```

`apps/core` additionally orders `cpf-storage` after `nro-service` through the
HelmRelease `dependsOn` field.

## Prerequisites per cluster

- FluxCD addon enabled on the Cozystack tenant cluster.
- Secrets `nro-certs` and `org-pki` present in the target namespace.
- Service Helm charts published to `oci://cerit.io/commonprovenanceframework`.
