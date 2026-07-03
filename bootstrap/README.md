# Bootstrap (one-time, manual)

Everything under `gitops/`, `operator/`, and `hub/` is managed declaratively by ArgoCD.
But ArgoCD itself doesn't exist on a fresh cluster yet, so it can't apply its own
installation — these two commands are the one deliberate exception to "everything is
GitOps." Run them once, in order, and never again unless the hub cluster is rebuilt.

## 1. Install the OpenShift GitOps operator

```sh
oc apply -f bootstrap/openshift-gitops-subscription.yaml
```

Wait for the operator to finish installing and for the default ArgoCD instance to come up:

```sh
oc get csv -n openshift-gitops-operator
oc get pods -n openshift-gitops
```

## 2. Point ArgoCD at this repo

```sh
oc apply -f bootstrap/root-application.yaml
```

This creates the root `Application` (`acm-install-aws-root`) in the `openshift-gitops`
namespace, pointed at `gitops/root` in this repo. From here on, ArgoCD manages the
RHACM operator install and the `MultiClusterHub` CR itself — see the top-level
[README.md](../README.md) for the sequencing details and verification steps.

If ArgoCD needs read access to this repo (private repo over SSH), register the deploy
key/credentials with the ArgoCD instance (`openshift-gitops` namespace) before applying
`root-application.yaml`, e.g. via a `repo-creds` Secret labeled
`argocd.argoproj.io/secret-type: repo-creds`. Not included here since it's
credential-specific to your setup.
