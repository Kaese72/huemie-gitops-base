# Local development environment how-to

## Gitea configuration

Commited kustomization files in this reposity under git-server
deployed by

```
kubectl kustomize .\local-development\git-server\ > git-server.yaml
kubectl create namespace git-server
kubectl -n git-server apply -f .\git-server.yaml
```

You need a way to access the GitServer. I have so far just forwarded it with minikube.

```
minikube service -n git-server gitea
```

> NOTE: This URL will be used to add a git origin, so we need to make this static. Works for now though.


Once in the UI. Click install.
Then register a new user. This user is now the admin.
Create a repository named `huemie-local-development`
- Make it open so ArgoCD can read it without credentials
Then add that repository as `local-development` origin in `huemie-gitops-base` where the local development stuff are

```
git remote add local-development http://127.0.0.1:57220/kaese/huemie-local-development.git
```

Then we can use whatever branch we want to use for local development and push it to the remove master

```
git checkout -b local-development
git push -u local-development local-development:master --force-with-lease
```

> NOTE: the `master` branch on the `local-development` remote will move around to whatever you need it to be, so the environment may need to be reset once in a while since the persistence may not match your development. This is designed to be reset rather than maintained for a long time.

> NOTE: You will need to authenticate iwth Oauth, which works pretty well

When you eventually want to push to the real code repository, you simply change the remote, but be wary with force-pushing

```
git push -u local-development local-development:master
```

> NOTE: You may need to rebase on top of `origin/master` and handle conflicts introduced by others before pushing without force.

## ArgoCD configuration

Installed by following https://argo-cd.readthedocs.io/en/stable/operator-manual/core/#installing

```
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.3.1/manifests/core-install.yaml
```

I also installed the argocd cli according to https://argo-cd.readthedocs.io/en/stable/cli_installation/

```
$version = (Invoke-RestMethod https://api.github.com/repos/argoproj/argo-cd/releases/latest).tag_name
(resulted in version "v3.3.1")
$url = "https://github.com/argoproj/argo-cd/releases/download/" + $version + "/argocd-windows-amd64.exe"
$output = "argocd.exe"
Invoke-WebRequest -Uri $url -OutFile $output
```

This is the binary I am using to access argocd

At this point I had issues with ArgoCD not starting, so I asked github copilot to look into it and it restarted a bunch of ArgoCD pods
and now it works... Unclear exactly why, but if it works it works.

ArgoCD core is quite minimal, so it does not come with a UI, but the `argocd` cli can give this to us.

```
./argocd --core login
./argocd -n argocd admin dashboard
```

Theo utput inidicates where you need to go to find what you are looking for.

Since you have configured gitea to use the `master` branch at all times, we need to create
and application that points to `master` of the internal gitea instance. Configure an application with 
the following important information

* URL - `http://gitea.git-server:3000/kaese/huemie-local-development.git`
* Namespace: Not Set (this is set in the manifests for now)
* Path: `local-development`
* Target revision: `master`
* Cluster: `in-cluster`

The `local-development` folder contains a patch that adds `local.huemie.space` to the ingress, and we need to 
add that to our `Hosts` file like so,

```
127.0.0.1 local.huemie.space
```

Then we need to tunnel Minikube the minikube ingress like so

```
minikube tunnel huemie-ingress
```

Once this is done, we should be able to access http://local.huemie.space and see it running
what is at the top of `master` of `gitea`.

## Building for local environment

In order to build for the minikube cluster you need to would need to run something like this

```
minikube -p minikube docker-env --shell powershell | Invoke-Expression
docker build -t <application>:<tag> .
```

You should tag it with the format of the image we already have in the cluster, and have
the tag just be something random since it will not be committed anyway.

This leads to `<application>:<tag>` being available in the minikube image registry.
