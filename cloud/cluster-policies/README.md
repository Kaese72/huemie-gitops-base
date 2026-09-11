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

## Bootstrapping (one-time, outside this repo)

Register this path as its own ArgoCD Application targeting `hallen`
(`destination.name: hallen`, same as `cloud/envs/hallen`'s own
Application), the same way every top-level kustomize root in this
ecosystem gets registered.

## Requirements

`ValidatingAdmissionPolicy` is GA in `admissionregistration.k8s.io/v1`
since Kubernetes 1.30. Confirm "hallen" is running that version or newer -
on an older cluster the apiserver rejects this object outright (a clear
sync failure, not a silently-missing protection).
