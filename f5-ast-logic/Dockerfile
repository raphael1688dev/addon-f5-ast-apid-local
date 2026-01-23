ARG BUILD_FROM=ghcr.io/hassio-addons/debian-base:7.3.0

# --- Stage 1: 下載資源 ---
FROM alpine:latest as fetcher
RUN apk add --no-cache curl jq
WORKDIR /downloads
RUN curl -o otel-config.yaml https://raw.githubusercontent.com/raphael1688/ast-api-discovery/refs/heads/main/services/otel_collector/config.yaml
RUN curl -o prometheus.yml https://raw.githubusercontent.com/raphael1688/ast-api-discovery/refs/heads/main/services/prometheus/prometheus.yml
RUN mkdir -p init_scripts
WORKDIR /downloads/init_scripts
RUN curl -s "https://api.github.com/repos/raphael1688/ast-api-discovery/contents/services/clickhouse/docker-entrypoint-initdb.d?ref=main" \
    | jq -r '.[].name' \
    | while read filename; do \
        curl -O "https://raw.githubusercontent.com/raphael1688/ast-api-discovery/refs/heads/main/services/clickhouse/docker-entrypoint-initdb.d/$filename"; \
      done

# --- Stage 2: 提取執行檔 ---
FROM docker.io/prom/prometheus:v2.54.1 as prometheus_source
FROM ghcr.io/f5devcentral/ast-api-discovery/api_discovery_log_analyzer:v0.2.0 as analyzer_source
FROM ghcr.io/f5devcentral/application-study-tool/otel_custom_collector:v0.7.0 as collector_source

# --- Final Stage ---
FROM ghcr.io/hassio-addons/debian-base:7.3.0

# 安裝 ClickHouse Client
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl gnupg2 ca-certificates lsb-release python3 netcat-openbsd bash \
    && curl -fsSL 'https://packages.clickhouse.com/rpm/lts/repodata/repomd.xml.key' | gpg --dearmor -o /usr/share/keyrings/clickhouse-keyring.gpg \
    && echo "deb [signed-by=/usr/share/keyrings/clickhouse-keyring.gpg] https://packages.clickhouse.com/deb stable main" > /etc/apt/sources.list.d/clickhouse.list \
    && apt-get update \
    && apt-get install -y --no-install-recommends clickhouse-client \
    && rm -rf /var/lib/apt/lists/*

# 部署元件
COPY --from=prometheus_source /bin/prometheus /usr/bin/prometheus
COPY --from=collector_source /otelcol-custom /usr/bin/otelcol-custom
RUN chmod +x /usr/bin/otelcol-custom
WORKDIR /app/analyzer
COPY --from=analyzer_source /app/. .

# 部署設定檔
RUN mkdir -p /etc/prometheus /etc/otel-collector /app/sql /data/prometheus
COPY --from=fetcher /downloads/prometheus.yml /etc/prometheus/prometheus.yml
COPY --from=fetcher /downloads/otel-config.yaml /etc/otel-collector/config.yaml
COPY --from=fetcher /downloads/init_scripts/ /app/sql/

# 建立服務資料夾
RUN mkdir -p /etc/services.d/init-remote-db /etc/services.d/otel-collector /etc/services.d/log-analyzer /etc/services.d/prometheus

# 1. 資料庫初始化腳本
RUN printf '#!/usr/bin/with-contenv bashio\n\
HOST=$(bashio::config "db_host"); PORT=$(bashio::config "db_port")\n\
USER=$(bashio::config "db_user"); PASS=$(bashio::config "db_password")\n\
if [ -f "/data/db_initialized.flag" ]; then exit 0; fi\n\
bashio::log.info "Waiting for ClickHouse at $HOST:$PORT..."\n\
while ! nc -z "$HOST" "$PORT"; do sleep 2; done\n\
\n\
mkdir -p /etc/clickhouse-client\n\
cat > /etc/clickhouse-client/config.xml <<EOF\n\
<config>\n\
    <host>$HOST</host>\n\
    <port>$PORT</port>\n\
    <user>$USER</user>\n\
    <password>$PASS</password>\n\
</config>\n\
EOF\n\
\n\
bashio::log.info "Running SQL initialization scripts..."\n\
for f in /app/sql/*; do [ -f "$f" ] || continue; chmod +x "$f"; bashio::log.info "Running $f"; /bin/bash "$f"; done\n\
touch "/data/db_initialized.flag"\n\
bashio::log.info "Database initialized successfully."\n' > /etc/services.d/init-remote-db/run

# 2. OTel Collector 腳本
RUN printf '#!/usr/bin/with-contenv bashio\n\
HOST=$(bashio::config "db_host"); PORT=$(bashio::config "db_port"); PASS=$(bashio::config "db_password")\n\
export OTEL_COLLECTOR_CLICKHOUSE_PASSWORD="$PASS"\n\
export CLICKHOUSE_PASSWORD="$PASS"\n\
USER=$(bashio::config "db_user")\n\
export OTEL_COLLECTOR_CLICKHOUSE_USER="$USER"\n\
\n\
sed -i "s|endpoint: tcp://.*:9000|endpoint: tcp://$HOST:$PORT|g" /etc/otel-collector/config.yaml\n\
\n\
# [Fix] 將 metrics exporter 的 prometheus 主機名改為 localhost\n\
sed -i "s|http://prometheus:9090|http://localhost:9090|g" /etc/otel-collector/config.yaml\n\
\n\
# Disable TLS\n\
sed -i "s|ca_file:|# ca_file:|g" /etc/otel-collector/config.yaml\n\
sed -i "s|cert_file:|# cert_file:|g" /etc/otel-collector/config.yaml\n\
sed -i "s|key_file:|# key_file:|g" /etc/otel-collector/config.yaml\n\
\n\
while [ ! -f "/data/db_initialized.flag" ]; do sleep 2; done\n\
exec /usr/bin/otelcol-custom --config=/etc/otel-collector/config.yaml\n' > /etc/services.d/otel-collector/run

# 3. Log Analyzer 腳本
RUN printf '#!/usr/bin/with-contenv bashio\n\
HOST=$(bashio::config "db_host"); PORT=$(bashio::config "db_port")\n\
USER=$(bashio::config "db_user"); PASS=$(bashio::config "db_password")\n\
INTERVAL=$(bashio::config "analyzer_interval")\n\
\n\
export CLICKHOUSE_HOST="$HOST"\n\
export CLICKHOUSE_PORT="$PORT"\n\
export CLICKHOUSE_USER="$USER"\n\
export CLICKHOUSE_PASSWORD="$PASS"\n\
export DATA_ANALYSIS_MIN_INTERVAL_SECONDS="$INTERVAL"\n\
export ACCESS_LOGS_FETCH_LIMIT=3000000\n\
export CLICKHOUSE_USE_TLS="False"\n\
\n\
while [ ! -f "/data/db_initialized.flag" ]; do sleep 2; done\n\
cd /app/analyzer\n\
exec ./api_discovery\n' > /etc/services.d/log-analyzer/run

# 4. Prometheus 腳本
# [關鍵] 使用 enable-feature=otlp-write-receiver 支援 v2.54 版
RUN printf '#!/usr/bin/with-contenv bashio\n\
exec /usr/bin/prometheus \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/data/prometheus \
    --web.listen-address="0.0.0.0:9090" \
    --enable-feature=otlp-write-receiver \
    --web.enable-remote-write-receiver\n' > /etc/services.d/prometheus/run

RUN chmod +x /etc/services.d/*/run
