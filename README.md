# Kratix and idpbuilder

The purpose of this project is to demonstrate that we can use [idpbuilder](https://cnoe.io/docs/intro/idpbuilder) as IDPlatform to install [Kratix](https://docs.kratix.io) with fewer efforts as it is needed when using bash scripts or manual steps.

Such a process is simplified as the tool `idpbuilder` allows a user to create locally a kind cluster embedding: Argo CD, Gitea and Nginx ingress 
and where we can easily deploy Kratix.

For that purpose, different Argo CD Applications manifests have been created under the path `idp/<PACKAGE_NAME>` in order to provision the cluster with the following packages:
- `foundation`: installing the mandatory components like: cert-manager, ...
- `kratix`: Deploy the kratix's controller; some default `Destination` and `GitstateStore` resources and register on the gitea server a new git organization: `kratix` and repository: `state`

To create such an IDPlatform, execute the following command:
```shell
idpbuilder create --recreate --color --dev-password \
  --name kratix \
  --port 8443 \
  -p idp/foundation \
  -p idp/kratix
```

When done, you can access the Argo CD: https://argocd.cnoe.localtest.me:8443 or Gitea - https://gitea.cnoe.localtest.me:8443 dashboards using the following credentials:
```shell
❯ idp get secrets
NAME                          NAMESPACE   USERNAME     PASSWORD    TOKEN                                      DATA
argocd-initial-admin-secret   argocd      admin        developer                                              
gitea-credential              gitea       giteaAdmin   developer   34d333ee4330d441c3da782fd4bc443a848e5a1b   
```

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

## Add new destinations

To simulate a more natural environment running in a company, we will now create some additional clusters representing either the `dev`, `test` and `prod` machines or different machines created for: `team-1`, `team-2` having different needs, services that kratix can deal with using `Promises` and `requests`.

For that purpose we will create top of the IDPlatform cluster some additional clusters using vcluster - https://www.vcluster.com/docs

So let's create some vclusters: `worker-1` and `worker-2`
```shell
idpbuilder create --color --dev-password \
        --name kratix \
        --port 8443 \
        -p idp/foundation \
        -p idp/kratix \
        -p idp/kratix-agents
```

**Note**: To access the cluster, it is needed to execute the command: `vcluster connect worker-2` responsible to create a kubectl's container acting as proxy able to access from your laptop the vcluster.

You can create more vclusters if you change the helm values of the Application CR of [kratix-agents.yaml](idp/kratix-agents/kratix-agents.yaml)
```yaml
        clusters:
          - name: worker-1
            path: worker-1
            organization: team-a
          - name: worker-2
            path: worker-2
            organization: team-b
          - name: worker-3
            path: worker-3
            organization: team-c
```

To uninstall a vcluster: `helm uninstall worker-2 -n worker-2`

TODO:
- Use argocd to install the resources of the kratix's agent instead of deploying them manually with the command `k --context "$WORKER1" apply -f kratix-destination/worker-1-resources.yml`
- Generate using a template engine (aka helm) the resources to be created to:
  - Create a vcluster, deploy argocd
  - Install on the vcluster the generated resources: Argo Applications and Secret to access the gitea server
  - Create and execute a new job able to add on the existing gitea server (under the org/repository: `kratix/state`) the folder of the new destination (aka vcluster)
  - Create for the IDPlatform running Kratix new resources: Destination & GitStateStore to register a new destination (aka vcluster)

Enjoy ;-)