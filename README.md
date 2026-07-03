# acm-install-aws

Installs Red Hat Advanced Cluster Management (RHACM) on an existing self-managed
OpenShift cluster on AWS, managed declaratively via ArgoCD (OpenShift GitOps).

## Architecture

```
bootstrap/        one-time, manual — installs OpenShift GitOps and points it at this repo
gitops/root/      root "app-of-apps" Application ArgoCD is bootstrapped against
gitops/apps/      two child Applications, sequenced by sync-wave:
                    01-operator-app.yaml (wave 1) -> operator/overlays/aws-prod
                    02-hub-app.yaml      (wave 2) -> hub/overlays/aws-prod
operator/         RHACM operator install: Namespace, OperatorGroup, Subscription
hub/              MultiClusterHub CR (triggers the actual RHACM component install)
spoke-clusters/   optional, opt-in spoke-cluster import — NOT part of the default sync
```

RHACM installs through OLM: Subscription -> CSV -> `MultiClusterHub` CRD -> `MultiClusterHub`
CR. The `MultiClusterHub` CRD doesn't exist until the operator's CSV succeeds, so the two
stages are separate ArgoCD `Application`s rather than one flat manifest set:

- Wave 1 (`acm-operator`) installs the OLM Subscription. ArgoCD's built-in health check
  for `Subscription` only reports `Healthy` once `status.state == AtLatestKnown` (the CSV
  really succeeded), which naturally gates wave 2.
- Wave 2 (`acm-multiclusterhub`) applies the `MultiClusterHub` CR. It sets
  `SkipDryRunOnMissingResource=true` so its first reconcile doesn't hard-fail before the
  CRD exists; combined with `syncPolicy.automated.selfHeal`, ArgoCD retries it
  automatically once wave 1 is healthy.

Observability and other optional RHACM add-ons (policy engine, submariner, app lifecycle)
are intentionally out of scope — `hub/base/multiclusterhub.yaml` uses a minimal `spec: {}`.

## Install

See [`bootstrap/README.md`](bootstrap/README.md) for the one-time manual bootstrap
(install OpenShift GitOps, then create the root `Application`). Everything after that is
GitOps-managed — commits to `main` are what drive changes.

## Verify

```sh
oc get subscription acm-operator-subscription -n open-cluster-management -o jsonpath='{.status.state}'   # expect AtLatestKnown
oc get csv -n open-cluster-management                                                                     # expect Succeeded
oc get crd multiclusterhubs.operator.open-cluster-management.io                                            # expect it to exist
oc get multiclusterhub multiclusterhub -n open-cluster-management -o jsonpath='{.status.phase}'            # expect Running
```

You can also watch progress in the ArgoCD UI: the `acm-install-aws-root` ->
`acm-install-aws-apps` -> `acm-operator` / `acm-multiclusterhub` Application tree.

## Bumping the RHACM version

Edit `operator/overlays/aws-prod/channel-patch.yaml` (`spec.channel`), commit, push.
ArgoCD auto-syncs the new Subscription channel and OLM upgrades the CSV in place — this
is the single place channel version is set.

## Spoke cluster import

Not installed by default. See [`spoke-clusters/README.md`](spoke-clusters/README.md) if/when
you need to attach a managed cluster to this hub.
