# Spoke cluster import (optional, opt-in)

This directory is **not** wired into the default GitOps sync (`gitops/apps/`). The hub
install (`operator/` + `hub/`) is fully functional and healthy without anything here.
Only use this if/when you actually need to attach a managed (spoke) cluster to the hub.

## To import a spoke cluster

1. Copy the template:
   ```sh
   cp -r spoke-clusters/_template spoke-clusters/<cluster-name>
   ```
2. Replace every `CLUSTER_NAME` placeholder in the copied files with the real cluster name.
3. Generate the real `auto-import-secret` out-of-band (see the comments in
   `_template/auto-import-secret.example.yaml`) — **never commit real spoke-cluster
   credentials to this repo.** Apply it directly to the hub:
   ```sh
   oc apply -f <path-to-your-real-secret>.yaml
   ```
4. Apply the ManagedCluster + KlusterletAddonConfig:
   ```sh
   oc apply -k spoke-clusters/<cluster-name>/
   ```
5. Once you're comfortable managing spoke import declaratively, you can wire
   `spoke-clusters/<cluster-name>/` into `gitops/apps/kustomization.yaml` as its own
   `Application` (excluding the secret) — it's deliberately left manual for now.

## Verify import

```sh
oc get managedcluster <cluster-name>
oc get pods -n <cluster-name>
```

The `ManagedCluster` should show `HUB ACCEPTED=true` and eventually `JOINED=true`,
`AVAILABLE=true`.
