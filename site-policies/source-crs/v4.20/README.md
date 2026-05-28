# Vendored policy source CRs (per OCP version)

Copy the reference CR library here from your extracted container:

    podman run --log-driver=none --rm \
      registry.redhat.io/openshift4/ztp-site-generate-rhel8:v4.20 \
      extract /home/ztp --tar | tar x -C ./out
    cp -r out/source-crs/* site-policies/source-crs/v4.20/

These are the files the PolicyGenerator `manifests[].path` entries reference
(PtpConfig.yaml, PerformanceProfile.yaml, SriovNetwork.yaml, etc.). Pinning them
per-version here is the searchPaths idea applied to the policy side: the repo no
longer depends on whatever container tag the pipeline happens to run.

When you move to 4.21, create a sibling v4.21/ directory, re-extract, and update
the version segment in your PolicyGenerator paths.
