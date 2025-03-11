# Pod Troubleshooting Guide

This module focuses on diagnosing and resolving common pod-related issues in Kubernetes.

## Table of Contents

1. [Pod Lifecycle States](#pod-lifecycle-states)
2. [Common Pod Issues](#common-pod-issues)
3. [Debugging Strategies](#debugging-strategies)
4. [Scripts and Commands](#scripts-and-commands)
5. [Decision Trees](#decision-trees)
6. [Case Studies](#case-studies)

## Pod Lifecycle States

Understanding the pod lifecycle is crucial for effective troubleshooting:

| Phase | Description | Potential Issues |
|-------|-------------|------------------|
| Pending | Pod accepted but containers not created | Resource constraints, scheduling issues |
| Running | Pod bound to node, all containers created | Application errors, resource utilization |
| Succeeded | All containers terminated successfully | None - expected completion |
| Failed | All containers terminated, at least one with failure | Crash, application error, config issues |
| Unknown | Pod state cannot be obtained | Node issues, communication problems |

### Pod Conditions

Pay attention to these pod conditions:

- **PodScheduled**: Pod has been scheduled to a node
- **ContainersReady**: All containers in the pod are ready
- **Initialized**: All init containers completed successfully
- **Ready**: Pod is able to serve requests

## Common Pod Issues

### 1. Pod Stuck in Pending

**Symptoms**:
- Pod remains in `Pending` state
- Events show scheduling issues

**Troubleshooting**:

```bash
# Check pod details
kubectl describe pod <pod-name>

# Look for node resource availability
kubectl describe nodes

# Check if pod requests are too high
kubectl get pod <pod-name> -o json | jq '.spec.containers[].resources'
```

**Common Causes**:
- Insufficient resources (CPU, memory)
- Node selector/affinity constraints not met
- PersistentVolumeClaim not bound
- Taints preventing scheduling

### 2. CrashLoopBackOff

**Symptoms**:
- Pod continuously restarting
- Status shows `CrashLoopBackOff`

**Troubleshooting**:

```bash
# Check logs
kubectl logs <pod-name> --previous

# Check pod details
kubectl describe pod <pod-name>

# Check resource usage when running
kubectl top pod <pod-name>
```

**Common Causes**:
- Application errors
- Configuration issues
- Resource constraints (OOMKilled)
- Liveness probe failures

### 3. ImagePullBackOff

**Symptoms**:
- Pod cannot start containers
- Status shows `ImagePullBackOff` or `ErrImagePull`

**Troubleshooting**:

```bash
# Check pod events
kubectl describe pod <pod-name>

# Verify image name and tag
kubectl get pod <pod-name> -o json | jq '.spec.containers[].image'

# Check if pull secrets exist
kubectl get pod <pod-name> -o json | jq '.spec.imagePullSecrets'
```

**Common Causes**:
- Image does not exist
- Wrong image tag
- Missing or invalid pull secrets
- Registry authentication issues
- Network connectivity to registry

### 4. Container Creating but Never Ready

**Symptoms**:
- Pod is Running but container is not ready
- Readiness probe fails

**Troubleshooting**:

```bash
# Check readiness probe configuration
kubectl get pod <pod-name> -o json | jq '.spec.containers[].readinessProbe'

# Check pod logs
kubectl logs <pod-name>

# Describe pod for events
kubectl describe pod <pod-name>
```

**Common Causes**:
- Application not listening on expected port
- Application startup time exceeds probe timeout
- Configuration errors
- Dependencies not available

## Debugging Strategies

### 1. Debug Container

Add a sidecar debugging container to the pod:

```yaml
# debug-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: debug-pod
spec:
  containers:
  - name: app
    image: original-app-image
  - name: debug
    image: busybox
    command:
    - sleep
    - "3600"
```

Apply with:
```bash
kubectl apply -f debug-pod.yaml
```

### 2. Ephemeral Debug Container (Kubernetes 1.18+)

```bash
kubectl debug -it <pod-name> --image=busybox --target=<container-name>
```

### 3. Replace Pod Image with Debug Image

```bash
kubectl debug <pod-name> -it --copy-to=<pod-name>-debug --container=<container-name> --image=busybox
```

### 4. Port-Forward for Application Testing

```bash
kubectl port-forward <pod-name> <local-port>:<pod-port>
```

Then test with:
```bash
curl localhost:<local-port>
```

## Scripts and Commands

### Pod State Analyzer

Save as `analyze-pod.sh`:

```bash
#!/bin/bash
# Analyzes pod state and provides troubleshooting recommendations

POD_NAME=$1
NAMESPACE=${2:-default}

if [ -z "$POD_NAME" ]; then
  echo "Usage: $0 <pod-name> [namespace]"
  exit 1
fi

echo "=== Pod Basic Info ==="
kubectl get pod $POD_NAME -n $NAMESPACE -o wide

echo -e "\n=== Pod Status Details ==="
kubectl get pod $POD_NAME -n $NAMESPACE -o json | jq '.status.phase, .status.conditions, .status.containerStatuses'

echo -e "\n=== Pod Events ==="
kubectl get events -n $NAMESPACE --field-selector involvedObject.name=$POD_NAME

echo -e "\n=== Pod Logs ==="
for container in $(kubectl get pod $POD_NAME -n $NAMESPACE -o jsonpath='{.spec.containers[*].name}'); do
  echo -e "\n--- Container: $container ---"
  kubectl logs $POD_NAME -n $NAMESPACE -c $container --tail=20 2>/dev/null || echo "No logs available"
  
  # If container is not running, try to get previous logs
  if [[ $(kubectl get pod $POD_NAME -n $NAMESPACE -o jsonpath="{.status.containerStatuses[?(@.name=='$container')].state.running}") == "" ]]; then
    echo -e "\n--- Previous container logs: $container ---"
    kubectl logs $POD_NAME -n $NAMESPACE -c $container --previous --tail=20 2>/dev/null || echo "No previous logs available" 
  fi
done

echo -e "\n=== Pod Resource Usage ==="
kubectl top pod $POD_NAME -n $NAMESPACE 2>/dev/null || echo "Metrics not available"

echo -e "\n=== Node Conditions ==="
NODE=$(kubectl get pod $POD_NAME -n $NAMESPACE -o jsonpath='{.spec.nodeName}')
kubectl describe node $NODE | grep -A 5 "Conditions:"

echo -e "\n=== Troubleshooting Recommendations ==="

# Check for common issues based on pod status
STATUS=$(kubectl get pod $POD_NAME -n $NAMESPACE -o jsonpath='{.status.phase}')
REASON=$(kubectl get pod $POD_NAME -n $NAMESPACE -o jsonpath='{.status.reason}')
CONTAINER_STATUSES=$(kubectl get pod $POD_NAME -n $NAMESPACE -o json | jq '.status.containerStatuses[]')

case $STATUS in
  "Pending")
    echo "Pod is pending. Possible issues:"
    echo "- Insufficient cluster resources"
    echo "- PersistentVolumeClaim not bound"
    echo "- Node selector/affinity constraints"
    echo "- Taints or tolerations preventing scheduling"
    ;;
  "Running")
    if echo "$CONTAINER_STATUSES" | grep -q "waiting"; then
      if echo "$CONTAINER_STATUSES" | grep -q "CrashLoopBackOff"; then
        echo "Container is crashing. Check for:"
        echo "- Application errors in the logs"
        echo "- Misconfiguration"
        echo "- Resource constraints"
        echo "- Command or args errors"
      elif echo "$CONTAINER_STATUSES" | grep -q "ImagePull"; then
        echo "Image pull issues. Check for:"
        echo "- Correct image name and tag"
        echo "- Valid image pull secrets"
        echo "- Registry availability"
        echo "- Network connectivity to registry"
      fi
    elif echo "$CONTAINER_STATUSES" | grep -q '"ready": false'; then
      echo "Container is running but not ready. Check for:"
      echo "- Readiness probe failures"
      echo "- Application not fully initialized"
      echo "- Dependencies not available"
    fi
    ;;
  "Failed")
    echo "Pod has failed. Check for:"
    echo "- Exit code in container status"
    echo "- Application crash in logs"
    echo "- OOMKilled events"
    ;;
  "Unknown")
    echo "Pod state is unknown. Check for:"
    echo "- Node problems"
    echo "- Kubelet not reporting status"
    echo "- Network issues between control plane and node"
    ;;
esac
```

### Pod Restart Counter

Save as `count-restarts.sh`:

```bash
#!/bin/bash
# Counts pod restarts across the cluster

NAMESPACE=${1:-"--all-namespaces"}
if [[ "$NAMESPACE" != "--all-namespaces" ]]; then
  NAMESPACE="-n $NAMESPACE"
fi

kubectl get pods $NAMESPACE -o json | \
jq '.items[] | select(.status.containerStatuses != null) | {name: .metadata.name, namespace: .metadata.namespace, restarts: [.status.containerStatuses[].restartCount] | add, containers: [.status.containerStatuses[] | {name: .name, restarts: .restartCount}] | sort_by(.restarts) | reverse}' | \
jq -s 'sort_by(.restarts) | reverse | .[:20]'
```

## Decision Trees

### Pod Startup Troubleshooting

```
Pod Not Running
├── Pod Status = Pending
│   ├── Check Events: "FailedScheduling"
│   │   ├── "Insufficient" in message → Resource constraints
│   │   ├── "Unschedulable" in message → Node selector/affinity issues
│   │   └── "PVC" in message → Volume issues
│   └── No Events → Check control plane (scheduler) health
├── Container Status = ImagePullBackOff
│   ├── "repository not found" in events → Check image name
│   ├── "unauthorized" in events → Check image pull secrets
│   └── Network connectivity issues → Check node connectivity to registry
├── Container Status = CrashLoopBackOff
│   ├── Check logs for application errors
│   ├── Check for OOMKilled → Increase memory limits
│   └── Check for configuration issues
└── Container Status = ContainerCreating
    ├── Check Events for volume mounting issues
    ├── Check kubelet logs on node
    └── Check CNI plugin status
```

### Pod Network Connectivity Troubleshooting

```
Pod Connectivity Issues
├── Pod to Pod Same Namespace
│   ├── Check if pods are Running
│   ├── Test connectivity with debug container
│   ├── Check NetworkPolicies
│   └── Check CNI plugin status
├── Pod to Service Same Namespace
│   ├── Check service endpoints exist
│   ├── Check service selector matches pods
│   ├── Check kube-proxy is running
│   └── Verify DNS resolution
├── Pod to External Services
│   ├── Check node connectivity to external services
│   ├── Check NetworkPolicies
│   ├── Check egress traffic rules
│   └── Verify external DNS resolution
└── Ingress to Pod
    ├── Check Ingress controller logs
    ├── Verify service and endpoints
    ├── Check NetworkPolicies for ingress
    └── Test node network access to pods
```

## Case Studies

### Case Study 1: OOMKilled Application

**Scenario**: A Java application pod continuously restarts with `CrashLoopBackOff`.

**Investigation**:

1. Checking pod description:
```bash
kubectl describe pod java-app
```
Events showed `OOMKilled`

2. Checking container resource limits:
```bash
kubectl get pod java-app -o json | jq '.spec.containers[].resources'
```
Memory limit set to 512Mi

3. Analyzing application memory usage before crash:
```bash
kubectl logs java-app --previous | grep "Memory usage"
```
Showed JVM using 480Mi, close to limit

**Resolution**:
- Increased memory limits to 1024Mi
- Added JVM memory settings: `-XX:MaxRAMPercentage=75.0`
-
