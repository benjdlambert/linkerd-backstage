# [Backstage](https://backstage.io)

This is your newly scaffolded Backstage App, Good Luck!

To start the app, run:

```sh
yarn install
```

### Setup

Setup the Backstage Kubernetes Plugin:

- There's a lot of ways to do this, we're going to cheat a little bit and use kind, but ideally you'd set it up with your K8s cluster provider.

```sh
brew install kind

kind create cluster

# enable metrics-server on kind
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.5.0/components.yaml
kubectl patch -n kube-system deployment metrics-server --type=json \
  -p '[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'

kubectl create sa backstage-service-account

kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: backstage
  namespace: default
  annotations:
    kubernetes.io/service-account.name: backstage-service-account
type: kubernetes.io/service-account-token
EOF

kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: backstage-read-only
rules:
  - apiGroups:
      - '*'
    resources:
      - pods
      - configmaps
      - services
      - deployments
      - replicasets
      - horizontalpodautoscalers
      - ingresses
      - statefulsets
      - limitranges
      - resourcequotas
      - daemonsets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - batch
    resources:
      - jobs
      - cronjobs
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - metrics.k8s.io
    resources:
      - pods
    verbs:
      - get
      - list
EOF

kubectl create clusterrolebinding backstage-read-only \
  --clusterrole=backstage-read-only \
  --serviceaccount=default:backstage-service-account

export KIND_SERVICE_ACCOUNT_TOKEN=$(kubectl -n default get secret backstage -o go-template='{{.data.token | base64decode}}')
```

- Install l5d into the cluster

```sh
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install-edge | sh

export PATH=$HOME/.linkerd2/bin:$PATH

linkerd check --pre

linkerd install --crds | kubectl apply -f -

linkerd install | kubectl apply -f -

linkerd viz install | kubectl apply -f -

linkerd check
```

- Run the emojivoto demo

```sh

curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/emojivoto.yml \
  | kubectl apply -f -

```

- Inject l5d into the emojivoto namespace

```sh
kubectl get -n emojivoto deploy -o yaml \
  | linkerd inject - \
  | kubectl apply -f -
```


- Start backstage

```sh
yarn dev
```
