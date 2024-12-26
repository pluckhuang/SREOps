## bitcoin 剪枝节点配置, 使用 aws 测试

### 客户端使用 bitcoin core
```
[ec2-user@ip-172-31-15-158 home]$ tree -L 2 bitcoin-28.0/
bitcoin-28.0/
├── README.md
├── bin
│   ├── bitcoin-cli
│   ├── bitcoin-qt
│   ├── bitcoin-tx
│   ├── bitcoin-util
│   ├── bitcoin-wallet
│   ├── bitcoind
│   └── test_bitcoin
├── bitcoin.conf
└── share
    ├── man
    └── rpcauth
```
### bitcoind 配置文件
```
 ~/.bitcoin/bitcoin.conf
# 启用服务器模式以允许 RPC 连接
server=1
# 设置RPC用户名和密码（请自定义）
rpcuser=yourusername
rpcpassword=yourpassword
# 启用剪枝模式，并设置剪枝大小为550 MB（550 MiB）
prune=550
# 减少区块验证的额外检查，适用于硬件资源有限的节点
par=1
# 禁用交易索引以节省空间
txindex=0
# 减少内存使用
dbcache=4
# 降低CPU使用率
maxorphantx=10
# 减少网络连接数量
maxconnections=10
# listen=0
# 默认情况下为1，Bitcoin Core 节点会开启监听功能，允许其他节点连接到它，接受传入的连接请求以提供区块和交易数据。
blocksonly=1 # 块模式, 只是参与区块的同步, 不参与交易的传播

```
### 常规命令
```
1.启动节点
bitcoind -daemon
2.检查节点状态：
bitcoin-cli getblockcount
3.停止节点
bitcoin-cli stop
```
### 设置服务为自动重启
sudo vim /etc/systemd/system/bitcoind.service
```
[Unit]
Description=Bitcoin Core Daemon
After=network.target
Wants=network-online.target

[Service]
Type=forking
ExecStart=/home/bitcoin-28.0/bin/bitcoind -daemon -pid=/home/bitcoin-28.0/.bitcoin/bitcoind.pid
User=ec2-user
Group=ec2-user
Restart=on-failure
RestartSec=20s

# 设置工作目录
WorkingDirectory=/home/bitcoin-28.0/.bitcoin

# 日志配置
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```
sudo systemctl daemon-reload
sudo systemctl start bitcoind.service

### rpc 命令测试
```
curl --user yourusernamePluckh:yourpasswordPluckh --data-binary '{"jsonrpc": "1.0", "method": "getblockcount", "params": [] }' -H 'content-type: text/plain;' http://localhost:8332/
```

### 总结
```
剪枝节点不存储所有历史区块数据,  只保存设定prune=大小数据, 作用有限,只能查询部分交易记录,  无法查询用户历史有所交易记录.
```