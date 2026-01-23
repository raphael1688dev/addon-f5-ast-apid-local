ARG BUILD_FROM=ghcr.io/hassio-addons/debian-base:7.3.0

# 強制使用 Debian Base
FROM ghcr.io/hassio-addons/debian-base:7.3.0

# 安裝 ClickHouse
RUN apt-get update && apt-get install -y --no-install-recommends \
    clickhouse-server clickhouse-client curl \
    && rm -rf /var/lib/apt/lists/*

COPY rootfs /

# [NUCLEAR OPTION] 建立啟動腳本
# 策略：完全捨棄系統預設的 users.xml，直接產生一份全新的
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
\n\
# 確保資料夾存在\n\
if [ ! -d "/data/clickhouse" ]; then\n\
    bashio::log.info "Creating data directory..."\n\
    mkdir -p /data/clickhouse\n\
fi\n\
chown -R clickhouse:clickhouse /data/clickhouse\n\
\n\
# [核心修正] 刪除預設設定檔，避免任何合併衝突\n\
rm -f /etc/clickhouse-server/users.xml\n\
rm -rf /etc/clickhouse-server/users.d/*\n\
\n\
# 產生全新的 users.xml (包含 Profile, Quota 和 User)\n\
# 這樣做保證 default 使用者只有一個密碼設定來源\n\
bashio::log.info "Generating fresh users.xml with configured password..."\n\
cat > /etc/clickhouse-server/users.xml <<EOF\n\
<clickhouse>\n\
    <profiles>\n\
        <default>\n\
            <max_memory_usage>10000000000</max_memory_usage>\n\
            <use_uncompressed_cache>0</use_uncompressed_cache>\n\
            <load_balancing>random</load_balancing>\n\
        </default>\n\
        <readonly>\n\
            <readonly>1</readonly>\n\
        </readonly>\n\
    </profiles>\n\
    <users>\n\
        <default>\n\
            <password>$PASSWORD</password>\n\
            <networks>\n\
                <ip>::/0</ip>\n\
            </networks>\n\
            <profile>default</profile>\n\
            <quota>default</quota>\n\
            <access_management>1</access_management>\n\
        </default>\n\
    </users>\n\
    <quotas>\n\
        <default>\n\
            <interval>\n\
                <duration>3600</duration>\n\
                <queries>0</queries>\n\
                <errors>0</errors>\n\
                <result_rows>0</result_rows>\n\
                <read_rows>0</read_rows>\n\
                <execution_time>0</execution_time>\n\
            </interval>\n\
        </default>\n\
    </quotas>\n\
</clickhouse>\n\
EOF\n\
\n\
# 產生效能與日誌設定\n\
cat > /etc/clickhouse-server/config.d/tuning.xml <<EOF\n\
<clickhouse>\n\
    <max_memory_usage>$MAX_MEM</max_memory_usage>\n\
    <max_server_memory_usage>$MAX_SERVER_MEM</max_server_memory_usage>\n\
    <logger><console>1</console><level>information</level></logger>\n\
</clickhouse>\n\
EOF\n\
\n\
bashio::log.info "Starting ClickHouse..."\n\
exec s6-setuidgid clickhouse "$BINARY" --config-file=/etc/clickhouse-server/config.xml\n\
' > /etc/services.d/clickhouse/run

# 給予執行權限
RUN chmod +x /etc/services.d/clickhouse/run
