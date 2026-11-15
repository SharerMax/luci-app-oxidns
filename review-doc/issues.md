# Known Issues

记录排查过程中发现的、按优先级排序的问题。每条包含：现象、证据、影响、建议修法。

最近核对：2026-11-16，逐条对照 `luci-app-oxidns@561bd16` 与上游 `../oxidns`（1.7.1）；定性结论全部成立，`luci.oxidns` 行号已校正到该版本。

约定：
- **P1** 会导致功能不可用或数据/文件损坏
- **P2** 安全、审计或健壮性缺口
- **P3** 上游约束（不是 bug，但会限制设计空间，记录以免重复踩坑）
- **P4** 仓库卫生与流程

状态：`待处理` / `处理中` / `已修复` / `已确认不修`

---

## P1

### 1. `minimal` bundle 在全新系统上必定安装失败

- **状态**：待处理
- **现象**：设备上没有 `/etc/oxidns/config.yaml` 时，用 `minimal` bundle 走 `core_install`，流程在 `write_openwrt_default_config` 阶段退出，返回 `default_config_unsupported`。**`core_upload_install` 同样中招**。
- **证据**：
  - `minimal` 归档里的 `config.yaml` 其实是 `config.minimal.yaml` 改名而来（`../oxidns/.github/workflows/release.yml:382-384`）。
  - `../oxidns/config.minimal.yaml:14-18` 把整个 `api:` 块注释掉了：`# api:` / `#   http:` / `#     listen: ":9199"` / `#     webui:` / `#       root: "./webui"`。workflow 自己的注释也写了 `# minimal: ship the dedicated minimal config (no api/webui block)`。
  - `root/usr/libexec/rpcd/luci.oxidns:1364` 的顶层键匹配 `/^[^[:space:]#][^:]*:/` 已经显式排除了注释行（说明作者当时考虑过注释），但 `listen_done` 只在 `luci.oxidns:1383` 才置位、`root_done` 只在 `luci.oxidns:1388` 才置位。`END`（`luci.oxidns:1392-1395`）无条件要求两者都为真，否则 `exit 42`。
  - 两个调用点都把 exit 42 转成 `default_config_unsupported`：`install_core_from_release`（`luci.oxidns:1515-1518`）和 `install_uploaded_core_from_unpack`（`luci.oxidns:1705-1708`）。
- **影响**：`minimal` bundle 实际不可用。`full` / `standard` 不受影响（它们用未注释的 `config.yaml`）。
- **建议**：`write_openwrt_default_config` 区分两种失败。若源文件根本不含未注释的 `api.http` 块，则不做改写、直接原样落盘（`minimal` 本来就没有管理 API，改写无意义）；只有"有 `api:` 块但缺 `listen` / `webui.root` 子键"才报 `default_config_unsupported`。需要在 `integration-check.sh` 里加一份注释掉 `api:` 块的 `config.yaml` 固件，两个安装路径都要覆盖。

### 2. `config_path` 校验放行 `/usr/bin/oxidns`，配置保存会覆盖内核二进制

- **状态**：待处理
- **现象**：`Settings` 里把 `Config path` 填成 `/usr/bin/oxidns` 能通过校验并保存；随后在 `Configuration` 页保存任意 YAML，就会把配置文件 `mv` 到内核二进制路径上。
- **证据**：
  - `valid_config_path_setting` → `safe_remove_config_path` → `oxidns_named_path`。后者只检查 basename 或父目录名是否含 `oxidns`（`luci.oxidns:809-820`，核心判断在 `814-818` 的 `case "$BASE_NAME:$DIR_BASE" in *oxidns*|*:oxidns*)`）。`/usr/bin/oxidns` 的 basename 就是 `oxidns`，拼出 `oxidns:bin`，命中 `*oxidns*` → 通过。
  - `emit_config_save` 校验的是**内容**（`validate_config_file "$TMP_FILE"`，对临时文件，`luci.oxidns:929`），不是目标路径。校验通过后直接 `mv "$TMP_FILE" "$CONFIG_PATH"`（`luci.oxidns:940`）。
  - 会先 `cp` 一份备份到 `$WORKING_DIR/config-backups/`（`luci.oxidns:936-939`），所以二进制通常还能找回，但服务已经起不来了。
- **影响**：误操作即可破坏内核安装。
- **建议**：`oxidns_named_path` 太宽松。建议单独加一个"绝不能作为写入目标"的显式黑名单（`$OXIDNS_BIN`、`$WEBUI_DIR`、`/etc/init.d/oxidns`、`/etc/config/oxidns`、`/etc/passwd`…）外加可执行/库目录前缀拒绝（`/bin/*`、`/sbin/*`、`/usr/bin/*`、`/usr/sbin/*`、`/lib/*`、`/usr/lib/*`、`/www/*`）。
- **参考**：~~`valid_log_path_setting` 里的黑名单 + 目录前缀拒绝~~ —— 2026-11-16 核对：`log_path` / `valid_log_path_setting` 在本仓库全 git 历史中均不存在（`git log --all -S` 验证），系误记。现成可借用的模式是 `safe_remove_working_dir` 的顶层目录黑名单（`luci.oxidns:833-846`），但它不含可执行/库目录前缀拒绝，需按本条建议补全。

### 3. Settings 无条件提供三种 bundle，但 slim bundle 只有两个 target

- **状态**：待处理
- **现象**：在 armv7 / armv6 / i686 设备上把 `Core bundle` 选成 `standard` 或 `minimal`，安装会失败并报 `release_asset_not_found`，错误信息完全没提示"这个架构没有 slim 包"。
- **证据**：
  - `../oxidns/.github/workflows/release.yml:310-320`：slim matrix 只有 `minimal`/`standard` × `x86_64-unknown-linux-musl`/`aarch64-unknown-linux-musl`。
  - `htdocs/.../settings.js` 的 `bundleSelect`（`settings.js:63-70`）固定渲染 `full`/`standard`/`minimal` 三项，不看架构。
  - `asset_name_for_bundle`（`luci.oxidns:1165-1179`）只按 bundle + target 拼名字（`full` → `oxidns-<target>.tar.gz`，注意 full 没有 bundle 段），拼完由 `select_release_asset` 去 release 里找，找不到就报 `release_asset_not_found`。
- **影响**：在非 x86_64/aarch64 设备上 UI 提供了必然失败的选项，排查成本高。
- **建议**：`settings_read` 或 `status` 额外返回当前 target 可用的 bundle 列表，前端据此只渲染可选项；或至少在 `asset_name_for_bundle` 失败时给出"该架构不发布 standard/minimal，请改用 full"的明确 `detail`。

### 4. `log_syslog` 改动后，`restart` 不生效，但 `stop` + `start` 生效

- **状态**：已修复（4c11591，app 层用 `run_init_action` 拆分 restart；rc.common/procd 本身属上游，未改也无法改）
- **现象**：`Settings` 里改“写入系统日志”并保存后：
  - `/etc/init.d/oxidns stop` 再 `/etc/init.d/oxidns start` → **生效**
  - `/etc/init.d/oxidns restart`（Overview 的 Restart 按钮）→ **不生效**，服务仍按旧的 `stdout`/`stderr` 跑
  - 两个方向都如此。`enable`/`disable` 完全无关（见 #7）。
- **根因**：procd 版的 `stop` 会 `procd_lock`，而解锁发生在脚本 EXIT 的 trap 里；procd 版的 `start` **不上锁**。
  - `restart` 走的是 `rc.common` 开头那个通用实现（procd 块只重定义了 `start`/`stop`/`reload`/`running`/`status`，**没有重定义 `restart`**），即单进程内 `stop` 再 `start`。于是 `procd_kill` 和随后的 `procd_close_service "set"`（在 `start_service` 里带新 `stdout`/`stderr`）**全程持有 procd 锁**，新实例定义走延迟处理路径，没有按新的 `log_syslog` 重新声明。
  - `stop` + `start` 是两次独立脚本执行：第一次退出时 EXIT trap 释放锁，第二次 `start` 不加锁，`procd_close_service "set"` 立即生效。
- **证据来源**：真机 `/etc/rc.common`（Raspberry Pi 4，OpenWrt 21.02 系列，用 `$IPKG_INSTROOT`）：
  ```sh
  restart() { trap '' TERM; stop "$@"; trap - TERM; start "$@"; }   # 通用版，procd 未覆盖
  stop()    { procd_lock; stop_service "$@"; procd_kill ...; }        # 上锁，EXIT trap 才解锁
  start()   { rc_procd start_service "$@"; }                          # 不上锁
  rc_procd(){ local method="set"; [ -n "$2" ] && method="add"
              procd_open_service ...; "$@"; procd_close_service "$method"; }
  ```
- **修法（已入库）**：`run_service` 里的 `run_init_action` 把 `restart` 拆成 `"$INIT" stop && "$INIT" start` 两次独立调用，从而让新实例定义在未加锁状态下提交。代价是两次脚本调用而非一次，重启窗口与原生 `restart` 基本一致。
- **未完全确认**：procd 在持锁状态下处理 `set` 请求的**具体**内部路径需要看 `package/system/procd/files/service.c` 才能 100% 闭环。不过「锁的作用域差异」是这份 `rc.common` 里可以直接读出的唯一结构性差异，且绕过方案用的正是实测有效路径，风险可控。
- **另一个独立遗留疑点**：即使实例定义重新声明了，`procd_set_param stdout 0` 本身能否完全摘掉 stdio 仍未验证 —— procd 把 `stdout` 当「是否转发到 syslog 的布尔量」，传字符串 `"0"` 与「省略参数」未必等价。若关闭后 console 仍出现 oxidns 输出（`dmesg | tail` 可验证），那是另一个问题（见 #8）。
- **待补测试**：`run_init_action` 的 restart 拆分仍无法单测 —— `init_script()` 返回硬编码的 `/etc/init.d/oxidns`，而 `load_settings` 不再读 `service_name`，测试无法替换该路径。验证只能靠真机：改设置 → Overview 点 Restart → 确认 `logread` 行为符合预期。已部分覆盖：561bd16 起 `integration-check.sh` 直接 source 仓库内真实 init 脚本，断言 `start_service` 写出的 `stdout`/`stderr` 参数随 `log_syslog` 0/1 变化；仍未覆盖的只剩 rpcd 侧 `run_init_action` 的拆分调用本身。

---

## P2

### 5. `minimal` bundle 不含 `webui/`，但配置改写仍会写入 `webui.root`

- **状态**：待处理（与 #1 同源，修 #1 时一并考虑）
- **证据**：`release.yml:376-385` 的 minimal 打包只 `tar czf ... oxidns config.yaml LICENSE`，没有 `webui`。而 `write_openwrt_default_config` 的 `luci.oxidns:1386-1390` 会把 `api.http.webui.root` 改写成 `/usr/share/oxidns/webui`。
- **注意**：`install_core_from_release` 拷贝 WebUI 的分支（`luci.oxidns:1536`）是 `if [ -d "$UNPACK_DIR/webui" ]`，所以 minimal 缺 `webui/` 不会报错，只是静默跳过 —— 也就是说配置被改写了但目录不存在，属于"半成功"状态。
- **影响**：修完 #1 之后，minimal 安装虽然能成功，但配置里会指向一个不存在的 WebUI 目录。
- **建议**：#1 里判断到"无 `api:` 块"时整体跳过改写，天然规避此问题。若将来让 minimal 支持 WebUI，需要先确认内核侧 `webui` cargo feature 在 minimal 下是否启用（按 `../oxidns` 的 feature 定义，`webui = ["api"]`，minimal 大概率没有）。

### 6. syslog 服务操作记录缺少操作者身份

- **状态**：已确认不修（rpcd 层面做不到）
- **现象**：`Overview` 的 Start/Stop/Restart/Enable/Disable 会写 `luci-app-oxidns` tag 的系统日志，但记录里只有动作和结果，无法归因到人。
- **原因**：rpcd 的 exec 脚本拿不到可靠的客户端 IP / 用户名，`$SSH_CLIENT` 之类在 rpcd 环境下不存在。
- **结论**：要审计到人只能靠 rpcd ACL 侧或 webserver 访问日志，rpcd 后端本身做不到。已在 README 注明当前记录范围。

### 7. `service_triggers` 不监听 `/etc/config/oxidns`

- **状态**：待处理
- **现象**：`service_triggers()`（`root/etc/init.d/oxidns`）只有 `procd_add_reload_trigger oxidns`（配置变更 → signal）。手动 `uci set oxidns.main.log_syslog=0` 后不触发任何动作；而且 `reload_service` 只发 signal、不重新读取 UCI、也不重建 procd 实例参数，所以 UCI 里的 `log_syslog`（以及 `config_path`、`working_dir`）在手动改 UCI 后都不生效，必须真正 restart。
- **影响**：绕过 LuCI 直接改 UCI 时容易误判「改了没用」；这和 #4 是两个独立问题 —— #4 是 restart 之后仍不生效，这里是根本不会自动 restart。
- **建议**：要么在 README 明确「这些设置需 `/etc/init.d/oxidns restart`」，要么加指向 UCI 包的 reload trigger 并把 `reload_service` 改成完整 restart。前者风险更低，建议先做前者。

---

## P3 上游约束（记录以免重复踩坑）

### 8. procd 的 `stdout` / `stderr` 只接受布尔值

- **证据**：OpenWrt 官方文档 —— `procd_set_param stdout: boolean whether to redirect commands stdout to syslog (default: 0)`；`1` = 用 `syslog(3)` 逐行转发，`0`/缺省 = 不转发。
- **影响**：无法用 procd 参数把服务输出重定向到文件。要落到文件只能自己在 init 脚本里包一层 `sh -c "exec … >>file 2>&1"`。
- **本次做法**：`log_to_syslog` 关闭时用 `sh -c "exec … >>/dev/null 2>&1"`，并依赖 `exec` 保持 PID 不变，这样 `procd_send_signal`（reload）和 respawn 仍然作用在真正的 oxidns 进程上。

### 9. OxiDNS 的 `log.file` 是叠加层，不是互斥

- **证据**：`../oxidns/src/infra/observability/logging.rs:39-47` —— console layer（`stdout`）总是安装，file layer 是额外叠加的。
- **影响**：只在 config 里设 `log.file` **停不掉** syslog 输出，停止必须从 procd 侧做。所以 LuCI 的 `log_to_syslog` 和内核的 `log.file` 是两条正交的轴，README 已说明。
- **附带信息**：内核日志控制台层写的是 **stdout**（不是 stderr），所以 `procd_set_param stderr 1` 对 tracing 输出没有作用，只是为了兜住启动 banner 之外的 stderr 诊断信息。日志级别来自 `log.level` 或 `start -l`，**不读 `RUST_LOG`**。

### 10. `oxidns --version` 和 `build-info` 都不含 target triple

- **证据**：
  - `--version` 输出 `oxidns <版本> (full)`，版本取 `CARGO_PKG_VERSION`（`src/cli/mod.rs:27` + `src/build_info.rs:26-37`；2026-11-16 核对时上游为 1.7.1）。
  - `build-info` JSON 字段为 `{version, bundle, enabled_bundles, enabled_features, supported_plugins}`（`src/build_info.rs:39-46`）。
  - triple 唯一的落点是编译期常量 `OXIDNS_BUILD_TARGET`（`build.rs:4-9`），只被 `src/infra/upgrade/release.rs:165` 读取，从不对外输出。
- **影响**：**当前没坏** —— LuCI 用 `targets.json` + `uname -m` 推断 target（`detect_rust_target`，`luci.oxidns:434-459`），不解析二进制。但如果将来想改成从二进制自身读 target，只能走 ELF header，`--version` / `build-info` 都行不通。

### 11. 完全没有 MIPS / mips64el 发布产物

- **证据**：`release.yml:62-165` 的 13 个 target 里没有任何 mips 变体；slim bundle 同样只有 x86_64 / aarch64。
- **影响**：MIPS OpenWrt 设备无法安装内核。`root/usr/share/oxidns/targets.json` 只映射了 5 个 target（x86_64 / aarch64 / i686 / armv6 / armv7），没有 mips；`detect_rust_target` 会返回 `unsupported_target` —— 行为是一致的，但错误文案没有直接说明"上游未发布 MIPS 包"。
- **建议**：在 `unsupported_target` 的 `detail` 里区分"架构未映射"和"上游未发布该架构"。

### 12. 归档布局是硬约束：根目录平铺，无套子目录

- **证据**：`release.yml:262-271` 用 `tar czf … -C "$DIR" "$EXE" config.yaml LICENSE webui`；`config.yaml` 恒定存在于根（minimal 是改名而来）。
- **影响**：`install_core_from_release` 硬编码 `$UNPACK_DIR/oxidns`（检查在 `luci.oxidns:1501`）、`$UNPACK_DIR/config.yaml`（`luci.oxidns:1509`）、`$UNPACK_DIR/webui`（`luci.oxidns:1536`）。上游哪天加一层目录就会静默失败。
- **建议**：可以在 integration-check 里加一条针对真实 release 归档布局的断言，或至少在 `AGENTS.md` 的"仓库关系"里写明这条契约。

---

## P4 仓库卫生与流程

### 13. `scripts/write-ar-archive.mjs` 是死代码

- **证据**：build 脚本已完全改用 tar（commit `2a5e0c5` "refactor: replace ar commands with tar equivalents"），全仓库只有 `scripts/check.sh:62` 还在对它做 `node --check`（2a5e0c5 删除的是另一文件 `scripts/write-ar.mjs`；`write-ar-archive.mjs` 本身仍在，仅剩这一处引用）。
- **建议**：删掉文件，同时从 `check.sh` 的脚本列表里移除。

### 14. 版本号硬编码在四处，`-r1` 重复两处

- **证据**：`Makefile:4-5`（`0.1.0` / `PKG_RELEASE:=1`，包文件名里的 `-r1` 由 `luci.mk` 生成）、`scripts/build-luci-package.sh:6-11`（默认值 + `PKG_FILE_BASE` / `I18N_FILE_BASE` 里的 `-r1`）、`.github/workflows/ci.yml:23`（`scripts/build-luci-package.sh 0.1.0 dist`）、`scripts/release-check.sh:5,8-9`。
- **影响**：发版目前只靠脚本参数驱动；`Makefile` 走 OpenWrt buildroot 是独立的一条路，容易和脚本产出的版本漂移。
- **建议**：单一来源（比如一个 `VERSION` 文件），各处读取。

### 15. `check.sh` 不检查 PO 里多余的 msgid

- **证据**：`check.sh:47-56` 只校验 required↔POT/PO、POT↔required 两个方向，从不做 PO↔POT 对比。当前 `po/zh_Hans/oxidns.po` 有 127 条 msgid 而 POT 是 126 条（2026-11-16 实测，与 `check.sh` 的 `poIds` 同口径，均不含头部 `msgid ""`），多出来的那条被静默放行：`Remove the OxiDNS core binary? Configuration and runtime data will be preserved.`（前端删除确认文案改动后遗留的陈旧翻译）。
- **建议**：补一个 "PO stale msgid" 检查。注意 PO 本来就允许比 POT 多（历史遗留），若要严格对齐，可直接删除上面那条 msgid（或恢复前端旧文案）。

### 16. rpcd 后端已突破 2000 行 shell

- **证据**：`root/usr/libexec/rpcd/luci.oxidns` 2017 行、93 个函数（2026-11-16 实测），同时承担 JSON 构造、HTTP 下载、归档校验、YAML 改写、配置写回、UCI 读写、日志读取、进度日志、锁管理等职责。
- **现状**：有 `check.sh` 的 `sh -n` 和 `integration-check.sh` 的行为断言兜底，暂未失控。
- **建议**：暂不重构。若继续加功能，优先考虑把"归档校验 + 下载"或"YAML 改写"抽成独立脚本，让主文件只做分发。2026-11-16 已定稿完整重构方案（拆分为 sourced 库文件，含分步落地与测试升级），见 review-doc/design/rpcd-backend-refactor.md。

### 17. 开发机没有 POSIX shell，本地跑不了校验脚本

- **现象**：当前环境（Windows + pwsh 7）PATH 里没有 `sh` / `bash` / `busybox`，WSL 也未安装（2026-11-16 复核：PATH 无 sh/grep/sed/wc，`wsl -l` 无已装发行版）。`scripts/check.sh` 的 `sh -n` 部分、`integration-check.sh`、`release-check.sh` 全部无法本地执行。
- **现成出路（未实测）**：`D:\ide\Git\bin\bash.exe` 存在 —— Git for Windows 自带完整 POSIX 工具链（sh/awk/sed/grep/tar），node 也在 PATH；pwsh 7 下 `& 'D:\ide\Git\bin\bash.exe' scripts/check.sh` 理论可直接跑。使用前先实测一次。
- **已尝试**：装 `bash-parser`（npm）做语法检查，但它在**未修改**的 `check.sh` / `build-luci-package.sh` 上就误报（heredoc、`${VAR:-default}` 构造），只能当"无新增回归"的弱信号。busybox.net 不提供 Windows 构建，busybox-w32 的 GitHub Releases 为空。
- **建议**：CI 仍是唯一可靠门禁。改 shell 代码时，本地至少用 `bash-parser` 跑一遍"当前 vs HEAD"对比，确认没有引入新的解析失败。

---

## 相关但未修的观察

- **`webui.root: "./webui"` 是相对 `-d` 而非相对配置文件目录解析的**（`../oxidns/src/api/static_files.rs:19-45` + `start` 在加载配置前 `set_current_dir(working_dir)`）。LuCI 安装到 `/usr/share/oxidns/webui` 并靠 `write_openwrt_default_config` 改写成绝对路径来规避（`config.yaml:17-18` 的注释本身已写明该规则：`Relative paths resolve from -d/--working-dir`，不存在误导）。Debian 包的做法是建 `/var/lib/oxidns/webui` → `/usr/share/oxidns/webui` 符号链接。
- **配置解析是宽松的**：顶层和 `api` / `log` 都没有 `deny_unknown_fields`，未知键被静默忽略（2026-11-16 核对：上游仅 outbound 相关三个子结构启用 `deny_unknown_fields`，见 `../oxidns/src/config/types.rs:353/418/464`）。所以用户手写错键名不会有任何提示。
