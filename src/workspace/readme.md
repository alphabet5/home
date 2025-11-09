# "workspace" docker image

This is used to replace a generic server to ssh into and do things.

```
docker build --platform linux/amd64,linux/arm64 -t alphabet5/workspace:latest .
docker push alphabet5/workspace:latest
```
