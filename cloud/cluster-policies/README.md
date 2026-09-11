# cloud/cluster-policies

Cluster-scoped admission policies for the "hallen" cluster (where
`cloud/envs/hallen` actually deploys) - see
`appliance-registry-secret-scope.yaml`'s own comment for what it does and
why.

## Why this is a separate kustomize root

`ValidatingAdmissionPolicy`/`ValidatingAdmissionPolicyBinding` are
cluster-scoped, not namespaced - but `cloud/envs/hallen/kustomization.yaml`
sets a blanket `namespace: huemie-cloud` on everything it aggregates, and
kustomize's namespace transformer doesn't recognize these newer kinds as
cluster-scoped, so it stamps `metadata.namespace: huemie-cloud` onto them
anyway (verified: `kubectl kustomize cloud/envs/hallen` does exactly this
if these objects are included there). The API server then rejects them at
sync time - loud, not silent, but still broken. Keeping them in their own
kustomize root with no namespace transformer avoids that entirely. Same
reasoning as `../application-crds` being separate, just for a different
kind of cluster-scoped object, on the other cluster ("hallen", not the
ArgoCD control-plane cluster - unlike `application-crds`, this targets the
same cluster as `cloud/envs/hallen`, just as its own separate Application).

## Bootstrapping

Registration is git-managed, not a manual `argocd app create`:
`hallen-gitops-base/application-crds/app-humi-cloud-cluster-policies.yaml`
is the Application object that points at this path (`destination.name:
hallen`, same as `cloud/envs/hallen`'s own Application via
`app-humi-cloud.yaml`), on a sync-wave before it so this policy is in place
before appliance-registry's ServiceAccount/Deployment come up. Once that
file is merged and `hallen-gitops-base/application-crds` syncs, this path
is registered - nothing to run by hand.

## Requirements

`ValidatingAdmissionPolicy` is GA in `admissionregistration.k8s.io/v1`
since Kubernetes 1.30. Confirm "hallen" is running that version or newer -
on an older cluster the apiserver rejects this object outright (a clear
sync failure, not a silently-missing protection).
