ARG BUILD_FROM=ghcr.io/hassio-addons/debian-base:7.3.0

# 強制使用 Debian Base
FROM ghcr.io/hassio-addons/debian-base:7.3.0

# 安裝 ClickHouse
RUN apt-get update && apt-get install -y --no-install-recommends \
    clickhouse-server clickhouse-client curl \
    && rm -rf /var/lib/apt/lists/*

COPY rootfs /

# [NUCLEAR FIX] 建立啟動腳本
# 這次改用 users.d 覆蓋法，保證密碼一定生效
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
# [關鍵修正] 使用 users.d 強制覆蓋密碼\n\
# 我們建立一個獨立的設定檔，利用 ClickHouse 的 merge 機制移除舊密碼並設定新密碼\n\
if [ ! -z "$PASSWORD" ]; then\n\
    bashio::log.info "Creating password config in users.d/set_password.xml..."\n\
    mkdir -p /etc/clickhouse-server/users.d\n\
    cat > /etc/clickhouse-server/users.d/set_password.xml <<EOF\n\
<clickhouse>\n\
    <users>\n\
        <default>\n\
            \n\
            <password remove="1" />\n\
            <password_sha256_hex remove="1" />\n\
            \n\
            <password>$PASSWORD</password>\n\
        </default>\n\
    </users>\n\
</clickhouse>\n\
EOF\n\
fi\n\
\n\
# 強制開啟控制台日誌\n\
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
exec s6-setuidgid clickhouse "$BINARY" --config-file=/etc/clickhouse-server/config.xml\n\
' > /etc/services.d/clickhouse/run

# 給予執行權限
RUN chmod +x /etc/services.d/clickhouse/run
