
> **Internet/User → LoadBalancer → Ingress Controller → Ingress Rules → Frontend/Backend Services → Pods**

We can build a small **e-commerce application** with:

* `shop.example.com` → Frontend
* `shop.example.com/api` → Backend API
* `shop.example.com/products` → Product service
* TLS/HTTPS
* Multiple replicas
* Path-based routing
* Host-based routing
* Health checks
* Troubleshooting


---


Before creating anything, explain this distinction:

### Service vs Ingress

Suppose we have:

```text
                     Kubernetes Cluster
                           
      Frontend Pods                 Backend Pods
      ┌─────────────┐              ┌─────────────┐
      │ frontend-1  │              │ backend-1   │
      │ frontend-2  │              │ backend-2   │
      └──────┬──────┘              └──────┬──────┘
             │                            │
       frontend-svc                 backend-svc
             │                            │
             └────────────┬───────────────┘
                          │
                       Ingress
                          │
                    Ingress Controller
                          │
                       Internet
```

A **Service** provides stable networking to Pods.

An **Ingress** defines HTTP/HTTPS routing rules.

The **Ingress Controller** is the component that actually implements those rules.

---

# 2. Real-time scenario


> "Imagine we have deployed an online shopping application."

We have:

```text
Frontend:
http://shop.example.com/

Backend:
http://shop.example.com/api/

Products:
http://shop.example.com/products/
```

Instead of exposing three different LoadBalancer services:

```text
frontend → LoadBalancer
backend  → LoadBalancer
products → LoadBalancer
```

we want:

```text
                         Internet
                            |
                            |
                    192.168.23.138
                            |
                    LoadBalancer
                            |
                  NGINX Ingress
                     Controller
                            |
               ┌────────────┼────────────┐
               |            |            |
               ↓            ↓            ↓
           /             /api        /products
               |            |            |
          frontend-svc  backend-svc  product-svc
               |            |            |
             Pods         Pods         Pods
```

This is the primary reason we use Ingress.

---

# 3. Prerequisites

For this lab, students need:

```bash
kubectl
```

and a Kubernetes cluster.

For example:

```bash
kubectl get nodes
```

Expected:

```text
NAME       STATUS   ROLES           AGE
master     Ready    control-plane   ...
worker01   Ready    <none>          ...
worker02   Ready    <none>          ...
```

Check the cluster:

```bash
kubectl cluster-info
```

---

# 4. Understand the Ingress architecture

There are **three important objects/components**:

### 1. Ingress

Contains routing rules.

Example:

```yaml
/api → backend-service
```

### 2. Ingress Controller

Reads those rules and configures the reverse proxy.

For example:

```text
NGINX
```

### 3. Service

Provides access to Pods.

---

# 5. Install NGINX Ingress Controller

For a training lab, you can install the controller using the official Kubernetes ingress-nginx deployment appropriate to your cluster environment.

After installation:

```bash
kubectl get pods -n ingress-nginx
```

You should see something similar to:

```text
NAME                                        READY
ingress-nginx-controller-xxxxx              1/1
```

Then:

```bash
kubectl get svc -n ingress-nginx
```

For a LoadBalancer-based environment:

```text
NAME                       TYPE           EXTERNAL-IP
ingress-nginx-controller   LoadBalancer   192.168.23.138
```

---

# 6. Important point about your MetalLB environment

Since your cluster is using **MetalLB**, this is especially useful for your students.

Your architecture becomes:

```text
                   Client
                     |
                     |
              192.168.23.138
                     |
                  MetalLB
                     |
           LoadBalancer Service
                     |
             NGINX Controller
                     |
              Kubernetes Ingress
                     |
        ┌────────────┴────────────┐
        |                         |
 frontend-service          backend-service
        |                         |
     Pods                      Pods
```

MetalLB allocates:

```text
192.168.23.138
```

to the LoadBalancer service.

---

# 7. Create the application namespace

Create:

```bash
kubectl create namespace ecommerce
```

Verify:

```bash
kubectl get ns
```

---

# 8. Deploy the frontend

Create:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: ecommerce
spec:
  replicas: 2
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
        image: nginx:1.27
        ports:
        - containerPort: 80
```

Save as:

```text
frontend-deployment.yaml
```

Apply:

```bash
kubectl apply -f frontend-deployment.yaml
```

Check:

```bash
kubectl get pods -n ecommerce
```

Expected:

```text
frontend-xxxxx   Running
frontend-yyyyy   Running
```

---

# 9. Create frontend Service

Create:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: ecommerce
spec:
  type: ClusterIP
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
```

Save:

```text
frontend-service.yaml
```

Apply:

```bash
kubectl apply -f frontend-service.yaml
```

Check:

```bash
kubectl get svc -n ecommerce
```

You should see:

```text
NAME               TYPE        CLUSTER-IP
frontend-service   ClusterIP   10.x.x.x
```

---

# 10. Why ClusterIP?

This is an important teaching point.

Students may ask:

> "Why don't we use LoadBalancer?"

Because the frontend doesn't need its own external IP.

We want:

```text
Internet
   |
Ingress Controller
   |
ClusterIP Service
   |
Pods
```

Only the **Ingress Controller** needs to be exposed externally.

---

# 11. Deploy the backend

For a simple training example, create another deployment.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: ecommerce
spec:
  replicas: 2
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
        image: hashicorp/http-echo:1.0
        args:
        - "-text=Hello from Backend API"
        ports:
        - containerPort: 5678
```

Apply:

```bash
kubectl apply -f backend-deployment.yaml
```

---

# 12. Backend Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: ecommerce
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 5678
```

Apply:

```bash
kubectl apply -f backend-service.yaml
```

Check:

```bash
kubectl get svc -n ecommerce
```

---

# 13. Now comes the important part — Ingress

Create:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecommerce-ingress
  namespace: ecommerce
spec:
  ingressClassName: nginx

  rules:
  - host: shop.example.com
    http:
      paths:

      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 80

      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

Save:

```text
ecommerce-ingress.yaml
```

Apply:

```bash
kubectl apply -f ecommerce-ingress.yaml
```

---

# 14. Examine the Ingress

Run:

```bash
kubectl get ingress -n ecommerce
```

You should get something like:

```text
NAME                CLASS   HOSTS               ADDRESS
ecommerce-ingress   nginx   shop.example.com    192.168.23.138
```

Describe it:

```bash
kubectl describe ingress ecommerce-ingress -n ecommerce
```

---

# 15. Understand the routing

The following rule:

```yaml
- path: /api
  pathType: Prefix
```

means:

```text
shop.example.com/api
shop.example.com/api/users
shop.example.com/api/orders
shop.example.com/api/products
```

will be routed to:

```text
backend-service
```

While:

```text
shop.example.com/
shop.example.com/login
shop.example.com/cart
shop.example.com/checkout
```

go to:

```text
frontend-service
```

---

# 16. The complete request flow

This is the most important part to teach.

A user enters:

```text
http://shop.example.com/api/users
```

### Step 1

DNS resolves:

```text
shop.example.com
        ↓
192.168.23.138
```

### Step 2

Request reaches the LoadBalancer:

```text
192.168.23.138:80
```

### Step 3

MetalLB directs traffic to the NGINX Ingress Controller.

```text
MetalLB
   ↓
NGINX Ingress Controller
```

### Step 4

NGINX examines:

```text
Host: shop.example.com
Path: /api/users
```

### Step 5

Ingress rule matches:

```text
shop.example.com
        +
/api
```

### Step 6

NGINX sends request to:

```text
backend-service:80
```

### Step 7

Kubernetes Service selects:

```text
backend pod 1
backend pod 2
```

### Step 8

One backend Pod receives the request.

So:

```text
Client
  |
  ↓
DNS
  |
  ↓
MetalLB
  |
  ↓
LoadBalancer Service
  |
  ↓
NGINX Ingress Controller
  |
  ↓
Ingress Rule
  |
  ↓
backend-service
  |
  ↓
Endpoint
  |
  ↓
backend Pod
```

That's the complete journey.

---

# 17. Test without DNS

You don't need real DNS for the classroom.

Suppose the LoadBalancer IP is:

```text
192.168.23.138
```

From your laptop:

```bash
curl -H "Host: shop.example.com" http://192.168.23.138/
```

And:

```bash
curl -H "Host: shop.example.com" http://192.168.23.138/api
```

The first request goes to:

```text
frontend-service
```

The second goes to:

```text
backend-service
```

---

# 18. Better testing using /etc/hosts

On Linux:

```bash
sudo vi /etc/hosts
```

Add:

```text
192.168.23.138 shop.example.com
```

Then:

```bash
ping shop.example.com
```

or:

```bash
curl http://shop.example.com/
```

and:

```bash
curl http://shop.example.com/api
```

This gives students a very realistic experience.

---

# 19. Host-based routing

Now introduce another application.

Suppose we have:

```text
shop.example.com
admin.example.com
api.example.com
```

Architecture:

```text
                    Internet
                       |
                192.168.23.138
                       |
                 NGINX Ingress
                       |
        ┌──────────────┼──────────────┐
        ↓              ↓              ↓
 shop.example.com admin.example.com api.example.com
        |              |              |
    frontend        admin-svc       api-svc
```

Ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecommerce-ingress
  namespace: ecommerce
spec:
  ingressClassName: nginx

  rules:

  - host: shop.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80

  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 80
```

Now:

```text
shop.example.com
       ↓
frontend-service
```

and:

```text
api.example.com
       ↓
backend-service
```

---

# 20. Path-based vs Host-based routing

Teach students this table:

| Routing | Example            | Destination     |
| ------- | ------------------ | --------------- |
| Host    | `shop.example.com` | Frontend        |
| Host    | `api.example.com`  | Backend         |
| Path    | `/api`             | Backend         |
| Path    | `/products`        | Product service |
| Path    | `/admin`           | Admin service   |

Real-world environments often combine both.

---

# 21. Add Product Service

Create:

```text
product-service
```

Then create this rule:

```yaml
- path: /products
  pathType: Prefix
  backend:
    service:
      name: product-service
      port:
        number: 80
```

Now:

```text
shop.example.com/
       ↓
frontend

shop.example.com/api
       ↓
backend

shop.example.com/products
       ↓
product-service
```

---

# 22. Important `pathType`

Students should understand these:

### Prefix

```yaml
pathType: Prefix
```

Matches:

```text
/api
/api/users
/api/orders
/api/products
```

### Exact

```yaml
pathType: Exact
```

Matches only:

```text
/api
```

but not:

```text
/api/users
```

This is a common interview question.

---

# 23. Very important concept — Ingress is not a proxy by itself

This is another excellent interview point.

Creating:

```yaml
kind: Ingress
```

doesn't automatically make traffic work.

You need:

```text
Ingress
+
Ingress Controller
```

For example:

```text
Ingress object
       ↓
Kubernetes API
       ↓
Ingress Controller watches it
       ↓
Controller configures NGINX
       ↓
NGINX handles traffic
```

---

# 24. How does the controller know about Ingress?

This is worth demonstrating.

Run:

```bash
kubectl get ingress
```

The API server stores the Ingress object.

The controller watches Kubernetes API resources.

Conceptually:

```text
               Kubernetes API Server
                       |
                       |
                Ingress Object
                       |
                       ↓
              Controller watches
                       |
                       ↓
                  NGINX config
                       |
                       ↓
                    Traffic
```

---

# 25. Troubleshooting exercise

This is where your training becomes much more valuable.

Ask students:

> "The application is returning 404. Find the problem."

Give them:

```bash
kubectl get ingress -n ecommerce
```

Then:

```bash
kubectl describe ingress ecommerce-ingress -n ecommerce
```

Then:

```bash
kubectl get svc -n ecommerce
```

Then:

```bash
kubectl get endpoints -n ecommerce
```

Then:

```bash
kubectl get pods -n ecommerce
```

---

# 26. Troubleshooting flow

Teach this sequence:

```text
Client
  |
  ↓
DNS
  |
  ↓
LoadBalancer
  |
  ↓
Ingress Controller
  |
  ↓
Ingress
  |
  ↓
Service
  |
  ↓
Endpoints
  |
  ↓
Pod
```

At each layer check:

```bash
kubectl get ...
```

For example:

### Controller

```bash
kubectl get pods -n ingress-nginx
```

### Ingress

```bash
kubectl get ingress -n ecommerce
```

### Service

```bash
kubectl get svc -n ecommerce
```

### Endpoints

```bash
kubectl get endpoints -n ecommerce
```

### Pods

```bash
kubectl get pods -n ecommerce
```

### Logs

```bash
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller
```

---

# 27. Common problem — Service selector

Suppose:

```yaml
Service:

selector:
  app: backend
```

But Pod has:

```yaml
labels:
  app: backend-api
```

Then:

```bash
kubectl get endpoints backend-service -n ecommerce
```

will show no endpoints.

Explain:

```text
Service selector
       ↓
      app=backend
       ↓
Does Pod have app=backend?
       ↓
       NO
       ↓
No endpoint
       ↓
Traffic fails
```

This is an excellent practical exercise.

---

# 28. TLS / HTTPS

After students understand HTTP, introduce HTTPS.

Architecture:

```text
              HTTPS
                |
                ↓
       shop.example.com
                |
                ↓
       NGINX Ingress
                |
          TLS termination
                |
                ↓
        frontend-service
```

Create a TLS secret:

```bash
kubectl create secret tls ecommerce-tls \
  --cert=tls.crt \
  --key=tls.key \
  -n ecommerce
```

Then add:

```yaml
tls:
- hosts:
  - shop.example.com
  secretName: ecommerce-tls
```

Complete example:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecommerce-ingress
  namespace: ecommerce
spec:
  ingressClassName: nginx

  tls:
  - hosts:
    - shop.example.com
    secretName: ecommerce-tls

  rules:
  - host: shop.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80

      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 80
```

Now:

```text
Client
  |
 HTTPS
  |
  ↓
Ingress Controller
  |
 TLS termination
  |
 HTTP
  |
  ↓
Service
  |
  ↓
Pod
```

---

# 29. Production architecture

Once students understand the basic lab, show them this:

```text
                         Internet
                            |
                            |
                       DNS / Route53
                            |
                            ↓
                     Cloud Load Balancer
                            |
                            ↓
                 NGINX Ingress Controller
                     /               \
                    /                 \
                   ↓                   ↓
              Ingress Rules        TLS
                   |
       ┌───────────┼────────────┐
       ↓           ↓            ↓
   Frontend      API       Product Service
       |           |            |
       ↓           ↓            ↓
   Service      Service       Service
       |           |            |
       ↓           ↓            ↓
    Pods         Pods          Pods
```

In AWS, for example:

```text
Route53
   ↓
AWS Load Balancer
   ↓
Ingress Controller
   ↓
Kubernetes Services
   ↓
Pods
```

In your MetalLB lab:

```text
DNS
 ↓
MetalLB IP
 ↓
Ingress Controller
 ↓
Services
 ↓
Pods
```

---

# 30. Ingress vs LoadBalancer Service

This is an important interview question.

### Without Ingress

```text
frontend → LoadBalancer → Public IP 1
backend  → LoadBalancer → Public IP 2
admin    → LoadBalancer → Public IP 3
```

Potentially:

```text
3 applications
3 LoadBalancers
3 external IPs
```

### With Ingress

```text
                    LoadBalancer
                         |
                    Ingress
                  /     |      \
                 /      |       \
          frontend    backend   admin
```

One entry point can route many applications.

---

# 31. Ingress Controller vs Ingress

Students frequently confuse these.

| Component          | Purpose                                                  |
| ------------------ | -------------------------------------------------------- |
| Ingress            | Routing rules                                            |
| Ingress Controller | Implements routing                                       |
| Service            | Exposes Pods internally                                  |
| Pod                | Runs application                                         |
| LoadBalancer       | Provides external access                                 |
| MetalLB            | Provides LoadBalancer IPs in bare-metal/on-prem clusters |

---

# 32. Hands-on lab progression

I'd conduct this as a **3–4 hour practical session**.

### Lab 1 — Deploy application

Students create:

```text
frontend Deployment
frontend Service

backend Deployment
backend Service
```

### Lab 2 — Install Ingress Controller

```text
NGINX Ingress Controller
```

### Lab 3 — Create basic Ingress

```text
/ → frontend
/api → backend
```

### Lab 4 — Host routing

```text
shop.example.com
api.example.com
```

### Lab 5 — TLS

```text
HTTPS
```

### Lab 6 — Scaling

```bash
kubectl scale deployment frontend --replicas=5
```

Explain that Service/Ingress doesn't need to change.

### Lab 7 — Failure testing

Delete a Pod:

```bash
kubectl delete pod <pod-name> -n ecommerce
```

Observe:

```text
Deployment
   ↓
creates replacement Pod
```

### Lab 8 — Troubleshooting

Intentionally break:

```text
Service selector
Ingress service name
Ingress port
Pod labels
```

Students diagnose the problem.

---

# 33. Final student exercise

Give them this requirement:

> Build an online shopping application with three microservices.

```text
shop.example.com
```

Requirements:

```text
/              → frontend-service
/api           → backend-service
/products      → product-service
/admin         → admin-service
```

Requirements:

* Frontend: 3 replicas
* Backend: 3 replicas
* Products: 2 replicas
* Admin: 2 replicas
* All Services must be `ClusterIP`
* Only Ingress Controller is externally accessible
* Configure HTTPS
* Configure host/path routing
* Test all endpoints
* Demonstrate Pod failure and recovery
* Troubleshoot one deliberately broken Service selector

---

# 34. The final architecture students should be able to draw

![Image](https://images.openai.com/static-rsc-4/CQEs3TQ4rtqUS1J4eCIeeY3kDNo4AD9JPMc6KnDzUvTLa6pwL_zfPiGYENKNqRSYsVyTx7Kt4cWMznvkEa-29Jy0l8zKohQkbdgNcim3jRsACjtgkM1Gbq-n2X05oMoTFGkAnCBhtPFmGmswZkW2lgHcpbqOwUwi7-iAlD5ZJTXdJqG7_gHGSYZaShE9-r52?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/p4PVk2zoUmjVBWQwo2-7TibLc537qCvhQGouDP04Q-WVvjIIIsooPgB29YbKkqvdAabQsNPohA8w4qGEGz74VKb_xsqYNbJOVq2ysm9PNUAOn74haXhLlJcBXEeTqDHx5nGVj5cIAYRdUzaOdhONNG_Yl-lgZjm-mIV7zhLt0vfALNRqTPVW6N_si96obWO2?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/3YcPp6ndLQsFBXyOlcVbmlwdk_jm_Jp3Z8Lw1akjz_UJg1tdq-LCXxnqFOSsVgkzR6f4YK6Mb2LAYZweVe_NCjKDz0ypAr89ytRAIA-lTh2zRE7HNdhA7XcEQDKAnqzQ1-l9p5bJGcLOenLIpN8GxSkHk91hy6GFEj9qvubKFObil3z2dcOFppbhMEYKPMuh?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/61RisAc_BvXVGMoC-KtoiuUaAVkBlOSlu8Y12MLevSemxPKmocBRqMg8qHGcyCCbnuagyQ_2gJ2JItKyzvmzkzbqg3e79Qh5VJp2yNjTlibdj0B7GRPt-xy_iEgJh3TbRzI-BIWPyhJAGWW0jCE5Yp_5skpt5tfogKHMpZkgTliYSkBMAXJNb1eCwLp_fVbg?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/IUtB3ydFKUZzWhHUSvAhFw9IUArFkZGRtqFHqFLZkuciHOcxGdXjPk5RP6cE34hrQ9NyYK6FWtz7tAIWh65XB3nR3_lpfZy7nPUs5nzLSTdquPhGnxL0lWZIZooBJMviKQtRYOq6A2xpjjYIdlpMo15n02UhTdSDE41GyFQiuyT9Pbo4KVHAaH5jwWqiiPzr?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/pFnDQHPCqUgql5ggdlrUml8KXO-zMXPMN1i0JSElCBgyKuBxOsmcbSNVu-KW6mlMvBTrVnzmqtsSubkdUe_GnpltYIjBxHa54-40LcRCcRSrsnOlBPB0R7wEPcxh7VJpm4xd8FDSKhfFA_fIFgfFtTH7wWLDM8HFxrcMqJAdd4C3JIH9RJ1BHvRLK0prhUuP?purpose=fullsize)

```text
                           USER
                            |
                            | HTTPS
                            ↓
                     shop.example.com
                            |
                            ↓
                    DNS / hosts file
                            |
                            ↓
                 ┌─────────────────────┐
                 │   MetalLB / LB      │
                 │   192.168.23.138    │
                 └──────────┬──────────┘
                            |
                            ↓
              ┌─────────────────────────┐
              │ NGINX Ingress Controller│
              └────────────┬────────────┘
                           |
                 ┌─────────┼─────────┐
                 |         |         |
                 ↓         ↓         ↓
                /        /api    /products
                 |         |         |
                 ↓         ↓         ↓
          frontend-svc backend-svc product-svc
                 |         |         |
              ┌──┴──┐   ┌──┴──┐   ┌──┴──┐
              ↓     ↓   ↓     ↓   ↓     ↓
             Pod   Pod Pod   Pod Pod   Pod
```

## The key message for students

Make them remember this one line:

> **Ingress defines HTTP/HTTPS routing rules; the Ingress Controller implements those rules; Services provide stable access to the Pods.**

And the request journey:

```text
DNS
 ↓
LoadBalancer
 ↓
Ingress Controller
 ↓
Ingress Rule
 ↓
Service
 ↓
Endpoint
 ↓
Pod
```

