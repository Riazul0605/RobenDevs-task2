## 1️⃣ Namespace: `namespace-dev.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

---

## 2️⃣ Configs: `app-configmap.yaml` & `app-secrets.yaml`

### `app-configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: dev
data:
  DB_HOST: postgres
  DB_PORT: "5432"
  DB_NAME: app_db
  API_BASE_URL: http://backend:4000
```

### `app-secrets.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: dev
type: Opaque
data:
  DB_USER: YXBwX3VzZXI=       # base64 of "app_user"
  DB_PASSWORD: YXBwX3Bhc3N3b3Jk  # base64 of "app_password"
```

> Note: Encode your secret values in base64:
>
> ```bash
> echo -n "app_user" | base64
> echo -n "app_password" | base64
> ```

---

## 3️⃣ PostgreSQL Persistent Storage

### `postgres-pv.yaml`

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv
  namespace: dev
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/data/postgres
```

### `postgres-pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: dev
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

---

## 4️⃣ PostgreSQL Deployment & Service

### `postgres-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: DB_NAME
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: DB_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: DB_PASSWORD
          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: postgres-storage
      volumes:
        - name: postgres-storage
          persistentVolumeClaim:
            claimName: postgres-pvc
```

### `postgres-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: dev
spec:
  selector:
    app: postgres
  ports:
    - protocol: TCP
      port: 5432
      targetPort: 5432
  type: ClusterIP
```

---

## 5️⃣ Backend Deployment & Service

### `backend-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: your-backend-image:latest
          ports:
            - containerPort: 4000
          envFrom:
            - configMapRef:
                name: app-config
            - secretRef:
                name: app-secrets
          readinessProbe:
            httpGet:
              path: /health
              port: 4000
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: 4000
            initialDelaySeconds: 10
            periodSeconds: 10
```

### `backend-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: dev
spec:
  selector:
    app: backend
  ports:
    - protocol: TCP
      port: 4000
      targetPort: 4000
  type: ClusterIP
```

---

## 6️⃣ Frontend Deployment & Service

### `frontend-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: your-frontend-image:latest
          ports:
            - containerPort: 80
          envFrom:
            - configMapRef:
                name: app-config
```

### `frontend-service.yaml` (NodePort)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: dev
spec:
  selector:
    app: frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30001
  type: NodePort
```

> After `minikube service frontend -n dev` you can open the frontend in browser.

---

## 7️⃣ Apply all resources

From **project root**:

```bash
kubectl apply -f namespace-dev.yaml
kubectl apply -f storage/
kubectl apply -f config/
kubectl apply -f postgreSQL/
kubectl apply -f backend/
kubectl apply -f frontend/
```

Check pods:

```bash
kubectl get pods -n dev
kubectl get svc -n dev
```

---

✅ **Features implemented**

* All resources in `dev` namespace
* Backend & Postgres → `ClusterIP` (internal)
* Frontend → `NodePort` (external)
* Postgres uses PVC
* Backend includes **readiness & liveness probes**
* Config & Secrets for DB credentials

---
