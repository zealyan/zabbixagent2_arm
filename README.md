# zabbix-agent2 (arm64 / aarch64) — Kylin V10 cross-compiled artifacts

AArch64 cross-compiled artifacts for **Zabbix Agent2**, targeting **Kylin V10 (Tercel) arm64 / glibc 2.28**.
Build approach (host x86_64 → aarch64, aarch64 cross gcc + CGO) is documented in the per-version SOP files.

## Versions and artifacts

> Both builds share the install prefix `/home/zabbix/zabbix_agent2` (CD config unchanged).
> **5.2.6 artifacts carry a `_5.2.6` suffix and never overwrite the 5.2.1 files.**

### Zabbix Agent2 5.2.1 (baseline)
- aarch64 / glibc ≤ 2.28 compatible
- Ships the **Docker** plugin; Oracle plugin excluded
- For server 5.2.1 (official support: agent ≤ server)

| File | Description |
|---|---|
| `zabbix_agent2` | aarch64 binary (18 MB, ELF64 AArch64, max GLIBC_2.28) |
| `zabbix_agent2-aarch64-kylin.tar.gz` | Turnkey deploy package (8.4 MB): binary + prefix-aligned config + systemd unit + start/stop script |
| `zabbix-agent2-arm64-crosscompile.md` | Full cross-compile SOP (glibc 2.28 alignment, resolv weak-symbol shim, pitfalls) |
| `产物清单.md` | Artifact verification checklist (arch/GLIBC/deps/docker plugin/qemu run/deploy) |

### Zabbix Agent2 5.2.6 (recommended, with MongoDB)
- aarch64 / glibc ≤ 2.28 compatible
- **Built-in MongoDB plugin** (`zabbix.com/plugins/mongodb`, `plugin 'Mongo'` loaded at runtime)
- Also bundles more plugins: Oracle / Ceph / Redis / Memcached / MQTT / Mysql / Postgres (all pure-Go drivers, no extra native client hard deps)
- Same minor line, minimal version skew; official for server 5.2.1 — **recommended for MongoDB monitoring**

| File | Description |
|---|---|
| `zabbix_agent2_5.2.6` | aarch64 binary (21 MB, ELF64 AArch64, max GLIBC_2.28) |
| `zabbix_agent2-aarch64-kylin-5.2.6.tar.gz` | Turnkey deploy package (10 MB); same layout as the 5.2.1 package (top dir `zabbix_agent2/`) |
| `zabbix-agent2-arm64-crosscompile-5.2.6.md` | 5.2.6 SOP (configure DNS-check patch, `sys/sysctl.h` dual `-isystem`, shim injection-point difference) |
| `产物清单-5.2.6.md` | 5.2.6 verification checklist (qemu version / MongoDB plugin / package integrity) |

## Quick deploy (Kylin V10 aarch64)

```bash
# example uses 5.2.6; for 5.2.1 drop the _5.2.6 suffix from package/binary names
cd /home/zabbix && tar -xzf zabbix_agent2-aarch64-kylin-5.2.6.tar.gz
useradd -M -s /sbin/nologin zabbix 2>/dev/null
chown -R zabbix:zabbix /home/zabbix/zabbix_agent2
# edit etc/zabbix_agent2.conf: Server / ServerActive / Hostname
systemctl enable --now zabbix-agent2     # or deploy/zabbix_agent2.sh start
```

- The binary's built-in default config path is `/home/zabbix/zabbix_agent2/etc/zabbix_agent2.conf` — no `-c` needed after extraction.
- MongoDB monitoring: add a MongoDB config under `etc/zabbix_agent2.d/` (URI / credentials) to load the built-in `Mongo` plugin.
- Change port: `ListenPort` in `etc/zabbix_agent2.conf` (default 10050), or `zabbix_agent2 -c <conf> -p <port>` for a one-off.

## Verification summary

| Check | 5.2.1 | 5.2.6 |
|---|---|---|
| Arch | aarch64 | aarch64 |
| glibc requirement | ≤ 2.28 | ≤ 2.28 |
| Version (qemu `--version`) | 5.2.1 | 5.2.6 |
| Key plugin | Docker | **MongoDB** (+ Docker/Oracle/...) |
| Dynamic deps | libdl/libresolv/libpthread/libc | same (no extra hard deps) |

> RabbitMQ: neither version has a native Agent2 plugin — monitor via the HTTP template.
