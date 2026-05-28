# ZTP / GitOps repo layout (OpenShift 4.20, ClusterInstance + PolicyGenerator)

A self-contained, version-pinned take on the Red Hat GitOps ZTP reference
structure. It keeps the **concepts** Red Hat is consistent about and drops the
churny container folder names.

## The durable concepts this encodes

1. **Two ArgoCD applications**, one per pipeline half:
   - `clusters` -> watches `site-configs/` (install, via ClusterInstance)
   - `policies` -> watches `site-policies/` (Day-2 config, via PolicyGenerator)
2. **Three policy tiers** bound by cluster labels:
   - `common` (all clusters) / `group` (a cluster class) / `site` (one cluster)
3. **Labels as the join key**: `spec.extraLabels.ManagedCluster` on the
   ClusterInstance is what the PolicyGenerator `placement.labelSelector` matches.
4. **Vendored reference CRs, pinned per OCP version** (`*/source-crs/v4.20/`,
   `reference-crs/v4.20/`). This is the searchPaths idea: the repo does not
   depend on whichever `ztp-site-generate` container tag the pipeline runs.

## Tree

    .
    ├── bootstrap/                 # hub-side wiring (= out/argocd/deployment)
    │   ├── clusters-app.yaml      #   ArgoCD App -> site-configs/
    │   ├── policies-app.yaml      #   ArgoCD App -> site-policies/
    │   └── kustomization.yaml
    ├── site-configs/              # INSTALL path (ClusterInstance)
    │   ├── kustomization.yaml
    │   ├── reference-crs/v4.20/   #   vendored extra-manifests (pinned)
    │   └── hub-1/                 #   per-hub / per-site cluster definitions
    │       └── example-sno.yaml
    └── site-policies/             # POLICY path (PolicyGenerator)
        ├── kustomization.yaml     #   resources: ns + generators: the PGs
        ├── ns.yaml
        ├── source-crs/v4.20/      #   vendored policy CR library (pinned)
        ├── common/                #   tier 1
        ├── group/                 #   tier 2
        └── site/                  #   tier 3

## Mapping from the old (4.11-era) layout

| Old                       | Here / current             | API change                                             |
|---------------------------|----------------------------|--------------------------------------------------------|
| `siteconfig/`             | `site-configs/`            | SiteConfig v1 -> ClusterInstance (SiteConfig Operator) |
| `policygentemplates/`     | `site-policies/`           | PolicyGenTemplate -> PolicyGenerator                   |
| `out/source-crs/`         | `*/source-crs/v4.20/`      | now vendored + version-pinned                          |
| `out/extra-manifest/`     | `reference-crs/v4.20/...`  | pluralized + vendored                                  |

## One-time hub setup (not GitOps-managed)

    # 1. Extract the reference container for your OCP version
    mkdir -p out
    podman run --log-driver=none --rm \
      registry.redhat.io/openshift4/ztp-site-generate-rhel8:v4.20 \
      extract /home/ztp --tar | tar x -C ./out

    # 2. Vendor the reference CRs into this repo (see the per-dir READMEs)
    cp -r out/source-crs/*     site-policies/source-crs/v4.20/
    cp -r out/extra-manifests/* site-configs/reference-crs/v4.20/extra-manifests/

    # 3. Patch the ArgoCD instance to enable the SiteConfig + PolicyGenerator plugins
    oc patch argocd openshift-gitops -n openshift-gitops --type=merge \
      --patch-file out/argocd/deployment/argocd-openshift-gitops-patch.json

    # 4. Apply the two ArgoCD Applications
    oc apply -k bootstrap/

## Per-cluster checklist (for each new site)

- Add a ClusterInstance under `site-configs/<hub>/` and list it in that
  kustomization.
- Create the namespace-scoped secrets the CR references (BMC secret +
  pull secret) in a namespace matching `clusterName`.
- Set `spec.extraLabels.ManagedCluster` so the right common/group/site policies
  bind.
- If the site needs unique config, add a `site/` PolicyGenerator and reference
  it in `site-policies/kustomization.yaml` under `generators:`.

## Caveats

- The `templateRefs` ConfigMap names (`ai-cluster-templates-v1`,
  `ai-node-templates-v1`) ship with the SiteConfig Operator; confirm the exact
  names in your RHACM version.
- `remediationAction` defaults to `inform` everywhere. Move tiers to `enforce`
  deliberately, common-first, once you trust the rollout.
- For migrating existing SiteConfig CRs, OCP 4.20 ships `siteconfig-converter`
  to translate them to ClusterInstance incrementally.
