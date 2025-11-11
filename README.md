# Welcome to my home.

This repo contains deployment manifests for things that I run at home.

A lot of the setup is not deployed with IAC. (drive and pi provisioning, k8s and flux bootstrapping, etc.)

For these, I've broken components out into separate docs.

- [Raspberry Pi Setup](./docs/rpi.md)
- [k3s setup](./docs/k3s.md)
- [Ceph setup](./docs/rook-ceph-provisioning.md)

## Updates

Updates are generally managed by renovate opening PRs against this repo. 

To preview changes, run this locally.

```bash
podman run --rm --entrypoint bash \
  -v "$PWD":/work -w /work \
  -e LOG_LEVEL=debug \
  -e RENOVATE_PLATFORM=local \
  -e RENOVATE_LOCAL_DIR=/work \
  -e RENOVATE_DRY_RUN=full \
  renovate/renovate:latest \
  -lc 'git config --global --add safe.directory /work && renovate'
```

##todo:

- change journald log location or map /var/log to a drive
- firecracker-containerd?
- https://docs.k3s.io/cli/agent
- --kubelet-arg='max-pods=220,system-reserved=memory=500Mi,cpu=500m,pid=2000,

- /etc/rancher/k3s/config.yaml
kubelet-arg:
  - "kube-reserved=cpu=500m,memory=1Gi,ephemeral-storage=2Gi"
  - "system-reserved=cpu=500m, memory=1Gi,ephemeral-storage=2Gi"
  - "eviction-hard=memory.available<500Mi,nodefs.available<10%"

```
sudo mkdir -p /etc/rancher/k3s
sudo cat << EOF | sudo tee /etc/rancher/k3s/config.yaml
kubelet-arg:
  - '--max-pods=220'
  - '--system-reserved=memory=200Mi,cpu=100m,pid=2000'
  - '--kube-reserved=memory=200Mi,cpu=100m,pid=2000'
  - '--eviction-hard=imagefs.available<15%,nodefs.available<10%,pid.available<10%,memory.available<100Mi'
  - '--image-gc-low-threshold=10'
  - '--image-gc-high-threshold=25'
  - '--pod-max-pids=10000'
  - '--root-dir=/local/kubelet'
EOF
```
