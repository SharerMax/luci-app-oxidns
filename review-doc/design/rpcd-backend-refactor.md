# rpcd 后端拆分重构方案

- **状态**：设计定稿，未实施
- **关联**：[issues.md](../issues.md) #16（rpcd 后端已突破 2000 行 shell）
- **基线**：`luci-app-oxidns@561bd16`（分支 `fix/restart`；截至本文撰写，其后 3 个提交尚未合入 `main`）。文中 `luci.oxidns` 行号均指该提交下的 `root/usr/libexec/rpcd/luci.oxidns`（2017 行 / 93 个函数）。

## 1. 背景与现状

单文件同时承担：JSON 构造、HTTP 下载、归档校验、YAML 改写、配置写回、UCI 读写、日志读取、进度日志、锁管理、服务控制、core 安装/上传/移除。

现状问题：

- 所有函数变量均为全局（全文件未用 `local`），嵌套调用存在变量遮蔽风险；
- 两段逐字节相同的脱敏 awk：`sanitize_progress_text`（110-127）与 `read_error_detail_file`（1034-1050）内嵌的是同一段；
- 安装二进制段（1530-1534 与 1719-1723）与 WebUI 拷贝段（1536-1544 与 1725-1733）在 release 安装与上传安装两条路径上逐行重复；
- lib 级逻辑（`asset_name_for_bundle`、`write_openwrt_default_config`、`oxidns_named_path` 等）只能通过整脚本端到端调用，无法单测——这也卡住了 issues.md #1 / #2 的测试铺设与 #4 的 `run_init_action` 覆盖。

## 2. 设计原则

1. **接口冻结**：ubus 对象名、`list`/`call` 方法名、stdin/stdout JSON 形状、错误码一律不变。对外零差异，纯内部搬移。
2. **纯搬移优先**：现有代码全部是函数定义，执行只发生在文件底部 `case`（基线 1931-2016 行），拆文件本身不改变任何行为。风险集中在「lib 目录定位」与「打包」两件事上，均有对策（见 §7）。
3. **每步绿灯**：每个 commit 后 `scripts/check.sh` + `scripts/integration-check.sh` 必须全过；现有端到端断言一行不删，它们就是「接口冻结」的证明。
4. **保持 busybox ash 方言**：不引入 bashism/数组；保持现状不使用 `set -e`（错误处理是手动的）；可以使用 `local`（busybox ash 原生支持），重构中函数内变量一律补 `local`。

## 3. 目标布局

库文件放 `/usr/share/oxidns/rpcd/lib/*.sh`（644、非可执行——避免 rpcd 扫描 `/usr/libexec/rpcd` 时误执行），主文件只留常量 + source + 分发：

| 文件 | 内容（基线行号） | 约行数 |
|---|---|---|
| `lib/fs.sh` | `stat_mtime`:588、`file_size_bytes`:237 | 25 |
| `lib/json.sh` | `json_*` 全家：33-107、`json_value_from_string`:393、`json_field_string`:620、`input_value/has_key/bool`:767-795、`safe_single_line_value`:611 | 160 |
| `lib/redact.sh` | 合并两处重复的脱敏 awk（§1 问题 2），导出 `redact_stream` | 40 |
| `lib/paths.sh` | `path_is_under`:797、`oxidns_named_path`:809、`safe_remove_config_path`:822、`safe_remove_working_dir`:833、`path_has_parent_ref`:848、`valid_config_path_setting`:857、`valid_working_dir_setting`:863 | 85 |
| `lib/target.sh` | `detect_rust_target_from_file`:403、`detect_rust_target`:434（保留 `OXIDNS_TARGETS_FILE` 环境覆写） | 65 |
| `lib/yaml.sh` | `yaml_scalar_clean`:308、`config_api_listen`:312、`webui_url_from_config`:337、`listen_host_from_config`:357、`is_loopback_host`:369、`is_wildcard_host`:378 | 95 |
| `lib/settings.sh` | `uci_get`:263、`load_settings`:273、`settings_value`:655、`emit_settings_read`:627、`emit_settings_save`:665 | 180 |
| `lib/progress.sh` | 进度目录/文件准备：129-158、`core_progress_*`:160-180、`emit_core_progress`:245、锁：182-235（`pid_is_running`/`clear_stale_core_lock`/`acquire_core_lock`/`release_core_lock`/`run_with_core_lock`） | 170 |
| `lib/download.sh` | `read_error_detail_file`:1031、`set_download_error_from_file`:1053、`run_download_command`:1065、`download_file`:1115、`require_download_transport`:1152 | 160 |
| `lib/archive.sh` | `sha256_file`:1161、`asset_name_for_bundle`:1165、`release_api_url`:1181、`version_tag`:1191、`select_release_asset`:1199、`valid_sha256`:1266、`validate_safe_archive`:1270 | 170 |
| `lib/service.sh` | `init_script`:288、`service_enabled/running`:292-306、`syslog_service_event`:533、`run_init_action`:542、`run_service`:553 | 110 |
| `lib/config-io.sh` | `validate_config_file`:869、`emit_config_read`:592、`emit_config_validate`:880、`emit_config_save`:910 | 170 |
| `lib/status.sh` | `valid_ipv4`:1306、`lan_ipv4_from_uci`:1318、`lan_ipv4_from_interface`:1328、`default_api_listen`:1341、`build_info_json`:387、`emit_status`:461 | 130 |
| `lib/logs.sh` | `emit_log_lines_json`:975、`read_logread_logs`:990、`emit_logs_recent`:1018 | 75 |
| `lib/core-install.sh` | `write_openwrt_default_config`:1352、`core_install_error`:1399、`install_core_from_release`:1412、`emit_core_install/reinstall`(+locked):1604-1637；新抽共享函数 `install_core_binary`、`install_webui_dir` | 340 |
| `lib/core-upload.sh` | `upload_core_error`:1639、`valid_upload_path`:1653、`validate_uploaded_binary`:1660、`install_uploaded_core_from_unpack`:1682、`emit_core_upload_install`(+locked):1801-1844 | 240 |
| `lib/core-remove.sh` | `emit_core_remove_locked`:1846、`emit_core_remove`:1927 | 115 |
| 主文件 `luci.oxidns` | 安装路径常量（`OXIDNS_BIN`/`WEBUI_DIR`/`DEFAULT_*`/`TMP_ROOT`）、`LIB_DIR` 解析、按序 source、`case list/call` 分发 | 120 |

第三处重复消除：release 安装与上传安装中的「安装二进制」（1530-1534 / 1719-1723）与「拷贝 WebUI」（1536-1544 / 1725-1733）抽成 `core-install.sh` 里的 `install_core_binary` / `install_webui_dir`，两条路径共用。

## 4. 状态与依赖规则

- **跨模块全局变量白名单**（唯一允许的隐式契约，写入每个模块头部注释）：
  - `load_settings` 产出：`CONFIG_PATH` `WORKING_DIR` `CORE_REPOSITORY` `CORE_BUNDLE` `LOG_SYSLOG` `DOWNLOAD_PROXY` `GITHUB_TOKEN`；
  - download 产出/自用：`DOWNLOAD_ERROR` `DOWNLOAD_RC`；
  - archive 产出：`SELECT_TAG` `SELECT_ASSET_NAME` `SELECT_URL` `SELECT_SHA256` `SELECT_SIZE`；
  - core 安装/上传/移除自用：`CORE_TMP_DIR` `UPLOAD_PATH`。
  - 白名单之外的跨模块全大写变量引用由 check.sh 门禁（§6）。
- **`LIB_DIR` 定位**：主文件 `LIB_DIR="${OXIDNS_LIB_DIR:-/usr/share/oxidns/rpcd/lib}"`。env 覆写仅供测试；不依赖 `$0` 推导，规避 rpcd 调用方式差异。source 失败时输出 `lib_missing` JSON 错误并退出，不静默。
- **source 顺序**（依赖可读性，shell 在调用时才解析函数，顺序不影响行为）：`fs → json → redact → paths → target → yaml → settings → progress → download → service → archive → config-io → status → logs → core-install → core-upload → core-remove`。
- **模块私有常量留在各自模块**（如 `CORE_PROGRESS_DIR` 归 progress.sh），安装路径类常量（`OXIDNS_BIN` 等）留在主文件。

## 5. 分步落地（每步一个 commit，全程绿灯）

1. `refactor(rpcd): extract leaf helpers` — fs/json/redact/paths/target/yaml 六个无状态模块。`redact.sh` 合并重复 awk 是本步唯一的非纯搬移，补断言：带 token 的进度日志输出 `[redacted]`。
2. `refactor(rpcd): extract settings, progress and download` — 跑通现有锁/进度/symlink 防护断言（integration-check.sh 基线 121-135、249-267 行）。
3. `refactor(rpcd): extract service, config-io, status and logs` — 含 `run_init_action`（issues.md #4 的修复逻辑原样搬移，不改动）。
4. `refactor(rpcd): extract core install/upload/remove` — 最大一步，含 `install_core_binary` / `install_webui_dir` 抽取与重复消除。
5. `refactor(rpcd): reduce entry file to dispatch` — 删除主文件中已迁出代码；AGENTS.md 的 Key Paths 增补新 lib 路径一行。

每步均为纯搬移或机械抽取，review 逐 commit 对 diff 即可。

## 6. 校验策略升级（重构的回报）

拆分后 `integration-check.sh` 可以 **source 单个 lib 直接测函数**——单文件时代做不到：

- `asset_name_for_bundle`：full/standard/minimal 三种命名断言；
- `select_release_asset`：fixture release.json 正反两例；
- `write_openwrt_default_config`：full 配置改写断言 + 注释掉 `api:` 块的配置 exit 42 断言（为 issues.md #1 的修复铺测试）；
- `oxidns_named_path` / `valid_config_path_setting`：接受/拒绝表（为 issues.md #2 的黑名单改造铺测试）；
- `config_api_listen`：各形态 YAML 提取断言。

配套：

- `check.sh` 的 `sh -n` 列表追加全部 lib 文件；
- check.sh 增加全局变量门禁：lib 文件中引用白名单外全大写变量即失败；
- integration-check.sh 打包段补两条断言：ipk/apk 的 data 归档中含 `usr/share/oxidns/rpcd/lib/*.sh`（打包脚本按 `root/` 树 find 生成文件列表，新文件自动进包，断言防将来打包逻辑改动遗漏）。

## 7. 风险与对策

| 风险 | 对策 |
|---|---|
| rpcd 环境下 lib 缺失（半升级状态） | 主文件 source 失败即输出 `lib_missing` JSON 错误退出，不静默 |
| busybox ash 方言差异 | 不引入新语法；`local` 为 ash 原生支持，integration-check 第一步先 source 一个 lib 验证 |
| 全局变量意外遮蔽 | 函数内补 `local` + 每步 diff review + 白名单 grep 门禁 |
| ipk/apk 打包遗漏 | §6 的两条 tar 成员断言；`postinst` 已重启 rpcd，无需改动 |
| 接口漂移 | 现有端到端断言一行不删；`list` 输出与方法名集合保持逐字一致 |

## 8. 实施顺序建议

**先做本重构，再修 issues.md #1 / #2 / #3**。三个修复分别落在 `core-install.sh`（#1、#5）、`paths.sh`（#2）、`settings.sh` + 前端（#3）——在数百行的模块里修比在 2000+ 行的单文件里修清晰得多，且本方案铺好的 lib 级测试正好接住这三个修复。#4 / #5 / #6 已闭环，不受影响。
