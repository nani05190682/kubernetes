Here are the steps to achieve this in Kubernetes:

### Step 1: Create the `ClusterRole`
A `ClusterRole` allows you to define permissions for certain Kubernetes resources. In this case, you'll grant create permissions for `Deployments`, `StatefulSets`, and `DaemonSets`.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: deployment-clusterone
rules:
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets", "daemonsets"]
    verbs: ["create"]
```

Save this file as `clusterrole.yaml` and apply it using:

```bash
kubectl apply -f clusterrole.yaml
```

---

### Step 2: Create the `ServiceAccount`
Create a new `ServiceAccount` named `cicd-token` in the `app-team1` namespace.

```bash
kubectl create serviceaccount cicd-token -n app-team1
```

---

### Step 3: Bind the `ClusterRole` to the `ServiceAccount` within the namespace
You will use a `RoleBinding` to bind the `ClusterRole` to the `ServiceAccount` in the namespace `app-team1`.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deployment-binding
  namespace: app-team1
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: deployment-clusterone
subjects:
  - kind: ServiceAccount
    name: cicd-token
    namespace: app-team1
```

Save this file as `rolebinding.yaml` and apply it using:

```bash
kubectl apply -f rolebinding.yaml
```

---

### Verification
You can verify that the `ServiceAccount` has the required permissions by checking the bindings:

```bash
kubectl get rolebindings -n app-team1
kubectl describe clusterrole deployment-clusterone
kubectl describe serviceaccount cicd-token -n app-team1
```
