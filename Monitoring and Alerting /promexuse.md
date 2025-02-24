### 项目目标
```
部署一个以太坊 PoS 全节点，包括执行客户端（Execution Client，如 Geth）和共识客户端（Consensus Client，如 Lighthouse）。

使用 OpenResty + Lua 实现流量调控。

配置 Prometheus 和 Alertmanager，监控节点状态并通过 Slack 发送告警。

提供维护脚本和文档，确保节点的高可用性和可升级性。
```
### 开发步骤
1. 部署以太坊 PoS 节点
```

以太坊 PoS 需要运行一对客户端：
执行客户端（Execution Client）：Geth，负责交易执行和状态管理。

共识客户端（Consensus Client）：Lighthouse，负责 PoS 共识和 Beacon Chain 管理。

环境：Ubuntu 22.04，4 vCPU，16GB 内存。

步骤：
安装 Geth：
bash

sudo apt update && sudo apt install -y git golang
git clone https://github.com/ethereum/go-ethereum.git
cd go-ethereum
make geth

启动 Geth（默认 HTTP RPC 端口 8545）：
bash

./build/bin/geth --http --http.api eth,web3,net,engine --syncmode snap

安装 Lighthouse：
bash

sudo apt install -y curl build-essential
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env
git clone https://github.com/sigp/lighthouse.git
cd lighthouse
make

启动 Lighthouse（Beacon Chain，默认端口 5052）：
bash

lighthouse beacon_node --network mainnet --execution-endpoint http://localhost:8551 --execution-jwt /path/to/jwt.hex

注意：需要生成一个 JWT 密钥用于 Geth 和 Lighthouse 通信：
bash

openssl rand -hex 32 > /path/to/jwt.hex

在 Geth 启动参数中添加 --authrpc.jwtsecret /path/to/jwt.hex。

配置 systemd 服务：
Geth 服务（/etc/systemd/system/geth.service）：
ini

[Unit]
Description=Ethereum Geth Execution Client
After=network.target
[Service]
ExecStart=/home/ubuntu/go-ethereum/build/bin/geth --http --http.api eth,web3,net,engine --syncmode snap --authrpc.jwtsecret /path/to/jwt.hex
Restart=always
User=ubuntu
[Install]
WantedBy=multi-user.target

Lighthouse 服务（/etc/systemd/system/lighthouse.service）：
ini

[Unit]
Description=Ethereum Lighthouse Consensus Client
After=network.target
[Service]
ExecStart=/home/ubuntu/lighthouse/target/release/lighthouse bn --network mainnet --execution-endpoint http://localhost:8551 --execution-jwt /path/to/jwt.hex
Restart=always
User=ubuntu
[Install]
WantedBy=multi-user.target

bash

sudo systemctl enable geth.service lighthouse.service
sudo systemctl start geth.service lighthouse.service

成果：一个以太坊 PoS 全节点，提供 RPC 服务。
```
2. 配置 OpenResty + Lua 流量调控
```
安装 OpenResty：
bash

sudo apt install -y software-properties-common
sudo add-apt-repository ppa:openresty/ppa
sudo apt update && sudo apt install -y openresty

# /usr/local/openresty/nginx/conf/nginx.conf
http {
    lua_package_path "/usr/local/openresty/lualib/?.lua;;";
    lua_shared_dict limit_dict 10m;  # 共享内存存储请求计数

    server {
        listen 80;
        server_name _;

        location / {
            access_by_lua_block {
                local limit = ngx.shared.limit_dict
                local ip = ngx.var.remote_addr
                local key = "rate:" .. ip
                local requests, err = limit:get(key)

                if not requests then
                    limit:set(key, 1, 60)  -- 初次请求，设为 1，过期时间 60s
                elseif requests >= 10 then  -- 每分钟限制 10 次请求
                    ngx.log(ngx.ERR, "IP " .. ip .. " exceeded rate limit")
                    return ngx.exit(ngx.HTTP_TOO_MANY_REQUESTS)
                else
                    limit:incr(key, 1)
                end
            }
            proxy_pass http://127.0.0.1:8545;
            proxy_set_header Host $host;
        }
    }
}

重启：
bash

sudo systemctl restart openresty

成果：请求限制功能，保护节点免受滥用。
```
3. 配置 Prometheus + Alertmanager 监控
```
目标：监控 Geth 和 Lighthouse 的状态，检测同步延迟或服务宕机。

步骤：
安装 Node Exporter（服务器资源监控）：
bash

wget https://github.com/prometheus/node_exporter/releases/download/v1.8.0/node_exporter-1.8.0.linux-amd64.tar.gz
tar xvfz node_exporter-1.8.0.linux-amd64.tar.gz
sudo mv node_exporter-1.8.0.linux-amd64/node_exporter /usr/local/bin/
sudo nohup /usr/local/bin/node_exporter &

安装 Prometheus：
bash

wget https://github.com/prometheus/prometheus/releases/download/v2.53.0/prometheus-2.53.0.linux-amd64.tar.gz
tar xvfz prometheus-2.53.0.linux-amd64.tar.gz
cd prometheus-2.53.0.linux-amd64

配置 prometheus.yml：
yaml

global:
  scrape_interval: 15s
scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']
  - job_name: 'geth'
    static_configs:
      - targets: ['localhost:8545']
  - job_name: 'lighthouse'
    static_configs:
      - targets: ['localhost:5052']
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['localhost:9093']
rule_files:
  - "alert_rules.yml"  # 报警规则文件

启动：
bash

sudo nohup ./prometheus --config.file=prometheus.yml &

安装 Alertmanager：
bash

wget https://github.com/prometheus/alertmanager/releases/download/v0.27.0/alertmanager-0.27.0.linux-amd64.tar.gz
tar xvfz alertmanager-0.27.0.linux-amd64.tar.gz
cd alertmanager-0.27.0.linux-amd64

配置 alertmanager.yml：
yaml
global:
  resolve_timeout: 5m  # 问题解决后5分钟停止报警
  slack_api_url: 'https://hooks.slack.com/services/TXXXXX/BXXXXX/XXXXX'  # Slack Webhook URL

route:
  group_by: ['alertname', 'job']  # 按报警名称和工作分组
  group_wait: 30s  # 等待30秒收集同一组报警
  group_interval: 5m  # 每5分钟重复报警
  repeat_interval: 1h  # 每小时重复未解决的报警
  receiver: 'slack-notifications'

receivers:
  - name: 'slack-notifications'
    slack_configs:
      - channel: '#eth-monitoring'  # Slack 频道
        send_resolved: true  # 问题解决时发送通知
        title: '{{ .CommonLabels.alertname }} - {{ .CommonLabels.job }}'
        text: '{{ .CommonAnnotations.summary }}'

获取 Slack Webhook：
在 Slack 中创建 App，启用 Incoming Webhooks，生成 URL。

启动：
bash

sudo nohup ./alertmanager --config.file=alertmanager.yml &

定义告警规则（alert_rules.yml）：
yaml
groups:
  - name: EthereumNodeAlerts
    rules:
      # 共识客户端离线
      - alert: ConsensusClientDown
        expr: up{job="lighthouse"} == 0
        for: 5m  # 持续5分钟触发
        labels:
          severity: critical
        annotations:
          summary: "Consensus client {{ $labels.instance }} is down"
          description: "Lighthouse has been offline for 5 minutes."

      # 执行客户端离线
      - alert: ExecutionClientDown
        expr: up{job="geth"} == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Execution client {{ $labels.instance }} is down"
          description: "Geth has been offline for 5 minutes."

      # 同步延迟（共识客户端）
      - alert: ConsensusSyncLag
        expr: beacon_chain_head_slot - max(beacon_chain_head_slot) by (instance) > 32
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Consensus client {{ $labels.instance }} sync lagging"
          description: "Head slot is behind by more than 32 slots (1 epoch)."

      # 对等节点数过低
      - alert: LowPeerCount
        expr: beacon_peer_count < 10
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Low peer count on {{ $labels.instance }}"
          description: "Peer count below 10, risking sync issues."

      # 验证者证明失败率高
      - alert: ValidatorAttestationFailure
        expr: rate(validator_attestation_failures_total[5m]) / rate(validator_attestation_successes_total[5m]) > 0.1
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "High attestation failure rate on {{ $labels.instance }}"
          description: "Attestation failure rate exceeds 10%."
规则说明：
up == 0：检测客户端是否离线。
beacon_chain_head_slot：监控同步延迟（32 slots ≈ 1 epoch ≈ 6.4 分钟）。
beacon_peer_count：确保足够对等节点。
validator_attestation_failures_total：跟踪验证者证明失败率。


成果：Prometheus 监控节点状态，Alertmanager 通过 Slack 发送告警。
```
4. 维护与升级脚本
```
脚本（maintain_node.sh）：
bash

#!/bin/bash
echo "Checking node status..."
sudo systemctl status geth.service lighthouse.service

echo "Updating Geth..."
cd ~/go-ethereum
git pull
make geth
sudo systemctl restart geth.service

echo "Updating Lighthouse..."
cd ~/lighthouse
git pull
make
sudo systemctl restart lighthouse.service

echo "Optimizing system..."
sudo sysctl -w vm.swappiness=10
sudo sysctl -w net.ipv4.tcp_fin_timeout=30

执行：
bash

chmod +x maintain_node.sh
./maintain_node.sh

成果：自动化维护脚本，支持 PoS 节点的升级。
```


5、集成与验证
```
重启 Prometheus：
确保加载新规则：
bash
检查 Web UI（http://localhost:9090/alerts），确认规则生效。
测试报警：
模拟离线：
bash
sudo systemctl stop lighthouse  # 或 kill Lighthouse 进程
等待 5 分钟，检查 Slack 是否收到：
Consensus client localhost:5054 is down
Lighthouse has been offline for 5 minutes.
验证恢复通知：
重启客户端：
bash
sudo systemctl start lighthouse
确认 Slack 收到“Resolved”消息。
```


6、高级优化
```
多渠道通知：
添加 Email：
yaml
receivers:
  - name: 'slack-and-email'
    slack_configs:
      - channel: '#eth-monitoring'
    email_configs:
      - to: 'admin@example.com'
        from: 'alerts@example.com'
        smarthost: 'smtp.example.com:587'
        auth_username: 'user'
        auth_password: 'password'
抑制重复报警：
在 alertmanager.yml 中添加：
yaml
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['instance']
避免同一实例的次要报警重复触发。
```