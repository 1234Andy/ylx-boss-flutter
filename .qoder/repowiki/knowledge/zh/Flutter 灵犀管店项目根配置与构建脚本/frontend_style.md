## 1. 采用的方案

该项目是 Flutter 移动端应用，没有使用 CSS/SCSS/Tailwind 等 Web 样式技术。其 UI 样式体系基于 Flutter `Material` 框架，通过以下层次组织视觉风格：

- **设计 Token（Design Tokens）**：颜色、字号、间距集中在 `lib/res/` 目录下，以静态 Dart 常量暴露。
- **组件级样式**：`TextStyles` 将字号/字重/颜色组合成可复用的 `TextStyle` 常量。
- **全局主题**：`ThemeProvider` 在 `MaterialApp` 上构建 `ThemeData`，统一 `primaryColor`、`scaffoldBackgroundColor`、`textTheme`、`inputDecorationTheme`、`appBarTheme`、`dividerTheme` 等。
- **深浅色模式**：每个 token 都提供 `_dark_` 前缀或同名变量，由 `ThemeProvider.getTheme(isDarkMode: true)` 切换。
- **状态栏/导航栏覆盖**：通过 `ThemeUtils.light` / `ThemeUtils.dark` 的 `SystemUiOverlayStyle` 与 `AnnotatedRegion` 在全局强制一致。

项目依赖中未引入任何第三方 UI 组件库（如 `flutter_material_color_utilities`、`fluent_ui`、`go_router` 等），仅使用官方 `flutter/material` 与 `flutter/cupertino`。图标资源通过 `assets/images/` 和 `assets/svg/` 以 `LoadAssetImage` 加载，字体仅声明了 `RobotoThin`（`pubspec.yaml`）。图表相关使用 `fl_chart` 包，但图表配色仍走项目的 `Colours`。

## 2. 关键文件

- `lib/res/resources.dart` — 统一导出 `colors.dart`、`dimens.dart`、`gaps.dart`、`styles.dart`，并提供 `Images` 图片入口。
- `lib/res/colors.dart` (`Colours`) — 全部颜色 Token，含浅色/深色两套命名。
- `lib/res/dimens.dart` (`Dimens`) — 字号（`font_sp10`…`font_sp18`）与间距（`gap_dp4`…`gap_dp50`）。
- `lib/res/gaps.dart` (`Gaps`) — 水平/垂直 `SizedBox` 占位、Divider、VerticalDivider、空 Widget。
- `lib/res/styles.dart` (`TextStyles`) — 预定义 `TextStyle`（`textSize12/16/18/24/26`、`textBold*`、`textGray*`、`textWhite14`、`text/textDark` 等）。
- `lib/setting/provider/theme_provider.dart` (`ThemeProvider`) — 构造 `ThemeData`，管理 `ThemeMode` 持久化（`sp_util`），启用 `useMaterial3: false`。
- `lib/util/theme_utils.dart` (`ThemeUtils`) — 深浅色判断、`SystemUiOverlayStyle` 常量、`BuildContext` 扩展。
- `lib/main.dart` (`MyApp._buildMaterialApp`) — 挂载 `MultiProvider(ThemeProvider)`、`OKToast`、`MediaQuery(textScaler: TextScaler.noScaling)`、路由、国际化。
- `pubspec.yaml` — 声明 `uses-material-design: true`、字体 `RobotoThin`、所有 `assets/` 目录。

## 3. 架构与约定

### 3.1 Design Token 分层

```text
lib/res/
├── colors.dart   → Colours (颜色)
├── dimens.dart   → Dimens (字号/间距数值)
├── gaps.dart     → Gaps (布局占位 Widget)
├── styles.dart   → TextStyles (TextStyle 组合)
└── resources.dart→ 统一 export + Images
```

- 颜色按用途语义命名：`app_main`、`bg_color`、`material_bg`、`text`、`text_gray`、`line`、`red`、`orange`、`gradient_blue`、`shadow_blue` 等，每个都有对应的 dark 版本（如 `dark_app_main`、`dark_bg_color`、`dark_text`、`dark_line`、`dark_red` 等）。
- 字号采用 `font_spX` 命名（sp = scale pixel），间距采用 `gap_dpX` 命名（dp = density pixel），体现对 Android/iOS 像素密度的关注。
- `Gaps` 为每个间距提供 `hGap*` / `vGap*` 两个方向的 `SizedBox` 实例，避免页面直接写 `SizedBox(width: 16)` 等硬编码值。

### 3.2 文本样式规范

`TextStyles` 将所有 `TextStyle` 集中声明，页面代码应引用 `TextStyles.textSize16` 等常量而非直接构造 `TextStyle`。已定义的样式包括：

- 字号系列：`textSize12`、`textSize16`、`textBold14`、`textBold16`、`textBold18`、`textBold24`、`textBold26`。
- 颜色系列：`textGray14`、`textDarkGray14`、`textWhite14`、`textGray12`、`textDarkGray12`、`textHint14`。
- 默认正文：`text`（浅色）、`textDark`（深色），均带 `textBaseline: TextBaseline.alphabetic` 修复基线问题。

### 3.3 主题与深浅色

`ThemeProvider.getTheme()` 返回完整的 `ThemeData`：

- `useMaterial3: false` —— 明确禁用 Material 3，维持 Material 2 外观。
- `primaryColor` / `secondary` / `indicatorColor` 取自 `Colours.app_main` 或 `Colours.dark_app_main`。
- `scaffoldBackgroundColor` / `canvasColor` 分别映射到 `bg_color` 与 `material_bg` 的深浅变体。
- `textTheme.titleMedium/bodyMedium/titleSmall` 绑定到 `TextStyles`。
- `inputDecorationTheme.hintStyle`、`appBarTheme.systemOverlayStyle`、`dividerTheme`、`cupertinoOverrideTheme` 均区分深浅色。
- `visualDensity: VisualDensity.standard` 固定密度，避免不同设备自动调整。

主题模式通过 `sp_util` 持久化为字符串 `'System' | 'Light' | 'Dark'`，`getThemeMode()` 将其映射为 `ThemeMode`；`setTheme()` 更新后 `notifyListeners()` 触发重建。

### 3.4 状态栏与系统 UI

`main.dart` 在 `runApp` 之前设置系统 UI 模式，并在 `MediaQuery.builder` 中使用 `AnnotatedRegion<SystemUiOverlayStyle>` 注入 `ThemeUtils.dark`，保证无 AppBar 页面也保持一致的状态栏图标亮度。

`ThemeUtils` 还提供 `isDark(context)`、`getIconColor(context)`、`getStickyHeaderColor(context)` 等方法，供业务层按当前主题动态取色。

### 3.5 资源与字体

- 图片通过 `assets/images/` 下的子目录分类（home、store、state、order、login、goods、shop、account、statistic），SVG 放在 `assets/svg/`。
- 自定义字体 `RobotoThin` 在 `pubspec.yaml` 中注册，路径 `assets/fonts/Roboto-Thin.ttf`。
- 图标不依赖 `Icons.*`，而是通过 `LoadAssetImage('ic_arrow_right', ...)` 方式加载。

## 4. 约定与约束

### 观察到的约定（描述性）

- 所有颜色必须从 `Colours` 引用，不应在业务文件中硬编码十六进制颜色。
- 所有字号应从 `Dimens.font_sp*` 或 `TextStyles.textSize*` 引用。
- 所有间距应从 `Dimens.gap_dp*` 或 `Gaps.hGap*` / `Gaps.vGap*` 引用。
- 新建 `TextStyle` 应加入 `TextStyles` 类，而不是在页面中重复构造。
- 新增颜色需同时提供浅色与深色两个常量（遵循现有命名约定）。
- 主题切换通过 `ThemeProvider` 的 `setTheme()` 修改 `sp_util` 中的 `Constant.theme`，再由 `MaterialApp.themeMode` 生效。
- 全局文本缩放被禁用（`MediaQuery(data: MediaQuery.of(context).copyWith(textScaler: TextScaler.noScaling))`），确保文字大小不受系统设置影响。
- 项目显式关闭 Material 3（`useMaterial3: false`），保持 Material 2 的视觉行为。

### 真正被代码/配置强制的规则

- `pubspec.yaml` 中 `uses-material-design: true`，Material Icons 随包打包；若需自定义字体，必须在同一文件的 `fonts:` 段声明，否则字体不可用。
- `lib/main.dart` 在 `MaterialApp` 构造函数中固定传入 `locale: const Locale('zh', 'CN')` 且 `supportedLocales` 仅包含中文，其他语言不会生效。
- `ThemeProvider.getTheme()` 始终设置 `useMaterial3: false` 与 `visualDensity: VisualDensity.standard`，这是运行时强制的主题约束。
- `MyApp._buildMaterialApp` 的 `builder` 始终包裹 `AnnotatedRegion<SystemUiOverlayStyle>(value: ThemeUtils.dark, ...)`，因此任意页面的状态栏样式都会被覆盖为 `ThemeUtils.dark` 的值。
- 主题模式值域由 `ThemeModeExtension.value` 枚举限定为 `['System', 'Light', 'Dark']`，超出该集合的字符串会被 `getThemeMode()` 回退到 `ThemeMode.system`。

### 未观察到（不代表不存在）的内容

- 未发现响应式断点系统（如 `LayoutBuilder` 断点表或自适应容器），`Dimens` 中的数值均为固定 dp/sp。
- 未发现 CSS-in-Dart 工具链（如 `csslib`、`sass_flutter`、`tailwind`），也没有 `.css` / `.scss` 文件。
- 未发现第三方主题生成工具（如 `flutter_gen`、`superglue`）对资源进行类型化生成，图片访问目前仍是手写字符串路径。