ARG BUILD_FROM=ghcr.io/hassio-addons/debian-base:7.3.0

# [FIX] 強制使用 Debian Base，忽略 Supervisor 傳入的 Alpine Base
FROM ghcr.io/hassio-addons/debian-base:7.3.0

RUN apt-get update && apt-get install -y --no-install-recommends \
    clickhouse-server clickhouse-client curl \
    && rm -rf /var/lib/apt/lists/*

COPY rootfs /

RUN mkdir -p /data/clickhouse \
    && chown -R clickhouse:clickhouse /data/clickhouse \
    && chmod +x /etc/services.d/*/run

CMD [ "/init" ]
