# Helm Chart for [K8s-Log-View](https://github.com/dennissobczak/k8s-log-view)

## Installation
```
# Render the Helm Chart
helm template k8s-log-view --debug .
# OR
helm install k8s-log-view --dry-run --debug .

# Check the HTTPRoute if enabled
helm template k8s-log-view . --set httpRoute.enabled=true --show-only templates/httproute.yaml

# Install
helm install k8s-log-view .
# OR with dedicated Namespace
helm install k8s-log-view ./k8s-log-view --namespace k8s-log-view --create-namespace

# Check installation
helm list -n <target namespace>
```

Check out the latest available [k8s-log-view Docker container image](https://hub.docker.com/r/dennissobczak/k8s-log-view/tags)
