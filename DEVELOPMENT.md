# Development

## Accessing ArgoCD on an appliance

The appliance runs only ArgoCD's core components (`core-install.yaml`):
application controller, ApplicationSet controller, repo server and Redis. It
has no ArgoCD API server, UI, Dex or notifications controller. To get the UI,
run it on your own machine with the `argocd` CLI, which talks straight to the
appliance's Kubernetes API.

### 1. Get the appliance's kubeconfig (once)

On the appliance, as root, print `/etc/rancher/k3s/k3s.yaml`:

```sh
ssh -t <user>@<appliance-host> "su -c 'cat /etc/rancher/k3s/k3s.yaml'"
```

Save the output, without the password prompt line, as e.g.
`~/.kube/humi-appliance.yaml`. Its server is `https://127.0.0.1:6443`, which
the tunnel in the next step makes work from your machine.

The file holds the cluster admin credentials. Keep it private.

### 2. Tunnel the Kubernetes API

```sh
ssh -N -L 6443:127.0.0.1:6443 <user>@<appliance-host>
```

Leave it running. If local port 6443 is taken, use another local port, e.g.
`-L 16443:127.0.0.1:6443`, and change the `server:` line in the kubeconfig to
match.

### 3. Run the dashboard

In another terminal:

```sh
argocd admin dashboard -n argocd --kubeconfig ~/.kube/humi-appliance.yaml
```

Then open <http://localhost:8080>. You don't need to log in: the dashboard
acts with the kubeconfig's permissions. Use `--port` to pick another local
port.

To use the CLI without the UI, log in in core mode:

```sh
export KUBECONFIG=~/.kube/humi-appliance.yaml
kubectl config set-context --current --namespace=argocd
argocd login --core
argocd app list
```

### Without the CLI

On the appliance itself, `sudo k3s kubectl -n argocd get applications` shows
the sync and health status of every Application.
