## 1. 使用的系统与工具

本项目是一个 Flutter 移动应用，采用三层依赖声明体系：
- Dart/Flutter 层：通过 `pubspec.yaml` + `pubspec.lock` 使用 **Dart pub**。
- iOS 原生层：通过 `ios/Podfile` 使用 **CocoaPods**。
- Android 原生层：通过 `android/app/build.gradle` + `settings.gradle` 使用 **Gradle / AAR**。

CI 使用 Codemagic (`codemagic.yaml`) 执行 `flutter pub get` 和构建脚本，不维护额外的私有仓库或锁文件更新机器人。

## 2. 关键文件

- `pubspec.yaml`：所有 Dart 依赖声明、版本约束与 `dependency_overrides`。
- `pubspec.lock`：锁定解析后的依赖树（由 `flutter pub get` 生成）。
- `ios/Podfile`：iOS 平台 CocoaPods 入口，统一将部署目标提升到 `15.5`。
- `android/app/build.gradle`：Android Gradle 依赖、签名配置、`flatDir` 本地 AAR 仓库。
- `android/app/libs/`：佳博打印机 SDK 的本地 AAR/JAR（`sdk2.aar`、`serial-port.aar`、`jzint.aar`、`fimage.aar`、`libs/*.jar`）。
- `packages/jpush_flutter/`、`packages/qr_code_scanner/`、`packages/scrollable_positioned_list/`：vendored 到仓库内的第三方 Flutter 插件源码。
- `codemagic.yaml`：CI 中 `flutter pub get` 触发依赖安装。

## 3. 架构与约定

### 3.1 Dart 依赖来源（三种方式并存）

| 方式 | 示例 | 说明 |
|---|---|---|
| pub.dev 版本约束 | `dio: ^5.9.0`、`provider: ^6.1.2` | 使用 caret 范围允许兼容升级 |
| 固定精确版本 | `oktoast: ^3.4.0`、`vibration: 3.1.3` | 部分依赖写死具体版本号 |
| 本地 path 引用 | `qr_code_scanner: path: packages/qr_code_scanner`、`jpush_flutter: path: packages/jpush_flutter` | 对需要打补丁的三方包进行本地 fork |
| git 引用 | `flutter_2d_amap` 通过 `git.ref: '2f9657e4'` | 指向特定 commit，未用 tag 或 branch |
| SDK 内依赖 | `flutter`、`flutter_localizations`、`flutter_test` | 通过 `sdk: flutter` 引入 |

### 3.2 本地 vendored 包策略

项目将三个重要的第三方库直接拷贝到根目录 `packages/` 下作为子模块：
- `packages/jpush_flutter`：极光推送 Flutter 插件，注释说明「已打补丁：通知消息 extras 透出下发通道 TYPE_PLATFORM」，通过 `path:` 引用。
- `packages/qr_code_scanner`：二维码扫描插件，包含完整 android/ios/example/test 结构。
- `packages/scrollable_positioned_list`：滚动定位列表组件，同时被 `dependency_overrides` 覆盖以强制使用本地版本。

这种「fork + path 引用」的方式替代了 `dependency_overrides` 仅做版本覆盖的方案，便于在仓库内长期维护补丁。

### 3.3 dependency_overrides 集中治理

`pubspec.yaml` 末尾的 `dependency_overrides` 用于解决上游冲突：
- 强制使用本地 vendored 的 `scrollable_positioned_list`。
- 修复 `record_linux 0.7.2` 与 `record_platform_interface` 接口不兼容问题，固定为 `1.3.1`。

这构成一个显式的「冲突修复清单」，集中出现在同一位置。

### 3.4 iOS CocoaPods 配置

- 全局最低部署目标提升至 `platform :ios, '15.5'`，注释说明「与 ytt_smdc_flutter 保持一致，兼容新版 Xcode 与各插件的部署目标要求」。
- 禁用 CocoaPods 统计：`ENV['COCOAPODS_DISABLE_STATS'] = 'true'`。
- 通过 `flutter_install_all_ios_pods` 自动安装 Flutter 插件对应的 pods。
- `post_install` 中统一将所有 pod 的 `IPHONEOS_DEPLOYMENT_TARGET` 不低于 `15.5`。

### 3.5 Android Gradle 与本地 AAR

- 使用 `dev.flutter.flutter-gradle-plugin` 接入 Flutter 构建。
- 启用华为 AGC 插件 `com.huawei.agconnect`，从 `agconnect-services.json` 生成资源。
- 通过 `flatDir { dirs 'libs' }` 引入佳博打印机 SDK 的本地 AAR/JAR。
- 极光厂商通道依赖：`cn.jiguang.sdk.plugin:oppo:6.2.1`、`cn.jiguang.sdk.plugin:huawei:6.2.1`，并配合 manifest placeholders 注入 AppKey。

### 3.6 CI 中的依赖安装

`codemagic.yaml` 的 `ios-test-build` workflow 执行顺序为：`flutter pub get` → 生成语音密钥 → `flutter build ios --release --no-codesign`。依赖安装完全由 Flutter SDK 驱动，无自定义缓存策略。

## 4. 观察到的约定与约束

- Dart 依赖的版本约束混合使用 caret (`^`) 与精确版本，没有统一的版本策略文档；可升级性由 `pubspec.lock` 锁定。
- 需要修改上游行为的第三方 Flutter 插件一律以 `packages/<name>/` 形式 vendored，并通过 `path:` 引用；无法通过 `dependency_overrides` 解决的则使用 git ref 精确提交。
- 针对上游版本冲突的修复集中在 `pubspec.yaml` 的 `dependency_overrides` 块，作为唯一的「覆盖点」。
- iOS 平台的原生依赖统一通过 CocoaPods 管理，且最低部署目标被强制拉齐到 `15.5`，任何低于此值的 pod 都会被 `post_install` 修正。
- Android 端非 pub.dev 的二进制依赖（佳博打印机 SDK、极光厂商通道）统一通过 Gradle `implementation` 声明，AAR/JAR 放在 `android/app/libs/` 并通过 `flatDir` 暴露。
- 敏感配置（JPush AppKey、OPPO/Huawei 厂商通道 Key、keystore 属性）以明文写在 `build.gradle` 和 `local.properties` 中，不在代码中硬编码——但这是安全层面的约定，不属于依赖管理范畴。
- 项目未使用私有 pub 仓库、`pubspec.yaml` 中未配置 `publish_to: 'none'`，也未见 `.npmrc`/`.yarnrc` 等其它语言包管理器配置。