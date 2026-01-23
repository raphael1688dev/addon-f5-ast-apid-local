ARG BUILD_FROM=ghcr.io/hassio-addons/debian-base:7.3.0

# 強制使用 Debian Base
FROM ghcr.io/hassio-addons/debian-base:7.3.0

# 安裝 ClickHouse
RUN apt-get update && apt-get install -y --no-install-recommends \
    clickhouse-server clickhouse-client curl \
    && rm -rf /var/lib/apt/lists/*

COPY rootfs /

# [NUCLEAR FIX v4] 加入 Console Logging 設定
RUN printf '#!/usr/bin/with-contenv bashio\n\
PASSWORD=$(bashio::config "password")\n\
MAX_MEM=$(bashio::config "max_memory_usage")\n\
MAX_SERVER_MEM=$(bashio::config "max_server_memory")\n\
\n\
export CLICKHOUSE_PASSWORD="$PASSWORD"\n\
\n\
# 自動尋找執行檔\n\
if [ -f "/usr/sbin/clickhouse-server" ]; then\n\
    BINARY="/usr/sbin/clickhouse-server"\n\
else\n\
    BINARY="/usr/bin/clickhouse-server"\n\
fi\n\
bashio::log.info "Found ClickHouse binary at: $BINARY"\n\
\n\
# 確保資料夾存在\n\
if [ ! -d "/data/clickhouse" ]; then\n\
    bashio::log.info "Creating data directory..."\n\
    mkdir -p /data/clickhouse\n\
fi\n\
\n\
# [NEW] 強制開啟控制台日誌 (讓我們能在 HA 看到輸出)\n\
cat > /etc/clickhouse-server/config.d/console_log.xml <<EOF\n\
<clickhouse>\n\
    <logger>\n\
        <level>information</level>\n\
        <console>1</console>\n\
    </logger>\n\
</clickhouse>\n\
EOF\n\
\n\
# 設定效能參數\n\
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
# 執行 Server\n\
exec s6-setuidgid clickhouse "$BINARY" --config-file=/etc/clickhouse-server/config.xml\n\
' > /etc/services.d/clickhouse/run

# 給予執行權限
RUN chmod +x /etc/services.d/clickhouse/run
