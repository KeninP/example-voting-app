# Voting App — CI/CD to Amazon EKS with GitHub Actions + OIDC

A multi-service voting app (based on [dockersamples/example-voting-app](https://github.com/dockersamples/example-voting-app)) that I extended with a full CI/CD pipeline: tests, container builds, keyless push to Amazon ECR, and automated rollouts to an EKS cluster.

## Architecture

```
 vote (Python/Flask) ──► redis ──► worker (.NET) ──► postgres ◄── result (Node.js)
```

| Service | Stack | Runs on |
|---|---|---|
| `vote` | Python / Flask | EKS (`vote-app` namespace) |
| `result` | Node.js | EKS |
| `worker` | .NET | EKS |
| `redis`, `postgres` | official images | EKS |

## Pipeline (`.github/workflows/vote-ci.yml`)

Triggered on pushes to `main` that touch `vote/**`:

1. **Check:** Python syntax check
2. **Test:** install dependencies, run `pytest`
3. **Build:** Docker image tagged with the **commit SHA** (traceable, easy rollback)
4. **Push:** GitHub Actions assumes an AWS IAM role via **OIDC** (no stored AWS keys) and pushes to **Amazon ECR**
5. **Deploy:** `aws eks update-kubeconfig` → `kubectl set image` → waits on `kubectl rollout status` and verifies the pods

`result-ci.yml` and `worker-ci.yml` build and push their images to Docker Hub using repository secrets.

## Infrastructure

- `eks-cluster.yaml`: eksctl cluster config (`vote-lab`, eu-north-1, managed node group)
- `k8s/`: Deployments, Services, ConfigMap and Ingress for every service
- `vote-app/`: **Helm chart** packaging the app (`vote-app-0.1.0.tgz`)

```bash
eksctl create cluster -f eks-cluster.yaml
kubectl create namespace vote-app
helm install vote-app ./vote-app -n vote-app
# or: kubectl apply -f k8s/ -n vote-app
```

## Run locally

```bash
docker compose up
# vote:   http://localhost:8080
# result: http://localhost:8081
```

## What I learned

- Setting up GitHub OIDC trust with AWS IAM, and debugging token claims until role assumption worked
- Splitting CI into gated stages so a broken test never reaches the cluster
- Moving from raw manifests to a Helm chart for repeatable installs
