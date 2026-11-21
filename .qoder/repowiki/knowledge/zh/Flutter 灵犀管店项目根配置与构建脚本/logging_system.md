## 1. 总体方案

本项目采用「两层日志」架构：
- **控制台/调试日志**：基于 `common_utils` 的 `LogUtil`，通过自定义 `lib/util/log_utils.dart` 中的 `Log` 类封装，统一 tag、按环境开关。
- **持久化日志**：自研 `FileLogWriter`（`lib/util/file_log_writer.dart`），异步批量写入本地磁盘，按日期+分类分文件；配合 `log_upload_utils.dart` 将最近 N 天的 `.log` 打包 zip 后通过 FTP 上传到远端服务器。

项目没有引入第三方日志框架（如 `logging`、`logger`、`pretty_dio_logger`），所有日志均由仓库内工具类实现。`main.dart` 中显式关闭 Release 模式的 `debugPrint`，并通过 `Log.init()` 初始化两套日志系统。

## 2. 关键文件与职责

| 文件 | 职责 |
|---|---|
| `lib/util/log_utils.dart` | 控制台日志门面 `Log`，封装 `d/e/json`，基于 `common_utils.LogUtil`，tag 固定为 `DEER-LOG` |
| `lib/util/file_log_writer.dart` | 本地文件日志写入器，异步队列 + 批量刷盘，分类：`json_writer`（运行日志）、`crash_log`（操作+接口审计） |
| `lib/util/log_upload_utils.dart` | 日志压缩上传：收集最近 N 天 `.log` → zip → FTP 上传 → 删除本地 zip |
| `lib/main.dart` | 入口初始化：Release 下禁用 `debugPrint`，调用 `Log.init()` 和 `FileLogWriter.instance.init()` |
| `lib/res/constant.dart` | `inProduction` 等常量，控制日志是否输出 |

## 3. 架构与设计决策

### 3.1 控制台日志 (`Log`)
- 统一 tag = `DEER-LOG`，可通过参数覆盖。
- `d(msg)` / `e(msg)`：仅在 `!Constant.inProduction` 时调用 `LogUtil.v/e`。
- `json(msg)`：尝试 JSON 解析，若为 Map/List 则走 `_printMap` / `_printList` 做缩进美化；列表长度 >50 或嵌套过深时直接扁平输出，避免单行过长被系统限制。
- `_canFlattenMap` 在 `list.length * map.length > 100` 时扁平化，防止日志行数爆炸。

### 3.2 本地文件日志 (`FileLogWriter`)
- **存储位置**：`getApplicationDocumentsDirectory()/log/{yyyy-MM-dd}_{category}.log`。
- **写入模型**：`StreamController.broadcast` + `_batchBuffer`，阈值 10 条或 500ms 触发一次 `_flushBatch`，降低高频 IO。
- **分类**：
  - `json_writer`：通用运行日志，`writeToFile(data, {tag})`，自动 `json.encode`。
  - `crash_log`：操作审计 + 接口错误，分别由 `writeOperationLog(page, action, detail?)` 和 `writeErrorLog(error, url, params, errorTips, {data})` 写入，形成按时间顺序的完整操作轨迹。
- **单条截断保护**：超过 `_maxLogLength = 650000` 字节时截断并追加原始长度提示。
- **设备信息**：`logDeviceInfo()` 在登录成功后记录设备基础信息。
- **过期清理**：`deleteOldLogs()` 保留天数按月区分：1-2月保留 20 天，其他月份保留 10 天。
- **Web 平台跳过**：`kIsWeb` 时不初始化文件系统相关逻辑。

### 3.3 日志上传 (`LogUploadUtils`)
- 收集文件名形如 `yyyy-MM-dd_category.log` 的文件，仅包含最近 N 天的日志。
- 压缩包命名：`{date}_{CLIENT}_{acc}_{storeCode}_logs.zip`，其中 CLIENT=`BOSS_APP`，acc/storeCode 从 `SpUtil(Constant.store)` 读取。
- 使用原生 `dart:io.Socket` 实现 FTP PASV 模式，硬编码服务器地址、端口、账号密码及远程目录。
- 上传完成后删除本地 zip。

### 3.4 初始化流程
`main.dart` 中：
```dart
debugPrint = (String? message, {int? wrapWidth}) {}; // Release 禁用
devToolEnable = false;
Log.init();           // 初始化 LogUtil + inProduction
// ...
FileLogWriter.instance.init(); // 启动文件日志
```
Release 构建会再次调用 `Log.init()`，确保生产环境不输出控制台日志。

## 4. 约定与约束

- **控制台日志**：业务代码应通过 `Log.d / Log.e / Log.json` 输出，而非直接使用 `print` / `debugPrint`；`LogUtil` 是底层实现，对外暴露的 API 是 `Log` 类。
- **生产环境抑制**：`Constant.inProduction` 为 true 时，`Log.d/e/json` 均不输出；`main.dart` 同时把全局 `debugPrint` 替换为空函数。
- **日志分类规范**：`FileLogWriter` 只接受两个 category —— `json_writer`（运行日志）和 `crash_log`（操作+接口日志）；新增分类需修改该常量定义。
- **文件名格式**：本地日志必须遵循 `{yyyy-MM-dd}_{category}.log`，否则 `LogUploadUtils` 无法识别日期进行过滤。
- **最大日志长度**：单条日志超过 650KB 会被截断，避免单个 log 文件过大影响传输。
- **批量策略**：写入以 10 条或 500ms 为一批刷新，非实时落盘，但保证最终一致。
- **保留天数规则**：1-2月保留 20 天，其他月份保留 10 天，由 `FileLogWriter._keepDays` 与 `LogUploadUtils._keepDays` 共同执行。
- **FTP 上传凭据**：主机、端口、用户名、密码、远程目录均为 `LogUploadUtils` 内的 `const` 字段，未配置外部化，属于仓库内硬编码约束。
- **Web 平台行为差异**：`FileLogWriter` 在 Web 上跳过文件系统初始化，不产生本地日志文件；`Log` 的控制台输出仍受 `inProduction` 控制。

## 5. 使用分布

约 825 处 `print/debugPrint/Log.` 调用分布在 105 个文件中，主要集中在网络层 (`net/http_helper.dart`)、推送 (`push/push_service.dart`)、页面 (`pages/*`) 以及工具类 (`util/*`) 中。`FileLogWriter` 的使用范围较窄，主要位于 `push_service.dart`、`http_helper.dart`、`login_page.dart`、`main_home_page.dart` 等业务关键路径，用于操作审计与接口追踪。