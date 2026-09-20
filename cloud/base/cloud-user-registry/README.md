# cloud-user-registry secrets

Unlike the appliance manifests (`appliance/base/default`), this folder does
**not** bootstrap its own secrets via an in-cluster Job. The appliance runs
on hardware you own, so generating credentials on first boot is fine; the
`hallen` cluster is shared cloud infrastructure, so `cloud-user-registry`'s
credentials and signing key are generated and applied out of band instead,
and only referenced by name here.

`cloud-user-registry.yaml` expects the following to already exist in the
`huemie-cloud` namespace before this kustomization is synced:

## `cloud-user-registry-secret`

A generic `Secret` with the following keys:

| Key | Used for |
|---|---|
| `database.password` | the `cloud-user-registry` MariaDB user's password (read by the `User` CR's `passwordSecretKeyRef` and by the app/migrater as `DATABASE_PASSWORD`) |
| `auth.refresh-secret` | HMAC secret the app signs refresh tokens with, as `AUTH_REFRESH_SECRET` |
| `smtp.host` | hostname of the SMTP relay used to send password-reset emails, as `SMTP_HOST` |
| `smtp.port` | SMTP relay port, as `SMTP_PORT` (587 if omitted) |
| `smtp.username` | SMTP auth username, as `SMTP_USERNAME` (leave empty/omit if the relay allows unauthenticated sends from the cluster egress IP) |
| `smtp.password` | SMTP auth password, as `SMTP_PASSWORD` |
| `smtp.from` | `From:` address on outgoing mail, as `SMTP_FROM` |
| `auth.service-tokens` | comma-separated bearer token(s) other cloud services use on the internal-only listener (port 8081, never routed by the ingress), as `AUTH_SERVICE_TOKENS`. `appliance-registry-secret`'s `user-registry.service-token` must be one of them |

Generate and apply it once:

```sh
DB_PASSWORD=$(head -c 32 /dev/urandom | base64 | tr -d '\n')
REFRESH_SECRET=$(head -c 32 /dev/urandom | base64 | tr -d '\n')
# Keep this - the same value goes into appliance-registry-secret as
# user-registry.service-token.
USER_REGISTRY_SERVICE_TOKEN=$(head -c 32 /dev/urandom | base64 | tr -d '\n')

kubectl create secret generic cloud-user-registry-secret -n huemie-cloud \
  --from-literal="database.password=$DB_PASSWORD" \
  --from-literal="auth.refresh-secret=$REFRESH_SECRET" \
  --from-literal="auth.service-tokens=$USER_REGISTRY_SERVICE_TOKEN" \
  --from-literal="smtp.host=$SMTP_HOST" \
  --from-literal="smtp.port=$SMTP_PORT" \
  --from-literal="smtp.username=$SMTP_USERNAME" \
  --from-literal="smtp.password=$SMTP_PASSWORD" \
  --from-literal="smtp.from=$SMTP_FROM"
```

The reset link's base URL (`PASSWORD_RESET_URL_BASE`, the cloud-ui page a
recipient lands on) is not secret and is set directly as a plain env var in
`cloud-user-registry.yaml`.

## `cloud-user-registry-rsa-signing-key` / `cloud-user-registry-rsa-verify-key`

The RSA keypair `cloud-user-registry` signs ("use" tokens) and verifies with.
Only the private key is currently mounted into the Deployment
(`AUTH_RSA_PRIVATE_KEY_PATH`); the public key is split into its own secret
so other cloud services can later verify these tokens without also holding
the private key, the same split the appliance uses for
`auth-rsa-signing-key`/`auth-rsa-verify-key`.

Generate and apply it once:

```sh
openssl genrsa -out private.pem 4096
openssl rsa -in private.pem -pubout -out public.pem

kubectl create secret generic cloud-user-registry-rsa-signing-key -n huemie-cloud \
  --from-file=private.pem=private.pem
kubectl create secret generic cloud-user-registry-rsa-verify-key -n huemie-cloud \
  --from-file=public.pem=public.pem

rm private.pem public.pem
```

## Rotation

Both secrets are read by the Deployment on pod start only. Rotating either
one means replacing the `Secret` (`kubectl create ... --dry-run=client -o
yaml | kubectl apply -f -`, or delete-and-recreate) and then restarting the
`cloud-user-registry` Deployment. Rotating the refresh secret or signing key
invalidates every outstanding refresh/use token, forcing all users to log
in again.
