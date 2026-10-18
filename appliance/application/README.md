# appliance/application

The ArgoCD `Application` objects every appliance runs:

- `argocd.yaml` -- ArgoCD managing itself, from `appliance/argocd`
- `humi.yaml` -- the Humi services, from `appliance/envs/appliance`

Nothing in ArgoCD watches this directory (there is deliberately no
app-of-apps). The appliance installer
(cloud-ui's `public/static-files/appliance/installation/install-appliance.sh`) fetches
each file from `master` and `kubectl apply`s it directly. So a change here
only reaches an existing appliance when the installer is re-run on it, or
the file is applied by hand. Adding a new Application here also means adding
it to the installer's list.
