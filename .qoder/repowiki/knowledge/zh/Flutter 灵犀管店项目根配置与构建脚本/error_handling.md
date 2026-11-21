## 1. 整体方案

本项目采用「Dio 拦截器 + 统一响应封装 + 业务码归一化 + 全局异常捕获」四层架构，覆盖从底层 HTTP 到 UI 层的错误传播。

- **HTTP 层**：基于 `dio`，通过 `AuthInterceptor`、`TokenInterceptor`、`LoggingInterceptor`、`AdapterInterceptor` 四个拦截器链式处理鉴权、token 刷新、日志、响应体适配。
- **响应层**：所有接口返回统一为 `BaseEntity<T>`（`code/message/data`），业务状态码由服务端定义（成功码 `0`，见 `res/constant.dart` 的 `Constant.code`）。
- **异常层**：`ExceptionHandle` 将 `DioException` / `SocketException` / `HttpException` / `FormatException` 等底层异常映射为统一的 `NetError(code, msg)`。
- **UI 层**：`handleError` 重写 `FlutterError.onError` 并通过 `runZonedGuarded` 捕获未捕获异常，debug 模式打印堆栈，release 模式接入上报（注释中预留了 flutter_bugly 集成点）。

## 2. 核心文件

| 文件 | 职责 |
|---|---|
| `lib/net/error_handle.dart` | 定义 HTTP 状态码常量、自定义错误码（`net_error=1000` 等）、`NetError` 类型、`ExceptionHandle.handleException` 映射逻辑、`DioErrorTypeExtension` 扩展 |
| `lib/net/base_entity.dart` | 通用响应实体 `BaseEntity<T>`，字段 `code?` / `message` / `data?` |
| `lib/net/dio_utils.dart` | Dio 单例工厂、`requestNetwork` / `asyncRequestNetwork` 统一请求入口，在 `.then` 中根据 `BaseEntity.code == 0` 分派 onSuccess/onError |
| `lib/net/intercept.dart` | 鉴权、自动刷新 token、日志、响应体适配器（把非标准 REST 响应统一成 `{"code":...,"message":"..."}` 或透传 `retcode` 格式） |
| `lib/util/handle_error_utils.dart` | 全局异常捕获 `handleError(body)`，按 `Constant.inProduction` 分支调试/生产行为 |
| `lib/main.dart` | 启动时调用 `handleError(() async { ... })` 包裹应用初始化 |

## 3. 架构与约定

### 3.1 错误码体系
`ExceptionHandle` 集中声明两类常量：
- HTTP 状态码：`success=200`、`success_not_content=204`、`not_modified=304`、`unauthorized=401`、`forbidden=403`、`not_found=404`
- 自定义网络错误码：`net_error=1000`、`parse_error=1001`、`socket_error=1002`、`http_error=1003`、`connect_timeout_error=1004`、`send_timeout_error=1005`、`receive_timeout_error=1006`、`cancel_error=1007`、`unknown_error=9999`

`_errorMap<int, NetError>` 将上述错误码映射到中文提示文本；`DioErrorTypeExtension` 把 `DioExceptionType.index` 映射到对应自定义码。

### 3.2 请求链路中的错误传播
```
Dio → AdapterInterceptor（统一响应体）→ BaseEntity.fromJson → DioUtils.requestNetwork
```- `validateStatus: (_) => true`：Dio 不根据 HTTP 状态码抛错，全部交给 `AdapterInterceptor` 和上层判断。
- `AdapterInterceptor` 对 `200/204` 直接透传原始响应；对 `404` 改写为标准失败体；对其他状态码尝试解析 `message`/`msg` 字段构造标准失败体。若响应体本身包含 `retcode` 字段则原样透传。
- `_request<T>` 内部 `catch(e)` 返回 `BaseEntity(ExceptionHandle.parse_error, '数据解析错误！', null)`。
- `requestNetwork` / `asyncRequestNetwork` 在 `.then` 中检查 `result.code == 0`，否则走 `_onError(code, message, onError)`；外层 `onError` 回调通过 `ExceptionHandle.handleException(e)` 生成 `NetError` 后传入 `_onError`。
- `_onError` 统一 `Log.e('接口请求异常： code: $code, mag: $msg')`，并调用业务层传入的 `NetErrorCallback`。

### 3.3 Token 过期处理
`TokenInterceptor.onResponse` 检测到 `statusCode == ExceptionHandle.unauthorized(401)` 时，使用独立 `Dio` 实例调用 `lgn/refreshToken` 刷新 token，成功后用新 token 重发原请求（避免递归触发拦截器）。

### 3.4 全局异常捕获
`handleError` 在 `main.dart` 启动时执行：
- debug：`FlutterError.dumpErrorToConsole(details)` + `debugPrintStack`。
- release：`Zone.current.handleUncaughtError(...)`，并预留上传服务器逻辑（注释提到可直接接入 `flutter_bugly`）。

## 4. 观察到的约定与约束

- **业务成功码固定为 0**：`BaseEntity.fromJson` 取 `json[Constant.code]`，`DioUtils.requestNetwork` 以 `result.code == 0` 作为成功分支——这是代码实际执行的约定。
- **所有 HTTP 请求必须经 `DioUtils` 的 `requestNetwork` / `asyncRequestNetwork`**：拦截器在 `DioUtils` 构造函数中统一注册，业务侧未出现绕过该入口的直接 `dio.request` 调用。
- **UI 层不直接 catch Dio 异常**：业务回调签名统一为 `NetSuccessCallback<T?>` / `NetErrorCallback = void Function(int code, String msg)`，错误信息以字符串形式传给 UI。
- **全局异常入口唯一**：应用启动只通过 `handleError` 包装 `runZonedGuarded`，并在 `FlutterError.onError` 内按 `Constant.inProduction` 分支选择控制台输出或上报。
- **无自定义 Dart Exception 类**：项目未定义专门的 `XxxException` 类型，而是用 `NetError` 值对象承载错误码+消息，配合枚举化的错误码常量传递语义。
- **未使用 `try/catch` 之外的 panic/recover 机制**：未见 `throw` 自定义异常上抛到调用方，也未发现 `try/catch` 后向上抛出的模式——错误在 `DioUtils` 层即被消费。
- **401 自动重试有防环保护**：`TokenInterceptor` 使用独立的 `_tokenDio` 发起 refresh 请求并重发原请求，避免重新进入拦截器链导致死循环。

## 5. 适用性判定

本仓库存在完整的网络错误映射、统一响应封装、拦截器链以及全局异常捕获，属于中等成熟度的错误处理体系，适用于 Flutter/Dio 移动端项目。