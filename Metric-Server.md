kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/high-availability-1.21+.yaml

---



### Resolution

#### 1. **Configure Metrics Server to Skip TLS Verification**

You can instruct the Metrics Server to bypass certificate validation by adding the `--kubelet-insecure-tls` argument to its Deployment. While this is less secure, it resolves the issue quickly.

Update the Metrics Server Deployment:
```bash
kubectl edit deployment metrics-server -n kube-system
```

Modify the `args` section to include:
```yaml
args:
  - --kubelet-insecure-tls
```

Save and exit. Restart the Metrics Server by deleting the pod:
```bash
kubectl delete pod -l k8s-app=metrics-server -n kube-system
```

#### 2. **Use Preferred Address Types**
Ensure the Metrics Server is using the correct address type for kubelet communication. Add the `--kubelet-preferred-address-types` argument to the Deployment:
```yaml
args:
  - --kubelet-insecure-tls
  - --kubelet-preferred-address-types=InternalIP,Hostname,ExternalIP
```

This ensures that the Metrics Server uses the `InternalIP` or `Hostname` instead of other address types.

#### 3. **Regenerate Certificates with IP SANs (More Secure)**
If you prefer a secure solution, you need to regenerate the kubelet certificates with the IP SANs included. This requires modifying the kubelet configuration and re-generating certificates.

- **Add IP SANs**: Ensure the kubelet's `--node-ip` and `--address` flags are set to the node's IP. Update the kubelet configuration file (e.g., `/var/lib/kubelet/config.yaml`) or the kubelet service definition.
- **Restart Kubelet**: Regenerate and apply the certificates, then restart the kubelet service.

#### 4. **Validate Configuration**
After making changes, verify that the Metrics Server is healthy:
```bash
kubectl get pods -n kube-system
```

Check if the readiness probe passes:
```bash
kubectl describe pod metrics-server-54bf7cdd6-h5c27 -n kube-system
```

#### 5. **Test Metrics**
Finally, test whether the Metrics Server is functioning:
```bash
kubectl top nodes
kubectl top pods
```
