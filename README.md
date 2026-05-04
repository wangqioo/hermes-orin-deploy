# hermes-orin-deploy

在 NVIDIA Jetson Orin Nano 上部署 Hermes AI Agent 的完整配置。

## 硬件环境

| 项目 | 规格 |
|------|------|
| 硬件 | NVIDIA Jetson Orin Nano |
| 架构 | ARM64（aarch64） |
| 内存 | 8GB 统一内存（CPU + GPU 共享） |
| 存储 | NVMe SSD 116GB |
| 系统 | Ubuntu 22.04（JetPack 6.x，内核 5.15-tegra） |
| GPU | Ampere 架构，1024 CUDA 核心 |

## 部署架构

```
Jetson Orin Nano
├── hermes-agent（端口 8642）   ← AI Agent 核心，ARM64 原生镜像
├── hermes-webui（端口 8787）   ← Web 界面
└── frpc                        ← 内网穿透，远程访问
        │
        └── frps（公网服务器 150.158.146.192）
                ├── SSH   → :6000
                ├── WebUI → :6158
                └── Web   → :6204 / :6205
```

## 目录结构

```
hermes-orin-deploy/
├── docker-compose.yml          # 两容器部署配置
├── opt/
│   └── data/
│       └── agent/
│           ├── config.yaml     # 模型和 Agent 配置
│           └── .env.example    # 环境变量模板（填入 Key 后改名为 .env）
└── README.md
```

## 快速部署

### 1. 前置条件

- Jetson Orin Nano（或其他 ARM64 设备）
- Docker 已安装
- `hermes-agent:arm64` 镜像已构建（见下方）

### 2. 克隆仓库

```bash
git clone https://github.com/wangqioo/hermes-orin-deploy.git
cd hermes-orin-deploy
```

### 3. 配置 API Key

```bash
cp opt/data/agent/.env.example opt/data/agent/.env
nano opt/data/agent/.env
# 填入 DEEPSEEK_API_KEY
```

### 4. 创建数据目录

```bash
mkdir -p opt/data/agent opt/data/webui workspace
```

### 5. 启动服务

```bash
docker compose up -d
```

### 6. 验证

```bash
docker ps | grep hermes
# 访问 http://localhost:8787
```

---

## 构建 hermes-agent:arm64 镜像

WebUI 镜像直接从 ghcr.io 拉取，Agent 镜像需要在 ARM64 设备上本地构建：

```bash
# 克隆 hermes-agent 源码（私有或自维护）
git clone <hermes-agent-repo> hermes-agent-src
cd hermes-agent-src

# 构建 ARM64 镜像
docker build -t hermes-agent:arm64 .
```

> **注意**：`hermes-agent:arm64` 是 ARM64 原生镜像，不能直接在 x86 机器上运行。
> 如需在 x86 设备部署，将镜像名改为 `hermes-agent:latest` 并重新构建对应平台版本。

---

## 模型配置

当前使用 **DeepSeek V4 Pro**（云端 API，非本地推理）。

配置文件位置：`opt/data/agent/config.yaml`

```yaml
model:
  default: "deepseek-v4-pro"
  provider: "deepseek"
  base_url: "https://api.deepseek.com/v1"
  context_length: 131072
```

切换其他模型只需修改 `config.yaml` 中的 `default` 和 `provider`，无需重建镜像。

支持的 provider 示例：

| Provider | 值 | 所需 Key |
|----------|-----|----------|
| DeepSeek | `deepseek` | `DEEPSEEK_API_KEY` |
| OpenRouter | `openrouter` | `OPENROUTER_API_KEY` |
| Anthropic | `anthropic` | `ANTHROPIC_API_KEY` |
| Google Gemini | `gemini` | `GOOGLE_API_KEY` |
| 本地 Ollama | `custom` | 无（配置 base_url） |

---

## 远程访问（FRP 内网穿透）

如需从公网访问，在 frpc.toml 中添加：

```toml
[[proxies]]
name = "orin-8787"
type = "tcp"
localIP = "127.0.0.1"
localPort = 8787
remotePort = 6158    # 修改为你的公网端口

[[proxies]]
name = "ssh"
type = "tcp"
localIP = "127.0.0.1"
localPort = 22
remotePort = 6000
```

重启 frpc 生效：

```bash
sudo systemctl restart frpc
```

---

## 显示器热插拔（Jetson 特有问题）

Jetson Orin Nano 开机时若未连接显示器，GDM 会以虚拟屏幕启动，后续插入显示器不会自动切换。

已配置 udev 热插拔规则自动修复，规则位于 `/etc/udev/rules.d/99-gdm-hotplug.rules`：

```
SUBSYSTEM=="drm", ACTION=="change", RUN+="/usr/bin/systemd-run --no-block /usr/local/bin/gdm-display-hotplug"
```

**效果**：插入显示器后 2~3 秒自动亮屏，无需手动操作。

如需手动修复黑屏：

```bash
sudo systemctl restart gdm
# 或远程执行
ssh -p 6000 nvidia@<公网IP> 'sudo systemctl restart gdm'
```

---

## 常见操作

```bash
# 查看容器状态
docker ps | grep hermes

# 查看日志
docker logs hermes-agent --tail 20
docker logs hermes-webui --tail 20

# 重启服务
docker compose restart

# 停止服务
docker compose down

# 更新 WebUI 镜像
docker pull ghcr.io/nesquena/hermes-webui:latest
docker compose up -d
```

---

## Nervus 生态

本设备同时运行 [Nervus](https://github.com/nervus) 系列服务，与 Hermes 共存于同一设备，通过 Caddy 反向代理统一管理。Hermes WebUI 独立运行，不依赖 Nervus。
