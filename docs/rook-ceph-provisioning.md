# rook-ceph

Most of the rook/ceph configuration is straightforward.



## Wiping a drive to be used by ceph

```bash
sudo sgdisk --zap-all /dev/sdXX
```

When replacing a spinning drive, the lvm metadatadevice should also be wiped. Delete the lv and recreate it.

After wiping, the osd won't be discovered by ceph right away. Restarting the operator will restart the prepare osd jobs.

```bash
kubectl rollout restart deployment -n rook-ceph rook-ceph-operator
```

## recovering from a node being offline

- mark the osd's as "in"
- restart the osd pods

```bash
kubectl rollout restart deployment -n rook-ceph -l app=rook-ceph-osd
```

This restarts all of the osd's, which is probably not what you want. You can just delete the pods for the specific osd (and the deployment will recreate them) or you can roll the specific osd deployment.

## repairing a pg

From ceph health detail

```text
[ERR] PG_DAMAGED: Possible data damage: 1 pg inconsistent
    pg 19.18 is active+clean+scrubbing+deep+inconsistent, acting [14,1,13]
```

```bash
john.burt@mba ~ % kubectl rook-ceph ceph pg scrub 19.18
Info: running 'ceph' command with args: [pg scrub 19.18]
instructing pg 19.18 on osd.14 to scrub
john.burt@mba ~ % kubectl rook-ceph ceph pg repair 19.18
Info: running 'ceph' command with args: [pg repair 19.18]
instructing pg 19.18 on osd.14 to repair
```

To automatically repair PG's when they're discovered to be inconsistent (during scrubbing)

```bash
kubectl rook-ceph ceph config set global osd_scrub_auto_repair true
```

This is currently configured in the cephcluster config.

```bash
 %  cat base/ceph/ceph-cluster.yaml | yq -o json | gron | grep 'osd_scrub_auto_repair'
json.spec.cephConfig.global.osd_scrub_auto_repair = "true";
json.spec.cephConfig.global.osd_scrub_auto_repair_num_errors = "10000";
```

## replacing or removing an osd


Make sure "useAllDevices" is set to false in the ceph-cluster.yaml file.

```bash
kubectl scale deployment -n rook-ceph rook-ceph-osd-0 --replicas=0
kubectl rook-ceph rook purge-osd 0 --force
```

If the osd fails to reprovision 

```
stderr: Error EEXIST: entity osd.6 exists but key does not match
```

```bash
ceph auth del osd.6
```

```bash
sudo sgdisk --zap-all /dev/sdd
sudo sgdisk --zap-all /dev/local/md1
sudo partprobe /dev/local/md1
```

Then a restart of the operator will run through the osd prepare job again and add the osd (with the new metadata device).

```bash
kubectl rollout restart deployment -n rook-ceph rook-ceph-operator
```

## Acking alerts

Sometimes daemons will crash, and trigger a warning for the cluster. 

These can be viewed with `kubectl rook-ceph ceph crash ls` 

They can be archived with `kubectl rook-ceph ceph crash archive <id>`

## Inconsistant PG

```text
[ERR] PG_DAMAGED: Possible data damage: 1 pg inconsistent
    pg 19.b9 is active+clean+inconsistent, acting [14,10,12]
```

```bash
~ % k ceph pg repair 19.b9
Info: running 'ceph' command with args: [pg repair 19.b9]
instructing pg 19.b9 on osd.14 to repair
```

```bash
ceph tell <pgid> query
```

```bash
k ceph health detail | grep 'not deep-scrubbed since' | awk '{print $2}' | xargs -L 1 kubectl rook-ceph ceph pg deep-scrub
```

## mgr issues

Sometimes stats or info can be wrong / a mgr can be stuck. Manually fail over with:

```
k ceph mgr fail
```

## pg stuck deep-scrubbing

Sometimes pg's can be stuck deep-scurbbing, you can repeer the pg's with this (assuming the deep-scrub command failed / they got stuck.)

```
k ceph health detail | grep 'not deep-scrubbed since' | awk '{print $2}' | xargs -L 1 kubectl rook-ceph ceph pg repeer
```

