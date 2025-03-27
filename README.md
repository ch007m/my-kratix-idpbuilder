# Kratix and idpbuilder

The purpose of this project is to demonstrate that we can use [idpbuilder](https://cnoe.io/docs/intro/idpbuilder) as IDPlatform to install [Kratix](https://docs.kratix.io) with fewer efforts as it is needed when using bash scripts or manual steps.

For that purpose, different Argo CD Applications manifests have been created in order to install the `foundation` needed to operate Kratix
which is currently: `cert-manager` and to create top of the existing gitea server a new git organization: `kratix` and repository: `platform`
to let Kratix to handle the resources to be installed using Promises.

To create a local kind cluster embedding: Argocd, Gitea, Nginx ingress and Kratix, execute the following command:
```
idpbuilder create --recreate --color --name kratix --port 8443 --dev-password -p idp/foundation -p idp/kratix
```

When done, you can access the Argo CD  https://argocd.cnoe.localtest.me:8443 or Gitea - https://gitea.cnoe.localtest.me:8443 dashboards using the following credentials:
```shell
❯ idp get secrets
NAME                          NAMESPACE   USERNAME     PASSWORD    TOKEN                                      DATA
argocd-initial-admin-secret   argocd      admin        developer                                              
gitea-credential              gitea       giteaAdmin   developer   34d333ee4330d441c3da782fd4bc443a848e5a1b   
```

**Remark**: This project only support at the moment to create a single kind cluster !

## Test the IDPlatform with a Postgresql promise  

To validate that the platform is working and can operate `promises`, execute the following commands:
```
kubectl apply -f https://raw.githubusercontent.com/syntasso/promise-postgresql/main/promise.yaml
kubectl apply -f https://raw.githubusercontent.com/syntasso/promise-postgresql/main/resource-request.yaml
```
Wait till the Postgresql operator finished to create the pod running the database and then access the pod

```shell
kubectl rollout status deployment postgres-operator -n default --timeout=180s

kubectl exec -it acid-example-postgresql-0 -- sh -c "
PGPASSWORD=$(kubectl get secret postgres.acid-example-postgresql.credentials.postgresql.acid.zalan.do -o 'jsonpath={.data.password}' | base64 -d) \
PGUSER=$(kubectl get secret postgres.acid-example-postgresql.credentials.postgresql.acid.zalan.do -o 'jsonpath={.data.username}' | base64 -d) \
psql bestdb"
```

## Adding some workers

```shell
kind create cluster --name worker-1
set -x WORKER kind-worker-1

kubectl --context {$WORKER} create ns argocd
kubectl --context {$WORKER} apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl --context {$WORKER} apply -f kratix-destination/

kubectl --context {$WORKER} get applications -A
NAMESPACE   NAME                           SYNC STATUS   HEALTH STATUS
argocd      kratix-workload-dependencies   Unknown       Healthy
argocd      kratix-workload-resources      Unknown       Healthy
```

TODO: Error to be fixed:
```text
Message: Failed to load target state: failed to generate manifest for source 1 of 1: rpc error: code = Unknown desc = failed to list refs: Get "https://gitea.cnoe.localtest.me:8443/kratix/state/info/refs?service=git-upload-pack": dial tcp 127.0.0.1:8443: connect: connection refused
```

To uninstall the helm release, delete the cluster, etc
```shell
#helm --kube-context {$WORKER} uninstall kratix-destination
kind delete cluster --name worker-1
```

New test using idpbuilder

```text
idp create --dev-password --port 7443 --name worker1 --recreate --color
idp create --dev-password --name worker2 --port 9443 --recreate --color --kind-config kind-worker2.cfg

set -x WORKER kind-worker2
kubectl --context {$WORKER} apply -f kratix-destination/
```


