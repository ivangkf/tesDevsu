# Demo Devops NodeJs

This is a simple application to be used in the technical test of DevOps.

## Getting Started

### Prerequisites

- Node.js 18.15.0

### Installation

Clone this repo.

```bash
git clone https://bitbucket.org/devsu/demo-devops-nodejs.git
```

Install dependencies.

```bash
npm i
```

### Database

The database is generated as a file in the main path when the project is first run, and its name is `dev.sqlite`.

Consider giving access permissions to the file for proper functioning.

## Usage

To run tests you can use this command.

```bash
npm run test
```

To run locally the project you can use this command.

```bash
npm run start
```

Open http://localhost:8000/api/users with your browser to see the result.

### Features

These services can perform,

#### Create User

To create a user, the endpoint **/api/users** must be consumed with the following parameters:

```bash
  Method: POST
```

```json
{
    "dni": "dni",
    "name": "name"
}
```

If the response is successful, the service will return an HTTP Status 200 and a message with the following structure:

```json
{
    "id": 1,
    "dni": "dni",
    "name": "name"
}
```

If the response is unsuccessful, we will receive status 400 and the following message:

```json
{
    "error": "error"
}
```

#### Get Users

To get all users, the endpoint **/api/users** must be consumed with the following parameters:

```bash
  Method: GET
```

If the response is successful, the service will return an HTTP Status 200 and a message with the following structure:

```json
[
    {
        "id": 1,
        "dni": "dni",
        "name": "name"
    }
]
```

#### Get User

To get an user, the endpoint **/api/users/<id>** must be consumed with the following parameters:

```bash
  Method: GET
```

If the response is successful, the service will return an HTTP Status 200 and a message with the following structure:

```json
{
    "id": 1,
    "dni": "dni",
    "name": "name"
}
```

If the user id does not exist, we will receive status 404 and the following message:

```json
{
    "error": "User not found: <id>"
}
```

If the response is unsuccessful, we will receive status 400 and the following message:

```json
{
    "errors": [
        "error"
    ]
}
```

## License

Copyright © 2023 Devsu. All rights reserved.

---

## DevOps Implementation - Added for this test

### Architecture and Diagram

Flow implemented for this test:

```
Developer (branch develop)
      |
      v
GitHub -> Push to develop
      |
      v
GitHub Actions (.github/workflows/ci.yml)
  1. Checkout code
  2. Setup Node.js 20
  3. npm ci (Code Build)
  4. ESLint (Static Code Analysis)
  5. npm test -- --coverage (Unit Tests + Coverage)
  6. Upload coverage/ as artifact
  7. Docker Build & Push to Docker Hub
  8. Trivy Vulnerability Scan
      |
      v
Docker Hub: ivangkf323gkf/devsu-demo-nodejs:latest
      |
      v
Kubernetes (Docker Desktop - local)
  Deployment (2 replicas) -> Service (NodePort 30080)
  ConfigMap + Secret -> HPA (2 to 5 replicas) -> Ingress (devsu.local)
```

Project structure:

```
.
├── Dockerfile
├── docker-compose.yml
├── .dockerignore
├── .env
├── index.js
├── users/
├── shared/
├── .github/workflows/ci.yml
└── k8s/
    ├── deployment.yaml
    ├── service.yaml
    ├── configmap.yaml
    ├── secret.yaml
    ├── hpa.yaml
    ├── ingress.yaml
    ├── ingress-tls.yaml
    └── networkpolicy.yaml
```

### Decisions Taken

1.  **Node.js 20 in CI, Node.js 18.15.0 in local:** The original README asks for 18.15.0. Local development keeps 18.15.0. CI uses Node 20 because it is the LTS version supported by `actions/setup-node` and it is compatible with the project.
2.  **Docker image:** One single image built from the root `Dockerfile`. It exposes port `8000`, uses `npm ci` for a clean install and `.dockerignore` to avoid copying `node_modules`, `.git` and `coverage/`.
3.  **Docker Compose for local:** `docker-compose.yml` is kept only for local development. Kubernetes does not use Compose, it uses the manifests in `k8s/`.
4.  **ConfigMap + Secret:** Non-sensitive configuration (`NODE_ENV`, `PORT`, `DATABASE_NAME`, `DB_FORCE_SYNC`) goes in a ConfigMap. Sensitive data (`DATABASE_USER`, `DATABASE_PASSWORD`) goes in a Secret.
5.  **DB sync:** In local, `sequelize.sync({ force: true })` was kept for easy testing. In Kubernetes, `DB_FORCE_SYNC=false` is set in the ConfigMap and the code was changed to `sequelize.sync({ force: process.env.DB_FORCE_SYNC === 'true' })` to avoid wiping the database on every pod restart.
6.  **2 replicas + HPA:** The Deployment starts with 2 replicas. The HPA can scale from 2 to 5 replicas when CPU average utilization is over 70%.

### CI/CD Pipeline

File: `.github/workflows/ci.yml`

It runs on every `push` and `pull_request` to `develop` and includes:

- **Code Build:** `npm ci`
- **Static Code Analysis:** `ESLint`
- **Unit Tests:** `npm test`
- **Code Coverage:** `npm test -- --coverage`, uploaded as artifact `coverage-report`
- **Docker Build & Push:** Image pushed to Docker Hub as `ivangkf323gkf/devsu-demo-nodejs:latest`
- **Vulnerability Scan:** `Trivy` scans the pushed image

Required GitHub Secrets:

- `DOCKERHUB_USERNAME`
- `DOCKERHUB_TOKEN`

### Kubernetes Deployment

Prerequisites:

- Docker Desktop with Kubernetes enabled: Kubernetes -> Create cluster
- `kubectl` installed

Deploy (without the TLS example file, it is only documentation for production):

```bash
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/secret.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/hpa.yaml
kubectl apply -f k8s/ingress.yaml
kubectl apply -f k8s/networkpolicy.yaml
```

Verify:

```bash
kubectl get pods
kubectl get svc
kubectl get hpa
kubectl get ingress
```

Expected result: 2 pods running.

Test locally:

The Service is defined as NodePort `8000:30080`, so the official URL is:

```bash
http://localhost:30080/api/users
```

In Docker Desktop on Windows, NodePort `30080` does not always open correctly on `localhost` because of WSL networking. For that reason, in local testing the reliable way is `kubectl port-forward`:

```bash
kubectl port-forward svc/devsu-service 8000:8000
```

Keep that terminal open and then test with:

```bash
http://localhost:8000/api/users
```

Create a user in Kubernetes:

```bash
curl -X POST http://localhost:8000/api/users -H "Content-Type: application/json" -d "{\"dni\":\"1234567890\",\"name\":\"Prueba K8s\"}"
```

### Production Considerations (Not enabled in local, documented for production)

1.  **DNS:** In production, `api.devsu.local` should point via an `A` record to the Ingress Controller LoadBalancer IP. In local, this is simulated with `127.0.0.1 devsu.local` in the hosts file.
2.  **TLS / HTTPS:** In production, use `cert-manager` + `Let's Encrypt` with an `Ingress` with `tls:` section. See `k8s/ingress-tls.yaml` as an example. It is not applied in local because `cert-manager` is not installed.
3.  **Secrets Management:** Do not commit `Secret` manifests with base64 values in production. Use `Sealed Secrets` or `External Secrets Operator` with AWS Secrets Manager / Vault.
4.  **Database:** Do not use SQLite in production. Migrate to PostgreSQL with a managed service (RDS / Cloud SQL).
5.  **NetworkPolicy:** Only Ingress should be able to reach the app Service. See `k8s/networkpolicy.yaml`.
6.  **Monitoring and Logs:** Use `Prometheus + Grafana` to monitor HPA, CPU, memory and pod restarts. In Docker Desktop local, HPA shows `<unknown>/70%` because metrics-server is not installed by default, but the HPA object is correctly created.
7.  **Image Tagging:** In production, do not use `latest`. Use immutable tags like `v1.0.0-<commit-sha>`.

### Limitations and Why

1.  **SQLite + 2 replicas:** Each pod has its own `dev.sqlite` file. A user created in Pod 1 may not be visible in Pod 2. This is a limitation of the original test application (file-based DB) and not a Kubernetes misconfiguration. In production, this is solved with PostgreSQL.
2.  **Automatic deploy from GitHub Actions to local Kubernetes:** Not implemented. GitHub Actions cannot reach a local Docker Desktop cluster. The pipeline builds and pushes the image. The deploy is done manually with `kubectl apply -f k8s/`. In a cloud cluster (EKS / GKE / AKS), the pipeline would include a deploy step using `KUBE_CONFIG` secret.
3.  **Ingress in local:** `devsu.local` requires a manual hosts entry and the NGINX Ingress Controller. In local Docker Desktop, `port-forward` is the reliable way to test.
4.  **NodePort in Windows:** `http://localhost:30080/api/users` does not always respond in Docker Desktop on Windows. The workaround documented and tested is `kubectl port-forward svc/devsu-service 8000:8000` and then `http://localhost:8000/api/users`.

### Best Practices and Code Quality

- `ESLint` runs in CI to enforce clean code.
- `npm test -- --coverage` runs in CI and the coverage report is saved as an artifact.
- `.dockerignore` prevents sending `node_modules` and secrets to Docker.
- `.env` is not committed with real production secrets.
- Resources `requests` / `limits` and `livenessProbe` / `readinessProbe` are defined in the Deployment.
