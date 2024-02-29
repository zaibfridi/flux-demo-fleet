# flux-demo-fleet

goto: https://github.com/settings/tokens
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>

KIND_CLUSTER_NAME=helm-flux-demo
kind create cluster --config kind-config.yaml

kubectl create ns production
kubectl create ns staging

kind load docker-image mariadb:0.1.1 --name $KIND_CLUSTER_NAME
kind load docker-image zaibfridi/backend:0.1.1 --name $KIND_CLUSTER_NAME
kind load docker-image mariadb:0.1.2 --name $KIND_CLUSTER_NAME
kind load docker-image zaibfridi/backend:0.1.2 --name $KIND_CLUSTER_NAME


flux bootstrap github \
    --token-auth \
    --owner=$GITHUB_USER \
    --repository flux-fleet \
    --branch main \
    --path apps \
    --personal

flux check --pre
flux check

flux create secret git flux-demo-fleet \
    --url https://github.com/zaibfridi/flux-demo-fleet \
    --username=$GITHUB_USER \
    --password=$GITHUB_TOKEN \
    --namespace default

flux create secret git flux-demo-staging \
    --url https://github.com/zaibfridi/flux-demo-staging \
    --username=$GITHUB_USER \
    --password=$GITHUB_TOKEN \
    --namespace flux-system

flux create secret git flux-demo-production \
    --url https://github.com/zaibfridi/flux-demo-production \
    --username=$GITHUB_USER \
    --password=$GITHUB_TOKEN \
    --namespace flux-system

flux create secret git flux-demo-microservices \
    --url https://github.com/zaibfridi/flux-demo-microservices \
    --username=$GITHUB_USER \
    --password=$GITHUB_TOKEN \
    --namespace flux-system

cd flux-fleet
flux create source git staging \
    --url https://github.com/zaibfridi/flux-demo-staging \
    --branch main \
    --interval 30s \
    --export \
    | tee apps/staging.yaml

flux create kustomization staging \
    --source staging \
    --path "./" \
    --prune true \
    --validation client \
    --interval 10m \
    --export \
    | tee -a apps/staging.yaml

flux create source git production \
    --url https://github.com/zaibfridi/flux-demo-production \
    --branch main \
    --interval 30s \
    --export \
    | tee apps/production.yaml

flux create kustomization production \
    --source production \
    --path "./" \
    --prune true \
    --validation client \
    --interval 10m \
    --export \
    | tee -a apps/production.yaml

flux create source git \
    flux-demo-microservices \
    --url https://github.com/zaibfridi/flux-demo-microservices \
    --branch main \
    --interval 30s \
    --export \
    | tee apps/flux-demo-microservices.yaml

cd flux-demo-staging
echo "image:
    tag: 0.1.1" \
    | tee values.yaml


flux create helmrelease \
    flux-demo-microservices-staging \
    --source GitRepository/flux-demo-microservices \
    --values values.yaml \
    --chart "backend/helm" \
    --target-namespace staging \
    --interval 30s \
    --export \
    | tee apps/flux-demo-microservices.yaml

cleanup
-------
helm delete backend
helm delete database
kind delete cluster --name=$KIND_CLUSTER_NAME



flux useful commands
---------------------
flux get sources git
flux get helmreleases
flux get kustomizations
kubectl get GitRepository -n flux-system
k describe gitrepository flux-demo-microservices
k describe kustomization flux-demo-microservices

