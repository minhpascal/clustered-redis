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

## Setting up Redis cluster in Docker Swarm (Mode)

Main sources of information when setting up Redis cluster:

Following ports need to be opened on all swarm nodes: `6379`, `16379`
and `26379`. The first one is used by client applications and the
second one is used for Redis cluster nodes to talk to each other. The
last one is used when Redis cluster is running in Sentinel mode (for
high cluster availability).

> Note that Redis will only let you configure the first port whether
the port for internal cluster communications will be calculated based
on that first node `client_port + 10000`.

Redis is using sharding model based on hash slots. There are `16384`
hash slots in total that are getting distributed between different
cluster nodes. Redis allows to add/remove nodes without a downtime by
taking time to re-distribute hash slots between cluster nodes.

Redis supports `master/slave` model where each master node can have
zero or more slave nodes service the same hash slot. In cases when
master node fails, a new master node will be elected from slave nodes
that have been attached to that master node. This allows to ensure
consistency of the whole cluster while some master nodes failing. And
there might be multiple `master` nodes serving different hash slots.

Redist cluster **does not** guarantee consistency in two scenarious:

* Writes are getting acknowledged by master node to a client
  before it is synchronized to connected slave nodes. In this scenario
  master node can fail and the slave that didn't recieve update might
  become a new master, thus loosing piece of information.
* During some period of time (`node timeout`) isolated master node (a
  master node in minority group in case of a network partition) will
  keep accepting writes from clients until it will realize that it is
  in a minority group. When it will realize that it is in minority it
  will stop accepting writes. In the same time in majority group a new
  master will be elected. Thus, when consistency of a cluster is
  restored the data that isolated master had accepted will be lost.

Before starting our cluster we will need to make a docker image for
sentinel redis based on standard redis alpine image.

**`sentinel.conf`**

```
port 26379
dir /tmp
sentinel monitor mymaster $REDIS_MASTER_HOST $REDIS_MASTER_PORT $SENTINEL_QUORUM
sentinel down-after-milliseconds mymaster $SENTINEL_DOWN_AFTER
sentinel parallel-syncs mymaster $SENTINEL_PARALLEL_SYNC
sentinel failover-timeout mymaster $SENTINEL_FAILOVER_TIMEOUT
```

**`sentinel-entrypoint.sh`**

```
#!/bin/sh
sed -i "s/\$REDIS_MASTER_HOST/$REDIS_MASTER_HOST/g" /etc/redis/sentinel.conf
sed -i "s/\$REDIS_MASTER_PORT/$REDIS_MASTER_PORT/g" /etc/redis/sentinel.conf
sed -i "s/\$SENTINEL_QUORUM/$SENTINEL_QUORUM/g" /etc/redis/sentinel.conf
sed -i "s/\$SENTINEL_DOWN_AFTER/$SENTINEL_DOWN_AFTER/g" /etc/redis/sentinel.conf
sed -i "s/\$SENTINEL_PARALLEL_SYNC/$SENTINEL_PARALLEL_SYNC/g" /etc/redis/sentinel.conf
sed -i "s/\$SENTINEL_FAILOVER_TIMEOUT/$SENTINEL_FAILOVER_TIMEOUT/g" /etc/redis/sentinel.conf
exec docker-entrypoint.sh redis-server /etc/redis/sentinel.conf --sentinel
```

And finally, **`Dockerfile`**

```
FROM redis:3.2.6-alpine
MAINTAINER Roman Kuznetsov <roman@kuznero.com>
EXPOSE 26379
ADD sentinel.conf /etc/redis/sentinel.conf
RUN chown redis:redis /etc/redis/sentinel.conf
ENV REDIS_MASTER_HOST redis-master
ENV REDIS_MASTER_PORT 6379
ENV SENTINEL_QUORUM 2
ENV SENTINEL_DOWN_AFTER 30000
ENV SENTINEL_PARALLEL_SYNC 1
ENV SENTINEL_FAILOVER_TIMEOUT 180000
COPY sentinel-entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/sentinel-entrypoint.sh
ENTRYPOINT ["sentinel-entrypoint.sh"]
```

We will then need to build it: `docker build -t redis-sentinel:3.2.6-alpine .`

Here is our first attempt into building our first Redis cluster with a
single master node:

```
# one single master node (it must not be scaled up and stay 1 at all times)
docker service create \
  --name redis \
  --network fx \
  --port 6379:6379 \
  redis:3.2.6-alpine

# a few slave nodes pointing to our master node
docker service create \
  --name redis-slave \
  --network fx \
  --replicas 3 \
  --port 6379:6379 \
  redis:3.2.6-alpine \
  redis-server --slaveof redis 6379

# and a few sentinel nodes pointing to our master node
docker service create \
  --name redis-sentinel \
  --network fx \
  --replicas 3 \
  -e REDIS_MASTER_HOST=redis \
  -e REDIS_MASTER_PORT=6379 \
  -e SENTINEL_DOWN_AFTER=5000 \
  -e SENTINEL_FAILOVER_TIMEOUT=15000 \
  redis-sentinel:3.2.6-alpine
```