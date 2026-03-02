# ServerStatus-Starter

基于 [cppla/ServerStatus](https://github.com/cppla/ServerStatus) 的快速部署工具，在原版基础上增加了：

- 节点管理交互式 CLI（增删改查）
- Telegram 上下线通知
- Agent 使用 systemd 管理，支持开机自启

---

## 前置要求

服务端机器需要：

- Docker
- Docker Compose
- Python 3
- root 权限

---

## 一、服务端部署（一键）

在**服务端**执行以下命令，将 `YOUR_TG_CHAT_ID` 和 `YOUR_TG_BOT_TOKEN` 替换成你自己的值：

```bash
mkdir sss && cd sss \
  && wget --no-check-certificate https://raw.githubusercontent.com/zqcccc/ServerStatus-Starter/master/sss.sh -O sss.sh \
  && chmod +x sss.sh \
  && sudo ./sss.sh YOUR_TG_CHAT_ID YOUR_TG_BOT_TOKEN
```

> **Telegram Bot 获取方式**
> - Bot Token：与 [@BotFather](https://t.me/BotFather) 对话创建机器人获取
> - Chat ID：与 [@userinfobot](https://t.me/userinfobot) 对话获取

安装完成后：
- 面板地址：`http://<服务端IP>:8081`
- 客户端上报端口：`35601`

> **注意**：需要确保服务端防火墙放行了 8081（Web 面板）和 35601（客户端上报）端口，详见 [防火墙配置](#六防火墙配置)。

---

## 二、更新管理脚本

在**已部署的服务端**上，重新运行 `sss.sh`（不需要带参数）即可拉取最新的 `_sss.py` 和 `bot.py`：

```bash
cd sss && sudo ./sss.sh
```

脚本会自动：
1. 检测到服务已在运行，跳过安装步骤
2. 从仓库拉取最新的 `_sss.py` 和 `bot.py`
3. 重新构建 bot 容器使更新生效
4. 进入节点管理 CLI

---

## 三、节点管理

部署完成后会自动进入节点管理 CLI，后续也可以在 `sss` 目录下随时运行：

```bash
cd sss && python3 _sss.py
```

菜单操作：

```
1. 查看  —— 列出所有节点，选择某个节点可查看完整信息和 Agent 安装命令
2. 添加  —— 添加新节点，添加后会输出该节点的 Agent 安装命令
3. 删除  —— 删除节点
4. 更新  —— 修改节点名称/位置/类型等信息
0. 退出
```

添加节点后，CLI 会自动输出一条安装命令，复制到对应节点机器上执行即可。

---

## 四、节点安装 Agent

在**需要监控的节点**上以 root 执行（命令由第二步的 CLI 生成）：

```bash
curl -L https://raw.githubusercontent.com/zqcccc/ServerStatus-Starter/master/sss-agent.sh \
  -o sss-agent.sh && chmod +x sss-agent.sh \
  && sudo ./sss-agent.sh <服务端IP> <username> <password>
```

Agent 通过 systemd 管理，安装后自动启动并开机自启。

常用管理命令：

```bash
# 查看运行状态
systemctl status sss-agent

# 查看日志
journalctl -u sss-agent -f

# 重启
systemctl restart sss-agent

# 卸载
curl -L https://raw.githubusercontent.com/zqcccc/ServerStatus-Starter/master/sss-agent.sh \
  -o sss-agent.sh && chmod +x sss-agent.sh && sudo ./sss-agent.sh
# 然后选择 2. 卸载Agent
```

---

## 五、服务端管理

所有命令在 `sss` 目录下执行：

```bash
# 查看运行状态
docker-compose ps

# 查看服务日志
docker-compose logs -f

# 重启服务
docker-compose restart

# 停止服务
docker-compose down

# 更新镜像（升级服务端）
docker-compose pull && docker-compose up -d
```

---

## 六、防火墙配置

客户端需要能访问服务端的 **35601** 端口（TCP）。如果节点连接超时，按以下步骤排查。

### 验证端口连通性

从节点机器测试：

```bash
python3 -uc "import socket;s=socket.create_connection(('<服务端IP>',35601),5);print(s.recv(1024));s.close()"
```

看到 `Authentication required` 表示端口通了，超时则需要配置防火墙。

### 通用 Linux（iptables）

```bash
# 放行入站
sudo iptables -I INPUT -p tcp --dport 35601 -j ACCEPT

# 放行 Docker 转发（FORWARD 默认策略为 DROP 时必须加）
sudo iptables -I FORWARD -p tcp --dport 35601 -j ACCEPT

# Docker 返回流量放行
BRIDGE=$(docker network inspect sss_default -f '{{(index .Options "com.docker.network.bridge.name")}}' 2>/dev/null || docker network ls --filter name=sss -q | head -1 | xargs -I{} docker network inspect {} -f 'br-{{.Id}}' | cut -c1-15)
sudo iptables -I FORWARD -o "$BRIDGE" -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
sudo iptables -I FORWARD -i "$BRIDGE" -j ACCEPT

# 持久化
sudo netfilter-persistent save
```

> 如果 `netfilter-persistent` 未安装：`sudo apt install -y netfilter-persistent`

### Oracle Cloud

Oracle Cloud 实例需要**两层**都放行：

1. **OS 防火墙**：执行上面的 iptables 命令

2. **VCN 安全列表**：Oracle Cloud 控制台 → Networking → Virtual Cloud Networks → 你的 VCN → Security Lists → Add Ingress Rules：
   - Source CIDR: `0.0.0.0/0`
   - IP Protocol: TCP
   - Destination Port Range: `35601`

3. **如果以上都配了还是不通**，重启 Docker 让它重建完整的转发规则：
   ```bash
   sudo systemctl restart docker && cd sss && docker-compose up -d
   ```

### 同机部署 Agent

如果服务端和 Agent 在同一台机器上，Agent 的 SERVER 地址应使用 `127.0.0.1` 而不是公网 IP，避免绕经防火墙：

```bash
sudo sed -i 's/SERVER=[^ ]*/SERVER=127.0.0.1/' /etc/systemd/system/sss-agent.service
sudo systemctl daemon-reload && sudo systemctl restart sss-agent
```

---

## 目录结构

```
sss/
├── docker-compose.yml   # Docker 编排配置
├── Dockerfile           # Telegram bot 镜像
├── config.json          # 节点配置（自动生成和维护）
├── json/                # 面板数据输出目录（自动生成）
├── bot.py               # Telegram 通知服务
└── _sss.py              # 节点管理 CLI
```

---

## 参考

- [cppla/ServerStatus](https://github.com/cppla/ServerStatus)
