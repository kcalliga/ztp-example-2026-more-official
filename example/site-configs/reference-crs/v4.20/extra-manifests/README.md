# Vendored install-time extra-manifests (per OCP version)

Copy the install reference manifests here from your extracted container:

    cp -r out/extra-manifests/* site-configs/reference-crs/v4.20/extra-manifests/

These are install-time MachineConfigs (workload partitioning, predefined
master/worker manifests, etc.). Wrap them in a ConfigMap and reference that
ConfigMap from the ClusterInstance `spec.extraManifestsRefs` field to pin them,
rather than relying on the operator's bundled defaults.
