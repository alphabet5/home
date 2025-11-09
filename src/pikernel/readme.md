## Building the kernel

This was taken from febus982/rpi-kernel

The only modification was also enabling geneve in the kernel.

```bash
cd src/pikernel
docker compose run --build --rm builder
rsync -ra debs k12:
rsync -ra debs k13:
rsync -ra debs k14:
cd ../../
run-commands k12 k13 k14 --command 'sudo apt install ./debs/*.deb'
```

