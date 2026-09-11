
> **Frontend Nginx → Backend Python API → Backend returns product information**

This lets students actually see the request travelling through a Kubernetes Service.

## 1. Architecture

```text
                 Kubernetes Cluster
┌──────────────────────────────────────────────────────┐
│                                                      │
│  Frontend Pod                                       │
│  ┌─────────────────────┐                            │
│  │ Nginx               │                            │
│  │ Website             │                            │
│  │ :80                 │                            │
│  └──────────┬──────────┘                            │
│             │                                       │
│             │ HTTP request                          │
│             │ http://backend-service:5000/products │
│             ▼                                       │
│  ┌─────────────────────┐                            │
│  │ backend-service     │                            │
│  │ ClusterIP           │                            │
│  │ 10.96.x.x:5000      │                            │
│  └──────────┬──────────┘                            │
│             │                                       │
│             ▼                                       │
│  ┌─────────────────────┐                            │
│  │ Backend Pod         │                            │
│  │ Python Flask        │                            │
│  │ :5000               │                            │
│  └─────────────────────┘                            │
│                                                      │
└──────────────────────────────────────────────────────┘
```

The important thing is:

```text
Frontend
   |
   | http://backend-service:5000
   ↓
ClusterIP Service
   |
   ↓
Backend Pod
```

---

# 2. Create the Backend

We'll create a small Python Flask application.

### `backend.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
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
          image: python:3.11-slim
          command:
            - /bin/sh
            - -c
            - |
              pip install flask
              cat <<'EOF' > /tmp/app.py
              from flask import Flask, jsonify

              app = Flask(__name__)

              @app.route("/products")
              def products():
                  return jsonify({
                      "product": "Laptop",
                      "price": 75000,
                      "message": "Response from Backend Pod"
                  })

              app.run(host="0.0.0.0", port=5000)
              EOF

              python /tmp/app.py
          ports:
            - containerPort: 5000
```

Apply:

```bash
kubectl apply -f backend.yaml
```

Check:

```bash
kubectl get pods
```

You should see:

```text
NAME                       READY   STATUS
backend-xxxxxxxxxx-xxxxx   1/1     Running
```

---

# 3. Create the ClusterIP Service

Now create:

### `backend-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 5000
      targetPort: 5000
```

Apply:

```bash
kubectl apply -f backend-service.yaml
```

Check:

```bash
kubectl get svc
```

Example:

```text
NAME              TYPE        CLUSTER-IP      PORT(S)
backend-service   ClusterIP   10.96.150.20    5000/TCP
```

Now you have:

```text
backend-service
       |
       ↓
10.96.150.20:5000
       |
       ↓
Backend Pod :5000
```

---

# 4. Create the Frontend

For the demonstration, let's use Nginx as the frontend.

### `frontend.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
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
          image: nginx:latest
          ports:
            - containerPort: 80
```

Apply:

```bash
kubectl apply -f frontend.yaml
```

---

# 5. Now the important part — frontend calls backend

We can enter the frontend Pod:

```bash
kubectl exec -it deploy/frontend -- bash
```

Inside the frontend container:

```bash
apt update
apt install curl -y
```

Now call:

```bash
curl http://backend-service:5000/products
```

You should get:

```json
{
  "message": "Response from Backend Pod",
  "price": 75000,
  "product": "Laptop"
}
```

🎯 **This is the actual demonstration of ClusterIP.**

The frontend doesn't know the backend Pod IP.

It only knows:

```text
backend-service:5000
```

---

# 6. Let's see what happened

When you execute:

```bash
curl http://backend-service:5000/products
```

the flow is:

```text
                    DNS
                     │
                     ▼
frontend Pod
     │
     │ backend-service
     ▼
CoreDNS
     │
     │ resolves
     ▼
10.96.150.20
     │
     ▼
ClusterIP Service
     │
     │ targetPort 5000
     ▼
Backend Pod
10.244.x.x:5000
     │
     ▼
Flask application
     │
     ▼
JSON response
```

---

# 7. Show students the DNS resolution

From the frontend Pod:

```bash
getent hosts backend-service
```

You might get:

```text
10.96.150.20 backend-service
```

This proves that:

```text
backend-service
       ↓
10.96.150.20
```

is being resolved through Kubernetes DNS.

---

# 8. Show students the Service endpoints

Run:

```bash
kubectl get endpoints backend-service
```

Example:

```text
NAME              ENDPOINTS
backend-service   10.244.1.15:5000
```

Now you can explain:

```text
ClusterIP
10.96.150.20
      |
      | Service routing
      ↓
Pod IP
10.244.1.15:5000
```

The Service provides the **stable address**.

The Pod IP can change.

---

# 9. Demonstrate the real benefit

Now delete the backend Pod:

```bash
kubectl delete pod -l app=backend
```

Kubernetes automatically creates a new backend Pod.

Check:

```bash
kubectl get pods -o wide
```

Suppose the old Pod was:

```text
10.244.1.15
```

and the new Pod becomes:

```text
10.244.1.25
```

The frontend doesn't care.

Run again:

```bash
kubectl exec -it deploy/frontend -- curl http://backend-service:5000/products
```

It still works.

Why?

Because:

```text
Frontend
   |
   ↓
backend-service
10.96.150.20
   |
   ↓
NEW Backend Pod
10.244.1.25
```

The **Service remains unchanged**, even though the Pod IP changed.

---

# 10. Make it even more interesting — 3 backend Pods

Change:

```yaml
replicas: 1
```

to:

```yaml
replicas: 3
```

Now:

```text
                    backend-service
                    10.96.150.20
                          |
             ┌────────────┼────────────┐
             ↓            ↓            ↓
       Backend Pod 1 Backend Pod 2 Backend Pod 3
       10.244.1.10   10.244.2.15   10.244.3.20
```

Run:

```bash
kubectl get endpoints backend-service
```

You'll see multiple endpoints.

Now the Service can distribute requests among the available backend Pods.

---

## Best classroom demonstration

I'd teach it in this exact sequence:

```text
STEP 1
Create Backend Pod
       ↓
STEP 2
Create ClusterIP Service
       ↓
STEP 3
Create Frontend Pod
       ↓
STEP 4
Frontend calls:
http://backend-service:5000/products
       ↓
STEP 5
Show CoreDNS
       ↓
STEP 6
Show ClusterIP
       ↓
STEP 7
Show Endpoints
       ↓
STEP 8
Delete Backend Pod
       ↓
STEP 9
Show new Pod IP
       ↓
STEP 10
Call backend again
       ↓
SUCCESS
```

