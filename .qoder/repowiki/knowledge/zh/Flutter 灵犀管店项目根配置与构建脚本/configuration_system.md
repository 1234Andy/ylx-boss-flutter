## 1. 总体方案

本项目是一个 Flutter（Dart）移动端应用，**没有使用通用的配置框架**（如 `flutter_config`、`dotenv`、`.env`、`config.json`、`application.properties` 等）。配置由三类来源组合而成：

| 配置类型 | 载体 | 加载时机 / 位置 |
|---|---|---|
| 全局常量（API 域名、缓存 key、图片 OSS 前缀、业务键名） | `lib/res/constant.dart` 中的 `Constant` 类 | 编译期常量，被各模块直接引用 |
| 运行时用户偏好 & 登录态 | `sp_util`（基于 `SharedPreferences`） | `main()` 中先调用 `SpUtil.getInstance()` 初始化，随后各处按需读写 |
| 网络基地址 | `lib/net/http_api.dart` 的 `HttpApi.baseUrlYlx` + `lib/net/dio_utils.dart` 的 `configDio(baseUrl: ...)` | `MyApp.initDio()` 在构建时注入；当前代码里写死为 GitHub 测试地址（见下条“约束”） |
| 平台/环境区分 | `kReleaseMode`、`Platform.isAndroid`/`isIOS`、`defaultTargetPlatform` | 通过 `Constant.inProduction` 及 `device_utils.dart`、`push_service.dart` 等平台判断分支 |
| 资源清单 | `pubspec.yaml` 的 `assets:` 和 `fonts:` 段 | Flutter 打包阶段静态嵌入 |

## 2. 关键文件

- `lib/res/constant.dart` — 所有字符串常量的单一入口（token、user、store、rolemap、sysSeat、loginParamResp、remember_*、scan_*、imageBaseUrl 等）。
- `lib/net/http_api.dart` — 所有后端接口路径集中声明，同时维护 API 基地址 `baseUrlYlx = 'https://yun.bypos.net/LxSvr/web/'`。
- `lib/net/dio_utils.dart` — Dio 单例工厂 `DioUtils`，暴露 `configDio({connectTimeout, receiveTimeout, sendTimeout, baseUrl, interceptors})` 用于覆盖默认超时与 baseUrl。
- `lib/net/http_helper.dart` — 统一请求封装，把 `HttpApi.baseUrlYlx` 拼接到相对 URL 上（若 URL 以 http(s):// 开头则跳过拼接）。
- `lib/main.dart` — 应用启动入口，负责 `SpUtil.getInstance()`、日志、推送、主题 Provider、Dio 拦截器注册、MaterialApp 构建。
- `lib/util/login_params_utils.dart` — 对齐后端 boss 项目的 App.vue onLaunch 行为，在首次登录后静默拉取 `/updateEmployeeStorePermission` 并合并 `loginParamResp` 到本地缓存。
- `lib/push/push_service.dart` — 极光推送初始化，根据 `Constant.inProduction` 控制 debug 模式，并读取 `Constant.token` 绑定设备。
- `pubspec.yaml` — 依赖版本、`environment.sdk/flutter` 范围、`assets` 清单、字体 `RobotoThin` 声明。

## 3. 架构与约定

### 3.1 常量集中原则
所有跨模块共享的配置键、URL、OSS 前缀都定义在 `Constant` 中，业务代码一律通过 `Constant.xxx` 引用，而不是散落字符串字面量。例如 token 存储 key 统一用 `Constant.token`，扫码设备枚举值用 `Constant.scanDevice`。

### 3.2 持久化层统一走 `sp_util`
- 登录态（token/user/store/rolemap/sysSeat/allLoginData）、记住密码开关、主题模式、扫码设置等全部经 `SpUtil.getString/putString`、`getInt/putInt`、`getBool/putBool` 存取。
- `main()` 启动时先 `await SpUtil.getInstance()`，保证后续任何地方都能安全读取。
- `LoginParamsUtils` 对 `loginParamResp` 做了 JSON 合并与一级嵌套对象解构（例如把 `{moduleA:{dynamicsCodeFlushTime:60}}` 拍平成顶层字段），并与服务端返回数据做覆盖式合并。

### 3.3 网络基地址的“注释式多环境”管理
`HttpApi.baseUrlYlx` 附近保留了多个开发机 IP 与 `dev.bypos.net` 的历史注释，当前线上值为 `https://yun.bypos.net/LxSvr/web/`。切换环境的唯一方式是编辑该常量——没有编译宏、没有配置文件、没有命令行参数。

### 3.4 运行环境与生产模式
- `Constant.inProduction = kReleaseMode`，Release 下关闭 `debugPrint` 输出。
- 部分功能按平台分支：`push_service.dart` 根据 `defaultTargetPlatform == TargetPlatform.iOS` 选择 APNs 通道；`device_utils.dart` 提供 `isAndroid`/`isIOS`（排除 Web）。
- `isDriverTest`/`isUnitTest` 两个静态标记用于在集成测试中绕过 isolate 解析（见 `DioUtils._request` 中 `!Constant.isDriverTest && data.length > 10*1024` 的 compute 条件）。

### 3.5 资源与字体配置
`pubspec.yaml` 的 `flutter.assets` 显式列出 `assets/data/`、`assets/images/*`、`assets/svg/` 等目录；字体 `RobotoThin` 通过 `fonts` 段声明。这是 Flutter 标准机制，无自定义扩展。

## 4. 观察到的约定与约束

- **约定：所有可序列化/跨进程共享的 key 必须写在 `Constant` 中**。证据：登录相关 key（token、user、store、rolemap、sysSeat、loginParamResp、allLoginData）、记住密码 key（remember_login_type、remember_phone、…）、扫码设置 key（scan_device、scan_qty_mode、scan_continuous）全部集中在 `Constant`，并在 `lib/models/scan_settings.dart`、`lib/pages/user/user_info_page.dart`、`lib/push/push_service.dart` 等处引用。
- **约定：所有后端接口路径集中在 `HttpApi` 类**。新增接口应追加 `static const String xxx = '...'`，不要直接写字符串字面量传给 `request()`。
- **约定：修改服务器基地址只改 `HttpApi.baseUrlYlx`**。`http_helper.dart` 会用它拼接相对路径；其他位置不应硬编码 `https://yun.bypos.net`。
- **约束：Release 模式下禁用 `debugPrint`**。`lib/main.dart` 在 `Constant.inProduction` 为真时把 `debugPrint` 重定向为空函数；`DioUtils` 中也依据 `!Constant.inProduction` 决定是否添加 `LoggingInterceptor`。
- **约束：异步 IO 必须先初始化 `SpUtil.getInstance()`**。`main()` 在 `runApp` 之前 await 它，避免任意组件在冷启动早期访问 SharedPreferences 导致空指针。
- **约束：网络响应默认以 plain 文本接收，再自行 json.decode**。`BaseOptions.responseType = ResponseType.plain`，并由 `parseData` 解析，错误统一走 `ExceptionHandle.handleException`。
- **约束：登录相关数据刷新仅在本 App 运行周期内执行一次**。`LoginParamsUtils._refreshed` 静态布尔防止重复请求，除非显式调用 `forceRefresh()`（用于席位解绑后同步）。
- **约束：图片 URL 统一加 OSS 前缀**。业务侧通过 `${Constant.imageBaseUrl}/$path` 拼接，而非直接使用后端返回的裸路径。
- **注意：`lib/main.dart` 中 `initDio()` 仍传入 `baseUrl: 'https://api.github.com/'`**，这是一个明显的遗留占位值，实际业务请求通过 `HttpApi.baseUrlYlx` 拼接相对路径后由 `http_helper.dart` 发出；因此 GitHub 地址不会成为真实请求基址，但属于易混淆的硬编码残留。
- **未发现 .env / dotenv / 动态配置文件加载逻辑**。项目未引入 `flutter_config`、`dart_dotenv` 等包，也没有 `*.env*` 或 `config/` 目录。
- **未发现独立 feature flag 系统**。功能开关通过 `Constant.inProduction`、`isDriverTest`、`isUnitTest` 以及平台判断实现，没有统一的 FeatureFlag 抽象。
