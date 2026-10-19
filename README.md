# Deploying BookShelf on Kubernetes with Minikube

This exercise deploys **BookShelf**, a small Python web application backed by PostgreSQL, on a local Kubernetes cluster. You will build and load the web image, deploy the application and database, expose the web application, and experiment with scaling, recovery, persistence, and updates.

> **Scope:** This is a teaching deployment, not a production-ready architecture. The database has one replica, the password is for demonstration only, and persistent storage is not a substitute for backups.

## Learning objectives

- Start and inspect a Minikube cluster.
- Deploy applications using Kubernetes Deployments.
- Use Services for stable networking and service discovery.
- Configure applications with environment variables and Secrets.
- Request persistent storage with a PersistentVolumeClaim (PVC).
- Inspect Pods, logs, rollouts, and application behaviour.

## 1. Prerequisites

This guide assumes macOS and a terminal. Install and start [Docker Desktop](https://www.docker.com/products/docker-desktop/), then install Minikube and `kubectl`:

```bash
brew install minikube kubectl
docker version
minikube version
kubectl version --client
```

Start a local cluster:

```bash
minikube start --driver=docker --cpus=4 --memory=6144
kubectl cluster-info
kubectl get nodes
kubectl get namespaces
kubectl get pods -n kube-system
```

The node should eventually report `Ready`. See the [Minikube getting-started guide](https://minikube.sigs.k8s.io/docs/start/) if the cluster fails to start.

## 2. Obtain the BookShelf application

Use the BookShelf project supplied with the Docker exercise. The expected layout is:

```text
bookshelf/
├── web/
│   ├── app.py
│   ├── Dockerfile
│   └── requirements.txt
└── k8s/
```

The web application listens on port `8080` and uses these environment variables:

| Variable | Value |
|---|---|
| `DB_HOST` | `db` |
| `POSTGRES_DB` | `bookshelf` |
| `POSTGRES_USER` | `postgres` |
| `POSTGRES_PASSWORD` | Provided by a Kubernetes Secret |

The application should provide `GET /` for the BookShelf page and `GET /health` for a health check. Adjust paths if your project uses a different layout.

## 3. Build and load the web image

From the project root:

```bash
docker build -t ictinfr/bookshelf-web:1.0 ./web
minikube image load ictinfr/bookshelf-web:1.0
minikube image ls
```

For this local exercise, no public registry is required. The Deployment uses `imagePullPolicy: IfNotPresent` so Kubernetes can use the image loaded into Minikube. If you rebuild the image later, load it into Minikube again. For version updates, prefer a new tag such as `1.1`.

## 4. Create the Kubernetes manifests

Save the following files in the `k8s/` directory.

### 4.1 Database password — `k8s/db-secret.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: bookshelf-db-secret
type: Opaque
stringData:
  POSTGRES_PASSWORD: "example"
```

This is a classroom credential. Never commit real passwords or tokens to a public GitHub repository. Production deployments should use an appropriate secret-management workflow and restrict access to Secrets.

### 4.2 Persistent storage — `k8s/db-pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: bookshelf-db-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

The PVC requests 1 GiB of storage for PostgreSQL. Minikube commonly has a default StorageClass that can dynamically provision a volume; this depends on the cluster configuration.

### 4.3 Database Deployment — `k8s/db-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bookshelf-db
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bookshelf-db
  template:
    metadata:
      labels:
        app: bookshelf-db
    spec:
      containers:
        - name: postgres
          image: postgres:17
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              value: bookshelf
            - name: POSTGRES_USER
              value: postgres
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: bookshelf-db-secret
                  key: POSTGRES_PASSWORD
          volumeMounts:
            - name: database-data
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: database-data
          persistentVolumeClaim:
            claimName: bookshelf-db-data
```

This Deployment runs one PostgreSQL Pod and mounts the PVC at the data directory used by the PostgreSQL 17 image.

### 4.4 Database Service — `k8s/db-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db
spec:
  type: ClusterIP
  selector:
    app: bookshelf-db
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432
```

The Service gives PostgreSQL a stable in-cluster name, `db`. The web application can use `DB_HOST=db` without knowing the database Pod's IP address.

### 4.5 Web Deployment — `k8s/web-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bookshelf-web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: bookshelf-web
  template:
    metadata:
      labels:
        app: bookshelf-web
    spec:
      containers:
        - name: web
          image: ictinfr/bookshelf-web:1.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
          env:
            - name: DB_HOST
              value: db
            - name: POSTGRES_DB
              value: bookshelf
            - name: POSTGRES_USER
              value: postgres
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: bookshelf-db-secret
                  key: POSTGRES_PASSWORD
```

The Deployment requests two web replicas. The application code retries database connections, helping it tolerate the database still starting.

### 4.6 Web Service — `k8s/web-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: bookshelf-web
spec:
  type: ClusterIP
  selector:
    app: bookshelf-web
  ports:
    - name: http
      port: 8080
      targetPort: 8080
```

The Service provides a stable in-cluster endpoint for the web Pods. We will use port forwarding to access it from the host machine.

## 5. Deploy BookShelf

Apply the manifests in this order:

```bash
kubectl apply -f k8s/db-secret.yaml
kubectl apply -f k8s/db-pvc.yaml
kubectl apply -f k8s/db-deployment.yaml
kubectl apply -f k8s/db-service.yaml
kubectl apply -f k8s/web-deployment.yaml
kubectl apply -f k8s/web-service.yaml
```

Wait for both Deployments:

```bash
kubectl rollout status deployment/bookshelf-db
kubectl rollout status deployment/bookshelf-web
```

Inspect the resources:

```bash
kubectl get deployments
kubectl get pods -o wide
kubectl get services
kubectl get pvc
```

Expected high-level result:

- `bookshelf-db` has one available replica.
- `bookshelf-web` has two available replicas.
- Both Services exist.
- `bookshelf-db-data` is `Bound`.

Pod names and IP addresses will vary. If a resource is not ready, inspect its events and logs rather than assuming deployment succeeded.

## 6. Access the application

Run port forwarding in a terminal and leave it running:

```bash
kubectl port-forward service/bookshelf-web 8080:8080
```

Open [http://localhost:8080](http://localhost:8080) in your browser. Add a book and refresh the page to verify it appears.

In a second terminal, check the health endpoint:

```bash
curl http://localhost:8080/health
```

The exact response depends on the implementation of `app.py`. Stop port forwarding with `Ctrl+C`.

## 7. How the resources work together

| Resource | Role |
|---|---|
| Web Deployment | Maintains two web application replicas |
| Web Pods | Run the Python application |
| Web Service | Provides a stable endpoint for web traffic |
| Database Deployment | Runs PostgreSQL |
| Database Service (`db`) | Provides a stable database endpoint |
| Secret | Supplies the database password |
| PVC | Requests persistent storage for PostgreSQL |

The Service selector must match the labels on its target Pods. The web application connects to PostgreSQL through the `db` Service, not through a hard-coded Pod IP.

## 8. Experiment with scaling and recovery

### 8.1 Scale the web application

Scale from two to four replicas:

```bash
kubectl scale deployment/bookshelf-web --replicas=4
kubectl get pods -l app=bookshelf-web
kubectl get deployment bookshelf-web
```

Kubernetes attempts to reach the desired replica count. Scaling the web Deployment does not automatically scale PostgreSQL.

### 8.2 Delete one web Pod

List the web Pods:

```bash
kubectl get pods -l app=bookshelf-web
```

Delete one of the names returned:

```bash
kubectl delete pod <web-pod-name>
kubectl get pods -l app=bookshelf-web -w
```

The ReplicaSet should create a replacement Pod. Press `Ctrl+C` to stop watching.

### 8.3 Verify database persistence

1. Add one or more books through the web interface.
2. Identify the database Pod:

   ```bash
   kubectl get pods -l app=bookshelf-db
   ```

3. Delete that Pod:

   ```bash
   kubectl delete pod <database-pod-name>
   ```

4. Wait for PostgreSQL to return:

   ```bash
   kubectl rollout status deployment/bookshelf-db
   ```

5. Refresh BookShelf and verify the books remain.

The replacement Pod should mount the same PVC. The web application may need a short time to reconnect.

**This demonstrates persistence across Pod replacement, not high availability.** The database has one replica, and a PVC is not a backup. Do not delete the PVC if you want to preserve the exercise data.

## 9. Update the web application

This exercise is meaningful if you have made a change to the application or otherwise prepared a distinct image.

1. Modify the application, for example by changing a page title.
2. Build a new image tag:

   ```bash
   docker build -t ictinfr/bookshelf-web:1.1 ./web
   ```

3. Load it into Minikube:

   ```bash
   minikube image load ictinfr/bookshelf-web:1.1
   ```

4. Update the Deployment:

   ```bash
   kubectl set image deployment/bookshelf-web \
     web=ictinfr/bookshelf-web:1.1
   ```

5. Monitor the rollout:

   ```bash
   kubectl rollout status deployment/bookshelf-web
   kubectl rollout history deployment/bookshelf-web
   ```

6. Verify the updated application in the browser.

If the new version fails or behaves unexpectedly, roll back:

```bash
kubectl rollout undo deployment/bookshelf-web
kubectl rollout status deployment/bookshelf-web
```

A Deployment rollback restores a previous workload revision when available. It does not reverse database schema changes or undo data modifications.

## 10. Troubleshooting

### Pod stuck in `Pending`

```bash
kubectl describe pod <pod-name>
kubectl get events
kubectl get pvc
```

Check scheduling events, resource availability, and whether the PVC is bound.

### `ImagePullBackOff` or `ErrImagePull`

```bash
kubectl describe pod <pod-name>
minikube image ls
```

Check that the image name and tag match the manifest and that the image is available to Minikube.

### Container repeatedly restarts

```bash
kubectl logs <pod-name>
kubectl describe pod <pod-name>
```

Look for application exceptions, incorrect environment variables, or startup failures.

### Web application cannot connect to PostgreSQL

```bash
kubectl get pods
kubectl get services
kubectl describe service db
kubectl logs deployment/bookshelf-db
kubectl logs deployment/bookshelf-web
```

Check database readiness, the `db` Service selector, and the database name, username, password, and `DB_HOST`.

### Service is unreachable

```bash
kubectl get endpointslices
kubectl describe service bookshelf-web
kubectl get pods --show-labels
```

Verify the Service selector, Pod labels, Service port, and application listening port.

## 11. Cleanup

Delete the resources when finished:

```bash
kubectl delete -f k8s/web-service.yaml
kubectl delete -f k8s/web-deployment.yaml
kubectl delete -f k8s/db-service.yaml
kubectl delete -f k8s/db-deployment.yaml
kubectl delete -f k8s/db-pvc.yaml
kubectl delete -f k8s/db-secret.yaml
```

Deleting the PVC may also delete the underlying storage, depending on the StorageClass reclaim policy. Only delete it if you no longer need the data.

Stop Minikube:

```bash
minikube stop
```

To remove the local cluster entirely:

```bash
minikube delete
```