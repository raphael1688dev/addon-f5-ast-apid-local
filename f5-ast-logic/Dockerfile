ARG BUILD_FROM=ghcr.io/hassio-addons/debian-base:7.3.0

# --- Stage 1: Fetch Resources ---
FROM alpine:latest as fetcher
RUN apk add --no-cache curl jq
WORKDIR /downloads

# 1. Configs
RUN curl -o otel-config.yaml https://raw.githubusercontent.com/raphael1688/ast-api-discovery/refs/heads/main/services/otel_collector/config.yaml
RUN curl -o prometheus.yml https://raw.githubusercontent.com/raphael1688/ast-api-discovery/refs/heads/main/services/prometheus/prometheus.yml

# 2. Scripts (.sh)
RUN mkdir -p init_scripts
WORKDIR /downloads/init_scripts
RUN curl -s "https://api.github.com/repos/raphael1688/ast-api-discovery/contents/services/clickhouse/docker-entrypoint-initdb.d?ref=main" \
    | jq -r '.[].name' \
    | while read filename; do \
        echo "Downloading $filename..."; \
        curl -O "https://raw.githubusercontent.com/raphael1688/ast-api-discovery/refs/heads/main/services/clickhouse/docker-entrypoint-initdb.d/$filename"; \
      done

# --- Stage 2: Extract Binaries ---
FROM prom/prometheus:v2.54.1 as prometheus_source
FROM ghcr.io/f5devcentral/ast-api-discovery/api_discovery_log_analyzer:v0.2.0 as analyzer_source
FROM ghcr.io/f5devcentral/application-study-tool/otel_custom_collector:v0.7.0 as collector_source

# --- Final Stage ---
FROM ghcr.io/hassio-addons/debian-base:7.3.0

# 雖然 Analyzer 是 Binary，但我們還是保留 python3 供 .sh 腳本或其他工具使用
# 移除 pip 安裝步驟，因為不需要了
RUN apt-get update && apt-get install -y --no-install-recommends \
    clickhouse-client python3 netcat-openbsd curl bash \
    && rm -rf /var/lib/apt/lists/*

# Deploy Binaries
COPY --from=prometheus_source /bin/prometheus /usr/bin/prometheus
COPY --from=collector_source /otelcol-custom /usr/bin/otelcol-custom
RUN chmod +x /usr/bin/otelcol-custom

# Deploy Analyzer Binary
WORKDIR /app/analyzer
# 複製資料夾內容 (包含 api_discovery 執行檔)
COPY --from=analyzer_source /app/. .

# [UPDATE] 移除了 pip install，因為沒有 requirements.txt

# Deploy Configs & Scripts
RUN mkdir -p /etc/prometheus /etc/otel-collector /app/sql /data/prometheus
COPY --from=fetcher /downloads/prometheus.yml /etc/prometheus/prometheus.yml
COPY --from=fetcher /downloads/otel-config.yaml /etc/otel-collector/config.yaml
COPY --from=fetcher /downloads/init_scripts/ /app/sql/

COPY rootfs /
RUN chmod +x /etc/services.d/*/run
CMD [ "/init" ]
