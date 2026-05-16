# CPA-Manager SERV00 FreeBSD 部署指南

本文档适用于 `cozers/CPA-Manager` 发布的 FreeBSD 原生包，面向 `serv00` 这类 FreeBSD 主机环境。

当前可直接部署的 FreeBSD release 例如：

```sh
cpa-manager-v1.2.1-freebsd-amd64.tar.gz
```

对应 GitHub Release:

```text
https://github.com/cozers/CPA-Manager/releases/tag/v1.2.1
```

## 先说结论

- `serv00` 上建议使用 **FreeBSD 原生包**
- 不建议在 `serv00` 上走 Docker 方案
- `CPA-Manager` 不是 `CPA` 本体，它是管理面板和 Usage Service
- 部署完成后，登录页还需要填写：
  - `CPA URL`
  - `Management Key`

## 前置条件

部署前请确认：

- 你已经有一个可访问的 `CPA` 服务
- `CPA` 已启用 Management
- 如果你要使用请求监控，`CPA` 需要启用 `usage-statistics-enabled: true`
- 推荐 `CPA` 版本 `>= v6.10.8`
- 你已经在 `serv00` 面板申请好一个可用端口，例如 `45890`

## 下载与上传

在 GitHub Releases 下载：

```text
cpa-manager-v1.2.1-freebsd-amd64.tar.gz
cpa-manager-v1.2.1-freebsd-amd64.tar.gz.sha256
```

上传到 `serv00` 后，推荐放在：

```sh
~/cpa-manager/
```

也可以放在你的域名目录，例如：

```sh
~/domains/你的域名/
```

## 解压

SSH 登录 `serv00` 后执行：

```sh
mkdir -p ~/cpa-manager
cd ~/cpa-manager
tar -xzf ~/cpa-manager-v1.2.1-freebsd-amd64.tar.gz
cd cpa-manager
chmod +x bin/cpa-manager start.sh daemon-start.sh stop.sh
```

解压后的目录结构大致如下：

```text
cpa-manager/
  bin/cpa-manager
  config.json
  start.sh
  daemon-start.sh
  stop.sh
  data/
  logs/
```

## 配置端口

编辑 `config.json`，把 `httpAddr` 改成 `serv00` 面板分配给你的端口。假设端口是 `45890`：

```json
{
  "httpAddr": "0.0.0.0:45890",
  "dataDir": "./data",
  "collectorMode": "auto",
  "queue": "usage",
  "popSide": "right",
  "batchSize": 100,
  "pollIntervalMs": 500,
  "queryLimit": 50000,
  "corsOrigins": ["*"]
}
```

## 重要坑位：只改 `config.json` 可能不生效

当前 FreeBSD 打包脚本生成的 `start.sh` 默认带了这一行：

```sh
export HTTP_ADDR="${HTTP_ADDR:-0.0.0.0:18317}"
```

而程序启动配置优先级是：

```text
环境变量 > config.json > 程序默认值
```

这意味着：

- 如果 `start.sh` 里仍然保留上面这一行
- 那么即使你把 `config.json` 改成了 `45890`
- 实际运行时仍然可能监听 `18317`

典型现象：

```text
cpa-manager listening on 0.0.0.0:18317
http server: listen tcp 0.0.0.0:18317: bind: operation not permitted
```

这是因为在 `serv00` 上，`18317` 不是你当前账号可用的端口。

## 正确处理方法

推荐直接修改 `start.sh`，让它和 `config.json` 保持一致。

编辑：

```sh
vi start.sh
```

把：

```sh
export HTTP_ADDR="${HTTP_ADDR:-0.0.0.0:18317}"
```

改成：

```sh
export HTTP_ADDR="${HTTP_ADDR:-0.0.0.0:45890}"
```

如果你希望完全以 `config.json` 为准，也可以直接删掉这一行。

更推荐的 `start.sh` 写法如下：

```sh
#!/bin/sh
set -eu

APP_DIR=`CDPATH= cd -- "`dirname -- "$0"`" && pwd`
export CPA_MANAGER_CONFIG="${CPA_MANAGER_CONFIG:-$APP_DIR/config.json}"

mkdir -p "$APP_DIR/data" "$APP_DIR/logs"
exec "$APP_DIR/bin/cpa-manager" >> "$APP_DIR/logs/cpa-manager.log" 2>&1
```

这样程序会直接读取 `config.json` 里的 `httpAddr`，不会再被 `18317` 覆盖。

## `start.sh` 和 `daemon-start.sh` 的区别

- `start.sh`
  - 前台启动
  - 适合首次调试
  - 当前 SSH 会话断开后，进程通常也会结束

- `daemon-start.sh`
  - 后台启动
  - 适合正式部署
  - 会写入 `cpa-manager.pid`
  - 可以配合 `stop.sh` 停止服务

`serv00` 正式运行建议使用：

```sh
./daemon-start.sh
```

## 启动

推荐顺序：

1. 先用前台模式确认能否正常启动
2. 确认无误后，再切换到后台模式

前台启动调试：

```sh
cd ~/cpa-manager/cpa-manager
./start.sh
```

如果输出正常，按 `Ctrl+C` 停掉。

然后后台启动：

```sh
./daemon-start.sh
```

查看日志：

```sh
tail -f logs/cpa-manager.log
```

停止：

```sh
./stop.sh
```

## 访问方式

如果 `serv00` 支持端口直接访问，可以打开：

```text
http://你的域名:45890/management.html
```

如果你在 `serv00` 面板中做了反向代理，把域名反代到：

```text
127.0.0.1:45890
```

然后访问你的 HTTPS 域名即可。

## 登录后怎么配置

打开面板后，填写：

- `CPA URL`
- `Management Key`

如果要启用请求监控，还需要确保：

- `CPA` 端开启 Management
- `CPA` 端开启 `usage-statistics-enabled: true`
- 最好使用 `CPA v6.10.8+`

## 校验文件完整性

上传后可以在 `serv00` 上执行：

```sh
sha256 cpa-manager-v1.2.1-freebsd-amd64.tar.gz
cat cpa-manager-v1.2.1-freebsd-amd64.tar.gz.sha256
```

两边 hash 一致即可。

## 升级

升级前建议先停止旧进程并备份数据：

```sh
cd ~/cpa-manager/cpa-manager
./stop.sh
cd ..
cp -a cpa-manager/data ./data.backup
```

重新解压新包后恢复数据：

```sh
tar -xzf ~/cpa-manager-v1.2.1-freebsd-amd64.tar.gz
cp -a ./data.backup/* ./cpa-manager/data/
cd cpa-manager
chmod +x bin/cpa-manager start.sh daemon-start.sh stop.sh
```

如果新包里的 `start.sh` 仍然带有 `18317` 默认值，请重新检查并修改端口相关配置。

完成后启动：

```sh
./daemon-start.sh
```

## 常见问题

### 1. 已经把 `config.json` 改成了 `45890`，日志还是显示 `18317`

原因：

- `start.sh` 里的 `HTTP_ADDR` 环境变量覆盖了 `config.json`

处理：

- 修改 `start.sh`
- 或删除其中的 `HTTP_ADDR` 导出行

### 2. 日志报错 `bind: operation not permitted`

通常表示：

- 当前进程尝试监听一个你没有权限使用的端口

先检查：

```sh
tail -n 100 logs/cpa-manager.log
sockstat -4 -l | grep cpa-manager
```

然后确认：

- `config.json` 的端口是否正确
- `start.sh` 是否仍然把端口覆盖成 `18317`

### 3. `Permission denied`

重新执行：

```sh
chmod +x bin/cpa-manager start.sh daemon-start.sh stop.sh
```

### 4. 后台启动后打不开页面

检查：

```sh
tail -n 100 logs/cpa-manager.log
sockstat -4 -l | grep 45890
```

如果没监听到，说明程序没有成功启动。

### 5. 请求监控没有数据

检查：

- `CPA` 是否启用了 `usage-statistics-enabled`
- `CPA` 是否启用了 Management
- `CPA URL` 和 `Management Key` 是否填写正确
- 是否只有一个 Usage Service 在消费同一个 CPA 实例

## 版本说明

历史上 `v1.0.0` 到 `v1.0.4` 没有 `usage-service` 目录，只是前端工程，无法生成 `serv00` 可直接运行的 FreeBSD 服务二进制。

因此部署 `serv00` 时，请优先使用包含 FreeBSD 包的 release，例如 `v1.2.1` 及后续版本。

## 仓库维护说明

上游发布新版本后，可在仓库目录执行：

```sh
git fetch upstream --tags
powershell -ExecutionPolicy Bypass -File ./tools/package-freebsd.ps1
```

脚本会选择本地最高版本号的 `v*` tag，输出最新 FreeBSD 包到：

```text
dist-freebsd/
```
