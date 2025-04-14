# Kratix and idpbuilder

The purpose of this project is to demonstrate that we can use [idpbuilder](https://cnoe.io/docs/intro/idpbuilder) as IDPlatform to install [Kratix](https://docs.kratix.io) with fewer efforts as it is needed when using bash scripts or manual steps.

Such a process is simplified as the tool `idpbuilder` allows a user to create locally a kind cluster embedding: Argo CD, Gitea and Nginx ingress 
and where we can easily deploy Kratix.

For that purpose, different Argo CD Applications manifests have been created under the path `idp/<PACKAGE_NAME>` in order to provision the cluster with the following packages:
- `foundation`: installing the mandatory components like: cert-manager, ...
- `kratix`: Deploy the kratix's controller; some default `Destination` and `GitstateStore` resources and register on the gitea server a new git organization: `kratix` and repository: `state`

To create such an IDPlatform, execute the following command which is creating a kind cluster using the following ports:

| Name              | Ingress port | Gitea HTTP port | Gitea SSH port | kind config file name                                              |
|-------------------|--------------|-----------------|----------------|--------------------------------------------------------------------|
| Kratix IDPlatform | 8443         | 32223           | 32222          | [kratix-idp-8443.cfg](idp/kratix-idp-8443.cfg) |

```shell
idpbuilder create --color --dev-password \
  --name kratix \
  --port 8443 \
  --kind-config idp/kratix-idp-8443.cfg \
  -p idp/foundation \
  -p idp/kratix
```

**IMPORTANT**: The previous command will only work if you use a version of idpbuilder >= [0.10](https://github.com/cnoe-io/idpbuilder/releases/tag/v0.10.0-nightly.20250407)

**TODO**: Convert the `idp/kratix` resources folder into a helm chart able to configure path of the resources, URL of the gitea server, destination's labels, etc

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

## Add new destinations - idpbuilder

To simulate a more natural environment running in a company, we will now create some additional clusters representing either the `dev`, `test` and `prod` machines or different machines created for: `team-1`, `team-2` having different needs, services that kratix can deal with using `Promises` and `requests`.

For that purpose we will create some additional clusters using the `idpbuilder` tool. 

But prior to create a new cluster, it is needed first to register on Kratix the new Destination, one for each worker cluster.

To achieve this goal we will install a new idpbuilder package on the cluster using the helm chart `kratix-new-destination` able to:
- Populate the needed resources
- Execute a kubernetes job executing some `gitea curl commands` to create the folders `dependencies` and `resources` for a worker on the GitStateStore

**Note**: The package `kratix-new-destination` uses an [ApplicationSet](idp/kratix-new-destination/kratix-new-destination.yaml) resource able to create an Argo CD Application for each needed worker cluster. Feel free to review the ApplicationSet file to add more workers, change the labels, etc

```shell
idpbuilder create --color --dev-password \
  --name kratix \
  --port 8443 \
  --kind-config idp/kratix-idp-8443.cfg \
  -p idp/foundation \
  -p idp/kratix \
  -p idp/kratix-new-destination
```

So let's create now some worker clusters having the following ports

| Name    | Ingress port | kind config file name                      |
|---------|--------------|--------------------------------------------|
| worker1 | 8444         | [worker-1-8444.cfg](idp/worker-1-8444.cfg) |
| worker2 | 8445         | [worker-1-8445.cfg](idp/worker-1-8445.cfg) |

As the gitea server is running on the Kratix IDPlatform at the following address: `http://<HOST_IP_ADDRESS>:32223`, we will have to patch the Application CR of the `kratix-agent` before to deploy. Execute then this command before to create the different workers 

```shell
yq e '.spec.source.helm.valuesObject.giteaServer.url = "http://<HOST_IP_ADDRESS>:32223"' -i idp/kratix-agent/kratix-agent.yaml
```

When done, execute the following command respectively for the `worker1` and `worker2`
```shell
idpbuilder create --color --dev-password --recreate \
  --name <WORKER_NAME> \
  --port <INGRESS_PORT> \
  --kind-config idp/<KIND_CONFIG_FILE_NAME> \
  -p idp/kratix-agent
```

When the cluster has been created and Applications deployed, verify their status to check if the Application CR is sync and healthy

```shell
export CONTEXT="kind-worker1" # set CONTEXT "kind-worker-1"
kubectl --context "$CONTEXT" -n argocd get application -lcluster=worker-1
NAME                                    SYNC STATUS   HEALTH STATUS
kratix-workload-worker-1-dependencies   Synced        Healthy
kratix-workload-worker-1-resources      Synced        Healthy
```
If Argocd is able to watch resources from the kratix IDPlatform gitea server under `kratix/state/<WORKKER_NAME/resources | dependencies`, then you can start to play with Kratix and deploy some promises and requests against the different environments !

Enjoy ;-)

## How to play wit Kratix

In order to test the IDPlatform which has been installed, you will have to either create a new `Promise` (see hereafter) and a `Request` or use an existing promise published on one of the marketplaces:
- https://github.com/syntasso/kratix-marketplace (mongodb, redis, kafka, dapr, vault, etc)
- https://github.com/syntasso?q=promise (postgresql, rabbitmq, etc)


## Populate a promise from a helm chart

To create a Promise CR (and its request CR) from a helm chart, execute the following command:
```shell
rm -rf helm-postgresql; mkdir -p helm-postgresql
pushd helm-postgresql
kratix init helm-promise postresql --chart-url oci://registry-1.docker.io/bitnamicharts/postgresql --group snowdrop.dev --kind database
```
**Note**: During the init process, kratix will convert the Helm `values` file (of the chart) into an `openAPIV3Schema` and assign it as the Kratix Promise API definition:

```yaml
spec:
  api:
    apiVersion: apiextensions.k8s.io/v1
    kind: CustomResourceDefinition
    metadata:
      name: databases.snowdrop.dev
    spec:
      group: snowdrop.dev # API Group
      names:
        kind: database # Api kind
        plural: databases # Api plural
        singular: database
...
```

## Configure the Request and generate the manifests

Now that we have created the Promise and its `API`, we will create the folders/files that kratix will use to populate the resources from the Helm chart.
For that purpose, execute the following commands:

```shell
mkdir -p kratix/{input,output}; touch kratix/{input,output}/object.yaml
```

The content of the `kratix/input/object.yaml` will be used as input to customize the resources generated by helm during the execution of the `template` command:
```shell
helm template $name $arguments --values values.yaml > $KRATIX_OUTPUT/object.yaml
```

The variable `name` is coming from the resource `.metadata.name` and the values.yaml file from `.spec`

To create the `kratix/input/object.yaml`, we will use the input of the `example-resource.yaml` and customize it using the Promise API properties (aka helm values).

```YAML
# Example of promise for postgresql
apiVersion: snowdrop.dev/v1alpha1
kind: database
metadata:
  name: fruits-db
  namespace: kratix-requests # This namespace MUST exist within the target cluster !!
spec:
  environment: dev
  auth:
    database: fruits_database
    username: healthy
    password: healthy
```

Copy the promise request example to the `kratix/input/object.yaml` file.
```shell
cp example-resource.yaml kratix/input/object.yaml
```

We can now generate the helm chart using either this script:
```shell
#set -x CHART_URL https://repo.broadcom.com/bitnami-files
#set -x CHART_NAME postgresql
set -x KRATIX_INPUT kratix/input
set -x KRATIX_OUTPUT kratix/output
set -x CHART_URL oci://registry-1.docker.io/bitnamicharts/postgresql
curl -s https://raw.githubusercontent.com/syntasso/kratix-cli/60c88937545266127a2faea0e1132650d005cc5a/aspects/helm-promise/pipeline.sh | bash
...
helm template fruits-db oci://registry-1.docker.io/bitnamicharts/postgresql --values values.yaml
```
or using the kratix `helm-resource-configure` container `ghcr.io/syntasso/kratix-cli/helm-resource-configure:v0.1.0`
```shell
podman run --rm \
  -e CHART_URL=oci://registry-1.docker.io/bitnamicharts/postgresql \
  -v $(pwd)/example-resource.yaml:/kratix/input/object.yaml \
  -v $(pwd)/kratix/output:/kratix/output \
  ghcr.io/syntasso/kratix-cli/helm-resource-configure:v0.1.0
popd
```
**Note**: The resources generated by the helm chart are stored under the `kratix/output/object.yaml`.

When done, you can install the kratix Promise and a request to install a `Postgresql fruit db` under the destination `dev`.

To test the connectivity with the database, connect to the `vcluster` where the service has been installed and execute the commands: 

```shell
vcluster connect worker-1

set -x POSTGRES_PASSWORD $(kubectl get secret -n kratix-requests fruits-db-postgresql -o jsonpath="{.data.password}" | base64 -d)
kubectl run postgresql-client --rm --tty -i --restart='Never' \
        -n kratix-requests --image docker.io/bitnami/postgresql:14.5.0-debian-11-r14 \
        --env="PGPASSWORD=$POSTGRES_PASSWORD" \
        --command -- psql --host fruits-db-postgresql -U healthy -d fruits_database -p 5432
If you don't see a command prompt, try pressing enter.

fruits_database=>
...
vcluster disconnect
```

## Add new destinations - vclusters (DEPRECATED)

To simulate a more natural environment running in a company, we will now create some additional clusters representing either the `dev`, `test` and `prod` machines or different machines created for: `team-1`, `team-2` having different needs, services that kratix can deal with using `Promises` and `requests`.

For that purpose we will create top of the IDPlatform cluster some additional clusters using vcluster - https://www.vcluster.com/docs

So let's create some vclusters: `worker-1` and `worker-2`
```shell
idpbuilder create --color --dev-password \
        --name kratix \
        --port 8443 \
        -p idp/foundation \
        -p idp/kratix \
        -p idp/vcluster
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