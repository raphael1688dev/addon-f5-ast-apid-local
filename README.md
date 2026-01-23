# F5 AST API Discovery (Local)

**Installation:**
Add this URL to Home Assistant Add-on Store:
`https://github.com/raphael1688dev/addon-f5-ast-apid-local`

**Setup:**

1.  **General ClickHouse**:
    * Install and start.
    * It opens port `19000` (TCP) and `18123` (HTTP).

2.  **F5 AST Logic**:
    * Install.
    * In **Configuration**, set `db_host` to your NUC's IP address (e.g., `192.168.1.X`) and `db_port` to `19000`.
    * Start the add-on.
    * **Prometheus Metrics** are available at port `19091`.

**Grafana Configuration:**
* **ClickHouse Source**: URL `http://<NUC_IP>:18123`
* **Prometheus Source**: URL `http://<NUC_IP>:19091`
