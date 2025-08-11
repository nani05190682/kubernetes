---

## **Kubernetes StatefulSet – Hands-on Lab for Students**

---

### **Pre-requisites**

* Kubernetes cluster ready (minikube, kind, EKS, etc.)
* `kubectl` installed and configured
* Basic understanding of Pods, Services, and PVCs

---

## **Step 1 – Create a Headless Service**

A headless service is needed for pod-to-pod communication with stable DNS records.

```yaml
# headless-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  clusterIP: None   # headless
  selector:
    app: nginx
  ports:
  - port: 80
    name: web
```

**Command:**

```bash
kubectl apply -f headless-svc.yaml
```

**Check:**

```bash
kubectl get svc nginx
```

> You should see `CLUSTER-IP` as `<none>`.

---

## **Step 2 – Create the StatefulSet**

```yaml
# statefulset-nginx.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"   # must match the headless service name
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.23
        ports:
        - containerPort: 80
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

**Command:**

```bash
kubectl apply -f statefulset-nginx.yaml
```

**Check:**

```bash
kubectl get statefulsets
kubectl get pods -l app=nginx
```

> Notice pod names: `web-0`, `web-1`, `web-2` (created one by one).

---

## **Step 3 – Observe the stable network identities**

```bash
kubectl exec -it web-0 -- hostname
kubectl exec -it web-1 -- hostname
kubectl exec -it web-2 -- hostname
```

> Output will match pod names (`web-0`, `web-1`, `web-2`).

Check DNS resolution:

```bash
kubectl run -it tmp --image=busybox --restart=Never -- sh
nslookup web-0.nginx
nslookup web-1.nginx
```

---

## **Step 4 – Test persistent storage**

Edit content inside `web-0`:

```bash
kubectl exec -it web-0 -- sh -c "echo 'Hello from web-0' > /usr/share/nginx/html/index.html"
```

Delete `web-0`:

```bash
kubectl delete pod web-0
```

Check that it comes back with the same data:

```bash
kubectl exec -it web-0 -- cat /usr/share/nginx/html/index.html
```

> The file should still contain “Hello from web-0” — proving persistence.

---

## **Step 5 – Scale up and scale down**

Scale to 5 pods:

```bash
kubectl scale statefulset web --replicas=5
kubectl get pods -l app=nginx
```

Scale down to 2 pods:

```bash
kubectl scale statefulset web --replicas=2
kubectl get pods -l app=nginx
```

> PVCs for deleted pods still exist — data is preserved if scaled up again.

---

## **Step 6 – View PVC and PV mapping**

```bash
kubectl get pvc
kubectl get pv
```

> You’ll see `www-web-0`, `www-web-1`, etc. mapped to PVs.

---

## **Step 7 – Cleanup**

```bash
kubectl delete statefulset web
kubectl delete svc nginx
kubectl delete pvc --all
```

---

### **What students will learn from this demo**

* Difference between **Deployment** and **StatefulSet**
* Importance of **headless service**
* How **pod identity** works (`podname-ordinal.service-name`)
* How **persistent storage** is retained across restarts
* Ordered pod creation and termination
* PVC → PV mapping for StatefulSets

---

