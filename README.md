# TechPathway Platform: DevOps Capstone

Production deployment of the TechPathway Platform on AWS: three Flask applications containerized with Docker, published to Amazon ECR, deployed to Amazon EKS, with autoscaling and monitoring (Prometheus and Grafana).

**Author:** Olawale Adebayo  
**AWS region used:** us-east-1  
**Cluster name:** `techpathway-cluster`

---

## 1. The platform

| App (folder) | Image / Service name | Port | Health check | Role |
|---|---|---|---|---|
| Weekly Call (`weekly-call/`) | `main-app` | 5111 | `/health` | Storefront and admin dashboard |
| Action Messages (`action-messages/`) | `action-messages` | 5001 | `/health` | Order-confirmation emails and tracking numbers |
| TechPathway Warehouse (`techpathway-warehouse/`) | `techpathway-warehouse` | 5002 | `/health` | Inventory and order fulfillment board |

`main-app` calls the other two over HTTP using their Kubernetes Service names. See `ARCHITECTURE.md` for the developers' service contract.

## 2. Repository structure

```
.
├── action-messages/          app code, Dockerfile, requirements.txt
├── techpathway-warehouse/    app code, Dockerfile, requirements.txt
├── weekly-call/              app code, Dockerfile, requirements.txt
├── k8s/                      Kubernetes manifests
│   ├── 00-namespace.yaml
│   ├── 10-configmaps.yaml
│   ├── 20-services.yaml
│   ├── 30-deploy-action-messages.yaml
│   ├── 31-deploy-techpathway-warehouse.yaml
│   ├── 32-deploy-main-app.yaml
│   ├── 40-hpa.yaml
│   └── troubleshooting/      manifests used for the TP-008 scenarios
├── ARCHITECTURE.md           developer-provided service contract
└── README.md
```

## 3. Prerequisites

AWS CLI v2, Docker, kubectl, eksctl, Helm, Git, Python 3.11+, and an AWS IAM user able to manage ECR, EKS, EC2, IAM and CloudFormation.

```bash
export AWS_REGION=<region>
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export ECR_REGISTRY=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
```

## 4. Containerization (Docker)

Each app has its own `Dockerfile` (Python 3.11-slim base, dependencies installed before code to use layer caching, gunicorn as the production server, a `HEALTHCHECK` on `/health`, and a `.dockerignore` that keeps `.env` and local databases out of the image).

```bash
docker build -t action-messages ./action-messages
docker build -t techpathway-warehouse ./techpathway-warehouse
docker build -t main-app ./weekly-call

docker run -d --name action-messages -p 5001:5001 action-messages
docker run -d --name techpathway-warehouse -p 5002:5002 techpathway-warehouse
docker run -d --name main-app -p 5111:5111 main-app
```

Clean up: `docker stop action-messages techpathway-warehouse main-app && docker rm action-messages techpathway-warehouse main-app`

## 5. Amazon ECR

```bash
for r in action-messages techpathway-warehouse main-app; do
  aws ecr create-repository --repository-name $r --region $AWS_REGION \
    --image-scanning-configuration scanOnPush=true
done

aws ecr get-login-password --region $AWS_REGION \
  | docker login --username AWS --password-stdin $ECR_REGISTRY

for r in action-messages techpathway-warehouse main-app; do
  docker tag $r:latest $ECR_REGISTRY/$r:v1.0
  docker tag $r:latest $ECR_REGISTRY/$r:latest
  docker push $ECR_REGISTRY/$r:v1.0
  docker push $ECR_REGISTRY/$r:latest
done
```

The manifests deploy the pinned `v1.0` tag, not `latest`, so each release is predictable.

## 6. Amazon EKS deployment

```bash
eksctl create cluster --name techpathway-cluster --region $AWS_REGION \
  --nodegroup-name techpathway-nodes --node-type t3.medium \
  --nodes 2 --nodes-min 2 --nodes-max 4 --managed

aws eks update-kubeconfig --name techpathway-cluster --region $AWS_REGION
kubectl get nodes
```

Deploy in this order (each step is required by the next):

```bash
kubectl apply -f k8s/00-namespace.yaml

# Secrets are created from the command line and never stored in Git
kubectl create secret generic main-app-secret -n techpathway \
  --from-literal=SECRET_KEY="$(openssl rand -hex 32)"
kubectl create secret generic action-messages-secret -n techpathway \
  --from-literal=SMTP_USER="" --from-literal=SMTP_PASSWORD=""

kubectl apply -f k8s/10-configmaps.yaml
kubectl apply -f k8s/20-services.yaml     # creates one AWS load balancer per app

# Give main-app the public hostnames of the other two (wait until EXTERNAL-IP is set)
NOTIF_HOST=$(kubectl get svc action-messages -n techpathway -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
WH_HOST=$(kubectl get svc techpathway-warehouse -n techpathway -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
kubectl patch configmap main-app-config -n techpathway --type merge \
  -p "{\"data\":{\"NOTIFICATION_PUBLIC_URL\":\"http://$NOTIF_HOST\",\"WAREHOUSE_PUBLIC_URL\":\"http://$WH_HOST\"}}"

# Deployments (envsubst fills in ${ECR_REGISTRY})
for f in k8s/30-*.yaml k8s/31-*.yaml k8s/32-*.yaml; do envsubst < $f | kubectl apply -f -; done

kubectl get deployments,pods,services -n techpathway
```

### What each manifest configures

| Requirement | Where |
|---|---|
| Namespace | `00-namespace.yaml` (`techpathway`) |
| ConfigMaps | `10-configmaps.yaml` (non-secret settings) |
| Secrets | created with `kubectl create secret` (see above) |
| CPU and memory requests/limits | each Deployment: requests 100m CPU, limits 500m CPU, memory 128 to 512Mi |
| Readiness and liveness probes | each Deployment: `GET /health` on the app's port |
| Services | `20-services.yaml` (type `LoadBalancer`, port 80 to the container port) |

## 7. Kubernetes operations

```bash
kubectl scale deployment --all --replicas=5 -n techpathway
kubectl scale deployment --all --replicas=2 -n techpathway
kubectl delete pod <pod-name> -n techpathway      # Kubernetes recreates it
kubectl logs <pod-name> -n techpathway
kubectl describe deployment <name> -n techpathway
kubectl exec -it <pod-name> -n techpathway -- /bin/bash
```

## 8. Autoscaling (HPA)

Requires the Metrics Server (not included in EKS):

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl apply -f k8s/40-hpa.yaml
kubectl get hpa -n techpathway -w
```

Each HPA targets 50% average CPU (of the pod's request), with 2 to 5 replicas. Load was generated with a `busybox` pod looping `wget` against each app's `/health` endpoint, then deleted to watch the pods scale back in.

## 9. Monitoring (Helm, Prometheus, Grafana)

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana-community https://grafana-community.github.io/helm-charts
helm repo update
kubectl create namespace monitoring

helm install prometheus prometheus-community/prometheus -n monitoring \
  --set server.persistentVolume.enabled=false \
  --set alertmanager.enabled=false \
  --set prometheus-pushgateway.enabled=false

helm install grafana grafana-community/grafana -n monitoring

kubectl get secret grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 --decode; echo
kubectl port-forward svc/grafana 3000:80 -n monitoring     # then open http://localhost:3000
```

Grafana data source URL: `http://prometheus-server.monitoring.svc.cluster.local`

Persistence is disabled (no EBS CSI driver was installed), so metric history is lost if the Prometheus pod restarts.

Example queries used:

| Panel | PromQL |
|---|---|
| Nodes | `count(kube_node_info)` |
| Pod CPU | `sum(rate(container_cpu_usage_seconds_total{namespace="techpathway", container!="", container!="POD"}[5m])) by (pod)` |
| Pod memory | `sum(container_memory_working_set_bytes{namespace="techpathway", container!="", container!="POD"}) by (pod)` |

## 10. Troubleshooting scenarios

| Scenario | How I triggered it | Root cause | Fix |
|---|---|---|---|
| ImagePullBackOff | Deployed `ts-imagepull` in the `troubleshooting` namespace with image tag `v9.9`, which was never pushed to ECR | The tag does not exist in the ECR repository (only `v1.0` and `latest` do), so the node cannot pull the image; the `kubectl describe pod` Events show the pull failing with "not found" | Changed the image to the existing tag `v1.0` and re-applied; the pod became `1/1 Running` |
| CrashLoopBackOff | Deployed a Deployment whose container start command was broken, so the process exits right after starting | The container keeps starting and crashing; Kubernetes restarts it with a growing back-off delay, so RESTARTS climbs and the status shows `CrashLoopBackOff`; `kubectl logs` (and `--previous`) shows why it exits | Corrected the start command and re-applied; the pod stayed `1/1 Running` |
| Pending Pod | Deployed a pod requesting far more CPU or memory than any node has | The scheduler cannot place it on any node; the `kubectl describe pod` Events show `FailedScheduling` with "Insufficient cpu" (or memory), so the pod stays `Pending` and no container starts | Lowered the resource requests to realistic values (100m CPU, 128Mi memory) and re-applied; the pod was scheduled and became `Running` |
| Incorrect container port | Deployed a pod with a wrong port (not 5111) in the container port, probes or Service targetPort | The app listens on 5111, so probes or Service traffic reach a port with nothing listening ("connection refused"); the pod is not Ready or the Service cannot reach it | Set the port to `5111` in the Deployment and Service and re-applied; the pod became `1/1 Running` and the app answered |
| Failed readiness probe | Deployed a pod with the readiness probe path set to a page that does not exist (for example `/wrong-ready`) | The probe gets HTTP 404, so Kubernetes marks the pod not ready: it shows `Running` but `0/1` READY, gets no Service traffic, and is not restarted; the Events show "Readiness probe failed" | Changed the readiness probe path to `/health` and re-applied; the pod became `1/1` Ready |
| Failed liveness probe | Deployed `ts-live` with the liveness probe path set to `/wrong-health` (checked every 5s, restart after 2 failures) | The app has no `/wrong-health` route, so the probe gets HTTP 404 and fails; Kubernetes keeps killing and restarting the container, so the RESTARTS count climbs and the Events show "Liveness probe failed" | Changed the liveness probe path to `/health` and re-applied; the restarts stopped and the pod stayed `1/1 Running` |

Diagnosis workflow used each time: `kubectl get pods`, then `kubectl describe pod` (read the Events), then `kubectl logs`.

## 11. Cleanup

Order matters: delete the Services first so AWS removes their load balancers before the cluster goes.

```bash
kubectl delete -f k8s/40-hpa.yaml
kubectl delete deployment --all -n techpathway
kubectl delete service --all -n techpathway
kubectl delete namespace techpathway

helm uninstall grafana -n monitoring
helm uninstall prometheus -n monitoring
kubectl delete namespace monitoring

eksctl delete cluster --name techpathway-cluster --region $AWS_REGION

for r in action-messages techpathway-warehouse main-app; do
  aws ecr delete-repository --repository-name $r --region $AWS_REGION --force
done
```

## 12. Known limitations and next steps

- `action-messages` and `techpathway-warehouse` store data in a local SQLite file per pod, so replicas do not share data. A production version needs a shared database (for example Amazon RDS) or persistent volumes.
- Each app is exposed through its own public HTTP load balancer. Production would use one Ingress/ALB with HTTPS.
- Images are deployed by pinned tag but pushed manually. A CI/CD pipeline (for example GitHub Actions building and pushing to ECR) would automate this.
- Prometheus and Grafana run without persistent storage.

## 13. Evidence

Screenshots for TP-001 to TP-009 are submitted separately with the assignment.
