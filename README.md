# Self Managed Argo CD - App of Everything

**Table of Contents**
https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/
- [Introduction](#introduction)
- [Intall Argo CD Using Helm](#intall-argo-cd-using-helm)
- [Cleanup](#cleanup)

# Introduction
This project aims to install a self-managed Argo CD using the App of App pattern. Full instructions and explanation can be found in the Medium article [Self Managed Argo CD — App Of Everything](https://medium.com/devopsturkiye/self-managed-argo-cd-app-of-everything-a226eb100cf0).

# Intall Argo CD Using Helm
Go to argocd directory.
```
cd argocd/argocd-install/
```

Intall Argo CD to *argocd* namespace using argo-cd helm chart overriding default values with *values-override.yaml* file. If argocd namespace does not exist, use *--create-namespace* parameter to create it.
```
helm install argocd ./argo-cd \
    --namespace=argocd \
    --create-namespace \
    -f values-override.yaml
```

Wait until all pods are running.
```
kubectl -n argocd get pods

NAME                                            READY   STATUS    RESTARTS
argocd-application-controller-bcc4f7584-vsbc7   1/1     Running   0       
argocd-dex-server-77f6fc6cfb-v844k              1/1     Running   0       
argocd-redis-7966999975-68hm7                   1/1     Running   0       
argocd-repo-server-6b76b7ff6b-2fgqr             1/1     Running   0       
argocd-server-848dbc6cb4-r48qp                  1/1     Running   0
```

Get initial admin password.
```
kubectl -n argocd get secrets argocd-initial-admin-secret \
    -o jsonpath='{.data.password}' | base64 -d
```

Forward argocd-server service port 80 to localhost:8080 using kubectl.
```
kubectl -n argocd port-forward service/argocd-server 8080:80
```

Browse http://localhost:8080 and login with initial admin password.

# Demo With Sample Application
Create an application project definition file called *sample-project*.
```
cat << EOF > argocd-appprojects/sample-project.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: sample-project
  namespace: argocd
spec:
  clusterResourceWhitelist:
  - group: '*'
    kind: '*'
  destinations:
  - namespace: sample-app
    server: https://kubernetes.default.svc
  orphanedResources:
    warn: false
  sourceRepos:
  - '*'
EOF
```

Push changes to your repository.
```
git add argocd-appprojects/sample-project.yaml
git commit -m "Create sample-project"
git push
```

Create a saple applicaiton definition yaml file called *sample-app* under argocd-apps.
```
cat << EOF >> argocd-apps/sample-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sample-app
  namespace: argocd
spec:
  destination:
    namespace: sample-app
    server: https://kubernetes.default.svc
  project: sample-project
  source:
    path: sample-app/
    repoURL: https://github.com/kurtburak/argocd.git
    targetRevision: HEAD
  syncPolicy:
    syncOptions:
    - CreateNamespace=true
    automated:
      selfHeal: true
      prune: true
EOF
```

Push changes to your repository.
```
git add argocd-apps/sample-app.yaml
git commit -m "Create application"
git push
```

# Cleanup
Remove application and applicaiton project.
```
rm -f argocd-apps/sample-app.yaml
rm -f argocd-appprojects/sample-project.yaml
git rm argocd-apps/sample-app.yaml
git rm argocd-appprojects/sample-project.yaml
git commit -m "Remove app and project."
git push
```
