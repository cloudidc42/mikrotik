# Part 82: Kubernetes Networking กับ MikroTik

## สารบัญ
1. [K8s Networking Concepts](#concepts)
2. [CNI Plugins](#cni)
3. [MikroTik as K8s Gateway](#gateway)
4. [Ingress Controller](#ingress)
5. [MetalLB Load Balancer](#metallb)
6. [BGP with K8s](#bgp)
7. [Lab: K8s Network Setup](#lab)

---

## 1. K8s Networking Concepts {#concepts}

### Kubernetes Network Model

```
External Traffic
    │
[MikroTik Router/Firewall]
    │ NodePort or LoadBalancer
    │
[K8s Nodes]
    ├── Node1 (192.168.10.11)
    ├── Node2 (192.168.10.12)
    └── Node3 (192.168.10.13)
    
[Pod Network (CNI)]
    ├── Pod CIDR: 10.244.0.0/16
    └── Service CIDR: 10.96.0.0/12
```

### Network Types

| Type | Description |
|------|-------------|
| Pod Network | เครือข่ายระหว่าง pods (CNI managed) |
| Service Network | ClusterIP, NodePort, LoadBalancer |
| Ingress | HTTP/HTTPS routing to services |
| Node Network | physical network ของ K8s nodes |

---

## 2. CNI Plugins {#cni}

```bash
# ============================================
# Flannel CNI Configuration
# ============================================

# ติดตั้ง Flannel
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# ตรวจสอบ pod network
kubectl get pods -n kube-flannel
kubectl get nodes -o wide

# Pod CIDR สำหรับแต่ละ node
kubectl get nodes -o jsonpath='{.items[*].spec.podCIDR}'

# ============================================
# Calico CNI (รองรับ BGP)
# ============================================

kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# Calico BGP configuration
cat <<EOF | kubectl apply -f -
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  logSeverityScreen: Info
  nodeToNodeMeshEnabled: false
  asNumber: 65100
EOF

# BGP Peer กับ MikroTik Router
cat <<EOF | kubectl apply -f -
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: mikrotik-peer
spec:
  peerIP: 192.168.10.1
  asNumber: 65000
EOF
```

---

## 3. MikroTik as K8s Gateway {#gateway}

```bash
# ============================================
# MikroTik Configuration สำหรับ K8s
# ============================================

# Route K8s pod network ผ่าน nodes
/ip route
# Pod CIDR routes ไปยัง K8s nodes
add dst-address=10.244.0.0/24 gateway=192.168.10.11 comment="Node1 pods"
add dst-address=10.244.1.0/24 gateway=192.168.10.12 comment="Node2 pods"
add dst-address=10.244.2.0/24 gateway=192.168.10.13 comment="Node3 pods"

# Service CIDR route (ไปยัง any node)
add dst-address=10.96.0.0/12 gateway=192.168.10.11 comment="K8s services"

# Firewall rules สำหรับ K8s traffic
/ip firewall filter
# NodePort services (30000-32767)
add chain=forward in-interface=ether1 \
    dst-address=192.168.10.0/24 \
    dst-port=30000-32767 protocol=tcp \
    action=accept comment="NodePort services"

# Allow pods to reach internet
add chain=forward in-interface=ether3 \
    src-address=10.244.0.0/16 action=accept \
    comment="K8s pods to internet"

# NAT สำหรับ pods
/ip firewall nat
add chain=srcnat out-interface=ether1 \
    src-address=10.244.0.0/16 action=masquerade \
    comment="NAT for K8s pods"

# Load Balance across K8s nodes
/ip firewall nat
# Round robin NodePort
add chain=dstnat dst-address=203.0.113.5 dst-port=80 \
    protocol=tcp action=dst-nat \
    to-addresses=192.168.10.11-192.168.10.13 to-ports=30080 \
    comment="NodePort 30080 load balance"
```

---

## 4. Ingress Controller {#ingress}

```yaml
# ============================================
# Nginx Ingress Controller
# ============================================

# ติดตั้ง nginx ingress
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/baremetal/deploy.yaml

# Ingress resource
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8000
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls-cert
```

```bash
# MikroTik - forward traffic to ingress controller
/ip firewall nat
add chain=dstnat dst-address=203.0.113.5 dst-port=80,443 \
    protocol=tcp action=dst-nat \
    to-addresses=192.168.10.11 to-ports=80,443 \
    comment="Forward to Ingress Controller"
```

---

## 5. MetalLB Load Balancer {#metallb}

```yaml
# ============================================
# MetalLB สำหรับ bare-metal K8s
# ============================================

# ติดตั้ง MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml

# IP Address Pool (ใช้ IP จาก MikroTik subnet)
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: local-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.10.200-192.168.10.220
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: local-advert
  namespace: metallb-system
spec:
  ipAddressPools:
  - local-pool
```

```bash
# MikroTik configuration สำหรับ MetalLB BGP mode
/routing bgp instance
add name=default as=65000 router-id=192.168.10.1

/routing bgp peer
add name=metallb-node1 remote-address=192.168.10.11 remote-as=65100
add name=metallb-node2 remote-address=192.168.10.12 remote-as=65100
add name=metallb-node3 remote-address=192.168.10.13 remote-as=65100

# MetalLB BGP Mode config
apiVersion: metallb.io/v1beta2
kind: BGPPeer
metadata:
  name: mikrotik-peer
  namespace: metallb-system
spec:
  myASN: 65100
  peerASN: 65000
  peerAddress: 192.168.10.1
```

---

## 6. Lab: K8s Network Setup {#lab}

### Lab Architecture

```
Internet
    │
[MikroTik Router]
192.168.10.1 (gateway)
    │
[K8s Cluster Network: 192.168.10.0/24]
    ├── master: 192.168.10.10
    ├── worker1: 192.168.10.11
    └── worker2: 192.168.10.12
    
Pod Network: 10.244.0.0/16
Service Network: 10.96.0.0/12
MetalLB Pool: 192.168.10.200-220
```

### Lab Steps

```bash
# Step 1: K8s cluster (kubeadm)
kubeadm init --pod-network-cidr=10.244.0.0/16

# Step 2: Flannel CNI
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# Step 3: MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml
kubectl apply -f metallb-config.yaml

# Step 4: MikroTik routes
/ip route add dst-address=10.244.0.0/16 gateway=192.168.10.11

# Step 5: Deploy test app with LoadBalancer
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=LoadBalancer

# Step 6: Verify
kubectl get svc nginx
# Expected: EXTERNAL-IP = 192.168.10.200 (from MetalLB pool)
curl http://192.168.10.200  # ควร return nginx page
```

### Verification Checklist

- [ ] K8s nodes Ready
- [ ] CNI pods running
- [ ] Pod-to-pod communication ทำงาน
- [ ] MikroTik routes ไปยัง pod networks
- [ ] MetalLB assigns IPs จาก pool
- [ ] LoadBalancer services accessible
- [ ] Ingress routing ทำงาน

> **Tip:** ใช้ `kubectl exec -it pod -- curl http://other-pod-ip` ทดสอบ pod connectivity

---

## Summary

Part นี้ครอบคลุม:
- **K8s network model** และ concepts
- **CNI plugins** - Flannel และ Calico
- **MikroTik configuration** สำหรับ K8s
- **MetalLB** สำหรับ LoadBalancer
- **Ingress** HTTP routing
- **BGP integration** ระหว่าง K8s และ MikroTik

---

[← Part 81: Cloud Environments](part-081-cloud-environments.md) | [Part 83: Ansible →](part-083-ansible.md)
