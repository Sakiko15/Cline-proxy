# 1Panel 编排（Compose）一键部署指南

在 VPS 上用 1Panel 的「编排」功能部署 Cline-proxy，两种方式任选：

- **方案 A（推荐）：镜像拉取** — 每个 Release 自动推送 `ghcr.io/sakiko15/cline-proxy` 镜像（linux/amd64 + arm64，`latest` + `v*` 双标签），编排里 `image:` 直接拉取，更新一条命令、无需源码
- **方案 B：源码构建** — VPS 上 clone 源码，首次部署自动构建镜像，适合想自己改代码的场景

## 1. 前置条件

- VPS（建议 ≥1 核 1G）
- 已安装 1Panel（安装时默认装好 Docker）
- 云厂商安全组已放行要用到的端口（3457 直连，或 80/443 走反代）

## 2. 方案 A：镜像拉取部署

**2.1 准备数据目录与可选 override.md**（1Panel 终端或 SSH）：

```bash
mkdir -p /opt/cline-proxy-data
touch /opt/cline-proxy-data/override.md
```

> `touch override.md` 很重要：compose 把宿主机该文件挂载进容器（`:ro`）。文件不存在时 Docker 会创建一个**目录**顶上去。空文件即可（override.md 是可选功能，内容为空时自动使用客户端自带系统提示词）。

**2.2 创建编排**：1Panel 左侧 **容器 → 编排 → 创建编排**

- 来源选择 **「编辑」**，文件夹名填 `cline-proxy`，粘贴以下内容（数据目录请按实际调整）：

```yaml
services:
  cline-proxy:
    image: ghcr.io/sakiko15/cline-proxy:latest
    container_name: cline-proxy
    restart: unless-stopped
    ports:
      - "3457:3457"
    volumes:
      - /opt/cline-proxy-data:/app/data
      - /opt/cline-proxy-data/override.md:/app/override.md:ro
    environment:
      - PORT=3457
      - TZ=Asia/Shanghai
```

- 确认后点**创建**，再点**部署**——1Panel 自动从 GHCR 拉取镜像并启动（无需源码、无需构建）

**2.3 更新**：1Panel 终端或 SSH 执行 `docker pull ghcr.io/sakiko15/cline-proxy:latest`，然后在编排详情页点**重新部署**（镜像变化会自动重建容器）。

## 3. 方案 B：源码构建部署

**3.1 拉取源码**（1Panel 终端或 SSH）：

```bash
cd /opt
git clone https://github.com/Sakiko15/Cline-proxy.git
cd Cline-proxy
touch override.md
```

**3.2 创建编排**：**容器 → 编排 → 创建编排** → 来源选 **「路径选择」**，选中 `/opt/Cline-proxy` 目录（1Panel 读取其中的 `docker-compose.yml`）→ 创建 → 部署。

> 为什么用「路径选择」而不是「编辑」：compose 里的 `build: .` 和 `./data` 相对路径都以 compose 文件所在目录为基准。路径选择指向源码目录，保证构建上下文和数据目录都落在 `/opt/Cline-proxy` 下（已核实：1Panel 路径选择是 `docker compose -f <path> up -d` 原地引用，不复制文件）。

**3.3 更新**：`git pull` 后**必须手动构建**（1Panel 重新部署只是 `up -d`，不会重建已有镜像）：

```bash
cd /opt/Cline-proxy
git pull
docker compose build && docker compose up -d
```

## 4. 防火墙

- **1Panel 防火墙**：放行 TCP 3457
- **云厂商安全组**：放行 3457（如果后面配了 HTTPS 反代，也可以只放行 80/443，不放 3457）

## 5. 安全配置（必做）

管理后台**没有登录鉴权**，且**未配置任何 API Key 时代理完全开放**。公网 VPS 上必须：

1. 浏览器打开 `http://<服务器IP>:3457/admin/`
2. 在**设置 → API Keys** 生成至少一个 Key
3. 之后所有客户端请求都必须携带 `x-api-key: <Key>`（或 `Authorization: Bearer <Key>`）

## 6. 客户端接入

Cline / Claude Code / Cursor 配置：

```
Base URL: http://<服务器IP>:3457/v1
API Key:  上一步生成的 Key
Model:    deepseek/deepseek-v4-flash    # 消耗账号额度
          或 deepseek-v4-flash-free     # zen 免费模型（匿名 key）
```

## 7. 添加账号

账号池数据（`data/.cline-accounts.json`）在容器重启/重建后不丢失，但**添加账号要进容器执行**：

- 方式一（推荐）：1Panel 编排详情页 → 进入容器 → 打开终端，执行：

  ```bash
  /app/cline-proxy -login
  ```

  按提示完成 WorkOS 设备登录，账号写入 `data/.cline-accounts.json`。

- 方式二：编排详情页 → 进入容器 → 终端执行 `/app/cline-proxy -add-account`（与 `-login` 相同）。

> 注意：`-login` 必须**在容器内**执行（主机上直接跑没有数据目录权限与二进制）。旧版单账号凭据文件 `.cline-credentials.json` 存放在 `/app` 下、不在 `data/` 卷里，**容器重建会丢失**，不要使用——账号一律通过上面的方式加入账号池。

## 8. （可选）HTTPS 域名反代

1. 域名解析 A 记录指向服务器 IP
2. 1Panel：**网站 → 创建网站 → 反向代理**，域名填你的域名，代理地址 `http://127.0.0.1:3457`
3. 在网站详情里申请 Let's Encrypt SSL 证书并开启强制 HTTPS
4. 客户端 Base URL 改为 `https://你的域名/v1`（端口 3457 可只在防火墙放行 127.0.0.1）

## 9. 常见问题

| 问题 | 处理 |
|------|------|
| 部署失败 / 容器异常退出 | 编排详情页 → 查看容器日志（`docker logs cline-proxy`） |
| 端口冲突 | 修改 compose 左侧宿主机端口，如 `"3458:3457"` |
| 429 / 账号冷却 | 正常现象，冷却到期自动恢复，后台账号列表可见预计恢复时间 |
| 数据目录在哪 | 方案 A：`/opt/cline-proxy-data/`；方案 B：`/opt/Cline-proxy/data/`（账号、API Key、日志、统计都在里面） |

## 数据与配置文件一览

| 路径（容器内） | 宿主机（方案 A / 方案 B） | 说明 |
|------|------|------|
| `/app/data/` | `/opt/cline-proxy-data/` / `/opt/Cline-proxy/data/` | 账号池、API Key、`zen-stats.jsonl`、`requests.jsonl`、日志——**全部持久化** |
| `/app/override.md` | `/opt/cline-proxy-data/override.md` / `/opt/Cline-proxy/override.md` | 可选：系统提示词覆盖（空文件 = 不生效） |

## 镜像发布说明

每次推送 `main`，CI 自动：
- 构建 6 平台二进制 → GitHub Release（`v0.0.x` 递增）
- 构建并推送 Docker 镜像 `ghcr.io/sakiko15/cline-proxy`（linux/amd64 + linux/arm64，标签 `latest` + `v0.0.x`）

> GHCR 是公开容器仓库（GitHub 免费提供），拉取无需登录。仓库主页 Packages 页可查看镜像与各版本。
