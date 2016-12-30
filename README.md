# Redis Cluster in Docker Swarm (mode)

> This repository is inspired by [another
  redis-cluster](https://github.com/AliyunContainerService/redis-cluster)
  GitHub repository.

There is a `Dockerfile` under `sentinel\` folder which will let you
create image with Redis Sentinel as an entry point. Here is how you
can build it:

```bash
cd sentinel/
docker build -t redis-sentinel:3.2.6-alpine .
```