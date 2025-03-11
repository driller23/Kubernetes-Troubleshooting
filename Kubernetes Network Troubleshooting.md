# Kubernetes Network Troubleshooting

This guide provides step-by-step procedures for troubleshooting common Kubernetes networking issues. Use this document as a reference when diagnosing connectivity problems between pods, services, and external resources.

## Table of Contents

1. [Diagnostic Tools](#diagnostic-tools)
2. [Common Network Issues](#common-network-issues)
3. [DNS Troubleshooting](#dns-troubleshooting)
4. [Service Connectivity](#service-connectivity)
5. [Ingress Troubleshooting](#ingress-troubleshooting)
6. [NetworkPolicy Issues](#networkpolicy-issues)
7. [CNI Troubleshooting](#cni-troubleshooting)
8. [Advanced Network Debugging](#advanced-network-debugging)

## Diagnostic Tools

### Network Debugging Container

Deploy this debugging container to troubleshoot network issues from within the cluster:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: netshoot
  namespace: default
spec:
  containers:
  - name: netshoot
    image: nicolaka/netshoot
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
  restartPolicy: Always
```

Save this as `netshoot.yaml` and apply with:

```bash
kubectl apply -f netshoot.yaml
```

### Network Diagnostic Commands

| Issue | Command |
|-------|---------|
| DNS Resolution | `kubectl exec -it netshoot -- nslookup kubernetes.default` |
| Service Connectivity | `kubectl exec -it netshoot -- curl -v <service-name>.<namespace>.svc.cluster.local:<port>` |
| Pod Connectivity | `kubectl exec -it netshoot -- ping <pod-ip>` |
| Port Verification | `kubectl exec -it netshoot -- nc -zv <service-name> <port>` |
| Trace Route | `kubectl exec -it netshoot -- traceroute <destination>` |
| Network Policies | `kubectl get networkpolicies -A` |

## Common Network Issues

### Pod-to-Pod Communication

**Symptoms:**
- Pods unable to communicate with each other
- Connection timeouts between pods

**Troubleshooting Steps:**

1. Verify pod IPs are in the same subnet:
```bash
kubectl get pods -o wide
```

2. Check if NetworkPolicies are blocking traffic:
```bash
kubectl get networkpolicies -A
```

3. Verify CNI plugin is functioning:
```bash
kubectl logs -n kube-system -l k8s-app=kube-proxy
```

4. Test connectivity from a debugging pod:
```bash
kubectl exec -it netshoot -- ping <target-pod-ip>
```

### Service Discovery Issues

**Symptoms:**
- Unable to connect to services by name
- DNS resolution failures

**Troubleshooting Steps:**

1. Check if CoreDNS/kube-dns is running:
```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

2. Check for DNS resolution:
```bash
kubectl exec -it netshoot -- nslookup kubernetes.default
kubectl exec -it netshoot -- nslookup <service-name>.<namespace>.svc.cluster.local
```

3. Verify service exists and has correct selector:
```bash
kubectl describe service <service-name> -n <namespace>
```

4. Check if endpoints exist:
```bash
kubectl get endpoints <service-name> -n <namespace>
```

## DNS Troubleshooting

### CoreDNS Debugging

1. Check CoreDNS pods are running:
```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

2. Check CoreDNS logs:
```bash
kubectl logs -n kube-system -l k8s-app=kube-dns
```

3. Verify CoreDNS configuration:
```bash
kubectl get configmap coredns -n kube-system -o yaml
```

4. Test DNS resolution from pod:
```bash
kubectl exec -it netshoot -- cat /etc/resolv.conf
kubectl exec -it netshoot -- dig kubernetes.default.svc.cluster.local
```

### Common DNS Issues

| Issue | Symptoms | Solution |
|-------|----------|----------|
| CoreDNS crashes | DNS resolution fails completely | Check memory limits and increase if needed |
| Incorrect DNS config | Wrong search domains | Check `/etc/resolv.conf` in pods |
| CNI network issues | Intermittent DNS failures | Restart CoreDNS and check CNI plugin status |
| Kube-proxy issues | Services not resolving | Check kube-proxy logs for errors |

## Service Connectivity

### Service Debugging

1. Verify service definition:
```bash
kubectl get service <name> -o yaml
```

2. Check if endpoints exist:
```bash
kubectl get endpoints <name>
```

3. Verify pods are matched by selector:
```bash
kubectl get pods -l <key>=<value>
```

4. Check if pods are running and ready:
```bash
kubectl get pods -l <key>=<value> -o wide
```

5. Test direct pod connectivity:
```bash
kubectl exec -it netshoot -- curl <pod-ip>:<port>
```

6. Test service connectivity:
```bash
kubectl exec -it netshoot -- curl <service-name>.<namespace>.svc.cluster.local:<port>
```

### Service Types and Troubleshooting

| Service Type | Common Issues | Verification Commands |
|--------------|---------------|------------------------|
| ClusterIP | Not accessible externally | `kubectl exec -it netshoot -- curl <service-name>:<port>` |
| NodePort | Node firewall blocking | `curl <node-ip>:<node-port>` |
| LoadBalancer | Provider issues | `kubectl describe service <name>` to check external IP |
| ExternalName | DNS resolution | `kubectl exec -it netshoot -- dig <external-name>` |

## Ingress Troubleshooting

### Ingress Controller Verification

1. Check if ingress controller is running:
```bash
kubectl get pods -n <ingress-namespace> -l app=ingress-nginx
```

2. View ingress controller logs:
```bash
kubectl logs -n <ingress-namespace> -l app=ingress-nginx
```

3. Verify ingress resource:
```bash
kubectl describe ingress <name>
```

### Common Ingress Issues

| Issue | Troubleshooting |
|-------|----------------|
| TLS certificate problems | `kubectl describe secret <tls-secret>` |
| Path routing issues | Check ingress rules with `kubectl describe ingress` |
| Backend service not found | Verify service exists and endpoints are available |
| Configuration errors | Check ingress controller logs |

## NetworkPolicy Issues

### NetworkPolicy Debugging

1. List all NetworkPolicies:
```bash
kubectl get networkpolicies --all-namespaces
```

2. Describe specific NetworkPolicy:
```bash
kubectl describe networkpolicy <name> -n <namespace>
```

3. Verify impact on pod connectivity:
```bash
# Create a test pod in the affected namespace
kubectl run test-pod -n <namespace> --image=busybox -- sleep 3600

# Try connecting to the service
kubectl exec -it test-pod -n <namespace> -- wget -O- <service>:<port>
```

### NetworkPolicy Troubleshooting Script

Save this as `check-network-policies.sh`:

```bash
#!/bin/bash
# Identifies NetworkPolicies affecting a specific pod

POD_NAME=$1
NAMESPACE=${2:-default}

if [ -z "$POD_NAME" ]; then
  echo "Usage: $0 <pod-name> [namespace]"
  exit 1
fi

# Get pod labels
echo "Pod labels:"
kubectl get pod $POD_NAME -n $NAMESPACE -o json | jq '.metadata.labels'

# Find NetworkPolicies in the same namespace
echo -e "\nNetworkPolicies in the same namespace:"
kubectl get networkpolicies -n $NAMESPACE -o json | jq '.items[] | {name: .metadata.name, podSelector: .spec.podSelector}'

# Check if any NetworkPolicy matches this pod
echo -e "\nAnalyzing which NetworkPolicies affect this pod..."
POD_LABELS=$(kubectl get pod $POD_NAME -n $NAMESPACE -o json | jq -c '.metadata.labels')

kubectl get networkpolicies -n $NAMESPACE -o json | jq --argjson podLabels "$POD_LABELS" '.items[] | select(.spec.podSelector.matchLabels | to_entries | all(. as $item | $podLabels | to_entries | any(.key == $item.key and .value == $item.value))) | {name: .metadata.name, affects: "This pod", policy: .spec}'
```

## CNI Troubleshooting

### CNI Plugin Verification

1. Check which CNI plugin is installed:
```bash
ls /etc/cni/net.d/
```

2. Verify CNI plugin pods are running:
```bash
# For Calico
kubectl get pods -n kube-system -l k8s-app=calico-node

# For Weave
kubectl get pods -n kube-system -l name=weave-net

# For Flannel
kubectl get pods -n kube-system -l app=flannel
```

3. Check CNI logs:
```bash
kubectl logs -n kube-system -l k8s-app=calico-node
```

### CNI-Specific Issues

| CNI | Common Issues | Troubleshooting |
|-----|---------------|-----------------|
| Calico | BGP peering issues | `calicoctl node status` |
|        | IP pool exhaustion | `calicoctl get ippool -o wide` |
| Weave | Peer connection issues | `kubectl exec -n kube-system weave-net-xxxx -c weave -- /home/weave/weave --local status` |
| Flannel | Backend configuration | Check `/etc/cni/net.d/10-flannel.conflist` |

## Advanced Network Debugging

### Packet Capture

Deploy a privileged pod for packet capturing:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: tcpdump
  namespace: default
spec:
  hostNetwork: true
  containers:
  - name: tcpdump
    image: nicolaka/netshoot
    command:
      - sleep
      - "3600"
    securityContext:
      privileged: true
    imagePullPolicy: IfNotPresent
  restartPolicy: Always
```

Capture packets:

```bash
# Capture traffic on a specific interface
kubectl exec -it tcpdump -- tcpdump -i eth0 -n host <ip-address>

# Capture service traffic
kubectl exec -it tcpdump -- tcpdump -i any -n port <service-port>
```

### DNS Traffic Analysis

```bash
# Capture DNS traffic
kubectl exec -it tcpdump -- tcpdump -i any -n port 53
```

### Connection Tracking

```bash
# Check connection tracking table
kubectl exec -it tcpdump -- conntrack -L

# Check NAT rules
kubectl exec -it tcpdump -- iptables -t nat -L -v
```

### Layer-by-Layer Verification

1. Physical/Node connectivity:
```bash
# Verify node connectivity
kubectl get nodes -o wide
ping <node-ip>
```

2. Pod networking:
```bash
# Get pod CIDR
kubectl get nodes -o jsonpath='{.items[*].spec.podCIDR}'

# Verify pod can reach outside
kubectl exec -it netshoot -- ping 8.8.8.8
```

3. Service networking:
```bash
# Get service CIDR
kubectl cluster-info dump | grep -m 1 service-cluster-ip-range

# Check if kube-proxy is creating iptables rules
kubectl exec -it tcpdump -- iptables-save | grep <service-ip>
```

4. External connectivity:
```bash
# Test outbound connectivity
kubectl exec -it netshoot -- curl https://www.google.com

# Test inbound (for NodePort/LoadBalancer)
curl <node-ip>:<node-port>
```

---

## Additional Resources

- [Kubernetes Networking Documentation](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
- [Debugging DNS Resolution](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/)
- [Common Networking Issues Diagram](./diagrams/network-troubleshooting-flow.png)
