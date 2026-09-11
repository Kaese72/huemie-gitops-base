# cloud/application-crds

Owns exactly one thing: the `ApplicationSet` that instantiates one
`cloud-connect-server-<id>` per active appliance in appliance-registry, plus
its ArgoCD Plugin generator registration - see
`applicationset-cloud-connect-server-fleet.yaml`'s own comments for the
mechanism, and appliance-registry's README ("Cloud-side cloud-connect-server
provisioning") for why it exists.

## Why this is a separate kustomize root

Everything else in this repo targets the `huemie-cloud` namespace on the
"hallen" cluster. This directory's two objects (`ApplicationSet`,
`ConfigMap`) belong in the `argocd` namespace on the ArgoCD **control-plane**
cluster instead - wherever the `applicationset-controller` itself runs,
which this repo assumes is a separate cluster from "hallen" (mirroring
hallen-gitops-base's own assumption that ArgoCD isn't running on the
cluster its Applications target). If your installation actually runs ArgoCD
on "hallen" itself, or uses a different namespace than `argocd`, adjust
`metadata.namespace` in `applicationset-cloud-connect-server-fleet.yaml`
accordingly before applying.

## Bootstrapping

1. **Registration is git-managed, not a manual `argocd app create`.**
   `hallen-gitops-base/application-crds/app-humi-cloud-application-crds.yaml`
   is the Application object that points at this path
   (`repoURL: huemie-gitops-base`, `path: cloud/application-crds`,
   `destination: server: https://kubernetes.default.svc` - the same
   control-plane cluster `hallen-gitops-base/application-crds` itself
   already runs on) - the same pattern that repo already uses for
   `cloud/envs/hallen` itself, via `app-humi-cloud.yaml`. Once that file is
   merged and `hallen-gitops-base/application-crds` syncs, this path is
   registered - nothing to run by hand.

2. **Register the plugin's token.** ArgoCD's Plugin generator reads it out
   of `argocd-secret` (a pre-existing Secret ArgoCD manages itself - see
   `applicationset-cloud-connect-server-fleet.yaml`'s comment on why that
   Secret isn't committed here). Add the key without disturbing the rest of
   what ArgoCD keeps there:

   ```sh
   # Use one of the values from appliance-registry's own auth.plugin-tokens
   # (NOT auth.service-tokens - they're deliberately separate, see
   # ../base/appliance-registry/README.md).
   kubectl patch secret argocd-secret -n argocd --type=merge -p \
     "{\"stringData\":{\"plugin.appliance-registry.token\":\"$PLUGIN_TOKEN\"}}"
   ```

3. Confirm `appliance-registry` is reachable at the `baseUrl` configured in
   `appliance-registry-plugin`
   (`https://cloud.humi.kaese.space/appliance-registry/v0/argocd-plugin`) -
   it needs its own public ingress path, added in
   `../base/ingress/ingress.yaml`, since the ArgoCD control-plane cluster
   generally can't reach "hallen"'s in-cluster Service DNS directly.

## Rotating the plugin token

See `../base/appliance-registry/README.md`'s "Rotation" section for the
two-step (add-then-remove) way to do this without a hard cutover window -
`auth.plugin-tokens` accepts a comma-separated list of currently-valid
values for exactly this reason. Whichever way you do it, until
`argocd-secret`'s `plugin.appliance-registry.token` matches one of the
values in appliance-registry's `auth.plugin-tokens`, the ApplicationSet
gets 401s from the plugin endpoint and stops picking up new/revoked
appliances - already-created `cloud-connect-server-<id>` Applications are
unaffected, they just stop being kept in sync until the token matches
again.
