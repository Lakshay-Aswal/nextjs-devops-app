This project demonstrates containerizing a Next.js app with **Docker**, automating image builds using **GitHub Actions & GHCR**, and deploying the app to **Kubernetes (Minikube)**.

## Prerequisites — Install These First

Make sure the following are installed and working:

| Tool               | Check Command              | 
| ------------------ | -------------------------- | 
| **Node.js (v18+)** | `node -v`                  | 
| **npm**            | `npm -v`                   | 
| **Docker Desktop** | `docker --version`         | 
| **Git**            | `git --version`            | 
| **Minikube**       | `minikube version`         | 
| **kubectl**        | `kubectl version --client` |

---

## Step 1: Create the Next.js App

//Create a folder and Next.js starter app
npx create-next-app@latest nextjs-devops-app

//Move inside it
cd nextjs-devops-app

//Run locally to test
npm run dev

visit: http://localhost:3000

---

## Step 2: Add a Dockerfile

FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

RUN npm run build

EXPOSE 3000

CMD ["npm", "start"]


Then run:
docker build -t nextjs-devops-app:v1 .
docker run -p 3000:3000 nextjs-devops-app:v1

visit: http://localhost:3000

---

## Step 3: Push Code to GitHub

```bash
git init
git remote add origin https://github.com/<your-username>/nextjs-devops-app.git
git add .
git commit -m "Initial commit - Next.js DevOps app"
git branch -M main
git push -u origin main
```

---

## Step 4: GitHub Actions Workflow (Build + Push to GHCR)

1. On your PC, create folders:

   ```
   .github/workflows/
   ```
2. Inside it, create a file named **`docker-build-push.yml`**:

```yaml
name: Build and Push Docker Image

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/lakshay-aswal/nextjs-devops-app:latest

```

3. Commit and push this:

```bash
git add .
git commit -m "Add GitHub Actions for Docker build"
git push
```

Go to **GitHub → Actions tab** — you’ll see a pipeline running.

GHCR image built successfully!

---

## Step 5: Deploy on Minikube (Kubernetes)

Create a folder:

```
k8s/
```

Add two files:

### `k8s/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nextjs-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nextjs-app
  template:
    metadata:
      labels:
        app: nextjs-app
    spec:
      containers:
      - name: nextjs-container
        image: ghcr.io/lakshay-aswal/nextjs-devops-app:latest
        ports:
        - containerPort: 3000
        readinessProbe:
          httpGet:
            path: /
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 15
```

### `k8s/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nextjs-service
spec:
  type: NodePort
  selector:
    app: nextjs-app
  ports:
    - port: 3000
      targetPort: 3000
      nodePort: 32000
```

Then run:

```bash
minikube start
kubectl apply -f k8s/
kubectl get pods
```

Once pods are running:

```bash
minikube service nextjs-service
```

Browser will open → your app runs inside Kubernetes!

---
