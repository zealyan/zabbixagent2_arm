# zabbix-agent2 (arm64 / aarch64) — Kylin V10 交叉编译产物

Zabbix Agent2 的 aarch64 交叉编译产物，目标平台 **银河麒麟 Kylin V10（Tercel）arm64 / glibc 2.28**。
构建方案（宿主机 x86_64 → aarch64，aarch64 交叉 gcc + CGO）见各版本 SOP 文档。

## 版本与产物

> 两套构建共享安装前缀 `/home/zabbix/zabbix_agent2`（CD 配置不变）。**5.2.6 制品均带 `_5.2.6` 后缀，与 5.2.1 互不覆盖。**

### Zabbix Agent2 5.2.1（基线）
- aarch64 / glibc ≤ 2.28 兼容
- 内置 **Docker 插件**；Oracle 插件已裁剪
- 对接 server 5.2.1（官方支持 agent ≤ server）

| 文件 | 说明 |
|---|---|
| `zabbix_agent2` | aarch64 二进制本体（18 MB，ELF64 AArch64，最高需求 GLIBC_2.28） |
| `zabbix_agent2-aarch64-kylin.tar.gz` | 开箱即用部署包（8.4 MB）：二进制 + 前缀对齐配置 + systemd 单元 + 启停脚本 |
| `zabbix-agent2-arm64-crosscompile.md` | 完整交叉编译 SOP（glibc 2.28 对齐、resolv 弱符号 shim 等踩坑） |
| `产物清单.md` | 产物校验清单（架构/GLIBC/依赖/docker 插件/qemu 运行结果/部署方式） |

### Zabbix Agent2 5.2.6（推荐，含 MongoDB）
- aarch64 / glibc ≤ 2.28 兼容
- **内置 MongoDB 插件**（`zabbix.com/plugins/mongodb`，运行时已加载 `plugin 'Mongo'`）
- 另含更多插件：Oracle / Ceph / Redis / Memcached / MQTT / Mysql / Postgres 等（均为纯 Go 驱动，无额外原生客户端库硬依赖）
- 同 minor 线、最小版本偏移，对接 server 5.2.1 官方支持，**监控 MongoDB 推荐此版本**

| 文件 | 说明 |
|---|---|
| `zabbix_agent2_5.2.6` | aarch64 二进制本体（21 MB，ELF64 AArch64，最高需求 GLIBC_2.28） |
| `zabbix_agent2-aarch64-kylin-5.2.6.tar.gz` | 开箱即用部署包（10 MB）：结构与 5.2.1 包一致（内部顶层 `zabbix_agent2/`） |
| `zabbix-agent2-arm64-crosscompile-5.2.6.md` | 5.2.6 专用 SOP（configure DNS 检查 patch、`sys/sysctl.h` 双 `-isystem`、shim 注入点差异） |
| `产物清单-5.2.6.md` | 5.2.6 产物校验清单（含 qemu 版本/MongoDB 插件/部署包完整性） |

## 快速部署（Kylin V10 aarch64）

```bash
# 以 5.2.6 为例（5.2.1 把包名/二进制名去掉 _5.2.6 即可）
cd /home/zabbix && tar -xzf zabbix_agent2-aarch64-kylin-5.2.6.tar.gz
useradd -M -s /sbin/nologin zabbix 2>/dev/null
chown -R zabbix:zabbix /home/zabbix/zabbix_agent2
# 改 etc/zabbix_agent2.conf 里的 Server / ServerActive / Hostname 占位
systemctl enable --now zabbix-agent2     # 或 deploy/zabbix_agent2.sh start
```

- 二进制内置默认配置路径 `/home/zabbix/zabbix_agent2/etc/zabbix_agent2.conf`，解压后无需 `-c`。
- MongoDB 监控：在 `etc/zabbix_agent2.d/` 增加 MongoDB 配置（URI / 账号），加载内置 `Mongo` 插件。
- 改端口：`etc/zabbix_agent2.conf` 中 `ListenPort`（默认 10050），或 `zabbix_agent2 -c <conf> -p <port>` 临时指定。

## 验证摘要

| 检查项 | 5.2.1 | 5.2.6 |
|---|---|---|
| 架构 | aarch64 | aarch64 |
| glibc 需求 | ≤ 2.28 | ≤ 2.28 |
| 版本 (qemu `--version`) | 5.2.1 | 5.2.6 |
| 关键插件 | Docker | **MongoDB** (+ Docker/Oracle/...) |
| 动态依赖 | libdl/libresolv/libpthread/libc | 同上（无额外硬依赖） |

> RabbitMQ：两版本均无原生 Agent2 插件，统一用 HTTP 模板监控。
