# Zabbix Agent2 5.2.6 交叉编译 SOP（aarch64 / 麒麟 V10 Tercel / glibc 2.28）

> 方案 B：主机交叉编译（aarch64 交叉 gcc + CGO），内置 MongoDB 插件，产出 glibc ≤ 2.28 兼容的二进制。
> 本文档为 **5.2.6 专用**，区别于 5.2.1 的 `zabbix-agent2-arm64-crosscompile.md`。
> **prefix 保持 `/home/zabbix/zabbix_agent2` 不变**（CD 配置不变）；仅 CI 工程目录与产物制品名称区分。

---

## 1. 目标与前提

| 项 | 值 |
|---|---|
| 目标系统 | 银河麒麟 V10 Tercel（aarch64 / arm64），glibc **2.28** |
| 源码版本 | Zabbix Agent2 **5.2.6**（branch `5.2.6`） |
| 编译主机 | Ubuntu 24.04 x86_64，Go 1.21.4，aarch64-linux-gnu-gcc 13.3.0 |
| 关键能力 | **内置 MongoDB 插件**（5.2.6 起 `plugins/mongodb` 默认编译进二进制） |
| 兼容约束 | agent 版本 ≤ server 版本（server 锁定 5.2.1，5.2.6 同 minor 线、最小偏移，推荐） |
| 安装前缀 | `/home/zabbix/zabbix_agent2`（与 5.2.1 一致，CD 不变量） |

---

## 2. 复用环境资产（与 5.2.1 完全相同）

| 资产 | 路径 | 说明 |
|---|---|---|
| buster sysroot | `/opt/aarch64-buster-sysroot` | glibc 2.28 头文件与库（编译/链接基准） |
| resolv shim | `/tmp/resolv_shim.o` | aarch64 弱符号跳板：`res_query`→`__res_query` 等 12 个别名 |
| 修复版 libresolv | `/usr/aarch64-linux-gnu/lib/libresolv-2.28.so` | 有效 ELF（2.28），供运行时/链接解析 `__res_*` |
| libgcc_s | `/usr/aarch64-linux-gnu/lib/libgcc_s.so.1` | bullseye 版（仅 GLIBC_2.17），避免 GLIBC_2.34 未定义引用 |
| 交叉工具链 | `aarch64-linux-gnu-gcc` / `ld` | gcc 13.3.0 |
| Go | `go1.21.4` | 模块构建 |

> shim 源码（aarch64 汇编，`.weak` 关键，强符号会被 cgo 多次追加导致 `multiple definition`）：
> ```asm
> .section .text
> .macro ALIAS name, target
> .weak \name
> .type \name, %function
> \name:
>     b \target
> .endm
> ALIAS dn_comp, __dn_comp
> ALIAS dn_count_labels, __dn_count_labels
> ALIAS dn_expand, __dn_expand
> ALIAS dn_skipname, __dn_skipname
> ALIAS res_mkquery, __res_mkquery
> ALIAS res_nmkquery, __res_nmkquery
> ALIAS res_nquery, __res_nquery
> ALIAS res_nquerydomain, __res_nquerydomain
> ALIAS res_nsend, __res_nsend
> ALIAS res_query, __res_query
> ALIAS res_querydomain, __res_querydomain
> ALIAS res_search, __res_search
> ```
> 编译：`aarch64-linux-gnu-gcc -c /tmp/resolv_shim.s -o /tmp/resolv_shim.o`

---

## 3. 与 5.2.1 的关键差异

| 维度 | 5.2.1 | 5.2.6（本文） |
|---|---|---|
| CI 工程目录 | `/tmp/zbx` | **`/tmp/zbx526`** |
| MongoDB 插件 | 不在内置列表 | **内置**（`plugins_linux.go` 已注册 `_ "zabbix.com/plugins/mongodb"`） |
| 生成二进制大小 | ~18 MB | ~21 MB（插件更多：含 Oracle/Ceph/Redis/Memcached/MQTT 等） |
| `configure` DNS 检查 | 当年靠系统 2.39 libresolv 通过 | buster 2.28 libresolv 不导出普通 `res_query` → **须 patch 强制通过** |
| shim 注入点 | `/tmp/resolv_shim.o` 置于 `-shared-libgcc` 之前 | 5.2.6 无 `-shared-libgcc`，置于 **`-Wl,--end-group` 之前**（组内） |
| 产物命名 | `zabbix_agent2` / `zabbix_agent2-aarch64-kylin.tar.gz` | **`_5.2.6` 后缀**，不覆盖原产物 |

---

## 4. 构建步骤（已验证可复现）

### 4.1 克隆源码（新目录，区别于 5.2.1）

```bash
rm -rf /tmp/zbx526
git clone --depth 1 --branch 5.2.6 https://git.zabbix.com/scm/zbx/zabbix.git /tmp/zbx526
```

### 4.2 生成 configure

```bash
cd /tmp/zbx526
sh bootstrap.sh          # 生成 configure（autoconf/automake）
```

### 4.3 修补 configure：强制 DNS 检查通过（关键坑）

glibc 2.28 的 `libresolv` 仅导出 `__res_query` 等带下划线符号，**不导出**普通 `res_query`。
configure 的 DNS 测试链接 `res_init()`+`res_query()`，用 buster sysroot 时失败 → `Unable to do DNS lookups`。
**不能**把 shim 放进 `LDFLAGS`（会导致基础编译器测试里 shim 的未定义弱符号无法解析，破坏 `C compiler cannot create executables`）。
正确做法：把 configure 的 DNS 错误改为成功分支：

```bash
cd /tmp/zbx526
python3 - <<'PY'
import re
p='configure'; s=open(p).read()
pat=re.compile(r'\t*as_fn_error \$\? "Unable to do DNS lookups \(libresolv check failed\)" "\$LINENO" 5')
assert pat.search(s), "DNS error line not found"
repl='\t\tfound_resolv="yes"\n\t\tRESOLV_LIBS="-lresolv"'
s=pat.sub(repl,s); open(p,'w').write(s)
PY
```

### 4.4 configure（prefix 不变）

```bash
cd /tmp/zbx526
rm -f config.cache
./configure --host=aarch64-linux-gnu --prefix=/home/zabbix/zabbix_agent2 \
  --enable-agent2 CC=aarch64-linux-gnu-gcc \
  CFLAGS=--sysroot=/opt/aarch64-buster-sysroot \
  CPPFLAGS=--sysroot=/opt/aarch64-buster-sysroot \
  LDFLAGS=--sysroot=/opt/aarch64-buster-sysroot
```

- `--prefix` 仍为 `/home/zabbix/zabbix_agent2`（CD 不变）。
- `--sysroot=buster` 使 C 静态库按 glibc 2.28 构建。
- 注意：`--enable-agent2` 会让顶层 `make` **递归进入 `src/go` 自动 build**，但该自动 build 用默认（glibc 2.39）头文件会失败（见 4.6），因此 C 库阶段即止，Go 二进制在 4.7 单独显式构建。

### 4.5 构建 C 静态库与生成 src/go/Makefile

```bash
cd /tmp/zbx526
make        # 生成 src/libs/*.a 与 src/go/Makefile（Go 递归 build 会失败，可忽略）
```

产物：`src/libs/zbxcommon/libzbxcommon.a`、`src/libs/zbxsysinfo/libzbxagent2sysinfo.a`、`src/go/Makefile` 等。

### 4.6 注入 resolv shim 到 CGO_LDFLAGS

编辑 `src/go/Makefile` 第 151 行 `CGO_LDFLAGS`，将 `/tmp/resolv_shim.o` 插入到 **`-Wl,--end-group` 之前**（5.2.6 行尾无 `-shared-libgcc`）：

```python
# 行首为 "CGO_LDFLAGS = " 的那一行，把 " -Wl,--end-group" 替换为 " /tmp/resolv_shim.o -Wl,--end-group"
```

注入后片段示例（节选）：
```
CGO_LDFLAGS = ... -lm -ldl -lresolv -lpcre /tmp/resolv_shim.o -Wl,--end-group
```

### 4.7 清理缓存并显式交叉编译 Go 二进制

顶层 `make` 的自动 Go build 用默认 2.39 头会找不到 `sys/sysctl.h`（buster 中位于 `usr/include/aarch64-linux-gnu/sys/sysctl.h`）。
须用**双 `-isystem`** 指向 buster 头文件，并显式传 `GOOS/GOARCH/CC/CGO_ENABLED`：

```bash
cd /tmp/zbx526/src/go
unset GOOS GOARCH CC CGO_ENABLED CGO_CFLAGS CGO_LDFLAGS GOFLAGS
go clean -cache     # 避免沿用 2.39 头文件的脏缓存
make GOOS=linux GOARCH=arm64 CC=aarch64-linux-gnu-gcc CGO_ENABLED=1 \
  CGO_CFLAGS="-isystem /opt/aarch64-buster-sysroot/usr/include -isystem /opt/aarch64-buster-sysroot/usr/include/aarch64-linux-gnu" \
  build
```

> `GOOS/GOARCH/CC/CGO_ENABLED` 必须作为 **make 命令行赋值**传递——Makefile 中 `GOOS = \`go env GOOS\`` 是字面反引号字符串，make 不会展开。

产物：`src/go/bin/zabbix_agent2`（约 21 MB，aarch64）。

---

## 5. 验证（qemu 模拟 aarch64 + 2.28 libs）

```bash
BIN=/tmp/zbx526/src/go/bin/zabbix_agent2
file "$BIN"                                   # => ELF 64-bit ARM aarch64
qemu-aarch64-static -L /usr/aarch64-linux-gnu "$BIN" --version
# => zabbix_agent2 (Zabbix) 5.2.6
aarch64-linux-gnu-objdump -T "$BIN" | grep -oP 'GLIBC_[\d.]+' | sort -V | uniq
# => GLIBC_2.17  GLIBC_2.28   (≤2.28，麒麟兼容)
aarch64-linux-gnu-readelf -d "$BIN" | grep NEEDED
# => libdl / libresolv / libpthread / libc / ld-linux (无 libclntsh/libpcre 等硬依赖)
strings "$BIN" | grep -c "plugins/mongodb"    # => 399（内置 MongoDB 插件）
```

配置解析冒烟（最小 conf 下 qemu 启动）：

```bash
qemu-aarch64-static -L /usr/aarch64-linux-gnu "$BIN" -c /tmp/zbx526/mini.conf
# Starting Zabbix Agent 2 (5.2.6)
# using plugin 'Mongo' providing following interfaces: exporter, runner, configurator  ✅
```

---

## 6. 产物命名（区分 5.2.1，不覆盖）

| 产物 | 路径 | 说明 |
|---|---|---|
| 二进制 | `/workspace/zabbix_agent2_5.2.6` | 5.2.6 aarch64 二进制 |
| 部署包 | `/workspace/zabbix_agent2-aarch64-kylin-5.2.6.tar.gz` | 内部顶层目录 `zabbix_agent2/`，结构与 5.2.1 包一致 |
| SOP | `/workspace/zabbix-agent2-arm64-crosscompile-5.2.6.md` | 本文档 |
| 清单 | `/workspace/产物清单-5.2.6.md` | 产物清单 |

> 部署包内部目录（解压至 `/home/zabbix/` 即 `/home/zabbix/zabbix_agent2/`）：
> `bin/ etc/ etc/zabbix_agent2.d/ logs/ run/ plugins/ share/ deploy/`

---

## 7. 部署（与 5.2.1 一致）

1. 拷贝部署包至目标机 `/home/zabbix/`，解压得到 `zabbix_agent2/`。
2. `etc/zabbix_agent2.conf` 已对齐 prefix：`LogFile`/`ControlSocket`/`PidFile`@827/`Include`@828 指向 `/home/zabbix/zabbix_agent2/...`。
3. systemd：`deploy/zabbix-agent2.service`（zabbix 用户）；或 `deploy/zabbix_agent2.sh start|stop|restart|status`。
4. MongoDB 监控：在 `etc/zabbix_agent2.d/` 下添加 MongoDB 配置（URI、密码等），加载 `zabbix.com/plugins/mongodb` 插件（已内置）。
5. 改端口：`etc/zabbix_agent2.conf` 中 `ListenPort`（默认 10050），或 `zabbix_agent2 -c <conf> -p <port>` 临时指定。

---

## 8. 已知约束

- **Oracle 插件**：5.2.6 默认编译进 `plugins/oracle`，但为纯 Go 驱动，NEEDED 列表无 `libclntsh`，不影响基础运行与 2.28 兼容。
- **RabbitMQ**：Agent2 无原生 RabbitMQ 插件（与 5.2.1 一致），需用 HTTP 模板监控。
- **向下兼容**：5.2.6 agent 对接 5.2.1 server 属同 minor 线、最小版本偏移，官方支持（agent ≤ server）。
