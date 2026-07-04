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
3. Generate a scoped, short-lived import credential **on the spoke cluster** (`oc`
   pointed at the spoke, not the hub). RHACM's auto-import genuinely needs
   cluster-admin-equivalent rights to do the import: it creates a CRD
   (`klusterlets.operator.open-cluster-management.io`), a namespace, and
   ClusterRoles/ClusterRoleBindings for the klusterlet operator's own service account —
   Kubernetes' RBAC escalation rules require the importing credential to already hold
   at least those permissions, so this can't be narrowed down to fewer verbs. What
   *can* be minimized is how long and how visibly that access exists:
   ```sh
   # On the SPOKE cluster
   oc create namespace rhacm-import
   oc create serviceaccount rhacm-import -n rhacm-import
   oc create clusterrolebinding rhacm-import-cluster-admin \
     --clusterrole=cluster-admin \
     --serviceaccount=rhacm-import:rhacm-import
   oc create token rhacm-import -n rhacm-import --duration=1h
   ```
   A dedicated, disposable namespace makes cleanup a single `oc delete namespace`; the
   clearly-named binding makes it easy to find; the 1h token self-expires whether or
   not anyone remembers to revoke it.
4. Build the `auto-import-secret` directly **on the hub** from that token — never
   write it to a file, never commit it:
   ```sh
   oc create secret generic auto-import-secret \
     -n <cluster-name> \
     --from-literal=token="<token from step 3>" \
     --from-literal=server="https://api.<spoke-cluster-domain>:6443" \
     --from-literal=autoImportRetry="5"
   ```
5. Apply the ManagedCluster + KlusterletAddonConfig:
   ```sh
   oc apply -k spoke-clusters/<cluster-name>/
   ```
6. Once `oc get managedcluster <cluster-name>` shows `JOINED=true`/`AVAILABLE=true`
   (see Verify below), clean up the import credential on the spoke — it's never used
   again, since ongoing hub↔spoke communication runs on a separate
   `bootstrap-hub-kubeconfig` that the klusterlet manages and rotates itself:
   ```sh
   # On the SPOKE cluster
   oc delete clusterrolebinding rhacm-import-cluster-admin
   oc delete namespace rhacm-import
   ```
   RHACM automatically deletes the `auto-import-secret` on the hub once import
   completes — no cleanup needed there.
7. Once you're comfortable managing spoke import declaratively, you can wire
   `spoke-clusters/<cluster-name>/` into `gitops/apps/kustomization.yaml` as its own
   `Application` (excluding the secret) — it's deliberately left manual for now.

## Verify import

```sh
oc get managedcluster <cluster-name>
oc get managedclusteraddon -n <cluster-name>
```

The `ManagedCluster` should show `HUB ACCEPTED=true` and eventually `JOINED=true`,
`AVAILABLE=true`. Add-ons (`application-manager`, `cert-policy-controller`,
`config-policy-controller`, `governance-policy-framework`, `search-collector`,
`work-manager`, `managed-serviceaccount`, `cluster-proxy`) should each show
`AVAILABLE=true` shortly after.
