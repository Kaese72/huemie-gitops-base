# appliance-registry secrets

Same reasoning as `cloud-user-registry` (see `../cloud-user-registry/README.md`):
this runs on shared cloud infrastructure, so its credentials are generated
and applied out of band instead of bootstrapped in-cluster.

`appliance-registry.yaml` expects the following to already exist in the
`huemie-cloud` namespace before this kustomization is synced:

## `appliance-registry-secret`

A generic `Secret` with four keys:

| Key | Used for |
|---|---|
| `database.password` | the `appliance-registry` MariaDB user's password (read by the `User` CR's `passwordSecretKeyRef` and by the app/migrater as `DATABASE_PASSWORD`) |
| `auth.service-tokens` | comma-separated bearer token(s) for the generic internal-caller listing path (`AUTH_SERVICE_TOKENS`) |
| `auth.plugin-tokens` | comma-separated bearer token(s) for the ArgoCD ApplicationSet Plugin generator specifically (`AUTH_PLUGIN_TOKENS`) |
| `user-registry.service-token` | the bearer token appliance-registry presents to cloud-user-registry's internal listener for group-membership lookups (`USER_REGISTRY_SERVICE_TOKEN`). Must be one of the values in `cloud-user-registry-secret`'s `auth.service-tokens` |

These are two **independent** token sets, not one shared secret - see
appliance-registry's README ("Architecture") for why: the plugin endpoint
is reachable over the public internet (ArgoCD generally can't reach this
service's in-cluster DNS), so a leak of that token shouldn't also
compromise whatever else calls the generic listing endpoint. Each is
comma-separated because appliance-registry accepts more than one valid
value at a time, so a token can be rotated by adding the new value first
and removing the old one later, instead of a single synchronized cutover.

Generate and apply it once:

```sh
DB_PASSWORD=$(head -c 32 /dev/urandom | base64 | tr -d '\n')
SERVICE_TOKEN=$(head -c 32 /dev/urandom | base64 | tr -d '\n')
PLUGIN_TOKEN=$(head -c 32 /dev/urandom | base64 | tr -d '\n')
# The value from cloud-user-registry-secret's auth.service-tokens (see
# ../cloud-user-registry/README.md).
USER_REGISTRY_SERVICE_TOKEN=<the same value>

kubectl create secret generic appliance-registry-secret -n huemie-cloud \
  --from-literal="database.password=$DB_PASSWORD" \
  --from-literal="auth.service-tokens=$SERVICE_TOKEN" \
  --from-literal="auth.plugin-tokens=$PLUGIN_TOKEN" \
  --from-literal="user-registry.service-token=$USER_REGISTRY_SERVICE_TOKEN"
```

Both `appliance-registry-secret` and `cloud-user-registry-secret` must contain
their new key **before** the corresponding Deployment is synced with this
change - the pods reference it without a default and will not start otherwise.
For an existing install, add the keys to the live Secrets first
(`kubectl patch secret ... --type merge -p '{"stringData":{...}}'`).

Keep `$PLUGIN_TOKEN` around - the same value goes into `argocd-secret` in
the ArgoCD control-plane cluster, per `../../application-crds/README.md`.
`$SERVICE_TOKEN` has no counterpart to keep in sync elsewhere yet - nothing
currently calls the generic listing path as a service (only the ArgoCD
plugin endpoint has a real caller today), so it's provisioned for whenever
something does.

## `cloud-user-registry-rsa-verify-key`

Not created here - this reuses the Secret `cloud-user-registry`'s own
kustomization already documents in `../cloud-user-registry/README.md`.
appliance-registry verifies the exact same use tokens cloud-user-registry
issues, so it needs that same public key, nothing appliance-registry-specific.

## Rotation

All three keys are read by the Deployment on pod start only - rotating any
of them means replacing the `Secret` and restarting the `appliance-registry`
Deployment. `auth.service-tokens`/`auth.plugin-tokens` accept a
comma-separated list specifically so this can be done as two steps instead
of one atomic cutover:

1. Add the new token to the relevant key alongside the old one
   (`auth.plugin-tokens=$OLD,$NEW`), restart the Deployment. Both are valid
   now.
2. Once every caller (for `auth.plugin-tokens`: `argocd-secret`'s
   `plugin.appliance-registry.token`, per `../../application-crds/README.md`)
   is confirmed using `$NEW`, replace the key with just `$NEW` and restart
   again.

Skipping straight to a single new value (the old one-step approach) still
works, it just means a window where the old and new values must be updated
in the same breath - miss it and the ApplicationSet Plugin generator starts
getting 401s and the fleet stops reconciling (existing
`cloud-connect-server-<id>` Applications keep running, they just stop being
kept in sync with appliance-registry's active-appliance list until the
token is fixed).
