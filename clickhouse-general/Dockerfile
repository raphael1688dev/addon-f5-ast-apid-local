ARG BUILD_FROM=ghcr.io/hassio-addons/debian-base:7.3.0

# 強制使用 Debian Base
FROM ghcr.io/hassio-addons/debian-base:7.3.0

# 更新 ClickHouse 安裝來源 (維持使用最新版)
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl gnupg2 ca-certificates lsb-release \
    && curl -fsSL 'https://packages.clickhouse.com/rpm/lts/repodata/repomd.xml.key' | gpg --dearmor -o /usr/share/keyrings/clickhouse-keyring.gpg \
    && echo "deb [signed-by=/usr/share/keyrings/clickhouse-keyring.gpg] https://packages.clickhouse.com/deb stable main" > /etc/apt/sources.list.d/clickhouse.list \
    && apt-get update \
    && apt-get install -y --no-install-recommends clickhouse-server clickhouse-client \
    && rm -rf /var/lib/apt/lists/*

COPY rootfs /

# [NUCLEAR OPTION] 建立啟動腳本
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
# [核心修正] 刪除預設設定檔\n\
rm -f /etc/clickhouse-server/users.xml\n\
rm -rf /etc/clickhouse-server/users.d/*\n\
\n\
# 產生全新的 users.xml\n\
# [FIX] 將 max_memory_usage 正確放置於 profiles/default 下\n\
bashio::log.info "Generating fresh users.xml with configured password and limits..."\n\
cat > /etc/clickhouse-server/users.xml <<EOF\n\
<clickhouse>\n\
    <profiles>\n\
        <default>\n\
            \n\
            <max_memory_usage>$MAX_MEM</max_memory_usage>\n\
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
# [FIX] 產生 tuning.xml\n\
# 移除 max_memory_usage (因為它是 User Setting)\n\
# 保留 max_server_memory_usage (因為它是 Server Setting)\n\
cat > /etc/clickhouse-server/config.d/tuning.xml <<EOF\n\
<clickhouse>\n\
    <max_server_memory_usage>$MAX_SERVER_MEM</max_server_memory_usage>\n\
    <logger><console>1</console><level>information</level></logger>\n\
    \n\
    <listen_host>0.0.0.0</listen_host>\n\
</clickhouse>\n\
EOF\n\
\n\
bashio::log.info "Starting ClickHouse..."\n\
exec s6-setuidgid clickhouse "$BINARY" --config-file=/etc/clickhouse-server/config.xml\n\
' > /etc/services.d/clickhouse/run

# 給予執行權限
RUN chmod +x /etc/services.d/clickhouse/run
