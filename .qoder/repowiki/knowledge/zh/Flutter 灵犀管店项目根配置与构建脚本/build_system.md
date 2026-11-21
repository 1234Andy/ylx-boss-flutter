## 1. 构建系统与工具链

本项目是 Flutter 移动端应用「灵犀管店」，采用以下构建体系：

- **Dart/Flutter SDK**：`pubspec.yaml` 锁定 `sdk: '>=3.4.0 <4.0.0'`、`flutter: '>=3.19.0'`。
- **Android 端**：Gradle（由 Flutter 脚手架生成，`android/build.gradle`、`android/app/build.gradle`）。
- **iOS 端**：Xcode / CocoaPods（`ios/Podfile`、`ios/Runner.xcodeproj`）。
- **CI 云构建**：[Codemagic](https://codemagic.io)（`codemagic.yaml`），专门用于在无 Mac 环境下产出 iOS `.ipa`。
- **本地辅助脚本**：PowerShell（`build_apk.ps1`、`build_ipa.ps1`、`download_flutter.ps1`、`get_short_path.ps1`）。

项目未使用 Makefile、Dockerfile 或 GitHub Actions；`.github/workflows/` 为空目录，所有自动化集中在 Codemagic。

## 2. 关键文件

| 文件 | 职责 |
|---|---|
| `pubspec.yaml` | 依赖声明、版本号 `1.0.1+112`、Flutter assets/fonts 清单、`dependency_overrides`（本地覆盖 `scrollable_positioned_list`、`record_linux`） |
| `codemagic.yaml` | Codemagic workflow：`flutter pub get` → 从环境变量注入语音密钥 → `flutter build ios --release --no-codesign` → zip 成 ipa 作为 artifact |
| `build_apk.ps1` | 本地 Android APK 构建：解析 `pubspec.yaml` 版本号 → 清理旧产物 → `--split-per-abi --target-platform android-arm64,android-arm` → 重命名为 `ylx_boss_flutter_<ver>_64.apk` / `_32.apk` |
| `build_ipa.ps1` | Windows 侧的 IPA 处理：查找最新 `.ipa` → 按版本号重命名为 `ylx_boss_flutter_<ver>.ipa` → 打印 Sideloadly 安装指引 |
| `android/app/build.gradle` | Android 应用 Gradle 配置 |
| `ios/Podfile` | iOS CocoaPods 依赖 |
| `lib/util/voice_secret.dart.example` | 语音识别密钥模板（CI 自动生成 `voice_secret.dart`，gitignore） |

## 3. 架构与约定

### 3.1 版本策略
版本号统一在 `pubspec.yaml` 中维护，格式为 `major.minor.patch+buildNumber`（当前 `1.0.1+112`）。两个 PowerShell 脚本通过正则 `^([^+]+)\+(.+)$` 将其拆分为 `versionName.buildNumber`（如 `1.0.1.112`），并通过 `--dart-define=FULL_VERSION=<fullVersion>` 注入到应用代码中供运行时读取。Codemagic 则用 `grep '^version:' pubspec.yaml | awk '{print $2}' | sed 's/+/./'` 做同样转换。

### 3.2 Android 构建流程
```powershell
flutter build apk --release --split-per-abi \
  --target-platform android-arm64,android-arm \
  --dart-define=FULL_VERSION=$fullVersion
```
- 仅打包 ARM 架构（arm64-v8a + armeabi-v7a），排除 x86_64 以减小包体。
- 输出自动重命名为 `ylx_boss_flutter_<ver>_64.apk` 和 `ylx_boss_flutter_<ver>_32.apk`。

### 3.3 iOS 构建流程（无 Mac 环境）
由于团队主要在 Windows 工作，iOS 构建委托给 Codemagic 云端 macOS runner：
1. 上传 zip 包至 Codemagic（排除 `build/`、`.dart_tool/`、`android/`、`ios/Pods`）。
2. CI 执行 `flutter build ios --release --no-codesign`，不签名。
3. 将 `Runner.app` 打包为 `Payload/` 并 zip 成 `ylx_boss_flutter_<ver>.ipa` 作为 artifact。
4. 开发者在 Windows 上运行 `build_ipa.ps1`，配合 [Sideloadly](https://sideloadly.io) 用免费 Apple ID 重新签名并安装到 iPhone（有效期 7 天）。

### 3.4 敏感信息注入
语音识别所需的 `VOICE_APP_KEY`、`VOICE_ACCESS_KEY_ID`、`VOICE_ACCESS_KEY_SECRET` 通过 Codemagic 的环境变量注入，CI 步骤动态生成 `lib/util/voice_secret.dart`（首行注释“本文件由 CI 自动生成，请勿提交”），本地保留 `voice_secret.dart.example` 作为模板。

### 3.5 依赖管理约定
- 常规依赖走 `pubspec.yaml` 的 `dependencies`。
- 需要本地修改的三方库通过 `dependency_overrides` 指向 `packages/<pkg>/` 下的子模块（`scrollable_positioned_list`、`record_linux`）。
- 内嵌了两个完整的 Flutter 插件仓库：`packages/jpush_flutter`（极光推送，已打补丁透出 extras 通道 TYPE_PLATFORM）和 `packages/qr_code_scanner`，通过 `path:` 引用。
- AMap 地图 SDK 直接引用 git ref `2f9657e4`。

## 4. 约定与约束

- **版本号单一来源**：`pubspec.yaml` 中的 `version` 字段是唯一权威值，三个构建入口（APK/iOS/CLI）均通过解析该文件获取版本号，禁止硬编码。
- **Android 产物命名**：APK 输出遵循 `ylx_boss_flutter_<versionName>.<buildNumber>_{32|64}.apk` 命名约定，由 `build_apk.ps1` 强制重命名。
- **iOS 产物命名**：IPA 输出遵循 `ylx_boss_flutter_<versionName>.<buildNumber>.ipa` 命名约定，由 Codemagic 打包阶段和 `build_ipa.ps1` 共同遵守。
- **架构裁剪**：Android 构建固定只包含 `android-arm64,android-arm` 两种 ABI，显式排除模拟器架构。
- **iOS 免签名**：Codemagic 使用 `--no-codesign` 构建，安装包必须由 Sideloadly 重新签名，不能直接使用 CI 产出的 ipa 上架。
- **密钥隔离**：`lib/util/voice_secret.dart` 由 CI 自动生成，不得提交；本地开发需从 `.example` 复制后填入占位值。
- **无 Docker/GitHub Actions**：整个构建管线基于 Codemagic + 本地 PowerShell，未在仓库中发现 Dockerfile 或 GitHub Actions workflow。
- **Flutter 版本基线**：`environment` 块要求 Dart SDK ≥3.4.0、Flutter ≥3.19.0，低于此版本的工具链无法解析依赖。