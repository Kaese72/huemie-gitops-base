# Development

## Accessing ArgoCD on an appliance

ArgoCD runs inside the appliance's k3s cluster and is not exposed outside it.
To reach its UI, tunnel a port-forward through SSH:

```sh
ssh -t -L 8080:127.0.0.1:8080 <user>@<appliance-host> "su -c 'k3s kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath={.data.password} | base64 -d; echo; k3s kubectl -n argocd port-forward svc/argocd-server 8080:443'"
```

Replace `<user>` with your SSH user on the appliance and `<appliance-host>` with
its IP address or hostname.

What it does:

1. `su -c` asks for the appliance's root password, because `k3s kubectl`
   needs root.
2. It prints the ArgoCD `admin` password from `argocd-initial-admin-secret`.
3. It port-forwards `argocd-server` to port 8080 on the appliance. `-L`
   carries that port to 8080 on your machine. The forward runs until you
   press Ctrl+C.

Then open <https://localhost:8080> and log in as `admin` with the printed
password. ArgoCD serves a self-signed certificate, so the browser shows a
warning.

The command has no `$` in it, so it can be pasted unchanged into PowerShell,
Git Bash or cmd.

If local port 8080 is taken, change the first number in `-L`, e.g.
`-L 9090:127.0.0.1:8080`, and open <https://localhost:9090> instead.

If the password line says the secret was not found, the initial admin
password has been changed and `argocd-initial-admin-secret` deleted. Use the
current password instead.
