# Task Manager Microservices on EKS (Day 13)

A 7-container task manager app, deployed on Amazon EKS (Auto Mode) with plain Kubernetes manifests and Kustomize, prepared for GitOps with ArgoCD.

## Architecture

```
Browser -> ALB (Ingress) -> frontend (nginx, :80)
                               | /api/
                               v
                          api-gateway (:4000)
                     /        |         \          \
              auth-service  user-service  task-service  notification-service
                (:4002)       (:4003)       (:4004)         (:4005)
                     \_________|____________/
                               v
                         mongo (:27017)
```

- Only the frontend is exposed publicly. Its nginx proxies `/api/` to `api-gateway`.
- Every other service is reachable only inside the cluster, through its Service name.
- The Service names are the same as in `docker-compose.yml`, so the internal URLs did not need to change.

## Repo layout

```
k8s/
  base/                     # environment-independent manifests
    namespace.yaml          # namespace "taskmanager"
    configmap.yaml          # shared non-secret config (Mongo URI, service URLs)
    mongo.yaml              # Mongo Deployment + Service (emptyDir storage)
    <service>.yaml          # Deployment + Service for each of 5 node services
    frontend.yaml           # nginx frontend Deployment + Service
    ingress.yaml            # public entry point (ALB)
    kustomization.yaml      # lists everything in base
  overlays/
    dev/kustomization.yaml  # maps image names to ECR repos + tags
  cluster-setup/
    ingressclass-alb.yaml   # one-time, applied by hand (EKS Auto Mode ALB)
argocd/
  application.yaml          # tells ArgoCD to watch k8s/overlays/dev
generate-k8s.sh             # script that generated the files above
```

## What each piece does

**Deployment.** Keeps the desired number of pods running and handles rolling updates and rollbacks. Each service has one, with `replicas: 1`.

**Service.** Gives pods a stable internal DNS name and load-balances across them. It finds pods through the label selector (`app: <name>`). If the selector doesn't match the pod labels, the Service has no endpoints.

**ConfigMap + `envFrom`.** Shared settings live in one place (`app-config`) and are injected into the containers as environment variables. Per-service values, such as `PORT`, are set directly in each Deployment.

**Secret (not in Git).** `JWT_SECRET` is read from a Secret named `app-secrets`, created by hand with `kubectl`. Committing it to Git would leak it permanently through the history.

**Probes.**
- Readiness: a pod only receives traffic once `/health` answers.
- Liveness: Kubernetes restarts a pod whose `/health` stops answering.

**Resource requests and limits.** A request is what the scheduler reserves for the pod when choosing a node. A limit is the cap, and a pod that exceeds its memory limit is OOMKilled. Without requests, the scheduler has nothing to place pods by.

**Ingress + IngressClass.** The Ingress defines the routing rule (`/` goes to `frontend`). The IngressClass tells EKS Auto Mode to build an AWS Application Load Balancer for it.

**Kustomize base + overlay.** The base holds the shared manifests, using short image names such as `api-gateway`. The overlay rewrites them to full ECR names and tags. In GitOps, CI changes only the tag in the overlay, and ArgoCD deploys the change.

**ArgoCD Application.** Points ArgoCD at the overlay path. With `automated`, `prune` and `selfHeal` on, Git becomes the source of truth, and manual changes in the cluster get reverted.

## Deploy steps

```bash
# 1. Create ECR repos (one per service)
for n in api-gateway auth-service user-service task-service notification-service frontend; do
  aws ecr create-repository --repository-name taskmanager/$n --region ap-south-1
done

# 2. Log in and push images
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com
ECR=<ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/taskmanager
for n in api-gateway auth-service user-service task-service notification-service frontend; do
  docker tag $n:v1 $ECR/$n:v1
  docker push $ECR/$n:v1
done

# 3. Namespace, secret, IngressClass, app
kubectl create namespace taskmanager
kubectl create secret generic app-secrets -n taskmanager --from-literal=JWT_SECRET=$(openssl rand -hex 32)
kubectl apply -f k8s/cluster-setup/ingressclass-alb.yaml
kubectl apply -k k8s/overlays/dev

# 4. Verify
kubectl get pods -n taskmanager
kubectl get ingress -n taskmanager
```

## Result

- All 7 pods `1/1 Running`, 0 restarts.
- App reachable on the public ALB address (Sign In page served).
- Register/login/create-task test: <fill in: worked / what failed>

## Cluster notes (EKS Auto Mode)

- The built-in `system` node is Graviton (ARM) and tainted `CriticalAddonsOnly`, so app pods cannot run on it.
- The `general-purpose` pool is x86 (`amd64`) only. Pods go Pending, then Auto Mode launches an x86 node automatically (about 1-2 minutes). My `amd64` images run there without an ARM rebuild.
- The extra node and the ALB cost money while they run.

## Troubleshooting drills

### Pending (done)
Set a 16 CPU request on `task-service`. The pod stayed Pending. `kubectl describe pod` showed:

1. `FailedScheduling`: `1 Insufficient cpu, 1 node(s) had untolerated taint(s)`. The scheduler found no node with room.
2. `Nominated`: Auto Mode planned a new node.
3. `InsufficientCapacityError ... VcpuLimitExceeded`: the AWS account's vCPU quota (8) blocked launching the node.

Lesson: Pending can be a scheduling problem (resources, taints, affinity) or a cloud problem (quota, capacity). `describe` events tell you which. The old pod kept serving traffic, because a rolling update does not remove it until the new pod is Ready. Fixed with `kubectl rollout undo`.

### ImagePullBackOff
<fill in: set a bad image tag, read the "Failed to pull image" event, rolled back>

## Cost and teardown

```bash
kubectl delete -k k8s/overlays/dev      # removes app, ALB, and (after a few minutes) the extra node
# delete the EKS cluster itself at the end of Day 14 to stop control-plane charges
```

## Next steps

- Push this repo to GitHub, set the real repo URL in `argocd/application.yaml`.
- Install ArgoCD and apply `argocd/application.yaml`.
- Day 14: HPA, rolling update, rollback, then delete the cluster.

## Lessons learned

- Service names and ports must match what the apps and nginx expect, or you get a Service with no endpoints or a 502 from nginx.
- Secrets stay out of Git. Create them by hand, and later use a secret manager.
- Check the node CPU architecture before pushing images.
- `kubectl describe` and its Events section are the first thing to read when a pod misbehaves.
