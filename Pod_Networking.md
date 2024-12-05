Troubleshooting pods and network issues in Kubernetes involves systematically diagnosing and resolving problems that can impact communication between pods, services, and external systems. 
---

### **1. Understand the Issue**
- **Symptoms**: Identify if the issue involves pod-to-pod, pod-to-service, service-to-external, or ingress/egress traffic.
- **Logs**: Check logs from affected pods, services, or controllers (e.g., kube-proxy, CNI plugin).

---

### **2. Verify Pod Networking**
- **Pod Status**: Use `kubectl get pods -o wide` to inspect pod IPs and statuses.
- **Ping Pod-to-Pod**: Use `ping` or `curl` to test connectivity between pods.
  ```bash
  kubectl exec <pod-name> -- ping <target-pod-IP>
  ```
- **DNS Resolution**: Verify DNS functionality with `nslookup` or `dig`.
  ```bash
  kubectl exec <pod-name> -- nslookup <service-name>
  ```
  If DNS fails, check the `kube-dns` or `CoreDNS` pods.

---

### **3. Check Service Configuration**
- **Service Details**: Verify the service type, ClusterIP, and port mapping.
  ```bash
  kubectl describe svc <service-name>
  ```
- **Endpoint Validation**: Confirm the service endpoints are correctly populated.
  ```bash
  kubectl get endpoints <service-name>
  ```
  If endpoints are missing, check pod readiness probes.

---

### **4. Inspect Network Policies**
- **NetworkPolicy Rules**: Ensure policies allow required ingress and egress traffic.
  ```bash
  kubectl get networkpolicy -o yaml
  ```
- Temporarily disable policies to isolate issues.

---

### **5. Test Node-Level Connectivity**
- **CNI Plugin Logs**: Review logs for the CNI plugin (e.g., Calico, Flannel, Weave).
- **Node IP Tables**: Inspect and troubleshoot IP tables rules.
  ```bash
  iptables -L -n -v
  ```
- **Node-to-Pod Communication**: Test connectivity between nodes and pods.

---

### **6. Investigate Load Balancing**
- **kube-proxy Logs**: Check for errors in kube-proxy logs on nodes.
- **Service IP Routing**: Validate routing rules for ClusterIP, NodePort, or LoadBalancer services.

---

### **7. Egress Issues**
- **Internet Access**: Ensure pods can access the internet if required.
- **NAT Rules**: Check if NAT rules are correctly applied on the node.

---

### **8. Ingress Issues**
- **Ingress Resources**: Verify ingress rules and backend service configurations.
  ```bash
  kubectl describe ingress <ingress-name>
  ```
- **Ingress Controller Logs**: Review ingress controller logs (e.g., NGINX, Traefik).

---

### **9. Common Commands for Troubleshooting**
- **Debug Pods**: Deploy a temporary debug pod for troubleshooting:
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: debug
  spec:
    containers:
    - name: debug
      image: nicolaka/netshoot
      command: ["sleep", "3600"]
  ```
- **TCP Dump**: Capture network traffic for in-depth analysis:
  ```bash
  tcpdump -i <interface> port <target-port>
  ```
- **Curl with Verbose Mode**: Test HTTP connections:
  ```bash
  curl -v http://<service-name>:<port>
  ```

---

### **10. Logs and Monitoring**
- **Logs**: Inspect `kubectl logs` for pods, kubelet, and kube-proxy.
- **Monitoring Tools**: Use monitoring tools like Prometheus, Grafana, or ELK Stack to gain insights into network metrics and errors.

---

