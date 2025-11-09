# Welcome to my home.

This repo contains deployment manifests for things that I run at home.

A lot of the setup is not deployed with IAC. (drive and pi provisioning, k8s and flux bootstrapping, etc.)

For these, I've broken components out into separate docs.

- [Raspberry Pi Setup](./docs/rpi.md)
- [k3s setup](./docs/k3s.md)
- [Flux setup](./docs/flux.md)
- [Ceph setup](./docs/ceph.md)

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





## ceph




## Special Nodes

Some nodes are dedicated to running ceph. A few nodes are dedicated to running {other services.}.

```bash
kubectl label node k6 apps=true
kubectl label node k7 apps=true
kubectl label node k8 apps=true
kubectl label node k6 role=apps
kubectl label node k7 role=apps
kubectl label node k8 role=apps
```




## Node Maintenance

Disable ceph from shuffling data around during maintenance.

```bash
kubectl rook-ceph ceph osd set noout
kubectl rook-ceph ceph osd set norebalance
kubectl rook-ceph ceph osd set nobackfill
```

Drain the node.

```bash
kubectl drain k6 --ignore-daemonsets --delete-local-data
```

Scale OSD's to 0

```bash
kubectl scale deployment -n rook-ceph rook-ceph-osd-0 --replicas=0
```

Resume after maintenance

```bash
kubectl rook-ceph ceph osd unset noout
kubectl rook-ceph ceph osd unset norebalance
kubectl rook-ceph ceph osd unset nobackfill
```

## back up stuff, switch to lvm

```bash
sudo /usr/local/bin/k3s-uninstall.sh
sudo rm -rf /local/rancher
sudo mkdir /local/rancher
sudo swapoff -a
sudo dphys-swapfile swapoff
sudo systemctl disable dphys-swapfile
sudo rsync -ra /local/ /mnt/local2/ --info=progress2 --exclude swap
```

```
sudo -E -s
rsync --rsync-path="sudo rsync" -ra /local/rook/ john.burt@k1:/local/rookk3/ --info=progress2
```

spare drive
```bash
sudo sgdisk --zap-all /dev/sdXX
sudo pvcreate /dev/sdXX
sudo vgcreate vg /dev/sdXX
sudo lvcreate -L 100G -n lv vg
sudo mkfs.ext4 /dev/vg/lv
```

actual drives
```bash
sudo pvcreate /dev/sda
sudo vgcreate local /dev/sda
sudo lvcreate -L 200G -n md0 local
sudo lvcreate -L 200G -n md1 local
sudo lvcreate -L 10G -n local local
sudo lvextend -l +100%FREE /dev/local/local
```

asdf

```bash
sudo lvcreate -L 80G -n local local
sudo lvcreate -L 120G -n md0 local
sudo lvcreate -L 120G -n md1 local
sudo mkfs.ext4 /dev/local/local
```

For 120g drives

```bash
sudo lvcreate -L 10G -n md0 local
sudo lvextend -l +100%FREE /dev/local/local
```

Then add this to /etc/fstab

```
/dev/local/local /local ext4 defaults 0 2
```

Then mount and re-copy

```bash
sudo systemctl daemon-reload
sudo mount -a
sudo rsync -ra /local2/ /local/
sudo mkdir /local/rancher
/usr/sbin/dphys-swapfile setup
sudo dphys-swapfile swapon
sudo systemctl enable dphys-swapfile
sudo systemctl start dphys-swapfile

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
