# hermes-orin-deploy

Hermes AI Agent 在 NVIDIA Jetson Orin Nano 上的 Docker 部署配置。

## 设备信息

- 硬件：NVIDIA Jetson Orin Nano（ARM64，8GB 统一内存）
- 系统：Ubuntu 22.04 (JetPack 6.x)
- 模型：DeepSeek V4 Pro（通过 DeepSeek API）

## 目录结构

```
hermes-orin-deploy/
├── docker-compose.yml          # 两容器部署配置
├── opt/
│   └── data/
│       └── agent/
│           ├── config.yaml     # Hermes 模型和 Agent 配置
│           └── .env.example    # 环境变量模板
└── static/                     # 自定义前端文件（如有）
```

## 快速部署

### 1. 前置条件

- Docker 已安装
- 已构建或拉取 `hermes-agent:arm64` 镜像

### 2. 克隆仓库

```bash
git clone https://github.com/wangqioo/hermes-orin-deploy.git
cd hermes-orin-deploy
```

### 3. 配置 API Key

```bash
cp opt/data/agent/.env.example opt/data/agent/.env
# 编辑 .env，填入你的 DEEPSEEK_API_KEY
nano opt/data/agent/.env
```

### 4. 创建必要目录

```bash
mkdir -p opt/data/agent opt/data/webui workspace
```

### 5. 启动服务

```bash
docker compose up -d
```

### 6. 访问

- WebUI：http://localhost:8787
- Agent API：http://localhost:8642

## 端口说明

| 端口 | 服务 | 说明 |
|------|------|------|
| 8787 | hermes-webui | Web 界面 |
| 8642 | hermes-agent | Agent API |

## FRP 内网穿透（可选）

如需远程访问，在 frpc.toml 中添加：

```toml
[[proxies]]
name = "orin-8787"
type = "tcp"
localIP = "127.0.0.1"
localPort = 8787
remotePort = 6158   # 修改为你的公网端口
```

## 常见问题

**图形界面黑屏**
```bash
sudo systemctl restart gdm
```

**查看容器日志**
```bash
docker logs hermes-agent
docker logs hermes-webui
```

**重启服务**
```bash
docker compose restart
```
