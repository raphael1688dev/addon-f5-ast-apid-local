ARG BUILD_FROM=ghcr.io/hassio-addons/debian-base:7.3.0

# 強制使用 Debian Base
FROM ghcr.io/hassio-addons/debian-base:7.3.0

# 安裝 ClickHouse
RUN apt-get update && apt-get install -y --no-install-recommends \
    clickhouse-server clickhouse-client curl \
    && rm -rf /var/lib/apt/lists/*

COPY rootfs /

# [關鍵] 1. 先建立乾淨的設定檔樣板 (Template)
# 這裡用 __PASSWORD__ 當作佔位符，避免在 shell script 裡處理複雜字串
RUN mkdir -p /etc/clickhouse-server/users.d && \
    printf '<clickhouse>\n\
    <users>\n\
        <default>\n\
            <password remove="1" />\n\
            <password_sha256_hex remove="1" />\n\
            <password>__PASSWORD__</password>\n\
            <networks>\n\
                <ip>::/0</ip>\n\
            </networks>\n\
        </default>\n\
    </users>\n\
</clickhouse>\n' > /etc/clickhouse-server/users.d/user_config_template.xml

# [關鍵] 2. 建立啟動腳本
# 啟動時只需要做簡單的文字替換，絕對不會出錯
RUN printf '#!/usr/bin/with-contenv bashio\n\
PASSWORD=$(bashio::config "password")\n\
MAX_MEM=$(bashio::config "max_memory_usage")\n\
MAX_SERVER_MEM=$(bashio::config "max_server_memory")\n\
\n\
# 自動尋找執行檔\n\
if [ -f "/usr/sbin/clickhouse-server" ]; then\n\
    BINARY="/usr/sbin/clickhouse-server"\n\
else\n\
    BINARY="/usr/bin/clickhouse-server"\n\
fi\n\
\n\
# 1. 處理資料夾\n\
if [ ! -d "/data/clickhouse" ]; then\n\
    bashio::log.info "Creating data directory..."\n\
    mkdir -p /data/clickhouse\n\
fi\n\
chown -R clickhouse:clickhouse /data/clickhouse\n\
\n\
# 2. [核心] 套用密碼\n\
# 如果使用者有設定密碼，就從樣板產生正式設定檔\n\
if [ ! -z "$PASSWORD" ]; then\n\
    bashio::log.info "Enforcing configured password..."\n\
    sed "s|__PASSWORD__|$PASSWORD|g" /etc/clickhouse-server/users.d/user_config_template.xml > /etc/clickhouse-server/users.d/default_password.xml\n\
else\n\
    bashio::log.warning "No password configured! Server is running in unsafe mode."\n\
fi\n\
\n\
# 3. 產生效能與日誌設定\n\
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
