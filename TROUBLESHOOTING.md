# Troubleshooting Kubernetes LDAP Webhook Authentication

## Issue Identification and Resolution

### Problem Description
When attempting to authenticate using `kubectl` against a Minikube cluster with LDAP webhook authentication, the authentication was failing with the error:
```
error: You must be logged in to the server (the server has asked for the client to provide credentials)
```

### Troubleshooting Process

#### 1. Initial Setup Verification
First, I verified that all components were running correctly:
- LDAP container was running and accessible on port 1389
- Auth service was running on port 7443
- The auth service was responding correctly to test requests

Test command used:
```bash
curl -X POST http://localhost:7443/auth \
  -H "Content-Type: application/json" \
  -d '{"spec":{"token":"alice:alicepassword"}}'
```

Response confirmed the auth service was working:
```json
{
  "apiVersion": "authentication.k8s.io/v1",
  "kind": "TokenReview",
  "status": {
    "authenticated": true,
    "user": {
      "username": "alice",
      "uid": "alice",
      "groups": ["dev"]
    }
  }
}
```

#### 2. Checking Minikube Configuration
Verified that the webhook configuration was properly loaded:
```bash
minikube ssh 'sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep authentication-token-webhook'
```

This confirmed the flag was set:
```
- --authentication-token-webhook-config-file=/var/lib/minikube/certs/webhook.yaml
```

#### 3. Analyzing API Server Logs
The critical step was examining the kube-apiserver logs:
```bash
minikube logs --file=kube-apiserver
grep -i 'webhook\|token\|auth' kube-apiserver | grep -E '(error|Error|ERROR|fail|Fail)'
```

This revealed the root cause:
```
E0703 12:36:49.304215       1 webhook.go:154] Failed to make webhook authenticator request: converting (v1.TokenReview) to (v1beta1.TokenReview): unknown conversion
```

### Root Cause
The Kubernetes API server was attempting to use the deprecated `v1beta1.TokenReview` API version, but the webhook configuration was specifying `v1`. This version mismatch was causing the authentication to fail.

### Solution

#### Fix 1: Update webhook-local.yaml
Remove the `api-version` field from the webhook configuration:

**Before:**
```yaml
apiVersion: v1
kind: Config
clusters:
  - name: webhook-authn
    cluster:
      insecure-skip-tls-verify: true
      server: http://host.docker.internal:7443/auth
      api-version: authentication.k8s.io/v1  # Remove this line
```

**After:**
```yaml
apiVersion: v1
kind: Config
clusters:
  - name: webhook-authn
    cluster:
      insecure-skip-tls-verify: true
      server: http://host.docker.internal:7443/auth
```

#### Fix 2: Add webhook version flag to Minikube
When starting Minikube, explicitly specify the webhook version:

```bash
minikube start \
  --extra-config=apiserver.authentication-token-webhook-config-file=/var/lib/minikube/certs/webhook.yaml \
  --extra-config=apiserver.authentication-token-webhook-version=v1
```

### Verification
After applying these fixes, authentication works correctly:

1. Create test user credentials:
```bash
kubectl --kubeconfig admin.conf config set-credentials alice --token=alice:alicepassword
kubectl --kubeconfig admin.conf config set-context alice-context --cluster=minikube --user=alice
kubectl --kubeconfig admin.conf config use-context alice-context
```

2. Test authentication (will fail with authorization error, which is expected):
```bash
kubectl --kubeconfig admin.conf get pods --all-namespaces
# Error from server (Forbidden): pods is forbidden: User "alice" cannot list resource "pods" in API group "" at the cluster scope
```

3. Grant permissions and verify:
```bash
kubectl create clusterrolebinding alice-admin --clusterrole=view --user=alice
kubectl --kubeconfig admin.conf get pods --all-namespaces
# Successfully lists pods
```

### Key Takeaways

1. **API Version Compatibility**: When configuring webhook authentication, be aware of API version compatibility between Kubernetes and the webhook configuration.

2. **Log Analysis**: The kube-apiserver logs are crucial for debugging authentication issues. Look for conversion errors and webhook-related messages.

3. **Testing Approach**: Always test the webhook service directly first to isolate whether the issue is with the webhook service or the Kubernetes configuration.

4. **Version Specification**: The `--extra-config=apiserver.authentication-token-webhook-version=v1` flag is important for forcing the API server to use the correct TokenReview API version.

### Updated Setup Instructions

The main README has been updated with the correct configuration. The key changes are:
- Removed the `api-version` field from webhook configuration examples
- Added the `--extra-config=apiserver.authentication-token-webhook-version=v1` flag to the Minikube start command
- Updated the webhook-local.yaml file to work correctly with current Kubernetes versions
