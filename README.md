## Instructions

## Kratix and idpbuilder

Use idpbuilder tool
```
idp create --color --recreate --name kratix --port 8443 --dev-password \
  -p kratix
```

## Deploy the postgresql promise
```
k delete -f https://raw.githubusercontent.com/syntasso/promise-postgresql/main/resource-request.yaml
k delete -f https://raw.githubusercontent.com/syntasso/promise-postgresql/main/promise.yaml

k apply -f https://raw.githubusercontent.com/syntasso/promise-postgresql/main/promise.yaml
k apply -f https://raw.githubusercontent.com/syntasso/promise-postgresql/main/resource-request.yaml

k get pods -n kratix-platform-system
NAME                                                 READY   STATUS      RESTARTS   AGE
kratix-platform-controller-manager-5db547858-hdt56   1/1     Running     0          8m43s
kratix-postgresql-promise-configure-c2b4b-zv8lh      0/1     Completed   0          17s
                              1/1     Running     0          3m26s

k exec -it acid-example-postgresql-0 -- sh -c "
PGPASSWORD=$(kubectl get secret postgres.acid-example-postgresql.credentials.postgresql.acid.zalan.do -o 'jsonpath={.data.password}' | base64 -d) \
PGUSER=$(kubectl get secret postgres.acid-example-postgresql.credentials.postgresql.acid.zalan.do -o 'jsonpath={.data.username}' | base64 -d) \
psql bestdb"
```

## Kratix

See: https://docs.kratix.io/main/quick-start

### All-in-one
```
kind delete cluster --name platform
kind create cluster --name platform
k apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.15.0/cert-manager.yaml
k apply -f https://github.com/syntasso/kratix/releases/latest/download/install-all-in-one.yaml
k apply -f https://github.com/syntasso/kratix/releases/latest/download/config-all-in-one.yaml
```

### Bash script

```
GITOPS_PROVIDER=argo RECREATE=true ./scripts/quick-start.sh -s -g
Looking for KinD... ✓
Looking for kubectl... ✓
Looking for docker... ✓
Looking for distribution/kratix.yaml...distribution/kratix.yaml not found; downloading latest version...
✓
Deleting pre-existing clusters... ✓
using podman due to KIND_EXPERIMENTAL_PROVIDER
enabling experimental podman provider
No kind clusters found.
Creating platform destination...
✓
Finished creating platform destination ✓
Setting up platform destination... ✓
Setting up worker destination... ✓
Waiting for local repository to be running... ✓
Waiting for system to reconcile... ✓
Kratix installation is complete!
```

## Temp

```shell
set -x GITEA_API_URL "https://gitea.cnoe.localtest.me:8443/api"
set -x GITEA_USERNAME "giteaAdmin"
set -x GITEA_PASSWORD "developer"

TOKEN=$(curl -s -k -H "Content-Type: application/json" -d '{"name":"init","scopes": ["write:user", "write:admin", "write:repository"]}' -u $GITEA_USERNAME:$GITEA_PASSWORD $GITEA_API_URL/v1/users/$GITEA_USERNAME/tokens | jq -r .sha1)
echo $TOKEN

set -x ORG_NAME "kratix"
curl -kv -X POST \
  "$GITEA_API_URL/v1/orgs" \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -u "$GITEA_USERNAME:$GITEA_PASSWORD" \
  -d '{"username": "'"$ORG_NAME"'"}'

## Create a repository in an org
set -x REPO_NAME "dummy"
set -x REPO_DESCRIPTION "dummy dummy"
set -x ORG_NAME "kratix"

curl -kv \
  "$GITEA_API_URL/v1/orgs/$ORG_NAME/repos" \
  -H 'accept: application/json' \
  -u "$GITEA_USERNAME:$GITEA_PASSWORD" \
  -H 'Content-Type: application/json' \
  -d '{
     "auto_init": true,
     "default_branch": "main",
     "description": "'"$REPO_DESCRIPTION"'",
     "name": "'"$REPO_NAME"'",
     "readme": "Default",
     "private": false
}'

export ORG_NAME="kratix"
export REPO_NAME="dummy"
curl -kv \
  "$GITEA_API_URL/v1/repos/$ORG_NAME/$REPO_NAME" \
  -H 'accept: application/json'
```
