# k3s setup

```bash
curl -sfL https://get.k3s.io | K3S_TOKEN="$(bw get password K3S_TOKEN)" sh -s - server --cluster-init --flannel-backend none --cluster-cidr=100.64.0.0/16 --service-cidr=100.65.0.0/16 --disable=traefik --disable=servicelb --disable-kube-proxy --disable-network-policy --disable-helm-controller --disable=local-storage --tls-san k8s-api.jburt.me --data-dir /local/rancher
```

## Install cilium

Cilium should be installed before you add more nodes.

cilium-values.yaml
```yaml
bpf:
  masquerade: true
ipam:
  mode: cluster-pool
  operator:
    clusterPoolIPv4PodCIDRList:
    - 100.64.0.0/16
    clusterPoolIPv4MaskSize: 24
kubeProxyReplacement: true
k8sServiceHost: k8s-api.jburt.me
k8sServicePort: 6443
autoDirectNodeRoutes: true
ipMasqAgent:
  enabled: true
nonMasqueradeCIDRs:
- 100.64.0.0/16
- 100.65.0.0/16
loadBalancer:
  mode: dsr
  serviceTopology: true
routingMode: "native"
ipv4NativeRoutingCIDR: 100.64.0.0/15
cgroup:
  autoMount:
    enabled: true
  # hostRoot: /sys/fs/cgroup
hubble:
  enabled: false
```

**Note: geneve is disabled in the rpi kernel and it takes forever to compile :(

```bash
helm repo add cilium https://helm.cilium.io/
helm upgrade --install cilium cilium/cilium --version 1.18.2 --namespace cilium-values.yaml
```

## Install additional nodes (or update nodes)

```bash
curl -sfL https://get.k3s.io | K3S_TOKEN="$(bw get password K3S_TOKEN)" sh -s - server --server https://k8s-api.jburt.me:6443 --flannel-backend none --cluster-cidr=100.64.0.0/16 --service-cidr=100.65.0.0/16 --disable=traefik --disable=servicelb --disable-kube-proxy --disable-network-policy --disable-helm-controller --disable local-storage --tls-san k8s-api.jburt.me --data-dir /local/rancher
```

For agent nodes

```bash
curl -sfL https://get.k3s.io | K3S_TOKEN="$(bw get password K3S_TOKEN)" sh -s - agent --server https://k8s-api.jburt.me:6443 --data-dir /local/rancher
```

You can add labels to nodes with `--node-label` arg. I typically just label nodes manually after they're added.

## Adding taints to osd nodes

```bash
kubectl taint nodes k1 role=osd:NoSchedule
kubectl taint nodes k2 role=osd:NoSchedule
kubectl taint nodes k3 role=osd:NoSchedule
kubectl taint nodes k4 role=osd:NoSchedule
kubectl taint nodes k5 role=osd:NoSchedule
kubectl taint nodes k9 role=osd:NoSchedule
```

```bash
kubectl label nodes k0 nvidia.com/gpu=true
kubectl label node k1 role=osd
kubectl label node k2 role=osd
kubectl label node k3 role=osd
kubectl label node k4 role=osd
kubectl label node k5 role=osd
kubectl label node k6 role=apps
kubectl label node k7 role=apps
kubectl label node k8 role=apps
kubectl label node k9 role=osd
kubectl label node k10 role=osd
kubectl label node k12 role=apps
kubectl label node k13 role=apps
kubectl label node k14 role=apps
```