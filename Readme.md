## Notes

* Built in docker now.  Using dockerhub as the repo.

## Process

### Build Locally

```bash
buildah bud --format docker -t toddwardzinski/resume:latest .
```

### Push to Repo

```bash
docker push toddwardzinski/resume:latest
```

### Test Locally

```bash
podman run -d -p 8080:80 localhost/toddwardzinski/resume
```

### Deployment

1. Follow the documentation here https://developers.cloudflare.com/cloudflare-one/tutorials/many-cfd-one-tunnel to appropriately set up tunnel.

