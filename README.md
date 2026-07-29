# zabbix-agent2 (arm64 / aarch64) — Kylin V10 交叉编译产物

Zabbix Agent2 **5.2.1** 的 aarch64 交叉编译产物，目标平台 **银河麒麟 Kylin V10（Tercel）arm64 / glibc 2.28**，
含官方 **Docker 插件**，已裁剪 Oracle 插件。

构建方案（宿主机 x86_64 → aarch64，aarch64 交叉 gcc + CGO）见 `zabbix-agent2-arm64-crosscompile.md`。

## 产物清单

| 文件 | 说明 |
|---|---|
| `zabbix_agent2` | aarch64 二进制本体（18 MB，ELF64 AArch64，最高需求 GLIBC_2.28） |
| `zabbix_agent2-aarch64-kylin.tar.gz` | 开箱即用部署包（8.4 MB）：含二进制 + 前缀对齐配置 + systemd 单元 + 启停脚本 |
| `zabbix-agent2-arm64-crosscompile.md` | 完整交叉编译 SOP（含 glibc 2.28 对齐、resolv 弱符号 shim 等踩坑记录） |
| `产物清单.md` | 产物校验清单（架构/GLIBC/依赖/docker 插件/qemu 运行结果/部署方式） |

## 快速部署（Kylin V10 aarch64）

```bash
cd / && tar -xzf zabbix_agent2-aarch64-kylin.tar.gz
useradd -M -s /sbin/nologin zabbix 2>/dev/null; usermod -aG docker zabbix
chown -R zabbix:zabbix /home/zabbix/zabbix_agent2
# 改 etc/zabbix_agent2.conf 里的 Server / ServerActive / Hostname 占位
systemctl enable --now zabbix-agent2     # 或 deploy/zabbix_agent2.sh start
```

二进制内置默认配置路径 `/home/zabbix/zabbix_agent2/etc/zabbix_agent2.conf`，解压后无需 `-c`。
验证：`zabbix_get -s 127.0.0.1 -k docker.info`。
