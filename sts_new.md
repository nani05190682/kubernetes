Absolutely. Let’s learn **Kubernetes StatefulSet end-to-end using a real production-style example**.

A very good example is a **MySQL database cluster**, because databases need:

* Stable pod names
* Persistent storage
* Stable network identity
* Data that survives pod restarts
* Ordered creation/deletion
* Each replica having its own storage

---

# 1. What is a StatefulSet?

A **StatefulSet** is a Kubernetes workload object designed for applications that need **stable identity and persistent state**.

Compare:

| Deployment                      | StatefulSet                         |
| ------------------------------- | ----------------------------------- |
| Stateless applications          | Stateful applications               |
| Pods have random names          | Pods have predictable names         |
| `web-7d8f...`                   | `mysql-0`, `mysql-1`, `mysql-2`     |
| Pods are interchangeable        | Pods have identity                  |
| Usually shared/external storage | Individual persistent storage       |
| Good for web servers            | Good for DB, Kafka, ZooKeeper, etc. |

For example:

```text
Deployment

web-7f6c8d9f5-x7abc
web-7f6c8d9f5-k2pqr
web-7f6c8d9f5-m9xyz
```

The pods are essentially interchangeable.

With StatefulSet:

```text
mysql-0
mysql-1
mysql-2
```

`mysql-0` is always `mysql-0`.

If `mysql-1` dies, Kubernetes creates another pod with the identity:

```text
mysql-1
```

---

# 2. Real-time example

Imagine an e-commerce application:

```text
                    Users
                      |
                      v
                Load Balancer
                      |
                      v
              Frontend / API
                      |
                      v
             MySQL Database
              /      |      \
             /       |       \
        mysql-0   mysql-1   mysql-2
           |         |         |
         PVC       PVC       PVC
           |         |         |
         Disk      Disk      Disk
```

We want:

```text
mysql-0 → its own disk
mysql-1 → its own disk
mysql-2 → its own disk
```

This is where StatefulSet becomes useful.

---

# 3. StatefulSet architecture

The important components are:

```text
                    StatefulSet
                         |
             +-----------+-----------+
             |           |           |
             v           v           v
          mysql-0     mysql-1     mysql-2
             |           |           |
            PVC         PVC         PVC
             |           |           |
            PV          PV          PV
             |           |           |
          Storage     Storage     Storage
```

And for networking:

```text
             Headless Service
                    |
        +-----------+-----------+
        |           |           |
        v           v           v
     mysql-0     mysql-1     mysql-2
```

---

# 4. Why do we need a Headless Service?

Normally a Kubernetes Service gives you a virtual IP:

```text
mysql-service
      |
      +---- mysql-0
      +---- mysql-1
      +---- mysql-2
```

But StatefulSets often need to reach **individual pods**.

A headless service:

```yaml
clusterIP: None
```

allows Kubernetes DNS to provide individual identities.

For example:

```text
mysql-0.mysql.default.svc.cluster.local
mysql-1.mysql.default.svc.cluster.local
mysql-2.mysql.default.svc.cluster.local
```

This is extremely important for systems such as:

* MySQL
* PostgreSQL
* MongoDB
* Kafka
* Elasticsearch
* ZooKeeper

---

# 5. Complete StatefulSet example

Let's create a simple MySQL StatefulSet.

## Step 1 — Headless Service

Create:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
    - port: 3306
      targetPort: 3306
```

Save as:

```text
mysql-service.yaml
```

Apply:

```bash
kubectl apply -f mysql-service.yaml
```

Check:

```bash
kubectl get svc
```

You should see:

```text
NAME    TYPE        CLUSTER-IP   PORT(S)
mysql   ClusterIP   None         3306/TCP
```

Notice:

```text
CLUSTER-IP = None
```

That's what makes it a **headless service**.

---

# 6. Create the StatefulSet

Now create:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:

  serviceName: mysql

  replicas: 3

  selector:
    matchLabels:
      app: mysql

  template:
    metadata:
      labels:
        app: mysql

    spec:
      containers:
        - name: mysql
          image: mysql:8.0

          ports:
            - containerPort: 3306
              name: mysql

          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "MyPassword123"

          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql

  volumeClaimTemplates:

    - metadata:
        name: mysql-data

      spec:
        accessModes:
          - ReadWriteOnce

        resources:
          requests:
            storage: 10Gi
```

Save:

```text
mysql-statefulset.yaml
```

Apply:

```bash
kubectl apply -f mysql-statefulset.yaml
```

---

# 7. What Kubernetes creates

Run:

```bash
kubectl get pods
```

You should get:

```text
NAME      READY   STATUS
mysql-0   1/1     Running
mysql-1   1/1     Running
mysql-2   1/1     Running
```

Notice the names:

```text
mysql-0
mysql-1
mysql-2
```

This is one of the biggest differences between Deployment and StatefulSet.

---

# 8. PVCs are automatically created

Run:

```bash
kubectl get pvc
```

You'll see something like:

```text
NAME                 STATUS   VOLUME
mysql-data-mysql-0   Bound    pvc-xxxxx
mysql-data-mysql-1   Bound    pvc-yyyyy
mysql-data-mysql-2   Bound    pvc-zzzzz
```

This comes from:

```yaml
volumeClaimTemplates:
```

This is a very important StatefulSet feature.

Kubernetes effectively creates:

```text
mysql-0
  |
  +---- mysql-data-mysql-0

mysql-1
  |
  +---- mysql-data-mysql-1

mysql-2
  |
  +---- mysql-data-mysql-2
```

Each pod gets its **own PVC**.

---

# 9. Why volumeClaimTemplates is important

Suppose:

```text
mysql-0
```

writes:

```text
Customer A
Customer B
Customer C
```

to:

```text
/var/lib/mysql
```

Those files are stored in:

```text
mysql-data-mysql-0
```

Now imagine the pod crashes.

Kubernetes recreates:

```text
mysql-0
```

and attaches the same PVC:

```text
mysql-data-mysql-0
```

Therefore:

```text
Old mysql-0
      |
      X
    crashed

      ↓

New mysql-0
      |
      |
      v
mysql-data-mysql-0
      |
      v
Existing database data
```

The pod disappeared.

**The identity and storage remain.**

---

# 10. Test persistence

Let's create some data.

First enter the pod:

```bash
kubectl exec -it mysql-0 -- bash
```

Connect to MySQL:

```bash
mysql -uroot -p
```

Password:

```text
MyPassword123
```

Create a database:

```sql
CREATE DATABASE ecommerce;
```

Create a table:

```sql
USE ecommerce;

CREATE TABLE customers (
    id INT PRIMARY KEY,
    name VARCHAR(100)
);
```

Insert data:

```sql
INSERT INTO customers VALUES
(1,'Vaithee'),
(2,'Raj'),
(3,'Kumar');
```

Check:

```sql
SELECT * FROM customers;
```

You should see:

```text
1 Vaithee
2 Raj
3 Kumar
```

Exit:

```bash
exit
```

---

# 11. Delete the pod

Now do:

```bash
kubectl delete pod mysql-0
```

Watch:

```bash
kubectl get pods -w
```

You'll see something like:

```text
mysql-0   Terminating
mysql-0   Pending
mysql-0   ContainerCreating
mysql-0   Running
```

Notice something important.

The new pod is again:

```text
mysql-0
```

Not:

```text
mysql-abc123
```

The PVC is still:

```text
mysql-data-mysql-0
```

---

# 12. Verify the data

Enter again:

```bash
kubectl exec -it mysql-0 -- bash
```

Then:

```bash
mysql -uroot -p
```

Run:

```sql
USE ecommerce;

SELECT * FROM customers;
```

Your data should still exist.

This demonstrates the fundamental StatefulSet concept:

```text
Pod identity + persistent storage
```

---

# 13. StatefulSet ordered creation

There is another interesting behavior.

When you create:

```yaml
replicas: 3
```

Kubernetes normally creates:

```text
mysql-0
   ↓
mysql-1
   ↓
mysql-2
```

Rather than creating all three simultaneously.

You can observe:

```bash
kubectl get pods -w
```

You'll see:

```text
mysql-0   Running
mysql-1   Running
mysql-2   Running
```

This ordered behavior is useful for distributed systems.

---

# 14. Scaling StatefulSet

Currently:

```text
replicas: 3
```

Scale to 5:

```bash
kubectl scale statefulset mysql --replicas=5
```

Check:

```bash
kubectl get pods
```

You'll get:

```text
mysql-0
mysql-1
mysql-2
mysql-3
mysql-4
```

And PVCs:

```text
mysql-data-mysql-0
mysql-data-mysql-1
mysql-data-mysql-2
mysql-data-mysql-3
mysql-data-mysql-4
```

Each instance gets its own storage.

---

# 15. Scale down

Now:

```bash
kubectl scale statefulset mysql --replicas=3
```

Pods:

```text
mysql-0
mysql-1
mysql-2
```

But something important happens.

The PVCs generally remain:

```text
mysql-data-mysql-0
mysql-data-mysql-1
mysql-data-mysql-2
mysql-data-mysql-3
mysql-data-mysql-4
```

Kubernetes does **not automatically delete the PVCs just because you scaled down**.

This protects your data.

---

# 16. StatefulSet DNS

This is one of the most important concepts to understand.

Because we created:

```yaml
serviceName: mysql
```

and:

```yaml
clusterIP: None
```

each pod gets a predictable DNS name.

For example:

```text
mysql-0.mysql.default.svc.cluster.local
```

```text
mysql-1.mysql.default.svc.cluster.local
```

```text
mysql-2.mysql.default.svc.cluster.local
```

The format is:

```text
<pod-name>.<service-name>.<namespace>.svc.cluster.local
```

So:

```text
mysql-0.mysql.default.svc.cluster.local
```

means:

```text
Pod:
mysql-0

Service:
mysql

Namespace:
default
```

---

# 17. Test DNS

Create a temporary pod:

```bash
kubectl run dns-test \
  --image=busybox:1.36 \
  -it --rm \
  --restart=Never -- sh
```

Inside:

```bash
nslookup mysql-0.mysql
```

You should get the IP of:

```text
mysql-0
```

Try:

```bash
nslookup mysql-1.mysql
```

and:

```bash
nslookup mysql-2.mysql
```

This demonstrates stable network identity.

---

# 18. Deployment vs StatefulSet — real example

Imagine you have a web application:

```text
Frontend
   |
   +--- web-abc
   +--- web-def
   +--- web-xyz
```

It doesn't matter which pod receives the request.

Therefore:

```text
Deployment
```

is appropriate.

But consider Kafka:

```text
Kafka-0
Kafka-1
Kafka-2
```

Each broker has a unique identity.

Or database:

```text
mysql-0
mysql-1
mysql-2
```

Or ZooKeeper:

```text
zk-0
zk-1
zk-2
```

These applications often need stable identity.

Therefore:

```text
StatefulSet
```

is appropriate.

---

# 19. Real production architecture

A more realistic architecture looks like this:

```text
                         Internet
                            |
                            v
                     Load Balancer
                            |
                            v
                    Application Pods
                   /       |       \
                  /        |        \
               API-1     API-2     API-3
                           |
                           v
                     Database Service
                           |
              +------------+------------+
              |            |            |
              v            v            v
           mysql-0      mysql-1      mysql-2
              |            |            |
              v            v            v
           PVC-0        PVC-1        PVC-2
              |            |            |
              v            v            v
          Storage       Storage       Storage
```

However, **don't assume that simply putting MySQL replicas in a StatefulSet creates a functioning MySQL cluster**.

This is an important production point.

Our example gives you:

* Stable identity
* Persistent storage
* Stable DNS
* Ordered lifecycle

But it does **not automatically configure MySQL replication/failover**.

For production database clustering, you would normally use something such as a MySQL operator or a database-specific clustering solution.

---

# 20. StatefulSet update strategy

StatefulSet also supports update strategies.

For example:

```yaml
updateStrategy:
  type: RollingUpdate
```

During an update, Kubernetes can update the StatefulSet pods while preserving their identities.

For more advanced workloads, you can also use:

```yaml
updateStrategy:
  type: OnDelete
```

With `OnDelete`, Kubernetes doesn't automatically replace existing pods when the StatefulSet template changes. You delete the pod yourself to trigger recreation.

---

# 21. PodManagementPolicy

By default:

```yaml
podManagementPolicy: OrderedReady
```

means pods are created/deleted in order.

For example:

```text
mysql-0
   ↓
mysql-1
   ↓
mysql-2
```

You can alternatively use:

```yaml
podManagementPolicy: Parallel
```

Then Kubernetes can create/delete pods without waiting for the previous ordinal pod to become ready.

This is useful for some distributed applications where strict ordering isn't required.

---

# 22. Complete production-style YAML

Here's a cleaner example you can use for your Kubernetes training lab:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
    - name: mysql
      port: 3306
      targetPort: 3306

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:

  serviceName: mysql

  replicas: 3

  podManagementPolicy: OrderedReady

  updateStrategy:
    type: RollingUpdate

  selector:
    matchLabels:
      app: mysql

  template:

    metadata:
      labels:
        app: mysql

    spec:

      containers:

        - name: mysql

          image: mysql:8.0

          ports:
            - name: mysql
              containerPort: 3306

          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "MyPassword123"

          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql

  volumeClaimTemplates:

    - metadata:
        name: mysql-data

      spec:

        accessModes:
          - ReadWriteOnce

        resources:
          requests:
            storage: 10Gi
```

---

# 23. Commands for your lab

### Create

```bash
kubectl apply -f mysql.yaml
```

### Check StatefulSet

```bash
kubectl get statefulset
```

### Check pods

```bash
kubectl get pods -o wide
```

### Check PVCs

```bash
kubectl get pvc
```

### Check PVs

```bash
kubectl get pv
```

### Check service

```bash
kubectl get svc
```

### Describe StatefulSet

```bash
kubectl describe statefulset mysql
```

### Describe pod

```bash
kubectl describe pod mysql-0
```

### Check logs

```bash
kubectl logs mysql-0
```

### Enter pod

```bash
kubectl exec -it mysql-0 -- bash
```

### Scale

```bash
kubectl scale statefulset mysql --replicas=5
```

### Delete one pod

```bash
kubectl delete pod mysql-1
```

### Watch recovery

```bash
kubectl get pods -w
```

---

# 24. The key concept to remember

Think about StatefulSet as:

```text
                 STATEFULSET
                      |
       +--------------+--------------+
       |              |              |
       v              v              v
    mysql-0        mysql-1        mysql-2
       |              |              |
       v              v              v
     PVC-0          PVC-1          PVC-2
       |              |              |
       v              v              v
    Storage        Storage        Storage
```

The **three most important StatefulSet concepts** are:

### 1. Stable identity

```text
mysql-0
mysql-1
mysql-2
```

### 2. Stable storage

```text
mysql-0 → PVC-0
mysql-1 → PVC-1
mysql-2 → PVC-2
```

### 3. Stable network identity

```text
mysql-0.mysql
mysql-1.mysql
mysql-2.mysql
```

So the mental model is:

> **Deployment = interchangeable pods.**
> **StatefulSet = identifiable pods with persistent identity/storage.**

