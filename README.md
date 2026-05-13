# prometheus.node_exporter

`prometheus.node_exporter` 是面向飞牛 NAS / FnOS 应用商店的 Prometheus Node Exporter 打包项目。项目不直接维护 Node Exporter 源码，而是在构建时下载 Prometheus 官方发布的 Linux 二进制文件，并封装为可在应用商店安装、启动、停止和访问的应用包。

安装后，应用会启动 `node_exporter` 服务并默认监听 `9100` 端口，用于向 Prometheus 提供主机 CPU、内存、磁盘、网络等监控指标。

> [!WARNING]
> 请谨慎配置防火墙、端口映射和访问授权。若配置不当，`9100` 端口可能被非预期网络访问，从而暴露宿主机性能指标、硬件状态和部分运行环境信息。

## 主要功能

- 基于 Prometheus 官方 `node_exporter` 二进制文件构建应用包。
- 支持 `amd64` 和 `arm64` 两种架构打包。
- 提供飞牛 NAS 应用商店所需的 `manifest`、权限、资源和生命周期脚本。
- 通过应用 UI 入口访问 `http://<设备地址>:9100`。
- 启动时尝试加载 `drivetemp` 内核模块，用于增强磁盘温度指标采集能力。
- 启动时将进程绑定到后 4 个 CPU 核心；当设备 CPU 数少于 4 时，绑定到全部可用核心。
- GitHub Actions 支持手动触发构建、生成校验和、创建预发布版本并上传产物。

## 目录结构

```text
.
|-- .github/
|   `-- workflows/
|       |-- main.yml          # 主流程：调度打包、汇总产物、可选创建 Release
|       |-- pack_amd64.yml    # amd64 架构应用包构建流程
|       `-- pack_arm64.yml    # arm64 架构应用包构建流程
|-- app/
|   `-- ui/
|       |-- config            # 应用商店 UI URL 入口配置
|       `-- images/
|           `-- ICON.PNG      # UI 图标
|-- node_exporter/
|   |-- manifest              # 应用商店清单模板，构建时替换版本与架构信息
|   |-- ICON.PNG
|   |-- ICON_256.PNG
|   |-- cmd/
|   |   |-- main              # 服务启动、停止、状态检查脚本
|   |   |-- install_init
|   |   |-- install_callback
|   |   |-- uninstall_init
|   |   |-- uninstall_callback
|   |   |-- upgrade_init
|   |   `-- upgrade_callback
|   `-- config/
|       |-- privilege         # 运行用户、用户组和 root 权限配置
|       `-- resource          # systemd 资源声明
`-- ui/
    `-- .gitkeep
```

## 环境要求

### 运行环境

- 飞牛 NAS / FnOS，最低系统版本：`0.8.10`。
- 应用以 `root` 安装类型运行。
- 默认服务端口：`9100`。
- 需要目标设备允许 Prometheus 或浏览器访问 `9100` 端口。

### 构建环境

GitHub Actions 使用 `ubuntu-latest` 构建。若在本地复现构建，需要准备：

- Linux shell 环境。
- `wget`
- `tar`
- `sed`
- `sha256sum`
- GitHub CLI `gh`，仅在需要发布 Release 时使用。

## 安装与使用

### 从 Release 安装

1. 在项目 Release 页面下载对应架构的应用包：
   - `appstore.prometheus.node_exporter.<包版本>.amd64.tgz`
   - `appstore.prometheus.node_exporter.<包版本>.arm64.tgz`
2. 将应用包上传到飞牛 NAS 应用商店或对应的手动安装入口。
3. 安装并启动应用。
4. 访问以下地址确认服务运行：

```text
http://<设备 IP>:9100
```

Prometheus 抓取配置示例：

```yaml
scrape_configs:
  - job_name: "fnos-node"
    static_configs:
      - targets:
          - "<设备 IP>:9100"
```

### 服务管理

应用商店会调用 `node_exporter/cmd/main` 管理服务。该脚本支持以下操作：

```bash
./node_exporter/cmd/main start
./node_exporter/cmd/main stop
./node_exporter/cmd/main status
```

运行日志默认写入：

```text
/var/log/node_exporter.log
```

## 构建与发布流程

项目使用 GitHub Actions 进行自动化打包和发布，入口文件为 `.github/workflows/main.yml`。

### 主流程

`main.yml` 通过 `workflow_dispatch` 手动触发，支持输入参数：

| 参数 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `make_release` | boolean | `false` | 是否创建 GitHub Release 并上传构建产物 |

主流程定义了两个版本变量：

| 变量 | 当前值 | 说明 |
| --- | --- | --- |
| `target_version` | `1.11.1` | Prometheus 官方 `node_exporter` 版本 |
| `this_pack_manifest_version` | `1.11.1-13` | 应用商店包版本，同时作为 Release tag |

执行流程：

1. `setup` 输出目标版本和应用包版本。
2. `pack-amd64` 调用 `.github/workflows/pack_amd64.yml` 构建 amd64 包。
3. `pack-arm64` 调用 `.github/workflows/pack_arm64.yml` 构建 arm64 包。
4. `upload-package` 下载所有构建产物，汇总到 `release-assets` 目录。
5. 生成 `SHA256SUMS.txt`。
6. 当 `make_release=true` 时，创建预发布版本并上传所有产物。

### 架构包构建

`pack_amd64.yml` 和 `pack_arm64.yml` 的核心步骤一致：

1. 下载 Prometheus 官方发布包：

```text
https://github.com/prometheus/node_exporter/releases/download/v<target_version>/node_exporter-<target_version>.linux-<arch>.tar.gz
```

2. 解压后将 `node_exporter` 可执行文件移动到：

```text
app/node_exporter
```

3. 替换 `node_exporter/manifest` 中的占位符：

| 占位符 | 替换内容 |
| --- | --- |
| `target_version` | 官方 Node Exporter 版本 |
| `target_pack_arch` | 构建架构，`amd64` 或 `arm64` |
| `this_pack_manifest_version` | 应用包版本 |
| `this_pack_manifest_arch` | 应用商店架构标识 |
| `this_pack_manifest_platform` | 应用商店平台标识 |

4. 打包内部应用文件：

```bash
tar zcvf app.tgz app ui
mv app.tgz node_exporter/
```

5. 打包最终应用商店安装包：

```bash
tar zcvf appstore.prometheus.node_exporter.<包版本>.<架构>.tgz node_exporter
```

### 发布产物

构建完成后会生成：

```text
appstore.prometheus.node_exporter.<包版本>.amd64.tgz
appstore.prometheus.node_exporter.<包版本>.arm64.tgz
SHA256SUMS.txt
```

当 `make_release=true` 时，Release 名称为：

```text
fnnas.appstore.prometheus.node_exporter.<包版本>
```

Release tag 为：

```text
<包版本>
```

当前流程将 Release 标记为 `prerelease`。

## 配置说明

### 应用清单

`node_exporter/manifest` 描述应用商店元数据，包括：

- 应用名：`prometheus.node_exporter`
- 展示名：`prometheus.node_exporter`
- 维护者：`prometheus`
- 分发者：`GreenDamTan`
- 官方项目地址：`https://github.com/prometheus/node_exporter`
- 当前项目地址：`https://github.com/fnnas/appstore.prometheus.node_exporter`
- 最低系统版本：`0.8.10`
- 服务端口：`9100`
- 安装类型：`root`

该文件是模板文件，包含版本和架构占位符；不要直接将未替换的模板作为最终安装包发布。

### UI 入口

`app/ui/config` 配置应用商店中的 URL 类型入口：

- 协议：`http`
- 端口：`9100`
- 图标：`images/ICON.PNG`
- 访问目标：`http://<设备地址>:9100`

### 启动参数

`node_exporter/cmd/main` 中的默认启动参数为：

```bash
--collector.rapl.enable-zone-label
```

如需启用更多采集器，可修改脚本中的 `EXEC_ARGS`。脚本中保留了可参考的扩展参数注释：

```bash
--collector.processes --collector.interrupts --collector.wifi --collector.tcpstat
```

### 权限配置

`node_exporter/config/privilege` 声明：

- 默认以 `root` 运行。
- 用户名：`prometheus`
- 用户组：`prometheus`

## 常见问题

### 访问 9100 端口无响应

先确认应用已启动，再检查设备防火墙、网络策略或应用商店端口映射配置。也可以查看日志：

```bash
cat /var/log/node_exporter.log
```

### Prometheus 抓取失败

确认 Prometheus 所在机器可以访问 NAS 的 `9100` 端口，并检查 `targets` 中配置的 IP 和端口是否正确。

### 应用启动后提示已在运行

启动脚本通过 `pgrep -c node_exporter` 判断进程是否存在。如果系统中已有其他 `node_exporter` 进程，脚本会认为服务已经运行。

### 磁盘温度指标缺失

启动脚本会尝试执行 `modprobe -v drivetemp`。如果目标系统不支持该模块或加载失败，磁盘温度相关指标可能不可用，但不影响基础指标采集。

### 构建出的包无法安装

请检查 `node_exporter/manifest` 中的版本、架构和平台占位符是否已在构建流程中正确替换，并确认下载的官方二进制文件与目标架构匹配。

## 维护说明

升级 Node Exporter 时，通常只需要修改 `.github/workflows/main.yml`：

```yaml
env:
  target_version: "1.11.1"
  this_pack_manifest_version: "1.11.1-13"
```

维护建议：

- `target_version` 与 Prometheus 官方 `node_exporter` Release 版本保持一致。
- `this_pack_manifest_version` 用于区分应用商店包版本，建议在同一上游版本下递增后缀。
- 修改启动参数后，检查 `node_exporter/cmd/main` 的 `start`、`stop`、`status` 行为是否正常。
- 发布前至少验证 amd64 和 arm64 两个产物都能生成。
- 发布前下载产物并校验 `SHA256SUMS.txt`。
- Release 产物应包含两个架构包和校验和文件。

## 许可证与来源

本项目用于封装和分发 Prometheus 官方 `node_exporter` 二进制文件。Node Exporter 的源码、许可证和使用说明请参考官方项目：

```text
https://github.com/prometheus/node_exporter
```
