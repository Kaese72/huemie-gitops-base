# appliance/optional/otel-collection

Optional telemetry for an appliance: traces from the Humi services, plus
container logs and node/kubelet metrics, shipped to the central monitoring
stack on hallen (`https://otlp.hallen.kaese.space`, Grafana at
`https://grafana.hallen.kaese.space`).

**Nothing links this in.** It is not part of `appliance/envs/*`, and the
installer only applies `appliance/application/`. An appliance without it
runs exactly as before. Its services find no `telemetry-env` ConfigMap,
so huemie-lib's `telemetry` package records and exports nothing.

## What's here

| File | What |
|---|---|
| `application.example.yaml` | ArgoCD Application to apply by hand: the collector Helm chart + `manifests/` |
| `values.yaml` | Helm values for the `otel-agent` DaemonSet |
| `manifests/namespace.yaml` | `otel-collection` namespace |
| `manifests/telemetry-env.yaml` | `telemetry-env` ConfigMap in `huemie-local`: the `OTEL_*` env vars the services pick up |
| `manifests/network-policies.yaml` | Egress from `huemie-local` (default-deny) to the agent on 4318 |

How data flows:

```
huemie-local pods --OTLP/HTTP :4318--> otel-agent (otel-collection)
node container logs ----------------/        |
                                             | OTLP/HTTP + bearer token
                                             v
                          otlp.hallen.kaese.space (otel-ingest) --> Tempo / Loki / Prometheus
```

## Installing on an appliance

1. **Ingest token on hallen.** In `hallen-gitops-base/instance-monitoring`, add
   the appliance's name to `CLUSTERS` in `bootstrap-secrets.yaml`. In
   `values/otel-ingest.yaml`, add a `TOKEN_<NAME>` env var and a `tokens`
   entry for it. Once synced, read the generated token:

   ```sh
   kubectl --context hallen -n monitoring get secret otlp-ingest-tokens \
     -o jsonpath='{.data.<name>}' | base64 -d
   ```

2. **Secret on the appliance.** `cluster-name` becomes `k8s.cluster.name` on
   everything the appliance sends. Use the same name as in step 1.

   ```sh
   kubectl create namespace otel-collection
   kubectl -n otel-collection create secret generic otlp-token \
     --from-literal=token=<token> --from-literal=cluster-name=<name>
   ```

3. **Application.** `kubectl apply -f application.example.yaml`, after
   editing it if needed (e.g. `targetRevision`). The agent waits in
   `CreateContainerConfigError` until the Secret exists, so the order of
   steps 2 and 3 doesn't matter.

4. **Restart the services** so they read `telemetry-env` (env vars are only
   read at pod start):

   ```sh
   kubectl -n huemie-local rollout restart deployment
   ```

   Each service logs `tracing enabled` on start when it picked the config up.
   This needs service images built against huemie-lib's `telemetry` package;
   older images ignore the variables.

## Uninstalling

Delete the Application (with cascade) and the `otlp-token` Secret, then
restart the `huemie-local` Deployments again. Remove the token on hallen too.

## Turning parts off

- **Logs, metrics, but no traces:** remove `telemetry-env.yaml` from
  `manifests/kustomization.yaml` (or create it by hand only where wanted).
- **Traces only:** disable the `logsCollection`, `kubeletMetrics` and
  `hostMetrics` presets in `values.yaml`.
- **Tracing in one service only:** set `OTEL_SDK_DISABLED=true` on the
  others' Deployments; they still pass trace context through.

## Towards something more rigorous

This is deliberately the minimum: a hand-made token per appliance and an
Application applied by hand. The pieces are laid out so that each step below
can replace one part without touching the services, which only ever see
standard `OTEL_*` env vars and an OTLP endpoint on the node:

- **Provisioning:** the ingest token could be issued by appliance-registry
  at enrollment (the appliance already holds cloud credentials), instead of
  being copied by hand. Rotation then follows the appliance secret's.
- **Identity binding:** otel-ingest currently accepts any listed token and
  trusts whatever `k8s.cluster.name` the sender sets. A gateway that
  maps the authenticated token to the cluster name (overwriting the
  attribute) would stop one appliance from posing as another.
- **Install:** once the above exists, the Application can move into
  `appliance/application/` behind an installer flag. Or the services can
  default to telemetry on, with the ConfigMap shipped in `base/default`.
- **Durability:** a persistent `file_storage` sending queue on the agent,
  so data survives agent restarts while an appliance is offline.
- **Sampling:** `OTEL_TRACES_SAMPLER` in `telemetry-env`, or tail sampling
  at the ingest.
- **Coverage:** RabbitMQ messages don't carry trace context yet, so a
  device update and the rule it triggers are separate traces. The adapters
  (sdup-lib) aren't instrumented yet either.
