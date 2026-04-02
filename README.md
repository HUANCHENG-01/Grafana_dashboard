部署 Prometheus + Node Exporter 主要分为服务端（Prometheus Server）和客户端（Node Exporter）两部分。为了让你快速上手，这里提供Linux 二进制部署（生产环境最常用）和Docker 快速启动两种方案。
一、方案选择：Linux 二进制部署（推荐生产环境） 
这是最稳定、可控性最高的方式，适合长期运行。
1. 部署 Node Exporter（在所有需要监控的机器上执行） 
Node Exporter 负责采集主机指标（CPU、内存、磁盘、网络等）。
步骤 1：下载并安装
# 以 root 用户操作
# 创建目录
mkdir -p /opt/node_exporter
cd /opt/node_exporter

# 下载最新版（请替换为实际版本号，如 1.7.0）
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz

# 解压并移动二进制文件
tar xvf node_exporter-*.tar.gz
cp node_exporter-*/node_exporter /usr/local/bin/
chmod +x /usr/local/bin/node_exporter
步骤 2：配置 Systemd 服务（守护进程）
创建服务文件 /etc/systemd/system/node-exporter.service：
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=root
ExecStart=/usr/local/bin/node_exporter
Restart=always

[Install]
WantedBy=multi-user.target
步骤 3：启动并验证
systemctl daemon-reload
systemctl start node-exporter
systemctl enable node-exporter

# 检查状态（应为 active）
systemctl status node-exporter

# 验证数据采集（应返回大量 metrics 文本）
curl http://localhost:9100/metrics
2. 部署 Prometheus Server（在监控服务器上执行） 
Prometheus Server 负责拉取并存储数据。
步骤 1：下载并安装
mkdir -p /opt/prometheus
cd /opt/prometheus

# 下载 Prometheus（替换为实际版本号，如 2.53.0）
wget https://github.com/prometheus/prometheus/releases/download/v2.53.0/prometheus-2.53.0.linux-amd64.tar.gz

tar xvf prometheus-*.tar.gz
cp prometheus-*/prometheus /usr/local/bin/
cp prometheus-*/promtool /usr/local/bin/
chmod +x /usr/local/bin/prometheus

# 创建配置文件目录
mkdir /etc/prometheus
cp prometheus-*/prometheus.yml /etc/prometheus/
步骤 2：配置抓取目标
编辑 /etc/prometheus/prometheus.yml，在 scrape_configs部分添加 Node Exporter 任务：
scrape_configs:
  # 监控 Prometheus 自身
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # 监控 Linux 主机（替换 IP 为你的实际机器 IP）
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['192.168.1.10:9100', '192.168.1.11:9100']  # 多个机器用逗号分隔
    # 可选：调整抓取间隔
    scrape_interval: 15s
步骤 3：启动 Prometheus
创建服务文件 /etc/systemd/system/prometheus.service：
[Unit]
Description=Prometheus Server
After=network.target

[Service]
User=root
ExecStart=/usr/local/bin/prometheus \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/var/lib/prometheus/data \
    --web.listen-address=:9090
Restart=always

[Install]
WantedBy=multi-user.target
启动服务：
systemctl daemon-reload
systemctl start prometheus
systemctl enable prometheus

# 访问验证：浏览器打开 http://<服务器IP>:9090

---
二、方案二：Docker 快速部署（适合测试/开发） 
如果你熟悉 Docker，可以使用以下命令快速拉起服务。
1. 启动 Node Exporter（在目标主机上）
docker run -d \
  --name node-exporter \
  --restart=always \
  --net="host" \
  -p 9100:9100 \
  prom/node-exporter:latest
2. 启动 Prometheus Server
首先创建配置文件 prometheus.yml（内容同上），然后运行：
docker run -d \
  --name prometheus \
  --restart=always \
  -p 9090:9090 \
  -v $(pwd)/prometheus.yml:/etc/prometheus/prometheus.yml \
  -v prometheus-data:/prometheus \
  prom/prometheus:latest

---
三、关键验证与排错 
部署完成后，请按以下步骤验证：
1. 检查端口连通性：在 Prometheus 服务器上执行 curl http://<目标机IP>:9100/metrics，确保能获取数据。
2. 查看 Targets：浏览器访问 http://<Prometheus_IP>:9090/targets，确认 node_exporter任务状态为 UP。
3. 查询数据：在 Prometheus 的 Graph 页面输入 node_memory_MemTotal_bytes，点击 Execute 查看是否有数据曲线。
四、安全建议（生产环境必做） 
- 防火墙：仅允许 Prometheus Server 的 IP 访问 9100 端口（Node Exporter 端口）。
- 内网部署：建议将 Node Exporter 监听地址改为内网 IP（启动参数 --web.listen-address=192.168.x.x:9100）。
- 反向代理：可通过 Nginx 添加 Basic Auth 或 HTTPS 加密。
五、使用 Grafana 持久化监控面板（推荐） 
Grafana 是专业的监控可视化工具，支持保存仪表盘（Dashboard）、设置告警、自定义样式，且能与 Prometheus 无缝集成。
1. 安装并启动 Grafana 
- Linux（Ubuntu/Debian）：
# 添加 Grafana 源
sudo apt-get install -y software-properties-common
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
sudo apt-get update
sudo apt-get install grafana

# 启动 Grafana 服务
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
- Docker：
docker run -d -p 3000:3000 --name=grafana grafana/grafana
2. 配置 Prometheus 数据源 
1. 访问 Grafana：打开浏览器，输入 http://<你的服务器IP>:3000（默认账号 admin/密码 admin，首次登录需修改密码）。
2. 进入 Configuration → Data Sources → Add data source，选择 Prometheus。
3. 填写 Prometheus 的访问地址（如 http://192.168.31.85:9090），点击 Save & Test 确保连接成功。
3. 创建并保存监控面板 
1. 进入 Create → Dashboard → Add a new panel。
2. 在 Query 标签页，选择 Prometheus 数据源，输入你的监控查询（如 CPU 使用率）：
#CPU使用率
100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

#内存使用率
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
CPU 使用率需要按 instance拆分查询，否则会聚合所有服务器的 CPU 使用率，无法区分。
内存使用率的查询语句利用了指标的 instance标签分组特性，一条语句就能自动按服务器分组计算，因此可以同时展示多个服务器的内存使用率。
1. 配置图表的 Title、Type（如 Graph、Stat、Table 等）、Legend、Axes 等样式。
2. 点击右上角 Apply 保存面板，然后点击 Save dashboard（输入名称，如“服务器监控”），选择 Save 即可永久保存。
4. 自定义Legend名称
1. 进入面板编辑模式，在查询编辑器下方找到 「Transformations」 选项卡。
2. 点击「Add transformation」，选择 「Rename by regex」（或「Rename」）。
3. 在「Regex」中输入匹配原图例名称的正则（如 .instance="129.204.xx.xx:9100".，或直接匹配默认图例）。
4. 在「Replacement」中输入新的名称（如 服务器A）。
5. 点击「Apply」，图例名称会被批量替换。

5. 复用已有面板（可选） 
Grafana 社区有大量现成的 Prometheus 仪表盘模板（如 Node Exporter Full），可直接导入：
1. 访问 Grafana Dashboards，搜索你需要的模板（如“Node Exporter”）。
2. 复制模板 ID（如 1860），在 Grafana 中进入 Create → Import，输入 ID 并选择 Prometheus 数据源，即可一键导入完整面板。
