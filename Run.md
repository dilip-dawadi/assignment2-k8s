# Assignment 2 — Full Run Script

---

## Pre-requisites (already installed on EC2)

- `kind`
- `kubectl`
- `aws cli`

---

## Step 1 — Configure AWS credentials (lab environment only)

```bash
aws configure
# Enter: Access Key ID, Secret Access Key, Session Token, region = us-east-1
```

Verify ECR access works:

```bash
aws ecr get-login-password --region us-east-1 | head -c 20 && echo "...token OK"
```

---

## Step 2 — Create the kind cluster

```bash
cd ~/assignment2-k8s/manifests
kind create cluster --name assignment2 --config kind-config.yaml
```

Verify the cluster is healthy (single-node):

```bash
kubectl cluster-info --context kind-assignment2
kubectl get nodes
kubectl get pods -n kube-system
```

> **Report Q1:** The K8s API server IP is shown in `kubectl cluster-info` output.
> **Report:** `kubectl get nodes` will show exactly 1 node — confirming single-node cluster.

---

## Step 3 — Create namespaces

```bash
kubectl apply -f 01-namespaces/
kubectl get namespaces
```

---

## Step 4 — Create ECR pull secrets in both namespaces

```bash
ECR_PASSWORD=$(aws ecr get-login-password --region us-east-1)

kubectl create secret docker-registry ecr-secret \
  --docker-server=677868816391.dkr.ecr.us-east-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password="$ECR_PASSWORD" \
  -n web-ns

kubectl create secret docker-registry ecr-secret \
  --docker-server=677868816391.dkr.ecr.us-east-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password="$ECR_PASSWORD" \
  -n mysql-ns

# Verify secrets exist
kubectl get secret ecr-secret -n web-ns
kubectl get secret ecr-secret -n mysql-ns
```

---

## Step 5 — Deploy Services FIRST (before pods)

> **Important:** The web app connects to `mysql-svc.mysql-ns.svc.cluster.local` at startup.
> If the Service doesn't exist yet, DNS won't resolve and the web pod will crash immediately.
> Always create Services before Pods.

```bash
kubectl apply -f 05-services/
kubectl get services -A
```

---

## Step 6 — Deploy Pods

```bash
kubectl apply -f 02-pods/
kubectl get pods -A -w
# Wait until BOTH mysql-pod and web-pod show Running, then Ctrl+C
```

Test pods using port-forward (required by assignment appendix):

```bash
# Forward web pod port in background
kubectl port-forward pod/web-pod 8080:8080 -n web-ns &

# Hit the app
curl localhost:8080

# Check logs to confirm the request was received in the log
kubectl logs web-pod -n web-ns

# Kill the port-forward background job
kill %1
```

> **Report Q2a:** Yes, both apps can listen on the same port (e.g. 8080) inside their
> containers because each pod has its own isolated network namespace — there is no conflict.

---

## Step 7 — Deploy ReplicaSets

```bash
kubectl apply -f 03-replicasets/
kubectl get pods -A -w
# Wait until all RS pods are Running, then Ctrl+C

kubectl get replicasets -A
```

> **Report Q3:** The pod created in Step 6 (mysql-pod / web-pod) shares the same labels
> (`app: mysql` / `app: employees`) as the RS selector. Kubernetes will adopt it — the pod
> becomes one of the RS-managed replicas (RS will only create 2 new pods to reach 3 total).

---

## Step 8 — Deploy Deployments

```bash
kubectl apply -f 04-deployments/
kubectl get pods -A -w
# NOTE: The standalone RS pods will show Terminating — this is expected.
# The Deployment creates its own RS with the same selector and takes ownership.
# Wait until Deployment pods are Running, then Ctrl+C

kubectl get replicasets -A
kubectl get deployments -A
```

> **Report Q4:** The RS created in Step 7 is adopted by the Deployment — it becomes the
> active RS under the Deployment's control. K8s matches the RS selector to the Deployment
> selector and sets an ownerReference on the RS.

---

## Step 9 — Verify the full stack

```bash
kubectl get pods -A
kubectl get services -A
kubectl get endpoints mysql-svc -n mysql-ns
curl localhost:30000
```

> For browser access: open `http://<EC2-public-IP>:30000` in browser.
> Make sure port 30000 is open in the EC2 Security Group inbound rules.
> **Report Q7:** Web uses NodePort so it is externally reachable from the internet/browser.
> MySQL uses ClusterIP because it should only be reachable from within the cluster —
> exposing a database externally would be a security risk.

---

## Step 10 — Update image version (rolling update)

Build and push a new image tag from your assignment1 app, then:

```bash
# Update the deployment to use the new image version
kubectl set image deployment/web-deploy \
  web=677868816391.dkr.ecr.us-east-1.amazonaws.com/clo835-webapp:v2 \
  -n web-ns

# Watch the rolling update happen
kubectl rollout status deployment/web-deploy -n web-ns

# Confirm new version is running
kubectl get pods -n web-ns
kubectl describe deployment web-deploy -n web-ns | grep Image

curl localhost:30000
```

---

## Tear Down

```bash
kind delete cluster --name assignment2
```
