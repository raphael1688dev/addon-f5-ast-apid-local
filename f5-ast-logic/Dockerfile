# 使用 Home Assistant 官方推薦的 Debian 基礎映像檔 [cite: 38, 56]
ARG BUILD_FROM=ghcr.io/hassio-addons/debian-base:7.3.0

# --- Stage 1: 下載資源 (使用 Alpine 以節省空間) [cite: 39, 57] ---
FROM alpine:latest as fetcher
RUN apk add --no-cache curl jq [cite: 23, 58]
WORKDIR /downloads

# 1. 下載 OTel 與 Prometheus 設定檔 [cite: 19, 20, 58]
RUN curl -o otel-config.yaml https://raw.githubusercontent.com/raphael1688/ast-api-discovery/refs/heads/main/services/otel_collector/config.yaml
RUN curl -o prometheus.yml https://raw.githubusercontent.com/raphael1688/ast-api-discovery/refs/heads/main/services/prometheus/prometheus.yml

# 2. 自動抓取 GitHub 上的所有資料庫初始化腳本 (.sh) [cite: 40, 41, 58, 59, 60]
RUN mkdir -p init_scripts
WORKDIR /downloads/init_scripts
RUN curl -s "https://api.github.com/repos/raphael1688/ast-api-discovery/contents/services/clickhouse/docker-entrypoint-initdb.d?ref=main" \
    | jq -r '.[].name' \
    | while read filename; do \
        echo "Downloading $filename..."; \
        curl -O "https://raw.githubusercontent.com/raphael1688/ast-api-discovery/refs/heads/main/services/clickhouse/docker-entrypoint-initdb.d/$filename"; \
      done

# --- Stage 2: 提取各元件執行檔 [cite: 2, 3, 6, 13, 14, 56] ---
FROM docker.io/prom/prometheus:v2.54.1 as prometheus_source
FROM ghcr.io/f5devcentral/ast-api-discovery/api_discovery_log_analyzer:v0.2.0 as analyzer_source
FROM ghcr.io/f5devcentral/application-study-tool/otel_custom_collector:v0.7.0 as collector_source

# --- Final Stage: 建立正式 Add-on 映像檔 ---
FROM ghcr.io/hassio-addons/debian-base:7.3.0

# 安裝必要套件 (注意：此環境不需 pip3，因為 analyzer 已編譯為執行檔) [cite: 9, 28, 43]
RUN apt-get update && apt-get install -y --no-install-recommends \
    clickhouse-client python3 netcat-openbsd curl bash \
    && rm -rf /var/lib/apt/lists/* [cite: 16, 29]

# 部署執行檔 (Binaries) [cite: 14, 15, 43, 44]
COPY --from=prometheus_source /bin/prometheus /usr/bin/prometheus
COPY --from=collector_source /otelcol-custom /usr/bin/otelcol-custom
RUN chmod +x /usr/bin/otelcol-custom [cite: 13, 27, 44]

# 部署 F5 Log Analyzer (api_discovery) [cite: 24, 31, 66]
WORKDIR /app/analyzer
COPY --from=analyzer_source /app/. .

# 部署設定檔與 SQL 腳本 [cite: 21, 22, 24, 29, 30, 58]
RUN mkdir -p /etc/prometheus /etc/otel-collector /app/sql /data/prometheus
COPY --from=fetcher /downloads/prometheus.yml /etc/prometheus/prometheus.yml
COPY --from=fetcher /downloads/otel-config.yaml /etc/otel-collector/config.yaml
COPY --from=fetcher /downloads/init_scripts/ /app/sql/

# 建立 S6 服務資料夾 [cite: 23, 26, 61]
RUN mkdir -p /etc/services.d/init-remote-db /etc/services.d/otel-collector /etc/services.d/log-analyzer /etc/services.d/prometheus

# 建立啟動腳本 (使用 printf 避開換行符號問題並修正路徑) [cite: 45, 62, 67, 72]

# 1. 資料庫初始化服務 [cite: 45, 46, 47, 48, 62, 63, 64, 65]
RUN printf '#!/usr/bin/with-contenv bashio\n\
HOST=$(bashio::config "db_host"); PORT=$(bashio::config "db_port")\n\
USER=$(bashio::config "db_user"); PASS=$(bashio::config "db_password")\n\
if [ -f "/data/db_initialized.flag" ]; then s6-svc -d /var/run/s6/services/init-remote-db; exit 0; fi\n\
bashio::log.info "Waiting for ClickHouse at $HOST:$PORT..."\n\
while ! nc -z "$HOST" "$PORT"; do sleep 2; done\n\
export CLICKHOUSE_HOST="$HOST"; export CLICKHOUSE_PORT="$PORT"; export CLICKHOUSE_USER="$USER"; export CLICKHOUSE_PASSWORD="$PASS"\n\
for f in /app/sql/*; do [ -f "$f" ] || continue; chmod +x "$f"; bashio::log.info "Running $f"; /bin/bash "$f"; done\n\
touch "/data/db_initialized.flag"\n\
s6-svc -d /var/run/s6/services/init-remote-db\n' > /etc/services.d/init-remote-db/run

# 2. OTel Collector 服務 [cite: 49, 67]
RUN printf '#!/usr/bin/with-contenv bashio\n\
HOST=$(bashio::config "db_host"); PORT=$(bashio::config "db_port"); PASS=$(bashio::config "db_password")\n\
export OTEL_COLLECTOR_CLICKHOUSE_PASSWORD="$PASS"\n\
sed -i "s|endpoint: tcp://.*:9000|endpoint: tcp://$HOST:$PORT|g" /etc/otel-collector/config.yaml\n\
while [ ! -f "/data/db_initialized.flag" ]; do sleep 2; done\n\
exec /usr/bin/otelcol-custom --config=/etc/otel-collector/config.yaml\n' > /etc/services.d/otel-collector/run

# 3. Log Analyzer 服務 [cite: 68, 69, 70, 71, 72]
RUN printf '#!/usr/bin/with-contenv bashio\n\
HOST=$(bashio::config "db_host"); PORT=$(bashio::config "db_port"); PASS=$(bashio::config "db_password"); INTERVAL=$(bashio::config "analyzer_interval")\n\
export CLICKHOUSE_HOST="$HOST"; export CLICKHOUSE_PORT="$PORT"; export CLICKHOUSE_PASSWORD="$PASS"; export DATA_ANALYSIS_MIN_INTERVAL_SECONDS="$INTERVAL"; export ACCESS_LOGS_FETCH_LIMIT=3000000; export CLICKHOUSE_USE_TLS="False"\n\
while [ ! -f "/data/db_initialized.flag" ]; do sleep 2; done\n\
cd /app/analyzer\n\
exec ./api_discovery\n' > /etc/services.d/log-analyzer/run

# 4. Prometheus 服務
RUN printf '#!/usr/bin/with-contenv bashio\n\
exec /usr/bin/prometheus --config.file=/etc/prometheus/prometheus.yml --storage.tsdb.path=/data/prometheus --web.listen-address="0.0.0.0:9090"\n' > /etc/services.d/prometheus/run

# 設定執行權限 [cite: 27]
RUN chmod +x /etc/services.d/*/run

# 注意：此處不加 CMD ["/init"] 以免發生 PID 1 衝突錯誤 [cite: 1, 12, 55]
