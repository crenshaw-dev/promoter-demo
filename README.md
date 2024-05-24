# GitOps Promoter Demo

## 1. Install Argo CD

```shell
git clone https://github.com/argoproj/argo-cd
cd argo-cd
git remote add crenshaw-dev https://github.com/crenshaw-dev/argo-cd
git checkout manifest-hydrator
kubectl create namespace argocd
kubectl config set-context --current --namespace=argocd
kubectl apply -f manifests/install.yaml
# Scale down the application-controller and applicationset-controller. We'll run those locally.
kubectl scale sts application-controller --replicas=0
kubectl scale deployment applicationset-controller --replicas=0
```

In a separate terminal, port-forward Redis.

```shell
kubectl port-forward svc/argocd-redis 6379:6379
```

Finally, start Argo CD.

```shell
make start-local ARGOCD_START="api-server ui applicationset-controller controller repo-server"
```

## 2. Install the Promoter

Follow the instructions here: https://github.com/zachaller/promoter/tree/main/docs

## 3. Set up Credentials

As part of the promoter installation, you should have created a secret with the private key, installation ID, and 
app ID. We need those same credentials for the source hydrator.

```shell
kubectl get secret -n default my-auth -ojson | jq '.data | {
  "apiVersion": "v1", 
  "kind": "Secret", 
  "type": "Opaque", 
  "metadata": {
    "name": "my-auth", 
    "namespace": "argocd", 
    "labels": {
      "argocd.argoproj.io/secret-type": "hydrator"
    }
  }, 
  "data": {
    "githubAppID": .appID, 
    "githubAppInstallationID": .installationID, 
    "githubAppPrivateKey": .privateKey, 
    "type": ("git" | @base64), 
    "url": ("https://github.com/crenshaw-dev" | @base64)
  }
}' | kubectl apply -n argocd -f -
```

## 4. Install an ApplicationSet that uses the source hydrator

Apply this manifest. It will create three applications, one for each environment and region. The application names 
correspond to the .argocd-source-<app name>.yaml files in the helm-guestbook directory. The branches correspond to those
in the PromotionStrategy from the promoter installation docs.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: appset
  namespace: argocd
spec:
  goTemplate: true
  generators:
    - matrix:
        generators:
        - list:
            elements:
            - name: dev
            - name: test
            - name: prod
        - list:
            elements:
            - region: east
            - region: west
  template:
    metadata:
      name: '{{.name}}-{{.region}}'
    spec:
      sourceHydrator:
        drySource:
          path: helm-guestbook
          repoURL: https://github.com/crenshaw-dev/promoter-demo
          targetRevision: HEAD
        hydrateTo:
          targetRevision: environments/{{.name}}-next
        syncSource:
          targetRevision: environments/{{.name}}
          path: "{{.name}}-{{.region}}"
      destination:
        namespace: default
        server: https://kubernetes.default.svc
      project: default
      syncPolicy:
        automated: {}
```

## 5. Push a change to the dry sources

Make a change to this repo in the helm-guestbook directory. Ideally, make it to a file that affects all three
environments and regions. Push the change to main. You should see three PRs automatically created in the repo
representing the promotion of the change through the environments.
