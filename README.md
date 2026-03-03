# Republic
Republic AI 测试网节点安装指南

# Republic AI 测试网节点安装指南

> **兼容 Ubuntu 22.04** - 包含使用 patchelf 方法修复 GLIBC 2.38+ 的解决方案

本指南通过解决 GLIBC 版本不兼容问题，帮助您在 Ubuntu 22.04 上运行 Republic AI 测试网节点。

## 📋 网络信息

| 属性 | 值 |
|----------|-------|
| Chain ID | `raitestnet_77701-1` |
| EVM Chain ID | `77701` |
| Denom | `arai` (基础单位), `RAI` (显示单位) |
| 小数位数 | `18` |
| 最低 Gas 价格 | `250000000arai` |
| 二进制版本 | `v0.1.0` |

### 公共端点

| 服务 | URL |
|---------|-----|
| Cosmos RPC | https://rpc.republicai.io |
| REST API | https://rest.republicai.io |
| gRPC | grpc.republicai.io:443 |
| EVM JSON-RPC | https://evm-rpc.republicai.io |

---

## 🔧 GLIBC 问题及解决方案

### 问题
官方的 `republicd` 二进制文件需要 GLIBC 2.38+，但 Ubuntu 22.04 只有 GLIBC 2.35。

```
republicd: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.38' not found
```

### 解决方案
我们使用 **patchelf** 让二进制文件使用独立的 GLIBC 2.39 库，而不触碰系统的 GLIBC。这样可以保持您的系统和其他应用程序完全安全。

```
系统 GLIBC (不触碰)          独立的 GLIBC 2.39
/lib/x86_64-linux-gnu/       /opt/glibc-2.39/lib/
├── libc.so.6 (2.35)         ├── libc.so.6 (2.39)
│                            │
│ ← 其他应用使用这个          │ ← 只有 republicd 使用这个
```

---

## 🚀 安装步骤

### 步骤 1：设置端口前缀

选择一个不与其他服务冲突的 2 位数端口前缀（例如 `36`、`37`、`38`）。

```bash
# 设置您的端口前缀（将 XX 改为您喜欢的 2 位数字）
PORT_PREFIX=36
```

**常用端口前缀参考：**
- 默认 Cosmos：`26`（26656、26657...）
- 自定义示例：`36`、`37`、`38`、`39`...

### 步骤 2：设置变量

```bash
# 节点配置
MONIKER="your-node-name"
CHAIN_ID="raitestnet_77701-1"
REPUBLIC_HOME="$HOME/.republicd"

# 端口（根据前缀自动计算）
P2P_PORT="${PORT_PREFIX}656"
RPC_PORT="${PORT_PREFIX}657"
GRPC_PORT="${PORT_PREFIX}090"
API_PORT="${PORT_PREFIX}317"
PPROF_PORT="${PORT_PREFIX}060"
PROMETHEUS_PORT="${PORT_PREFIX}660"
EVM_RPC_PORT="${PORT_PREFIX}545"
EVM_WS_PORT="${PORT_PREFIX}546"

# 保存到配置文件
echo "export REPUBLIC_HOME=$REPUBLIC_HOME" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

### 步骤 3：安装依赖项

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git jq patchelf
```

### 步骤 4：安装 GLIBC 2.39（隔离安装）

```bash
# 下载预编译的 GLIBC 2.39
cd $HOME
wget -O glibc-2.39-ubuntu24.tar.gz https://raw.githubusercontent.com/coinsspor/coinsspor/main/glibc-2.39-ubuntu24.tar.gz

# 解压
tar -xzvf glibc-2.39-ubuntu24.tar.gz

# 移动到 /opt（隔离位置）
sudo mkdir -p /opt/glibc-2.39/lib
sudo mv glibc-transfer/* /opt/glibc-2.39/lib/

# 验证
/opt/glibc-2.39/lib/ld-linux-x86-64.so.2 --version

# 清理
rm -rf glibc-transfer glibc-2.39-ubuntu24.tar.gz
```

您应该看到：`ld.so (Ubuntu GLIBC 2.39...) stable release version 2.39`

### 步骤 5：安装 Go（如果尚未安装）

```bash
# 检查是否已安装 Go
go version || {
    cd $HOME
    GO_VERSION="1.22.5"
    wget "https://golang.org/dl/go${GO_VERSION}.linux-amd64.tar.gz"
    sudo rm -rf /usr/local/go
    sudo tar -C /usr/local -xzf "go${GO_VERSION}.linux-amd64.tar.gz"
    rm "go${GO_VERSION}.linux-amd64.tar.gz"

    echo 'export GOROOT=/usr/local/go' >> $HOME/.bash_profile
    echo 'export GOPATH=$HOME/go' >> $HOME/.bash_profile
    echo 'export GO111MODULE=on' >> $HOME/.bash_profile
    echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.bash_profile
    source $HOME/.bash_profile
}

go version
```

### 步骤 6：安装 Cosmovisor（如果尚未安装）

```bash
which cosmovisor || go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@latest
```

### 步骤 7：下载并修补 Republic 二进制文件

```bash
# 下载二进制文件
cd $HOME
VERSION="v0.1.0"
curl -L "https://media.githubusercontent.com/media/RepublicAI/networks/main/testnet/releases/${VERSION}/republicd-linux-amd64" -o republicd
chmod +x republicd

# 修补二进制文件以使用 GLIBC 2.39
patchelf --set-interpreter /opt/glibc-2.39/lib/ld-linux-x86-64.so.2 republicd
patchelf --set-rpath /opt/glibc-2.39/lib republicd

# 移动到 bin 目录
sudo mv republicd /usr/local/bin/republicd

# 测试 - 应该显示版本而不出现 GLIBC 错误
republicd version
```

### 步骤 8：初始化节点

```bash
# 初始化
republicd init $MONIKER --chain-id $CHAIN_ID --home $REPUBLIC_HOME

# 下载创世文件
curl -s https://raw.githubusercontent.com/RepublicAI/networks/main/testnet/genesis.json > $REPUBLIC_HOME/config/genesis.json

# 验证创世文件
sha256sum $REPUBLIC_HOME/config/genesis.json
```

### 步骤 9：设置 Cosmovisor

```bash
# 创建目录
mkdir -p $REPUBLIC_HOME/cosmovisor/genesis/bin
mkdir -p $REPUBLIC_HOME/cosmovisor/upgrades

# 复制二进制文件
cp /usr/local/bin/republicd $REPUBLIC_HOME/cosmovisor/genesis/bin/

# 验证
ls -la $REPUBLIC_HOME/cosmovisor/genesis/bin/
```

### 步骤 10：配置对等节点

```bash
PEERS="e281dc6e4ebf5e32fb7e6c4a111c06f02a1d4d62@3.92.139.74:26656,cfb2cb90a241f7e1c076a43954f0ee6d42794d04@54.173.6.183:26656,dc254b98cebd6383ed8cf2e766557e3d240100a9@54.227.57.160:26656"
sed -i "s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $REPUBLIC_HOME/config/config.toml
```

### 步骤 11：配置自定义端口

```bash
# config.toml
sed -i "s|laddr = \"tcp://0.0.0.0:26656\"|laddr = \"tcp://0.0.0.0:$P2P_PORT\"|" $REPUBLIC_HOME/config/config.toml
sed -i "s|laddr = \"tcp://127.0.0.1:26657\"|laddr = \"tcp://127.0.0.1:$RPC_PORT\"|" $REPUBLIC_HOME/config/config.toml
sed -i "s|pprof_laddr = \"localhost:6060\"|pprof_laddr = \"localhost:$PPROF_PORT\"|" $REPUBLIC_HOME/config/config.toml
sed -i "s|prometheus_listen_addr = \":26660\"|prometheus_listen_addr = \":$PROMETHEUS_PORT\"|" $REPUBLIC_HOME/config/config.toml

# app.toml
sed -i "s|address = \"tcp://localhost:1317\"|address = \"tcp://localhost:$API_PORT\"|" $REPUBLIC_HOME/config/app.toml
sed -i "s|address = \"localhost:9090\"|address = \"localhost:$GRPC_PORT\"|" $REPUBLIC_HOME/config/app.toml
sed -i "s|address = \"127.0.0.1:8545\"|address = \"127.0.0.1:$EVM_RPC_PORT\"|" $REPUBLIC_HOME/config/app.toml
sed -i "s|ws-address = \"127.0.0.1:8546\"|ws-address = \"127.0.0.1:$EVM_WS_PORT\"|" $REPUBLIC_HOME/config/app.toml

# 验证端口
echo "=== 您的端口配置 ==="
echo "P2P:        $P2P_PORT"
echo "RPC:        $RPC_PORT"
echo "gRPC:       $GRPC_PORT"
echo "API:        $API_PORT"
echo "EVM RPC:    $EVM_RPC_PORT"
echo "EVM WS:     $EVM_WS_PORT"
echo "Prometheus: $PROMETHEUS_PORT"
```

### 步骤 12：配置 Gas 价格和修剪

```bash
# 设置最低 gas 价格
sed -i 's/minimum-gas-prices = "0arai"/minimum-gas-prices = "250000000arai"/' $REPUBLIC_HOME/config/app.toml

# 配置修剪（可选 - 节省磁盘空间）
sed -i -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-keep-every *=.*|pruning-keep-every = "0"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "19"|' \
  $REPUBLIC_HOME/config/app.toml
```

### 步骤 13：创建 Systemd 服务

```bash
sudo tee /etc/systemd/system/republicd.service > /dev/null << EOF
[Unit]
Description=Republic AI Node
After=network-online.target

[Service]
User=$USER
ExecStart=$(which cosmovisor) run start --home $REPUBLIC_HOME
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_HOME=$REPUBLIC_HOME"
Environment="DAEMON_NAME=republicd"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=false"
Environment="DAEMON_RESTART_AFTER_UPGRADE=true"
Environment="UNSAFE_SKIP_BACKUP=true"

[Install]
WantedBy=multi-user.target
EOF
```

### 步骤 14：启动节点

```bash
# 重新加载 systemd
sudo systemctl daemon-reload

# 启用服务
sudo systemctl enable republicd

# 启动节点
sudo systemctl start republicd

# 查看日志
sudo journalctl -u republicd -f -o cat
```

---

## ✅ 验证安装

### 检查同步状态

```bash
republicd status --home $REPUBLIC_HOME 2>&1 | jq '.sync_info'
```

### 检查服务状态

```bash
sudo systemctl status republicd
```

### 查看日志

```bash
sudo journalctl -u republicd -f -o cat
```

---

## 👛 钱包操作

### 创建新钱包

```bash
republicd keys add wallet --home $REPUBLIC_HOME
```

> ⚠️ **重要提示：** 请安全保存您的助记词！

### 导入现有钱包

```bash
republicd keys add wallet --recover --home $REPUBLIC_HOME
```

### 查看钱包地址

```bash
republicd keys show wallet -a --home $REPUBLIC_HOME
```

### 查看余额

```bash
republicd query bank balances $(republicd keys show wallet -a --home $REPUBLIC_HOME) --home $REPUBLIC_HOME
```

---

## 🗳️ 创建验证者

> 等待您的节点完全同步（`catching_up: false`）

### 获取测试网代币

从水龙头请求代币：https://points.republicai.io/faucet

### 创建验证者

```bash
republicd tx staking create-validator \
  --amount=1000000000000000000000arai \
  --pubkey=$(republicd comet show-validator --home $REPUBLIC_HOME) \
  --moniker="$MONIKER" \
  --identity="" \
  --details="您的验证者描述" \
  --website="https://your-website.com" \
  --chain-id=$CHAIN_ID \
  --commission-rate="0.10" \
  --commission-max-rate="0.20" \
  --commission-max-change-rate="0.01" \
  --min-self-delegation="1" \
  --gas=auto \
  --gas-adjustment=1.5 \
  --gas-prices="250000000arai" \
  --from=wallet \
  --home $REPUBLIC_HOME \
  -y
```

> **注意：** 最低自我委托为 1000 RAI = `1000000000000000000000arai`

---

## 📚 常用命令

### 服务管理

```bash
# 检查状态
sudo systemctl status republicd

# 重启
sudo systemctl restart republicd

# 停止
sudo systemctl stop republicd

# 查看日志
sudo journalctl -u republicd -f -o cat
```

### 质押操作

```bash
# 委托
republicd tx staking delegate <VALIDATOR_ADDRESS> <AMOUNT>arai \
  --from wallet --chain-id $CHAIN_ID \
  --gas auto --gas-adjustment 1.5 --gas-prices 250000000arai \
  --home $REPUBLIC_HOME -y

# 解除监禁
republicd tx slashing unjail \
  --from wallet --chain-id $CHAIN_ID \
  --gas auto --gas-adjustment 1.5 --gas-prices 250000000arai \
  --home $REPUBLIC_HOME -y

# 提取奖励
republicd tx distribution withdraw-all-rewards \
  --from wallet --chain-id $CHAIN_ID \
  --gas auto --gas-adjustment 1.5 --gas-prices 250000000arai \
  --home $REPUBLIC_HOME -y
```

### 节点信息

```bash
# 同步状态
republicd status --home $REPUBLIC_HOME 2>&1 | jq '.sync_info'

# 节点 ID
republicd comet show-node-id --home $REPUBLIC_HOME

# 验证者信息
republicd query staking validator $(republicd keys show wallet --bech val -a --home $REPUBLIC_HOME) --home $REPUBLIC_HOME
```

---

## 🔄 升级二进制文件（未来更新）

当发布新版本时：

```bash
# 下载新的二进制文件
curl -L "https://media.githubusercontent.com/media/RepublicAI/networks/main/testnet/releases/NEW_VERSION/republicd-linux-amd64" -o republicd_new
chmod +x republicd_new

# 修补它
patchelf --set-interpreter /opt/glibc-2.39/lib/ld-linux-x86-64.so.2 republicd_new
patchelf --set-rpath /opt/glibc-2.39/lib republicd_new

# 对于 Cosmovisor 自动升级，放置在 upgrades 文件夹中：
mkdir -p $REPUBLIC_HOME/cosmovisor/upgrades/<upgrade-name>/bin
mv republicd_new $REPUBLIC_HOME/cosmovisor/upgrades/<upgrade-name>/bin/republicd
```

---

## 🛠️ 故障排除

### 仍然出现 GLIBC 错误

```bash
# 验证 patchelf 设置
patchelf --print-interpreter /usr/local/bin/republicd
patchelf --print-rpath /usr/local/bin/republicd

# 应该显示：
# /opt/glibc-2.39/lib/ld-linux-x86-64.so.2
# /opt/glibc-2.39/lib
```

### 端口冲突

```bash
# 检查端口是否被占用
sudo lsof -i :${P2P_PORT}
sudo lsof -i :${RPC_PORT}
```

### 节点未同步

```bash
# 检查对等节点
curl -s localhost:${RPC_PORT}/net_info | jq '.result.n_peers'

# 如果需要，手动添加更多对等节点
```

---

## 🔗 资源

- **GitHub：** https://github.com/RepublicAI/networks
- **Discord：** https://discord.gg/xxvuGddz
- **文档：** https://github.com/RepublicAI/networks/issues

---

## 📝 致谢

- Republic AI 团队提供测试网

---

