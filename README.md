# Kratix and idpbuilder

## TODO

- [ ] Move the kratix and related packages to `github.com/cnoe-io/stacks`
- [x] Merge the local packages with: `github.com/ch007m/my-idp-packages`
- [x] Review the `HowTo` guide to separate the process to generate the resources locally for testing purposes
- [x] Add step to include the `matchingSelector` of the Promise
- [x] Mention part of the `HowTo` guide what the kratix client generated under Promise's workflow

## Introduction

The purpose of this project is to demonstrate that we can use [idpbuilder](https://cnoe.io/docs/intro/idpbuilder) as IDPlatform to install [Kratix](https://docs.kratix.io) with fewer efforts as it is needed when using bash scripts or manual steps.

Such a process is simplified as the tool `idpbuilder` allows a user to create locally a kind cluster embedding: Argo CD, Gitea and Ingress Nginx and where we can easily deploy Kratix.

For that purpose, different `packages` containing Argo CD Application(Set) files, manifests, helm chart, etc. have been created within a Git repository in order to provision the IDPlatform.

To use the scenario 1 or 2, it is then needed first to git clone the following project or to fork it and to export the path:
```shell
git clone https://github.com/ch007m/my-idp-packages
export PACKAGES_REPO=$(pwd)/my-idp-packages # set -x PACKAGES_REPO $(pwd)/my-idp-packages
```
**Note**: If you fork it, then you can create a new branch to push your local changes when you will modify the Argo Application(Set) files of the packages to customize the Helm values, etc 

## Scenario 1: IDPlatform with multi-kind clusters

To create a Kratix - IDPlatform, you will then execute the following command able to create a kind cluster exposing different ports listed hereafter. The HTTP port of the Gitea Server is also mapped to a container engine port and can be accessed using either the NodePort: `http://<HOST_IP_ADDRESS>:32223` or Ingress route: `https://gitea.cnoe.localtest.me:8443`.

| Name              | Ingress port | Gitea HTTP port | Gitea SSH port | kind config file name                          |
|-------------------|--------------|-----------------|----------------|------------------------------------------------|
| Kratix IDPlatform | 8443         | 32223           | 32222          | [kratix-idp-8443.cfg](idp/kratix-idp-8443.cfg) |

As you can see, we pass as arguments different packages to install: cert-manager, kratix controller, etc. The purpose of the package `kratix-gitstore-job` as documented on the git repository package is to create a new Git organization `kratix` and repository `state` under the Gitea server and to deploy the `GitStateStore` resource. 
```shell
idpbuilder create --color --dev-password \
  --name kratix \
  --port 8443 \
  --kind-config idp/kratix-idp-8443.cfg \
  -p $PACKAGES_REPO/cert-manager \
  -p $PACKAGES_REPO/kratix \
  -p $PACKAGES_REPO/kratix-gitstore-job
```
**IMPORTANT**: The previous command will only work if you use a version of idpbuilder >= [0.10](https://github.com/cnoe-io/idpbuilder/releases/tag/v0.10.0-nightly.20250407)

When created, you can access the Argo CD: https://argocd.cnoe.localtest.me:8443 or Gitea - https://gitea.cnoe.localtest.me:8443 dashboards using the following credentials:
```shell
❯ idp get secrets
NAME                          NAMESPACE   USERNAME     PASSWORD    TOKEN                                      DATA
argocd-initial-admin-secret   argocd      admin        developer                                              
gitea-credential              gitea       giteaAdmin   developer   34d333ee4330d441c3da782fd4bc443a848e5a1b   
```

To simulate a more natural environment running in a company, we will now create some additional clusters representing either the `dev`, `test` and `prod` environments or machines created for teams/projects: `team-1`, `team-2` having different needs, services that kratix can deal with using `Promises` and `Requests`.

Prior to create a new cluster, it is needed first to register on the Kratix IDPlatform the new `Destination`, one for each `worker` cluster.

To achieve this goal we will install a new package on the cluster using the helm chart `kratix-new-destination` able to:
- Generate the `Destination` 
- Install a kubernetes job responsible to create the folders `dependencies` and `resources` for a new destination (aka worker) on the GitStateStore

**Note**: The package `kratix-new-destination` uses an [ApplicationSet](idp/kratix-new-destination/kratix-new-destination.yaml) resource able to create an Argo CD Application for each needed `worker` cluster. Feel free to review the ApplicationSet file to add more workers, change the labels, etc

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: kratix-new-destination
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - clusterName: worker-1
            repoPath: worker-1
            environment: dev
            team: team-a
          - clusterName: worker-2
            repoPath: worker-2
            environment: test
            team: team-b
...
```

When the file has been reviewed, you can install the package using the command:

```shell
idpbuilder create --color --dev-password \
  --name kratix \
  --port 8443 \
  --kind-config idp/kratix-idp-8443.cfg \
  -p $PACKAGES_REPO/cert-manager \
  -p $PACKAGES_REPO/kratix \
  -p $PACKAGES_REPO/kratix-gitstore-job \
  -p $PACKAGES_REPO/kratix-new-destination
```

**Note**: You can verify if a new `destination` folder has been created under the Kratix StateStore repository. Example: `https://gitea.cnoe.localtest.me:8443/kratix/state/src/branch/main/worker-1`

When done, we will create two new clusters having the following name and ports

| Name    | Ingress port | kind config file name                      |
|---------|--------------|--------------------------------------------|
| worker-1 | 8444         | [worker-1-8444.cfg](idp/worker-1-8444.cfg) |
| worker-2 | 8445         | [worker-1-8445.cfg](idp/worker-1-8445.cfg) |

and we will provision each using the package `kratix-new-agent`. 

This package generates the following resources:

- Argo CD Application CR able to watch the `dependencies` under the Gitea server repository path: `ORG/REPO/<WORKER_NAME>/dependencies`
- Argo CD Application CR able to watch the `resources` under the Gitea server repository path: `ORG/REPO/<WORKER_NAME>/respurces`
- A Secret to allow Argo CD to access the Gitea Server and to be authenticated

As this `package` has been designed as an Argo CD ApplicationSet file, it is needed to perform some changes in order to set up properly the `worker` cluster for this scenario.

Edit the `kratix-new-agent/kratix-new-agent.yaml` file to perform the following modifications: 
- Change the `helm` key `.spec.source.helm.valuesObject.giteaServer.url` with the IP address of your host machine include the nodePort: `http://<HOST_IP_ADDRESS>:32223`
- Uncomment the destination server and comment the name
- Review the `.spec.generator.list.elements` to define the key/value for a `worker`. This step must be repeated for each cluster to be created !
```yaml
spec:
  generators:
    - list:
        elements:
          - clusterName: worker-1
            repoPath: worker-1
          #- clusterName: worker-2
          #  repoPath: worker-2
...
    spec:
      project: default
      destination:
        # name: "{{clusterName}}" # name of the target cluster where the resources should be created
        server: "https://kubernetes.default.svc"
      source:
        repoURL: cnoe://kratix-worload
        targetRevision: HEAD
        path: .
        helm:
          valuesObject:
            ...
            giteaServer:
              url: http://<HOST_IP>:32223
```
**Note**: Alternatively you could also duplicate the folder `kratix-new-agent` to create a new package for each cluster to be provisioned (example: kratix-new-agent-worker-1, kratix-new-agent-worker-2, etc.)

When done, execute the following command:
```shell
idpbuilder create --color --dev-password \
  --name <WORKER_NAME> \
  --port <INGRESS_PORT> \
  --kind-config idp/<WORKER_KIND_CONFIG_FILE> \
  -p $PACKAGES_REPO/kratix-new-agent
```

When the worker cluster has been created and resources deployed, verify the status of the Argo CD Application which should be `synced` and `healthy`.

```shell
export CONTEXT="kind-worker-1" # set CONTEXT "kind-worker-1"
kubectl --context "$CONTEXT" -n argocd get application -lcluster=worker-1
NAME                                    SYNC STATUS   HEALTH STATUS
kratix-workload-worker-1-dependencies   Synced        Healthy
kratix-workload-worker-1-resources      Synced        Healthy
```
If Argocd is able to watch resources from the kratix IDPlatform gitea server under `kratix/state/<WORKKER_NAME/resources | dependencies`, then you can start [to play with Kratix](#how-to-play-wit-kratix) and deploy some promises and requests against the different environments !

Enjoy ;-)

## Scenario 2: IDPlatform and vclusters as alternative

Instead of using several kind clusters running as IDPlatform, you can also follow these instructions where we add as new
Kratix `agent`, a [vcluster](https://www.vcluster.com/docs) on an existing IDPlatform.

### Instructions

Create an IDPlatform cluster, deploy kyverno and create 2 vclusters: worker-1 and worker-2
```shell
idpbuilder create \
  --kind-config idp/kind-32223.cfg \
  --color \
  --dev-password \
  --name idplatform \
  --port 8443 \
  -p $PACKAGES_REPO/vcluster \
  -p $PACKAGES_REPO/kyverno --recreate
```
**Note: Kyverno is used as policy engine to create secrets. We could use as alternative: [External-Secrets](https://external-secrets.io/).

Deploy a Kyverno policy able to Create the `TLSconfig secret` used by Argo Secret to access the Kubernetes API of the different vclusters.
```shell
  idpbuilder create \
  --color \
  --dev-password \
  --name idplatform \
  --port 8443 \
  -p $PACKAGES_REPO/vcluster \
  -p $PACKAGES_REPO/kyverno \
  -p $PACKAGES_REPO/kyverno-policy-secret
```

Install now the kratix pre-requisites (cert manager, etc) and kratix
```shell
  idpbuilder create \
  --color \
  --dev-password \
  --name idplatform \
  --port 8443 \
  -p $PACKAGES_REPO/vcluster \
  -p $PACKAGES_REPO/kyverno \
  -p $PACKAGES_REPO/kyverno-policy-secret \
  -p $PACKAGES_REPO/cert-manager \
  -p $PACKAGES_REPO/kratix
```

Execute the job creating the Gitea Org `kratix` and StateStore repository `state`
```shell
  idpbuilder create \
  --color \
  --dev-password \
  --name idplatform \
  --port 8443 \
  -p $PACKAGES_REPO/vcluster \
  -p $PACKAGES_REPO/kyverno \
  -p $PACKAGES_REPO/kyverno-policy-secret \
  -p $PACKAGES_REPO/cert-manager \
  -p $PACKAGES_REPO/kratix \
  -p $PACKAGES_REPO/kratix-gitstore-job
```

Register 2 new destination(s) for `worker-1` and `worker-2` on the main cluster running kratix
```shell
  idpbuilder create \
  --color \
  --dev-password \
  --name idplatform \
  --port 8443 \
  -p $PACKAGES_REPO/vcluster \
  -p $PACKAGES_REPO/kyverno \
  -p $PACKAGES_REPO/kyverno-policy-secret \
  -p $PACKAGES_REPO/cert-manager \
  -p $PACKAGES_REPO/kratix \
  -p $PACKAGES_REPO/kratix-gitstore-job \
  -p $PACKAGES_REPO/kratix-new-destination
```  

Install Argocd on each vcluster. TODO: Find a way to install argocd agent, remove non needed servers: dex, etc
```shell
vcluster connect worker-1
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
vcluster disconnect

vcluster connect worker-2
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
vcluster disconnect
```

Edit the ApplicationSet file: `kratix-new-agent.yaml` of the package `kratix-new-agent` to set the IP address of your local machine
where the gitea server runs.

```yaml
  giteaServer:
    url: http://192.168.129.0:32223 # instead of https://gitea.cnoe.localtest.me:8443
```

Register the new agents for `worker-1` and `worker-2`.
```shell
  idpbuilder create \
  --color \
  --dev-password \
  --name idplatform \
  --port 8443 \
  -p $PACKAGES_REPO/vcluster \
  -p $PACKAGES_REPO/kyverno \
  -p $PACKAGES_REPO/kyverno-policy-secret \
  -p $PACKAGES_REPO/cert-manager \
  -p $PACKAGES_REPO/kratix \
  -p $PACKAGES_REPO/kratix-gitstore-job \
  -p $PACKAGES_REPO/kratix-new-destination \
  -p $PACKAGES_REPO/kratix-new-agent
```  

Continue with the [How to play](#how-to-play-wit-kratix) now ;-)

# How to play wit Kratix

In order to test if the Kratix IDPlatform which has been created is working properly, you will have to deploy a `Promise` and a `Request` 

**Note: You can either create a new `Promise` - see hereafter or to use an existing promise published on one of the marketplaces:
- https://github.com/syntasso/kratix-marketplace (mongodb, redis, kafka, dapr, vault, etc)
- https://github.com/syntasso?q=promise (postgresql, rabbitmq, etc)

## Populate a promise from a helm chart

To create a Kratix `Promise` from by example an existing helm chart like `bitnamicharts/postgresql`, execute the following command using the kratix [client](https://docs.kratix.io/main/kratix-cli/intro):
```shell
rm -rf helm-postgresql; mkdir -p helm-postgresql
pushd helm-postgresql
kratix init helm-promise postresql --chart-url oci://registry-1.docker.io/bitnamicharts/postgresql --group halkyon.io --kind database
```
**Note**: During the init process, kratix will convert the Helm `values` file (of the chart) into an `openAPIV3Schema` and assign it as the Kratix Promise API definition:

```yaml
spec:
  api:
    apiVersion: apiextensions.k8s.io/v1
    kind: CustomResourceDefinition
    metadata:
      name: databases.halkyon.io
    spec:
      group: halkyon.io # API Group
      names:
        kind: database # Api kind
        plural: databases # Api plural
        singular: database
      scope: Namespaced
      versions:
        - name: v1alpha1
          schema:
            openAPIV3Schema:
              type: object
              properties:
                spec:
                  type: object
                  properties:
                    architecture:
                      type: string
...
```
**Note**: Change the parameters `group` and `kind` according to your needs/preferences !

Part of the `promise.yaml` generated, you can see that a `Workflow Pipeline` has been created to generate from the helm chart the resources using the container `ghcr.io/syntasso/kratix-cli/helm-resource-configure:v0.1.0`
```yaml
...
workflows:
  promise:
    configure:
      []
  resource:
    configure:
      - apiVersion: platform.kratix.io/v1alpha1
        kind: Pipeline
        metadata:
          name: instance-configure
        spec:
          containers:
            - env:
                - name: CHART_URL
                  value: oci://registry-1.docker.io/bitnamicharts/postgresql
              image: ghcr.io/syntasso/kratix-cli/helm-resource-configure:v0.1.0
              name: instance-configure
```

As for this `HowTo` guide we will deploy a Postgresql DB on the `worker-1` cluster, which corresponds in fact to the `dev` destination, a small change is then required
in order to specify the destination. Edit the Promise file and include, within the spec part, a new field including a `Selector`
```yaml
spec:
  destinationSelectors:
    - matchLabels:
        environment: dev
```
**Note**: Different scheduling options are available and described [here](https://docs.kratix.io/main/reference/destinations/multidestination-management).

## Configure the Request

Now that we have created the Promise (Api, Workflow parts and destinationSelectors), we can create using the `example-resource.yaml` a Request named `fruits-db.yaml` and customize some fields to request to create a `fruits_database` with username/password: `healthy`.

```YAML
# Example of promise for postgresql
apiVersion: halkyon.io/v1alpha1
kind: database
metadata:
  name: fruits-db
  namespace: <KRATIX_REQUESTS_NAMESPACE> # This namespace MUST exist within the target cluster too !
spec:
  auth:
    database: fruits_database
    username: healthy
    password: healthy
```

When your `Request` file is ready, you can install the Kratix `Promise` and the `Request` on the Kratix IDPlatform. After a few moments, Kratix should have created a new pod on the target cluster/namespace.

```shell
kubectl get promise/postresql
NAME        STATUS      KIND       API VERSION           VERSION
postresql   Available   database   halkyon.io/v1alpha1   

kubectl get database/fruits-db
NAME        STATUS
fruits-db   Resource requested

export CONTEXT="kind-worker-1" # set CONTEXT "kind-worker-1"
kubectl --context "$CONTEXT" get pod/fruits-db-postgresql-0
NAME                     READY   STATUS    RESTARTS   AGE
fruits-db-postgresql-0   1/1     Running   0          2m1s
```

To test the connectivity with the database, connect to the `cluster` where the service has been installed and execute the commands:

```shell
export POSTGRES_PASSWORD=$(kubectl --context "$CONTEXT" get secret -n default fruits-db-postgresql -o jsonpath="{.data.password}" | base64 -d)
export PGPASSWORD=$(kubectl --context "$CONTEXT" get secret -n default fruits-db-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)
# set -x POSTGRES_PASSWORD $(kubectl --context "$CONTEXT" get secret -n default fruits-db-postgresql -o jsonpath="{.data.password}" | base64 -d)
# set -x PGPASSWORD $(kubectl --context "$CONTEXT" get secret -n default fruits-db-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)
kubectl --context "$CONTEXT" run postgresql-client --rm --tty -i --restart='Never' \
        -n default --image docker.io/bitnami/postgresql:14.5.0-debian-11-r14 \
        --env="PGPASSWORD=$POSTGRES_PASSWORD" \
        --command -- psql --host fruits-db-postgresql -U healthy -d fruits_database -p 5432
If you don't see a command prompt, try pressing enter.

fruits_database=>
fruits_database-> \l
                                     List of databases
      Name       |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-----------------+----------+----------+-------------+-------------+-----------------------
 fruits_database | healthy  | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =Tc/healthy          +
                 |          |          |             |             | healthy=CTc/healthy
 postgres        | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0       | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
                 |          |          |             |             | postgres=CTc/postgres
 template1       | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
                 |          |          |             |             | postgres=CTc/postgres
(4 rows)
```

Enjoy !

## Generate the manifests (optional)

If you would like to view/test the resources generated from the helm chart according to the Promise request resource, execute the following commands locally:

```shell
mkdir -p kratix/{input,output}; touch kratix/{input,output}/object.yaml
```

The content of the `kratix/input/object.yaml` will be used as input to customize the resources generated by helm during the execution of the `helm template` command:
```shell
helm template $name $arguments --values values.yaml > $KRATIX_OUTPUT/object.yaml
```

The variable `name` is coming from the Promise Request `.metadata.name` and the values.yaml file from the `.spec` (aka Promise customized API !)

To create the `kratix/input/object.yaml`, we will create using the `example-resource.yaml` file a Request.

```YAML
# Example of promise for postgresql
apiVersion: halkyon.io/v1alpha1
kind: database
metadata:
  name: fruits-db
  namespace: default # Changeme according to the user's namespace
spec:
  environment: dev
  auth:
    database: fruits_database
    username: healthy
    password: healthy
```

Copy the Promise request to the `kratix/input/object.yaml` file.
```shell
cp example-resource.yaml kratix/input/object.yaml
```

We can now generate the helm chart using either the `pipeline` script:
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