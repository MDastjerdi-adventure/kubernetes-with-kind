# Phase 1 — Core objects
## Big Picture
<img width="1840" height="940" alt="image" src="https://github.com/user-attachments/assets/928d2a59-fdea-47c3-b58d-75c8d80b1664" />

### Control plane — the brain
- API server — the only component anyone talks to. kubectl, the scheduler, kubelet, controllers — all of them read and write cluster state exclusively through the API server. It validates requests and is the sole gateway to etcd.
- etcd — a distributed key-value store holding the entire cluster state (every object's spec and status). Nothing else touches etcd directly — only the API server does. If etcd is lost, you've lost the cluster's memory.
- Scheduler — watches the API server for Pods with no node assigned, decides which node they should run on (based on resource requests, taints, affinity rules), and writes that decision back to the API server. It doesn't actually run anything.
- Controller manager — runs the reconciliation loops (Deployment controller, ReplicaSet controller, Node controller, etc). Each loop constantly compares "what you asked for" (desired state in etcd) vs "what's actually running" (observed state) and issues API calls to close the gap. This is the mechanism behind "delete a pod and a new one appears."
### Worker node — the muscle (this repeats on every node)
- kubelet — an agent that watches the API server for Pods assigned to its node, and makes sure those containers are actually running via the container runtime. It also reports node/pod health back up.
- kube-proxy — maintains network rules on the node so traffic sent to a Service's virtual IP gets routed to the right Pod, wherever it's actually running.
- Container runtime (containerd, in your kind setup) — actually pulls images and starts/stops containers. kubelet talks to it, not the other way around.>


## install kubectl (Linux example — adjust for your OS)
```bash
$ curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
$ chmod +x kubectl && sudo mv kubectl /usr/local/bin/
```

## install kind
```bash
$ curl -Lo kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
$ chmod +x kind && sudo mv kind /usr/local/bin/
```


# First of all: cluster and nodes
```yml
## kind-multi-node.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
```

```bash
$ kind create cluster --name kuber-cluster --config kind-multi-node.yaml
$ kubectl get nodes -o wide
```

## pods
```yml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
```

```bash
$ kubectl apply -f pod.yaml
$ kubectl get pods -o wide
$ kubectl describe pod nginx-pod
$ kubectl exec -it nginx-pod -- /bin/bash
$ kubectl port-forward pod/nginx-pod 8080:80
```

## deployments
```yml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

```bash
$ kubectl apply -f deployment.yaml
$ kubectl get deploy,rs,pods -o wide
$ kubectl delete pod nginx-pod
```
**Understand the chain: Deployment → creates ReplicaSet → ReplicaSet creates Pods.**

## Rolling update exercise
``` bash
$ kubectl set image deployment/nginx-deploy nginx=nginx:1.26
$ kubectl rollout status deployment/nginx-deploy
$ kubectl rollout history deployment/nginx-deploy
$ kubectl rollout undo deployment/nginx-deploy
```

## service
```yml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

```bash
$ kubectl apply -f service.yaml
$ kubectl port-forward svc/nginx-svc 8080:80

$ kubectl get endpoints
$ kubectl get endpoints nginx-svc
```
**Services route by label selector matching pod labels, and a mismatch is the #1 cause of "my service isn't working."**

## namespace
```bash 
$ kubectl create namespace dev
$ kubectl apply -f deployment.yaml -n dev
$ kubectl get pods -n dev
$ kubectl config set-context --current --namespace=dev   # avoid typing -n dev constantly
```
# Errors in this phase
- ImagePullBackOff
- ErrImagePull

# Phase 1 summry
1. Pod vs Deployment — a Pod is one running instance; a Deployment is a desired-state declaration that owns ReplicaSets which own Pods, giving you self-healing + rolling updates.
2. Scale 3→5 — you edit the Deployment's replicas field → controller manager's reconcile loop notices the ReplicaSet is short by 2 → creates 2 new Pod objects → scheduler assigns them to nodes → kubelet on those nodes starts them.
3. Service with 0 endpoints — the Service's selector labels don't match any live Pod's labels. kubectl get endpoints <svc> is the first command to run when "nothing's coming through."
4. port vs targetPort vs containerPort — port = what other things inside the cluster call the Service on; targetPort = the port on the Pod the Service forwards to; containerPort = the port the container actually listens on inside itself (should match targetPort).
