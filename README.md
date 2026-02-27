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
  && wget --no-check-certificate https://raw.githubusercontent.com/zqcccc/ServerStatus-Starter/master/sss.sh \
  && chmod +x sss.sh \
  && sudo ./sss.sh YOUR_TG_CHAT_ID YOUR_TG_BOT_TOKEN
```

> **Telegram Bot 获取方式**
> - Bot Token：与 [@BotFather](https://t.me/BotFather) 对话创建机器人获取
> - Chat ID：与 [@userinfobot](https://t.me/userinfobot) 对话获取

安装完成后：
- 面板地址：`http://<服务端IP>:8081`
- 客户端上报端口：`35601`

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
