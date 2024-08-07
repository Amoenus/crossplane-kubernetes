# Google GKE Example

## Setup

> Make sure that Docker is up-and-running. We'll use it to create a KinD cluster.

> Watch https://youtu.be/WiFLtcBvGMU if you are not familiar with Devbox. Alternatively, you can skip Devbox and install all the tools listed in `devbox.json` yourself.

```sh
devbox shell

kind create cluster

helm upgrade --install crossplane crossplane \
    --repo https://charts.crossplane.io/stable \
    --set args='{"--enable-usages"}' \
    --namespace crossplane-system --create-namespace --wait

kubectl apply --filename config.yaml

kubectl apply \
    --filename providers/provider-kubernetes-incluster.yaml

kubectl apply \
    --filename providers/provider-helm-incluster.yaml

gum spin --spinner dot \
    --title "Waiting for Crossplane providers to be deployed..." \
    -- sleep 60

gum spin --spinner dot \
    --title "Waiting for Crossplane providers to be healthy..." \
    -- kubectl wait \
    --for=condition=healthy provider.pkg.crossplane.io --all \
    --timeout 5m
```

> Replace `[...]` with the AWS access key ID

```sh
export AWS_ACCESS_KEY_ID=[...]
```

> Replace `[...]` with the AWS secret access key

```sh
export AWS_SECRET_ACCESS_KEY=[...]

echo "[default]
aws_access_key_id = $AWS_ACCESS_KEY_ID
aws_secret_access_key = $AWS_SECRET_ACCESS_KEY
" >aws-creds.conf

kubectl --namespace crossplane-system \
    create secret generic aws-creds \
    --from-file creds=./aws-creds.conf

kubectl apply --filename providers/provider-config-aws.yaml

kubectl create namespace a-team
```

> Replace `[...]` with the GitHub username or organization

```sh
export GITHUB_USER=[...]

gh repo create $GITHUB_USER/crossplane-kubernetes-gitops --public

yq --inplace \
    ".spec.parameters.apps.argocd.repoURL = \"https://github.com/$GITHUB_USER/crossplane-kubernetes-gitops\"" \
    examples/aws-eks-full.yaml
```

## Simple Cluster

```sh
kubectl --namespace a-team apply --filename examples/aws-eks.yaml

crossplane beta trace clusterclaim a-team --namespace a-team
```

> Wait until all the resources are `Available`.

## Package

```sh
cat package/compositions.yaml

cat kcl/aws.k

echo "https://marketplace.upbound.io/configurations/devops-toolkit/dot-kubernetes"
```

> Open the URL from the output in a browser

## Simple Cluster (cont.)

```sh
aws eks update-kubeconfig --region us-east-1 --name a-team \
    --kubeconfig kubeconfig.yaml

kubectl --kubeconfig kubeconfig.yaml get namespaces

chmod +x examples/get-traefik-eks.sh

export INGRESS_IP=$(./examples/get-traefik-eks.sh)
```

## Complete Cluster

```sh
cat examples/aws-eks-full.yaml

yq --inplace \
    ".spec.parameters.apps.argocd.host = \"argocd.$INGRESS_IP.nip.io\"" \
    examples/aws-eks-full.yaml

kubectl --namespace a-team apply \
    --filename examples/aws-eks-full.yaml

crossplane beta trace clusterclaim a-team --namespace a-team

kubectl --kubeconfig kubeconfig.yaml get namespaces

echo "http://argocd.$INGRESS_IP.nip.io"
```

> Open the URL from the output in a browser

## Destroy

```sh
kubectl --namespace a-team delete \
    --filename examples/aws-eks.yaml

kubectl get managed
```

> Wait until all managed resources are removed

```sh
kind delete cluster

gh repo view $GITHUB_USER/crossplane-kubernetes-gitops --web
```

> Delete the repo
