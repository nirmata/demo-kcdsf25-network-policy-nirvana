# Network Policy Nirvana
**Automating Security & Delivering Self-Service with Kyverno**

This repository contains the demo for a session presented at KCD San Francisco 2025.

Here are the accompanying [slides](https://docs.google.com/presentation/d/1iSDuCzh7KqvPNylnD2hXdUPwYGCcRSYG/edit?slide=id.g37c5a709afd_0_49#slide=id.g37c5a709afd_0_49) for the session.

Follow the steps below to run the demo:

## Setup

### 1. Create a kind cluster

Create a kind cluster:

```sh
kind create cluster --config=config/kind/kind-config.yaml --name=demo-kcdsf25
```

### 2. Install Cilium

From: https://docs.cilium.io/en/latest/installation/kind/#install-cilium

NOTE: The Cilium Helm chart is already downloaded to the config/cilium folder.

<!-- ``` sh
curl -LO https://github.com/cilium/cilium/archive/main.tar.gz
tar xzf main.tar.gz
mv cilium-main/install/kubernetes/cilium charts/cilium
``` -->

Load images:

```sh
docker pull quay.io/cilium/cilium:v1.18.1
kind load docker-image quay.io/cilium/cilium:v1.18.1
```

Install Cilium CNI:

``` sh
helm install cilium ./charts/cilium \
   --namespace kube-system \
   --set image.pullPolicy=IfNotPresent \
   --set ipam.mode=kubernetes
```

Install Cilium CLI:

```sh
brew install cilium
```

Check status:

```sh
cilium status --wait
```

This should show the output similar to:

```sh
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    OK
 \__/¯¯\__/    Hubble Relay:       disabled
    \__/       ClusterMesh:        disabled

DaemonSet              cilium                   Desired: 4, Ready: 4/4, Available: 4/4
DaemonSet              cilium-envoy             Desired: 4, Ready: 4/4, Available: 4/4
Deployment             cilium-operator          Desired: 2, Ready: 2/2, Available: 2/2
Containers:            cilium                   Running: 4
                       cilium-envoy             Running: 4
                       cilium-operator          Running: 2
                       clustermesh-apiserver
                       hubble-relay
Cluster Pods:          3/3 managed by Cilium
Helm chart version:    1.19.0-dev
Image versions         cilium             quay.io/cilium/cilium-ci:latest: 4
                       cilium-envoy       quay.io/cilium/cilium-envoy:v1.35.2-1756960567-b574a2f016bb07a129be6c3e500a886e02b1d599@sha256:f7dabfd2b4baff57b2bb9467a033ae4f870ea9a4546677685f5dfc0f66e95253: 4
                       cilium-operator    quay.io/cilium/operator-generic-ci:latest: 2
```

### 3. Install & Configure Kyverno

From https://kyverno.io/docs/installation/methods/#install-kyverno-using-helm


```sh
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
helm install kyverno kyverno/kyverno -n kyverno --create-namespace
```

Configure Kyverno permissions to allow managing Cilium network policies:

```sh
kubectl apply -f config/kyverno/rbac/
```

### 4. Install Kyverno policies

```sh
kubectl apply -f config/kyverno/policies
```

Check policies:

```sh
kubectl get cpol
```

The output should match:
```sh
NAME                             ADMISSION   BACKGROUND   READY   AGE   MESSAGE
add-labels                       true        false        True    14s   Ready
check-images                     true        false        True    14s   Ready
check-network-policies           true        false        True    14s   Ready
generate-netpol-allowdns         true        false        True    14s   Ready
generate-netpol-allowns          true        false        True    14s   Ready
generate-netpol-allowworkspace   true        false        True    14s   Ready
generate-netpol-denyall          true        false        True    14s   Ready
require-labels                   true        false        True    14s   Ready
```


# Demo

## 1. Test Kyverno policies

### Require namespace labels

Try creating a namespace `test`:

```sh
kubectl create ns test
```

This will be blocked by Kyverno:

```sh
Error from server: admission webhook "validate.kyverno.svc-fail" denied the request:

resource Namespace//test was blocked due to the following policies

require-labels:
  check-ns: 'validation error: The namespace must have a labels workspace and tier.
    rule check-ns failed at path /metadata/labels/tier/
```

Use the provided sample with required labels:

```sh
kubectl apply -f config/kubernetes/ns.yaml
```

This should succeed. Check the namespace labels:

```sh
kubectl get ns test -o yaml |  grep -e "tier" -e "workspace"
```

### Mutate pod labels

Run a redis image and check labels. Kyverno will automatically apply required namespace labels to the pods:

```sh
kubectl -n test run redis --image redis --dry-run=server -o yaml | grep -e "tier" -e "workspace"
```

### Generate network policies

Check the Cilium network policies:

```sh
kubectl -n test get cnp
```

This should show the default deny-all policy:

```sh
NAME       AGE   VALID
deny-all   10m   True
```

Add a label `allow-dns-traffic`:

```sh
kubectl label ns test allow-dns-traffic=true
```

Check the Cilium network policies again. This should now show an additional `allow-dns` policy:

```sh
k -n test get cnp
NAME        AGE   VALID
allow-dns   5s    True
deny-all    13m   True
```

Try setting the label to `false`:

```sh
kubectl label ns test allow-dns-traffic=false --overwrite
```

Then recheck the policy, the `allow-dns` policy should be deleted.

### Restrict images

Run an unauthorized image in the `backend` tier:

```sh
k -n test run nginx --image nginx --dry-run=server
```

This will be blocked by Kyverno:

```sh
Error from server: admission webhook "validate.kyverno.svc-fail" denied the request:

resource Pod/test/nginx was blocked due to the following policies

check-images:
  backend-images: 'validation failure: image nginx is not allowed in the backend tier'
```

Run a valid image; this should be allowed:

```sh
kubectl -n test run redis --image=redis --dry-run=server
pod/redis created (server dry run)
```

### Restrict traffic to the backend tier

Try creating a Cilium network policy that allows internet traffic to the `backend` tier:

```sh
kubectl -n test apply -f config/cilium/cnp-ingress-world.yaml
```

This will be blocked by kyverno:

```sh
Error from server: error when creating "config/cilium/cnp-ingress-world.yaml": admission webhook "validate.kyverno.svc-fail" denied the request:

resource CiliumNetworkPolicy/test/backend-ingress was blocked due to the following policies

check-network-policies:
  check-ingress-backend: only ingress traffic from the frontend tier is allowed; fromEntities
    are not allowed
```

You can also try the policy that allows egress traffic from the `backend` tier:

```sh
kubectl -n test apply -f config/cilium/cnp-egress-world.yaml
```

This will also be blocked by Kyverno:

```sh
Error from server: error when creating "config/cilium/cnp-egress-world.yaml": admission webhook "validate.kyverno.svc-fail" denied the request:

resource CiliumNetworkPolicy/test/egress-world was blocked due to the following policies

check-network-policies:
  deny-egress-backend: egress traffic from backend tier is not allowed; toEntities
    are not allowed
```

### Cleanup Test Namespace

```sh
kubectl delete ns test
```

## 2. Run the Guestbook app in 2 different workspaces

The Guestbook app a 2-tier application with `guestbook` running in the `frontend` tier and `redis` running in the `backend` tier. The application is configured across 2 namespaces, one for each tier.

Create 2 instances of the Guestbook application in different workspaces `wks1` and `wks2`. A workspace is a network segmentation boundary used to isolate applications.

```sh
helm install guestbook1 charts/guestbook/ --set workspace=wks1
helm install guestbook2 charts/guestbook/ --set workspace=wks2
```

Check connectity for the applications:

```sh
kubectl port-forward service/guestbook 8080:80 -n wks1-guestbook
```

Navigate to http://localhost:8080 and make sure you an enter a value and it is shown as a guestbook entry after clicking the submit button.

Repeat the test for `guestbook2` in `wks2`.

## 3. Test network connectivity within and across workspaces

Attach `netshoot` as a debug container to the `guestbook` pod in the `frontend` tier.

Check the running pods:

```sh
kubectl -n wks1-guestbook get pods
```

Sample output:
```sh
NAME                         READY   STATUS    RESTARTS   AGE
guestbook-7dc7786f46-78hkk   1/1     Running   0          3m7s
```

Note: you will need to update the pod name to match your deployment:

```sh
kubectl -n wks1-guestbook debug -i -t --image nicolaka/netshoot --profile general {{POD_NAME}}
```

In the netshoot session check connectivity from the frontend pod in wks1 to the backend service in wks1:

```sh
nc -w 2 -v redis-leader.wks1-redis.svc.cluster.local 6379
```

This should show a message similar to:

```sh
Connection to redis-leader.wks1-redis.svc.cluster.local (10.11.75.0) 6379 port [tcp/redis] succeeded!
```

Now, try connecting across workspaces:

```sh
nc -w 2 -v redis-leader.wks2-redis.svc.cluster.local 6379
```

This should timeout and fail:

```sh
nc: connect to redis-leader.wks2-redis.svc.cluster.local (10.11.147.254) port 6379 (tcp) timed out: Operation in progress
```

Type `exit` to quite the debug container.

Cleanup:

Delete Guestbook instances:

```sh
helm uninstall guestbook1
helm uninstall guestbook2
```

Delete demo kind cluster:

```sh
kind delete cluster --name=demo-kcdsf25
```
