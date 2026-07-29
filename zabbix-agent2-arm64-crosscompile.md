# Zabbix Agent2 交叉编译方案（x86 → Kylin V10 Tercel aarch64）

> 目标：在当前 **x86_64** 构建机上，交叉编译出 **Zabbix Agent2 5.2.1**（含官方 **Docker 插件**），
> 部署到目标采集宿主机 **银河麒麟 Kylin V10（Tercel）aarch64 / arm64**。
> Zabbix Server 版本 **5.2.1**（agent 与 server 同主版本，协议兼容）。
>
> **采用方案：方案 B（宿主机交叉编译，aarch64 交叉 gcc + CGO）。**
> **安装 prefix（安装根目录）：`/home/zabbix/zabbix_agent2`** —— 贯穿编译输出、目录布局与运行配置。
>
> **状态：✅ 已在 Ubuntu 24.04 x86_64 实测产出可在 qemu 下运行的 aarch64 二进制（GLIBC_2.28 / 含 docker 插件 / 已裁剪 oracle）。**

---

## 1. 环境检查结果（已实测）

| 项目 | 当前值 | 是否满足交叉编译 | 备注 |
|------|--------|------------------|------|
| 架构 | `x86_64` | ? | 即用户所说的 x86 |
| OS | Ubuntu 24.04.3 LTS | ? | 构建机 |
| Go | `go1.21.4 linux/amd64` | ?（偏高但可用） | 源码 `go.mod` 声明 `go 1.13`，新编译器向下兼容 |
| Docker | `amd64`（Server Arch） | ? | 仅用于备选方案 A |
| gcc | `gcc (Ubuntu 13.3.0)` | — | x86 原生 gcc，不直接用于 arm64 |
| aarch64 交叉 gcc | **gcc-aarch64-linux-gnu 13.3.0** | ? 已装 | `gcc-aarch64-linux-gnu` |
| git / make | 2.43 / 4.3 | ? | 拉源码用 |
| 网络 | 可访问公网 | ? | 需 `go mod download`，建议走国内镜像 |

---

## 2. 5.2.1 源码关键事实（已核对 `tag 5.2.1`）

| 事实 | 结论 | 对编译的影响 |
|------|------|--------------|
| 模块路径 / Go 声明 | `module zabbix.com` / `go 1.13` | 源码用 Go module；但 agent2 还依赖 C 静态库，需 autotools |
| 是否含 `vendor/` | **无** | 必须 `go mod download`（联网 / GOPROXY） |
| **Docker 插件是否默认编入** | ? 是。`src/go/plugins/plugins_linux.go` 第 24 行 `_ "zabbix.com/plugins/docker"` | **无需手动启用**，Linux 构建自动包含 |
| docker 插件是否依赖 cgo | ? 否（纯 Go，依赖 moby/docker client） | docker 监控本身无需交叉 gcc |
| **核心插件是否依赖 cgo** | ? `system/cpu`、`proc`、`vfs/file` 用 `import "C"` | 整个二进制**必须带 cgo 编译**，不能 `CGO_ENABLED=0` |
| `oracle` 插件 | 依赖 `github.com/godror/godror`，链接时需 **Oracle client 库** | 交叉编译最大障碍 → 已裁剪 |
| `vfs` diskcache | 依赖 `mattn/go-sqlite3`（cgo，但仅需 libc） | 有交叉 gcc 即可编，无需外部库 |
| 构建入口 | `src/go/cmd/zabbix_agent2/zabbix_agent2.go` | 包路径 `zabbix.com/cmd/zabbix_agent2` |
| 版本号来源 | `zabbix.com/pkg/version`（内嵌常量） | **不依赖 git**，干净环境也能出版本 |

> 结论先行：**5.2.1 的 agent2 默认构建不是纯 Go**，必须 `CGO_ENABLED=1`。
> 且它依赖一堆 C 静态库（`libzbxcommon.a` 等），需用 `./configure` + `make` 先生成，再走 Makefile 的 `build` 目标。
> 纯静态 `CGO_ENABLED=0` 会因 `system/proc/vfs/oracle` 失败（除非把这些插件也裁掉，得不偿失）。

---

## 3. 目标平台：Kylin V10 Tercel (aarch64) 兼容性

| 特性 | 说明 | 影响 |
|------|------|------|
| 架构 | `aarch64` / `arm64` | 编译目标 `GOARCH=arm64` |
| libc | glibc **2.28**（openEuler / CentOS8 谱系） | cgo 二进制需对齐；Ubuntu 24.04 交叉 gcc 头文件为 2.39，存在 **glibc 版本错位风险** |
| 内核 | 4.19 | Go 运行无压力 |
| Docker 守护进程 | 通常默认 unix socket `/var/run/docker.sock` | docker 插件默认 endpoint 即可 |
| 用户/组 | 建议建 `zabbix` 用户并加入 `docker` 组 | 否则无 socket 读权限 → `docker.info` 报 `ZBX_NOTSUPPORTED` |

---

## 4. 方案对比与采用结论

| 方案 | 原理 | 优点 | 缺点 | 结论 |
|------|------|------|------|------|
| **B. 宿主机交叉编译**（aarch64 交叉 gcc + CGO） | `GOOS=linux GOARCH=arm64 CC=aarch64-linux-gnu-gcc` | 最快、最贴合“x86 交叉编译”诉求；无需模拟 | 需手动对齐 glibc 2.28（见第 6 节） | ? **采用（主方案，已跑通）** |
| A. 容器内原生编译（buildx + QEMU，openEuler/CentOS8） | Docker 模拟 aarch64，在 glibc 2.28 环境 `go build` | **glibc 与 Kylin 完全对齐**，最稳；可复现 | 需 QEMU 注册 + 首次拉镜像较慢 | 备选（仅当 B 不可行时） |
| C. 纯静态精简（CGO_ENABLED=0 + 裁掉 cgo 插件） | 去掉 system.cpu/proc/vfs/oracle/sqlite | 单文件静态、零依赖 | **丢失 CPU/进程/文件系统核心监控** | ? 不推荐，仅应急 |

> 本文以 **方案 B** 为主线给出全部命令；方案 A 作为兜底保留在附录。

---

## 5. PREFIX 约定（重要）

> **Zabbix Agent2 是单一 Go 二进制，`go build` 没有 `--prefix` 编译参数**（它不像 Zabbix C 程序那样 `./configure --prefix`）。
> 这里的 **`prefix = /home/zabbix/zabbix_agent2`** 指**安装根目录**，体现在两处：
> 1. **编译产物输出路径**：`go build -o /home/zabbix/zabbix_agent2/bin/zabbix_agent2`
> 2. **运行时路径**：由配置文件里的 `LogFile` / `PidFile` / `Include` / `ControlSocket` 决定，全部用 prefix 下的子目录。
> 3. **默认配置路径**（`main.confDefault`）由 Makefile 的 `AGENT2_CONFIG_FILE = ${prefix}/etc/zabbix_agent2.conf` 通过 `-ldflags -X main.confDefault=...` 写进二进制。

### 5.0 推荐目录布局（构建机与 Kylin 一致）

```
/home/zabbix/zabbix_agent2/
├── bin/
│   └── zabbix_agent2              # 编译产物（go build -o 输出）
├── etc/
│   ├── zabbix_agent2.conf         # 主配置
│   └── zabbix_agent2.d/           # Include 子配置目录
├── logs/
│   └── zabbix_agent2.log          # 日志
└── run/
    ├── zabbix_agent2.pid          # PID 文件
    └── agent2.sock                # ControlSocket
```

---

## 6. 方案 B（交叉编译，主方案）—— 详细步骤（实测产出可用二进制）

> ⚠️ **重要纠正**：agent2 **不能** 直接 `go build ./cmd/zabbix_agent2`。它依赖一堆 C 静态库
> （`libzbxcommon.a`、`libcommonsysinfo.a` 等，由 autotools 用 `./configure` + `make` 生成），
> 且 cgo 链接期还踩了若干 glibc 2.28 对齐的坑。下面是基于 **Ubuntu 24.04 x86_64 → Kylin V10 aarch64**
> 实测跑通的全部步骤（已产出可在 qemu 下 `zabbix_agent2 --version` 运行的二进制）。

### 6.1 安装 aarch64 交叉工具链

```bash
sudo apt-get update
sudo apt-get install -y gcc-aarch64-linux-gnu binutils-aarch64-linux-gnu
aarch64-linux-gnu-gcc --version   # 实测 13.3.0
```

### 6.2 准备 glibc 2.28 sysroot（对齐 Kylin 的关键）

Ubuntu 24.04 自带交叉 gcc-13 的默认 sysroot 是 `/usr/aarch64-linux-gnu`，里面是 **glibc 2.39**，与 Kylin 2.28 错位。两点事实：

1. gcc-13 交叉编译器的 `--sysroot` 参数被**硬编码忽略**（始终用 `/usr/aarch64-linux-gnu`），所以只能把 2.28 库直接放进该目录；
2. 取一份 **Debian buster arm64** 的 sysroot（headers + libs）放在 `/opt/aarch64-buster-sysroot`，仅用于 `-isystem` 头文件与作为 2.28 库的来源。

做法：把 buster 的 2.28 运行库（`libc`/`ld`/`libm`/`libpthread`/`librt`/`libdl`/`libutil`/`libnsl`/`libcrypt`/`libanl`）覆盖到 `/usr/aarch64-linux-gnu/lib/`（原 2.39 备份到 `/usr/aarch64-linux-gnu.glibc39.bak/`）。

### 6.3 修复 libresolv（否则链接卡死 40+ 分钟 / OOM）

> **本次最大的坑 / 真正的根因**：`/usr/aarch64-linux-gnu/lib/libresolv-2.28.so` 最初是**损坏的 ASCII 文本而非 ELF**，
> 导致 `ld` 解析时内存爆炸（实测 44 分钟、RSS 2.7GB+）。
> **与 `--fix-cortex-a53-843419` 无关**（已用 `gcc -###` 验证该 flag 不会传给 ld）。
> 修复：从 buster 源拷贝有效 ELF：

```bash
cp /opt/aarch64-buster-sysroot/lib/aarch64-linux-gnu/libresolv-2.28.so \
   /usr/aarch64-linux-gnu/lib/libresolv-2.28.so
file /usr/aarch64-linux-gnu/lib/libresolv-2.28.so   # 应是 "ELF 64-bit LSB shared object"
```

### 6.4 安装 bullseye 版 libgcc_s.so.1（避免 GLIBC_2.34 未定义引用）

gcc-13 链接时会要 `libgcc_s.so.1`，但 2.28 源里没有匹配版本（其要求 `GLIBC_2.34`）。放一个 bullseye 的（仅依赖 `GLIBC_2.17`）即可：

```bash
cp /tmp/gcc_extract/lib/aarch64-linux-gnu/libgcc_s.so.1 /usr/aarch64-linux-gnu/lib/libgcc_s.so.1
```

### 6.5 生成 resolv 弱符号 shim（补齐 buster libresolv 缺失的公开别名）

buster 的 `libresolv-2.28.so` 动态符号表**只导出 `__dn_expand` / `__res_nmkquery` 等 `__`-前缀符号**，
而 Zabbix 的 `libcommonsysinfo.a(dns.o)` 引用的是**裸符号** `dn_expand` / `res_nmkquery` / `res_nsend` / `dn_skipname`。
需要一段 aarch64 跳板 shim 补齐公开别名（用 `b __dn_expand` 尾调用转发）。

> **关键坑（multiple definition 的来源）**：shim **必须声明 `.weak`**。
> 因为 `go build ./...` 会链接 `zabbix_agent2` 与 `mock_server` 两个 cgo 主包，且 agent2 内部还有多个 cgo 包，
> cgo 会**每个 cgo 包都追加一次 `CGO_LDFLAGS`**，于是 `/tmp/resolv_shim.o` 在最终链接里出现**两次**；
> 若为强符号会报 `multiple definition of dn_expand`。弱符号重复定义不报错，仍能解析 `dns.o` 的裸引用。

`/tmp/resolv_shim.s`：

```asm
.section .text
.macro ALIAS name, target
.weak \name
.type \name, %function
\name:
    b \target
.endm
ALIAS dn_comp, __dn_comp
ALIAS dn_count_labels, __dn_count_labels
ALIAS dn_expand, __dn_expand
ALIAS dn_skipname, __dn_skipname
ALIAS res_mkquery, __res_mkquery
ALIAS res_nmkquery, __res_nmkquery
ALIAS res_nquery, __res_nquery
ALIAS res_nquerydomain, __res_nquerydomain
ALIAS res_nsend, __res_nsend
ALIAS res_query, __res_query
ALIAS res_querydomain, __res_querydomain
ALIAS res_search, __res_search
```

```bash
aarch64-linux-gnu-gcc -c /tmp/resolv_shim.s -o /tmp/resolv_shim.o
nm /tmp/resolv_shim.o   # 应为 "W dn_expand ... U __dn_expand"（W=弱定义，U=待 libresolv 解析）
```

### 6.6 配置 + 编译 C 静态库（libzbx*.a）

```bash
cd /tmp/zbx                                   # 5.2.1 源码根
./configure --host=aarch64-linux-gnu --prefix=/home/zabbix/zabbix_agent2 \
  --enable-agent2 CC=aarch64-linux-gnu-gcc \
  CFLAGS=--sysroot=/opt/aarch64-buster-sysroot \
  CPPFLAGS=--sysroot=/opt/aarch64-buster-sysroot \
  LDFLAGS=--sysroot=/opt/aarch64-buster-sysroot
make                                          # 用 buster sysroot 头文件编译 libzbx*.a
```

### 6.7 把 shim 注入 Makefile 的 CGO_LDFLAGS

编辑 `src/go/Makefile`，在 `CGO_LDFLAGS` 的 `-shared-libgcc` 之前追加 `/tmp/resolv_shim.o`：

```
CGO_LDFLAGS = ... -Wl,--end-group /tmp/resolv_shim.o -shared-libgcc
```

### 6.8 交叉编译产出二进制

> **关键**：`GOOS`/`GOARCH`/`CC`/`CGO_ENABLED` **必须作为 make 命令行赋值传入，不能当环境变量**。
> Makefile 第 344 行 `GOOS = \`go env GOOS\`` 是字面反引号字符串（make 不展开反引号），会覆盖同名环境变量。

```bash
cd /tmp/zbx/src/go
unset GOOS GOARCH CC CGO_ENABLED CGO_CFLAGS
mkdir -p /home/zabbix/zabbix_agent2/{bin,etc/zabbix_agent2.d,logs,run}

make GOOS=linux GOARCH=arm64 CC=aarch64-linux-gnu-gcc CGO_ENABLED=1 \
  CGO_CFLAGS="-isystem /opt/aarch64-buster-sysroot/usr/include -isystem /opt/aarch64-buster-sysroot/usr/include/aarch64-linux-gnu" \
  build
# 产物：/tmp/zbx/src/go/bin/zabbix_agent2
cp /tmp/zbx/src/go/bin/zabbix_agent2 /home/zabbix/zabbix_agent2/bin/zabbix_agent2
```

### 6.9 产物校验（构建机即可全量验证）

```bash
file /home/zabbix/zabbix_agent2/bin/zabbix_agent2
# ELF 64-bit LSB executable, ARM aarch64, ... dynamically linked, interpreter /lib/ld-linux-aarch64.so.1

readelf -V /home/zabbix/zabbix_agent2/bin/zabbix_agent2 | grep -oE 'GLIBC_2\.[0-9]+' | sort -V | tail -1
# 期望 GLIBC_2.28 → 兼容 Kylin V10

readelf -d /home/zabbix/zabbix_agent2/bin/zabbix_agent2 | grep NEEDED
# libresolv.so.2 / libdl.so.2 / libpthread.so.0 / libgcc_s.so.1 / libc.so.6 / ld-linux-aarch64.so.1

strings /home/zabbix/zabbix_agent2/bin/zabbix_agent2 | grep 'zabbix.com/plugins/docker'   # docker 插件在
strings /home/zabbix/zabbix_agent2/bin/zabbix_agent2 | grep 'zabbix.com/plugins/oracle'   # 应为空（已裁剪）

# 运行时验证（构建机用 qemu-aarch64-static 模拟 aarch64）：
qemu-aarch64-static -L /usr/aarch64-linux-gnu /home/zabbix/zabbix_agent2/bin/zabbix_agent2 --version
# 输出：zabbix_agent2 (Zabbix) 5.2.1 ...  → 证明二进制真实可执行
```

---

## 7. 配置文件模板（prefix 路径版）

写入 `/home/zabbix/zabbix_agent2/etc/zabbix_agent2.conf`（构建机生成，随 bin 一起传到 Kylin）：

```ini
# ---- 服务端 ----
Server=<ZABBIX_SERVER_IP>            # Zabbix Server IP（5.2.1）
ServerActive=<ZABBIX_SERVER_IP>      # 主动模式地址
Hostname=<HOSTNAME>                  # 与前端配置一致

# ---- 运行路径（均基于 prefix /home/zabbix/zabbix_agent2）----
LogFile=/home/zabbix/zabbix_agent2/logs/zabbix_agent2.log
PidFile=/home/zabbix/zabbix_agent2/run/zabbix_agent2.pid
Include=/home/zabbix/zabbix_agent2/etc/zabbix_agent2.d/*.conf
ControlSocket=/home/zabbix/zabbix_agent2/run/agent2.sock

# ---- Docker 官方插件 ----
Plugins.Docker.Endpoint=unix:///var/run/docker.sock   # 默认即此，可省略
Plugins.Docker.Timeout=5
```

> **docker 插件 item key（5.2.1）示例**：`docker.info`、`docker.containers`、`docker.data`、`docker.images`、
> `docker.ping`（对应 `docker.info` 里 `ServerVersion` 等字段）。server 端用官方 “Docker” 模板即可。

---

## 8. 部署到 Kylin V10（arm64）

```bash
# 1) 在 Kylin 上建好 prefix 目录结构并赋权给 zabbix 用户
sudo mkdir -p /home/zabbix/zabbix_agent2/{bin,etc/zabbix_agent2.d,logs,run}
sudo useradd -M -s /sbin/nologin zabbix 2>/dev/null || true
sudo usermod -aG docker zabbix           # docker 插件读 socket 必需
sudo chown -R zabbix:zabbix /home/zabbix/zabbix_agent2

# 2) 从构建机传二进制 + 配置（prefix 内路径保持一致）
scp /home/zabbix/zabbix_agent2/bin/zabbix_agent2 kylin:/home/zabbix/zabbix_agent2/bin/
scp /home/zabbix/zabbix_agent2/etc/zabbix_agent2.conf kylin:/home/zabbix/zabbix_agent2/etc/

# 3) 在 Kylin 上启动并自测
ssh kylin "chmod +x /home/zabbix/zabbix_agent2/bin/zabbix_agent2 && \
  /home/zabbix/zabbix_agent2/bin/zabbix_agent2 -c /home/zabbix/zabbix_agent2/etc/zabbix_agent2.conf -f"
# 另开一终端验证 docker 插件（可在 Kylin 本机或 Server 端 zabbix_get）
zabbix_get -s 127.0.0.1 -k docker.info        # 期望返回 Docker 守护进程信息
zabbix_get -s 127.0.0.1 -k docker.containers   # 容器列表
```

> 若 Kylin 上已有其它 agent 安装（如 `/usr/local/sbin`），用本 prefix 独立部署互不冲突；
> 记得在 Kylin 上停掉旧 agent 或改端口/Hostname，避免与 Server 注册冲突。


### 8.1 开箱即用部署包（推荐）

构建机已产出 **`/workspace/zabbix_agent2-aarch64-kylin.tar.gz`**（8.4 MB），内含 `bin/` + `etc/`（已对齐 prefix 的配置）+ `deploy/`（`zabbix-agent2.service` systemd 单元与 `zabbix_agent2.sh` 启停脚本）+ 运行时目录。解压到 `/` 即得到完整 prefix：

```bash
# Kylin 上：解压即用
cd / && tar -xzf zabbix_agent2-aarch64-kylin.tar.gz
# 赋权 + 启动
useradd -M -s /sbin/nologin zabbix 2>/dev/null; usermod -aG docker zabbix
chown -R zabbix:zabbix /home/zabbix/zabbix_agent2
chmod +x /home/zabbix/zabbix_agent2/bin/zabbix_agent2 /home/zabbix/zabbix_agent2/deploy/zabbix_agent2.sh
# systemd:
cp /home/zabbix/zabbix_agent2/deploy/zabbix-agent2.service /usr/lib/systemd/system/
systemctl daemon-reload && systemctl enable --now zabbix-agent2
# 或脚本: /home/zabbix/zabbix_agent2/deploy/zabbix_agent2.sh start
```

> 配置文件 `etc/zabbix_agent2.conf` 已在构建机侧对齐 prefix（`LogFile`/`PidFile`/`ControlSocket`/`Include` 均指向 `/home/zabbix/zabbix_agent2/...`），
> 且二进制内置 `main.confDefault=/home/zabbix/zabbix_agent2/etc/zabbix_agent2.conf`，解压后无需 `-c` 也能找到配置。
> 部署前只需把 `Server` / `ServerActive` / `Hostname` 三处占位改成真实值。


---

## 9. Agent2 ? Server 5.2.1 兼容性

- agent2 与 server **同主版本（5.2.x）**，主动/被动协议一致，无需特殊兼容处理。
- 小版本不必严格相等（5.2.1 agent 对 5.2.1 server 最佳；对 5.2.7 server 通常也 OK，但建议对齐到 server 的实际补丁号）。
- agent2 向后兼容旧 server 的 item key；docker 插件 key 在 5.2 已稳定。

---

## 10. 常见坑与排查（实测踩过的）

| 现象 | 原因 | 解决 |
|------|------|------|
| `cgo: C compiler "aarch64-linux-gnu-gcc" not found` | 未装交叉 gcc | 执行 6.1 |
| 链接期 `ld` 卡死 40+ 分钟 / 内存爆炸（RSS 数 GB） | `libresolv-2.28.so` 损坏为 ASCII 文本而非 ELF | 6.3：从 buster 源拷贝有效 ELF |
| `multiple definition of dn_expand` | shim 被 cgo 每个 cgo 包追加一次 → 链接两次；强符号冲突 | 6.5：shim 用 `.weak` 声明 |
| `undefined reference to dn_expand` / `res_nmkquery` / `res_nsend` / `dn_skipname` | buster libresolv 只导出 `__`-前缀符号 | 6.5：加 resolv shim 跳板 |
| `undefined reference to ... GLIBC_2.34` | 自带 libgcc_s 要求过高 | 6.4：换 bullseye libgcc_s.so.1（仅 GLIBC_2.17） |
| `go: unsupported GOOS/GOARCH pair \`go env GOOS\`` | Makefile 第 344 行 `GOOS = \`go env GOOS\`` 覆盖环境变量 | 6.8：GOOS/GOARCH 作为 **make 命令行赋值** 传入 |
| 链接报 `cannot find -lclntsh` / oracle 相关 | oracle 插件要 Oracle client | 裁剪 oracle 插件（本方案已裁） |
| `build constraints exclude all Go files` / `cgo disabled` | 误用 `CGO_ENABLED=0` | 改为 `CGO_ENABLED=1`（见 6.8） |
| Kylin 上运行报 `version 'GLIBC_2.3x' not found` | 构建机 glibc 比 Kylin 新 | 用 6.2 的 2.28 sysroot；本方案已对齐到 GLIBC_2.28 |
| `docker.info` 返回 `ZBX_NOTSUPPORTED` | zabbix 用户无 `/var/run/docker.sock` 权限 | 把 zabbix 加入 `docker` 组并重启 agent |
| `go mod download` 超时 | 默认走 proxy.golang.org | 设 `GOPROXY=https://goproxy.cn,direct` |
| 编译报旧依赖不兼容（Go 1.21 过严） | go.mod 太老 | `GOFLAGS=-mod=mod` 或 `go mod tidy` 后再编 |
| 想加 `--prefix` 编译参数 | **agent2 是 Go 二进制，无此参数** | prefix 体现在 `go build -o <prefix>/bin/...` 与配置文件的路径（见第 5 节） |

---

## 11. 一句话执行清单（方案 B，含 prefix，实测版）

```bash
# 1) 工具链
sudo apt-get install -y gcc-aarch64-linux-gnu binutils-aarch64-linux-gnu
# 2) glibc 2.28 sysroot：buster 源放 /opt/aarch64-buster-sysroot；2.28 运行库覆盖到 /usr/aarch64-linux-gnu/lib/
#    （含 6.3 修复 libresolv、6.4 换 bullseye libgcc_s、6.5 生成弱符号 shim /tmp/resolv_shim.o）
# 3) 源码 + configure + make（生成 libzbx*.a）
cd /tmp/zbx
./configure --host=aarch64-linux-gnu --prefix=/home/zabbix/zabbix_agent2 --enable-agent2 \
  CC=aarch64-linux-gnu-gcc CFLAGS=--sysroot=/opt/aarch64-buster-sysroot \
  CPPFLAGS=--sysroot=/opt/aarch64-buster-sysroot LDFLAGS=--sysroot=/opt/aarch64-buster-sysroot
make
# 4) 注入 shim 到 Makefile CGO_LDFLAGS，然后交叉编译
cd src/go
# 编辑 Makefile：CGO_LDFLAGS = ... -Wl,--end-group /tmp/resolv_shim.o -shared-libgcc
mkdir -p /home/zabbix/zabbix_agent2/{bin,etc/zabbix_agent2.d,logs,run}
make GOOS=linux GOARCH=arm64 CC=aarch64-linux-gnu-gcc CGO_ENABLED=1 \
  CGO_CFLAGS="-isystem /opt/aarch64-buster-sysroot/usr/include -isystem /opt/aarch64-buster-sysroot/usr/include/aarch64-linux-gnu" \
  build
cp /tmp/zbx/src/go/bin/zabbix_agent2 /home/zabbix/zabbix_agent2/bin/zabbix_agent2
# 5) 校验
readelf -V /home/zabbix/zabbix_agent2/bin/zabbix_agent2 | grep -oE 'GLIBC_2\.[0-9]+' | sort -V | tail -1   # GLIBC_2.28
qemu-aarch64-static -L /usr/aarch64-linux-gnu /home/zabbix/zabbix_agent2/bin/zabbix_agent2 --version          # 5.2.1
```

---

## 附录：方案 A（buildx + QEMU 容器内原生编译，glibc 对齐 Kylin）—— 仅当方案 B 不可行时兜底

用 Docker 在 x86 上模拟 aarch64，基础镜像选 **glibc 2.28** 的 openEuler/CentOS8，编出的二进制与 Kylin 同 libc，部署零兼容风险。

```bash
# 1) 注册 QEMU 多架构支持
docker run --privileged --rm tonistiigi/binfmt --install all
# 2) 创建 buildx builder
docker buildx create --name arm64builder --use
docker buildx inspect --bootstrap
# 3) 在 openEuler(arm64) 容器中编译（dockerfile 片段）
# FROM openeuler/openeuler:20.03   # glibc 2.28，与 Kylin V10 对齐
# RUN dnf install -y go git gcc
# ... 同方案 B 的 6.6~6.8，但无需 CC/交叉 gcc，直接 go build -o /home/zabbix/zabbix_agent2/bin/zabbix_agent2
docker buildx build --platform linux/arm64 -t zbx-agent2-arm64 . --load
docker create --name tmp zbx-agent2-arm64
docker cp tmp:/home/zabbix/zabbix_agent2/bin/zabbix_agent2 /home/zabbix/zabbix_agent2/bin/zabbix_agent2
docker rm tmp
```
