# unikraft-deploy（通用版）

在 [Unikraft Cloud](https://unikraft.cloud) 上一键部署任意镜像。每次部署都会「构建 → 推送到你的组织 → 运行」。一个统一工作流 + 根目录 `<project>.yaml` 配置，新增项目只需加一个 yaml。

```
.
├── .github/workflows/
│   ├── deploy.yml        # 统一部署
│   └── delete.yml        # 删除实例
├── uptime-kuma.yaml      # 项目配置
├── 其他项目.yaml
└── README.md
```

## 准备

1. 登录 [Unikraft Cloud Console](https://console.unikraft.cloud/)，在 **Organization → API Keys** 获取 API Token。
2. Fork 本仓库，在 **Settings → Secrets and variables → Actions** 添加：

| Secret | 必填 | 说明 |
|--------|------|------|
| `UNIKRAFT_TOKEN` | 是 | Unikraft API Token |

## 部署

1. **Actions → Deploy Unikraft Instance → Run workflow**
2. 选择 `project`（对应根目录 yaml）和 `metro`（地区）
3. 完成后在 **Job Summary** 查看访问地址（FQDN）与实例名

实例名固定为 `<name>-<metro>`（如 `uptime-kuma-sin`）。同名实例已存在时会部署失败，这是有意的保护：有持久化卷的应用不应在同一个卷上跑两个实例。需要重新部署时先删除旧实例。

## 删除

**Actions → Delete Unikraft Instance**，选择地区，填入实例名（如 `uptime-kuma-sin`）。`delete_all` 会删除该地区**全部**实例，请谨慎。

> 删除实例**不会**删除数据卷，数据会保留，重新部署同一项目会复用。需要清空时请在控制台或用 `unikraft volume delete` 手动删除。

## yaml 配置字段

| 字段 | 必填 | 说明 |
|------|------|------|
| `name` | 否 | 实例名前缀，默认取文件名 |
| `build.base_image` | 与 `build.dockerfile` 二选一 | 基于该镜像构建：生成 `FROM <base_image>` 的 Dockerfile，构建并推送到 `<你的组织>/<name>:latest` 后运行。 |
| `build.dockerfile` | 与 `base_image` 二选一 | 完整 Dockerfile 文本，优先于 `base_image`（适合需要下载二进制的项目）。不支持 `COPY` 仓库里的本地文件 |
| `build.cmd` | 否 | 写入 Kraftfile 的 `cmd`；不填则沿用镜像自带启动命令 |
| `build.runtime` | 否 | 默认 `base-compat:latest` |
| `memory` | 否 | 默认 `512M` |
| `vcpus` | 否 | 不填则用平台默认 |
| `ports` | 否 | 列表，每项对应一个 `--publish`，如 `443:3001/http+tls` |
| `env` | 否 | 列表，每项对应一个 `--env`，如 `KEY=VALUE` |
| `volumes` | 否 | 列表，含 `name` / `path` / `size`（默认 `1G`），自动创建并挂载 |
| `scale_to_zero` | 否 | 留空 = 不休眠（常驻）；否则原样传给 `--scale-to-zero` |

## 新增一个项目

1. 复制 `_template.yaml`（或 `uptime-kuma.yaml`）为 `myapp.yaml`，修改 `name`、`build.base_image`、`ports`、`volumes` 等标有 ▶ 的位置
2. 在 `deploy.yml` 的 `project` 选项里追加一行 `- myapp`（GitHub 的下拉选项必须写死在工作流里，无法动态读取）

## 关于休眠

Unikraft 的 scale-to-zero 默认关闭：不传 `--scale-to-zero` 就不会因空闲而缩容。本项目只有在 yaml 里显式填写 `scale_to_zero` 时才会启用。Uptime Kuma 这类需要持续运行的监控服务应保持留空。

## 注意事项

- `UNIKRAFT_TOKEN` 只放在 GitHub Secrets，不要写入代码。
- yaml 中的 `env` 会出现在 Actions 日志里，**不要在 yaml 中写密码、Token 等敏感值**。
- `unikraft run --image` 使用的是 Unikraft 自己的镜像仓库，并不能直接拉取 Docker Hub 镜像，所以每次部署都会先构建并推送到你的组织。
- 构建模式下，实例若启动失败，先看构建日志里打印出的 Dockerfile / Kraftfile，再用 `build.cmd` 显式指定启动命令。
