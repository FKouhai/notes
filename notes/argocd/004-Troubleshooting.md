---
id: 004-Troubleshooting
aliases:
  - argocd-troubleshoot
tags:
  - argocd
  - kubernetes
  - gitops
  - troubleshooting
---

# Argo CD Troubleshooting Guide

## 1. Argo CD Pod Issues

### Problem: Pods in CrashLoopBackOff or Pending state
```bash
# Check pod status and events
kubectl get pods -n argocd
kubectl describe pod <pod-name> -n argocd

# View logs for specific pods
kubectl logs <pod-name> -n argocd
kubectl logs <pod-name> -n argocd --previous

# Check resource requests and limits
kubectl get pod <pod-name> -n argocd -o jsonpath='{.spec.containers[*].resources}'
```
**Common Solutions:**
- Insufficient resources: Adjust resource requests/limits in deployment
- Image pull errors: Verify image registry access and credentials
- Config errors: Check ConfigMaps and Secrets for correct configuration
- Missing dependencies: Ensure required services (Redis, etc.) are running

### Problem: Argo CD Server unable to start
```bash
# Check for configuration issues
kubectl get cm argocd-cm -n argocd -o yaml
kubectl get secret argocd-secret -n argocd -o yaml

# Verify service account permissions
kubectl get clusterrolebinding argocd-application-controller -o yaml
kubectl get serviceaccount argocd-application-controller -n argocd -o yaml
```
**Common Solutions:**
- Check TLS certificate configuration if enabled
- Verify database (Redis) connectivity
- Ensure proper RBAC permissions are in place
- Review configuration syntax in ConfigMaps

## 2. Application Synchronization Issues

### Problem: Application stuck in "Out of Sync" state
```bash
# Check application status
argocd app get <app-name>

# View sync operation logs
argocd app logs <app-name>

# Diff desired vs live state
argocd app diff <app-name>

# Check application controller logs
kubectl logs -n argocd -l app.kubernetes.io/component=application-controller
```
**Common Solutions:**
- Manifest syntax errors: Validate YAML files in Git repository
- Resource conflicts: Check for existing resources with same names
- Permission issues: Ensure Argo CD has appropriate RBAC permissions
- Repository access problems: Verify Git repository credentials and connectivity

### Problem: Application sync fails with "manifest generation error"
```bash
# Check repository server logs
kubectl logs -n argocd -l app.kubernetes.io/component=repo-server

# Test manifest generation manually
argocd app manifest <app-name> --revision <git-revision>

# Check for missing dependencies in manifests
kubectl get all -n <app-namespace>
```
**Common Solutions:**
- Verify Helm chart/Kustomize structure is correct
- Check for missing required values files
- Ensure all referenced files exist in repository
- Review template syntax errors (especially for Helm/Kustomize)

### Problem: Application shows "Degraded" status
```bash
# Check application health status
argocd app get <app-name> --refresh

# List resources and their health
argocd app resources <app-name>

# Check pod logs for application
kubectl logs -n <namespace> -l app=<app-name>
```
**Common Solutions:**
- Check resource health status individually
- Review application pod logs for errors
- Verify dependencies between resources are satisfied
- Check for insufficient resource quotas/limits

## 3. Repository Connectivity Issues

### Problem: Unable to add Git repository
```bash
# Test repository connection
argocd repo get <repo-url>

# Check repository server logs
kubectl logs -n argocd -l app.kubernetes.io/component=repo-server

# Verify repository credentials
kubectl get secret <git-cred-secret> -n argocd -o yaml | grep -A 10 data
```
**Common Solutions:**
- Verify SSH key or password credentials are correct
- Check Git repository access permissions
- Ensure network connectivity to Git repository
- Verify branch/tag exists and is accessible
- Check for rate limiting on Git repository

### Problem: Repository syncs are slow or delayed
```bash
# Check repository server resource usage
kubectl top pod -n argocd -l app.kubernetes.io/component=repo-server

# Review sync intervals
argocd app get <app-name> -o yaml | grep -i sync

# Check for large repositories or complex manifests
du -sh .git/
```
**Common Solutions:**
- Increase repository server resources (CPU/memory)
- Reduce sync interval for frequently changing apps
- Consider using shallow clones or sparse checkouts
- Optimize repository structure to reduce complexity

## 4. Application Controller Issues

### Problem: Controller not reconciling applications
```bash
# Check controller status
kubectl get deployment argocd-application-controller -n argocd
kubectl logs -n argocd -l app.kubernetes.io/component=application-controller

# Check Redis connectivity
kubectl exec -n argocd -it <argocd-redis-pod> -- redis-cli ping

# View controller metrics
kubectl logs -n argocd -l app.kubernetes.io/component=application-controller --tail=100 | grep metrics
```
**Common Solutions:**
- Verify Redis is healthy and accessible
- Check if controller has sufficient resources
- Review application sync configuration
- Check for controller startup errors or crashes

### Problem: Application sync takes too long
```bash
# Check controller performance metrics
kubectl top pod -n argocd -l app.kubernetes.io/component=application-controller

# Monitor sync operations
argocd app list -o name | xargs -I {} argocd app get {} --refresh

# Check for complex applications or many resources
argocd app resources <app-name> | wc -l
```
**Common Solutions:**
- Scale up controller replicas
- Increase controller resource limits
- Optimize application manifests (reduce complexity)
- Check for resource contention on cluster

## 5. Argo CD Server Access Issues

### Problem: Unable to access UI or API
```bash
# Check service status
kubectl get svc argocd-server -n argocd
kubectl get endpoints argocd-server -n argocd

# Check server logs
kubectl logs -n argocd -l app.kubernetes.io/component=server

# Test server connectivity
kubectl port-forward svc/argocd-server -n argocd 8080:443
curl -k https://localhost:8080
```
**Common Solutions:**
- Verify service is properly configured and exposed
- Check Ingress or LoadBalancer configuration
- Ensure TLS certificates are valid if HTTPS enabled
- Verify network policies allow access
- Check server is running and healthy

### Problem: CLI authentication fails
```bash
# Test server connectivity
argocd version

# Re-login with debug mode
argocd login <server-url> --insecure --username <username> --password <password> --loglevel debug

# Check server configuration
argocd version --client=false
```
**Common Solutions:**
- Verify correct server URL and port
- Check credentials (username/password/token)
- Ensure network connectivity to Argo CD server
- Check if using correct authentication method
- Verify account permissions and roles

## 6. Cluster Connectivity Issues

### Problem: Cannot add cluster to Argo CD
```bash
# Check cluster connection status
argocd cluster list

# Test kubectl context
kubectl config use-context <target-cluster>
kubectl get nodes

# Check application controller permissions on target cluster
kubectl get serviceaccount argocd-manager -n kube-system
kubectl get clusterrole argocd-manager-role -o yaml
```
**Common Solutions:**
- Ensure correct kubeconfig context is selected
- Verify cluster admin permissions on target cluster
- Check network connectivity between Argo CD and target cluster
- Verify service account and RBAC setup on target cluster
- Check for conflicting cluster IDs or server URLs

### Problem: Applications cannot sync to remote cluster
```bash
# Check cluster connectivity from Argo CD
kubectl exec -n argocd <argocd-server-pod> -- ping <cluster-ip>

# Verify cluster credentials are correct
argocd cluster get <cluster-server-url>

# Check application sync logs
argocd app logs <app-name>
```
**Common Solutions:**
- Verify cluster credentials are properly configured
- Check network policies allow inter-cluster traffic
- Ensure proper RBAC permissions on target cluster
- Verify Argo CD can reach target cluster API server
- Check for authentication token expiration

## 7. Performance Issues

### Problem: Argo CD is slow or unresponsive
```bash
# Check resource usage across all components
kubectl top pods -n argocd

# Check for resource limits
kubectl get deploy -n argocd -o jsonpath='{.items[*].spec.template.spec.containers[*].resources}'

# Review Argo CD metrics
kubectl logs -n argocd -l app.kubernetes.io/component=application-controller --tail=100 | grep -E "metrics|cache"
```
**Common Solutions:**
- Increase resource limits for affected components
- Scale up replicas for server, controller, repo-server
- Check for resource contention on Kubernetes cluster
- Optimize sync intervals and resource monitoring
- Consider HA setup for production workloads

## 8. Common Mistakes and Solutions

### Mistake: Incorrect Git repository configuration
```bash
# Verify repository is accessible
argocd repo list

# Test repository connection
argocd repo get <repo-url>

# Check for correct path specification
argocd app create test-app \
  --repo <repo-url> \
  --path correct/path/to/manifests
```

### Mistake: Namespace not created before application sync
```bash
# Create namespace manually
kubectl create namespace <target-namespace>

# Or use sync hooks in manifests
apiVersion: v1
kind: Namespace
metadata:
  name: <target-namespace>
```

### Mistake: Missing RBAC permissions
```bash
# Create proper service account
kubectl create serviceaccount argocd-application-controller -n <target-namespace>

# Create cluster role with needed permissions
cat > role.yaml << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: argocd-role
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
EOF
kubectl apply -f role.yaml

# Bind role to service account
kubectl create clusterrolebinding argocd-binding \
  --clusterrole=argocd-role \
  --serviceaccount=<target-namespace>:argocd-application-controller
```

## Debugging Tips

### Enable debug logging
```bash
# Enable debug logs for server
kubectl set env deployment/argocd-server -n argocd ARGOCD_LOG_LEVEL=debug

# Enable debug logs for controller
kubectl set env deployment/argocd-application-controller -n argocd ARGOCD_LOG_LEVEL=debug

# View debug logs
kubectl logs -f -n argocd -l app.kubernetes.io/component=application-controller
```

### Use Argo CD built-in debugging
```bash
# Get detailed application information
argocd app get <app-name> --refresh

# View application operations
argocd app operation get <app-name>

# Cancel running operation
argocd app operation cancel <app-name>
```

### Export and inspect configuration
```bash
# Export application configuration
argocd app get <app-name> -o yaml > app-config.yaml

# Export repository configuration
argocd repo get <repo-url> -o yaml > repo-config.yaml

# Export cluster configuration
argocd cluster get <cluster-url> -o yaml > cluster-config.yaml
```

This troubleshooting guide covers common issues you may encounter when working with Argo CD, providing practical commands and solutions to help diagnose and resolve problems effectively.