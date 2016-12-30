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