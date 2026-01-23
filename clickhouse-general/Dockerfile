ARG BUILD_FROM=ghcr.io/hassio-addons/debian-base:7.3.0

# 強制使用 Debian Base
FROM ghcr.io/hassio-addons/debian-base:7.3.0

# 安裝 ClickHouse
RUN apt-get update && apt-get install -y --no-install-recommends \
    clickhouse-server clickhouse-client curl \
    && rm -rf /var/lib/apt/lists/*

# 複製其他設定檔 (XML 等)
COPY rootfs /

# 建立資料夾並設定權限
RUN mkdir -p /data/clickhouse \
    && chown -R clickhouse:clickhouse /data/clickhouse

# [NUCLEAR FIX] 直接用 printf 產生 run 腳本，避開所有換行與路徑問題
# 注意：這裡修正了 Shebang 為 /usr/bin/with-contenv
RUN printf '#!/usr/bin/with-contenv bashio\n\
PASSWORD=$(bashio::config "password")\n\
MAX_MEM=$(bashio::config "max_memory_usage")\n\
MAX_SERVER_MEM=$(bashio::config "max_server_memory")\n\
\n\
export CLICKHOUSE_PASSWORD="$PASSWORD"\n\
\n\
cat > /etc/clickhouse-server/config.d/tuning.xml <<EOF\n\
<clickhouse>\n\
    <max_memory_usage>$MAX_MEM</max_memory_usage>\n\
    <max_server_memory_usage>$MAX_SERVER_MEM</max_server_memory_usage>\n\
    <max_threads>4</max_threads>\n\
    <background_pool_size>4</background_pool_size>\n\
</clickhouse>\n\
EOF\n\
\n\
chown -R clickhouse:clickhouse /data/clickhouse\n\
bashio::log.info "Starting ClickHouse..."\n\
exec s6-setuidgid clickhouse /usr/bin/clickhouse-server --config-file=/etc/clickhouse-server/config.xml\n\
' > /etc/services.d/clickhouse/run

# 給予執行權限
RUN chmod +x /etc/services.d/clickhouse/run
