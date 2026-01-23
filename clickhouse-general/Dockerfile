ARG BUILD_FROM=ghcr.io/hassio-addons/debian-base:7.3.0

# [FIX] 強制使用 Debian Base
FROM ghcr.io/hassio-addons/debian-base:7.3.0

# 安裝 ClickHouse (不需要安裝 dos2unix，因為我們改用內建的 sed)
RUN apt-get update && apt-get install -y --no-install-recommends \
    clickhouse-server clickhouse-client curl \
    && rm -rf /var/lib/apt/lists/*

COPY rootfs /

# 1. 建立資料夾
# 2. [關鍵修正] 使用 sed 移除 Windows 換行符號 (\r)
# 3. 給予執行權限
RUN mkdir -p /data/clickhouse \
    && chown -R clickhouse:clickhouse /data/clickhouse \
    && sed -i 's/\r$//' /etc/services.d/clickhouse/run \
    && chmod +x /etc/services.d/clickhouse/run

# [FIX] 確保移除 CMD /init 避免 PID 1 錯誤
