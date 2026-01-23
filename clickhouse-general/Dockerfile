ARG BUILD_FROM=ghcr.io/hassio-addons/debian-base:7.3.0
FROM ghcr.io/hassio-addons/debian-base:7.3.0

RUN apt-get update && apt-get install -y --no-install-recommends \
    clickhouse-server clickhouse-client curl \
    && rm -rf /var/lib/apt/lists/*

COPY rootfs /

RUN mkdir -p /data/clickhouse \
    && chown -R clickhouse:clickhouse /data/clickhouse \
    && chmod +x /etc/services.d/*/run

# [FIX] 移除了 CMD [ "/init" ]
