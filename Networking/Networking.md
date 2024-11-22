### **Lab Exercise: Kubernetes Networking**  

This lab exercise is designed to give hands-on experience with Kubernetes networking concepts, including **pod communication**, **services**, and **network policies**.

---

#### **Objective**
1. Understand pod-to-pod communication within the cluster.  
2. Set up and test different types of services: **ClusterIP**, **NodePort**, and **LoadBalancer**.  
3. Configure and enforce **network policies** to restrict or allow traffic.  

---

### **Prerequisites**
1. A running Kubernetes cluster (e.g., Minikube, EKS, GKE, or AKS).  
2. `kubectl` installed and configured to access the cluster.  
3. Basic knowledge of Kubernetes concepts.

---

### **Exercise Steps**

#### **Part 1: Pod-to-Pod Communication**
1. **Create Two Pods**
   - Deploy two pods (`pod1` and `pod2`) using the `busybox` image.
   ```bash
   kubectl run pod1 --image=busybox --command -- sleep 3600
   kubectl run pod2 --image=busybox --command -- sleep 3600
   ```

2. **Check Pod Details**
   - Get the IP addresses of both pods:
   ```bash
   kubectl get pods -o wide
   ```

3. **Test Communication**
   - Use `ping` from `pod1` to `pod2`.
   ```bash
   kubectl exec pod1 -- ping <pod2-ip>
   ```

---

#### **Part 2: ClusterIP Service**
1. **Deploy an Application**
   - Create a deployment for an `nginx` server.
   ```bash
   kubectl create deployment nginx --image=nginx
   ```

2. **Expose the Deployment**
   - Create a `ClusterIP` service to expose the deployment.
   ```bash
   kubectl expose deployment nginx --type=ClusterIP --port=80
   ```

3. **Access the Service**
   - Deploy a `curl` pod to test access to the service using its name:
   ```bash
   kubectl run curl-pod --image=radial/busyboxplus:curl --command -- sleep 3600
   kubectl exec curl-pod -- curl http://nginx
   ```

---

#### **Part 3: NodePort Service**
1. **Expose the Deployment**
   - Update the service to a `NodePort` type:
   ```bash
   kubectl delete service nginx
   kubectl expose deployment nginx --type=NodePort --port=80
   ```

2. **Get Service Details**
   - Note the assigned `NodePort` and access the service using `<NodeIP>:<NodePort>`.
   ```bash
   kubectl get services
   curl http://<NodeIP>:<NodePort>
   ```

---

#### **Part 4: LoadBalancer Service (Optional, if using a cloud environment)**  
1. **Expose the Deployment**
   - Create a `LoadBalancer` service:
   ```bash
   kubectl expose deployment nginx --type=LoadBalancer --port=80
   ```

2. **Access the Service**
   - Get the external IP and access the application:
   ```bash
   kubectl get services
   curl http://<External-IP>
   ```

---

#### **Part 5: Network Policies**
1. **Create a Namespace**
   - Create a namespace to isolate resources:
   ```bash
   kubectl create namespace netpol-demo
   ```

2. **Deploy Pods**
   - Deploy two pods: `frontend` and `backend` in the `netpol-demo` namespace.
   ```bash
   kubectl run frontend --image=busybox --namespace=netpol-demo --command -- sleep 3600
   kubectl run backend --image=busybox --namespace=netpol-demo --command -- sleep 3600
   ```

3. **Test Communication**
   - Verify that `frontend` can ping `backend`:
   ```bash
   kubectl exec -n netpol-demo frontend -- ping backend
   ```

4. **Apply a Network Policy**
   - Restrict traffic to `backend` so only `frontend` can communicate with it:
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: allow-frontend
     namespace: netpol-demo
   spec:
     podSelector:
       matchLabels:
         app: backend
     ingress:
     - from:
       - podSelector:
           matchLabels:
             app: frontend
   ```
   ```bash
   kubectl apply -f network-policy.yaml
   ```

5. **Verify Policy**
   - Test communication:
     - From `frontend`: **Should Succeed**
     ```bash
     kubectl exec -n netpol-demo frontend -- ping backend
     ```
     - From another pod: **Should Fail**
     ```bash
     kubectl run random --image=busybox --namespace=netpol-demo --command -- sleep 3600
     kubectl exec -n netpol-demo random -- ping backend
     ```

---

### **Deliverables**
- Screenshots or logs of the following:
  1. Pod-to-pod communication test (Part 1).  
  2. Successful access to services using `ClusterIP`, `NodePort`, and `LoadBalancer` (Parts 2, 3, and 4).  
  3. Network policy enforcement (Part 5).

---

### **Outcome**
By completing this lab, you will:
- Understand Kubernetes networking basics.  
- Configure services to expose applications.  
- Apply and test network policies for access control.  
