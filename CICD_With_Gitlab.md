```markdown
## mykubernetes/.gitlab/agents/mycluster
# edit config.yaml

ci_access:
    projects:
        -id: vaithee3/mykubernetes/


### Create Variables

```yaml
KUBE_CONTEXT: vaithee3/demo-kube-deployment:vaithee
KUBE_NAMESPACE: gitlab-
```

### Go to Operate -> Kubernetes clusters -> Connect cluster

Then copy the agents installation steps and paste it on your k8s cluster:

Example:

```bash
helm repo add gitlab https://charts.gitlab.io
helm repo update
helm upgrade --install mycluster gitlab/gitlab-agent \
    --namespace gitlab-agent-mycluster \
    --create-namespace \
    --set image.tag=v17.1.0 \
    --set config.token=glagent-jhkfwekflkewlkewfkllknd\
    --set config.kasAddress=wss://kas.gitlab.com
```

### Create a pipeline file

.gitlab-ci.yml

```yaml
validate_k8s:
    image: 
        name: bitnami/kubectl:latest
        entrypoint: ['']

    script:
        - kubectl config use-context $KUBE_CONTEXT
        - kubectl get po -A
        - kubectl apply -f manifests/*.yaml
```
```
