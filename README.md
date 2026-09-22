# Saber 精简自定义版（基于官方 v1.36.1）

在官方 [Saber](https://github.com/saber-notes/saber)（Flutter 手写笔记 App）**v1.36.1** 之上做的一个个人自用版本：
**删掉**遥测、云同步与各类外链，**补上**官方当时没有的图片编辑手感与 PDF 选页导出，目标是"干净、离线、图片能改"。

本仓库**不是完整源码**，而是**补丁集**：`patches/` 里的 14 个补丁用 `git format-patch` 从上游 tag `v1.36.1` 导出，可逐条复现。

> 许可：**GPL-3.0**（沿用上游许可），见 [LICENSE](LICENSE)。再分发请保留许可与上游署名。

## 这 14 个补丁在做什么

| # | 主题 | 内容 |
|---|---|---|
| 0001 | 瘦身 | 删除 Sentry 遥测、更新检查、赞助/隐私外链、Nextcloud 云同步、官方账号登录入口 |
| 0002 | 新功能 + 去依赖 | 新增 7 个功能，移除 `super_clipboard`（连带不再需要 Rust 工具链） |
| 0003–0004 | 收尾 | 修测试断言、更新 `.gitignore` |
| 0005 | **修闪退** | 补删 `AndroidManifest.xml` 里 `super_native_extensions` 的遗留 `<provider>` |
| 0006 | 导入与打开 | 导入图片改走系统相册（多选）、支持「用 Saber 打开」外部文件 |
| 0007 | 导出 | PDF 按页范围导出（「PDF 选页」） |
| 0008 | 前置修复 | 让 `srcRect` 真正参与渲染（裁剪功能的前置条件） |
| 0009–0014 | 图片编辑 | 裁剪 + 动作条、把手命中范围、复制/置顶/置底/真·反色、自由旋转 + 水平翻转、旋转手感（箭头跟手、不裁边、动作条不跳） |

详细的技术记录（根因、被证伪的猜测、验证方式）写在每个补丁的提交信息里，`git log` 可直接读。

## 怎么用

```bash
git clone https://github.com/saber-notes/saber.git
cd saber
git checkout v1.36.1

git am /path/to/patches/*.patch     # 顺序应用 14 个补丁
flutter pub get

flutter build apk --release --split-per-abi
# 产物：build/app/outputs/flutter-apk/app-arm64-v8a-release.apk
```

**必须用 `flutter build apk`，不要直接调 `gradlew assembleRelease`**：直接调 gradlew 时 Flutter 不会过滤 dev 依赖插件，`GeneratedPluginRegistrant` 会注册 `integration_test`，而它的类不在编译类路径上，报 `程序包 dev.flutter.plugins.integration_test 不存在`。

### 安装前必读

- 本版 APK 用项目自带的 `android/fallback-key.jks` 签名，**与官方版签名不同** → 必须先**卸载官方 Saber**，否则提示"应用未安装/签名冲突"。
- 卸载会连带清空应用私有目录（上游清单里写了 `allowBackup="false"`）。**已有笔记的话，先用"设置 → 自定义数据目录"把笔记挪到共享文件夹，或先导出备份，再卸载。**

## 主要改动详情

### 删除（瘦身）

Sentry 遥测与崩溃上报、更新检查、赞助/隐私外链、Nextcloud 云同步与配额 UI、官方账号登录入口。相应地，设置页不再有这些开关。

### 导入图片走系统相册

原来走 SAF 文档选择器（看不到文件在哪的那种）。现在安卓/iOS 走 `image_picker.pickMultiImage()` 并开启系统 Photo Picker（安卓 13+ 直接是相册 UI，可多选、无需权限），不支持时回退到相册式 `ACTION_GET_CONTENT`；桌面平台仍用文件选择器。

- **扩展名以魔数（文件头）判定**，因为相册返回的是系统临时副本、文件名不可靠，而扩展名会被写进 `.sbn2` 笔记数据，判错会让笔记里的图片坏掉。
- 不支持的格式（典型如 HEIC）会明确提示并跳过，不把坏数据写进笔记；SVG/PSD/EXR 等仍有「从文件选择」入口。

### 「用 Saber 打开」报 `GoException: no routes for location: content://...`

根因两层（均经源码/反编译确认）：Flutter 安卓嵌入层默认开启自动深链，把 `ACTION_VIEW` 的 `content://` URI 送进了路由通道；且 `flutter_sharing_intent` 把 `ACTION_VIEW` 当作 URL 而非文件。

修法：`MainActivity` 覆盖 `shouldHandleDeeplinking() = false`（清单里另加 `flutter_deeplinking_enabled=false` 作为冗余保险）；新增 MethodChannel 把 `content://` 复制到 `cacheDir/incoming/` 再把真实路径交给 Dart；`ACTION_SEND` 与 `ACTION_VIEW` 共用同一段导入逻辑，并处理冷启动与热启动两条路径。**无需新增权限**（`ACTION_VIEW` 自带读权限）。

### PDF 选页导出

导出条新增「PDF 选页」：填起止页（1-based 闭区间），带「当前页」「全部」快捷按钮与实时校验（越界/颠倒时导出按钮禁用）。文件名带页码范围，例如 `笔记名_p3-7.pdf`；原来的「PDF」按钮行为与文件名完全不变。

- 末尾那一页空白页（Saber 一直留一页供书写）永远不在可选范围内。
- 一处**有意的行为变更**：空白笔记以前会静默生成 0 页 PDF，现在会提示"本笔记没有可导出的页"。

### 图片裁剪与动作条

选中图片后，图片**下方**浮出动作条：**裁剪 / 重置 / 反色 / 水平翻转 / 置顶 / 置底 / 复制一份 / 删除**。

- 裁剪框 8 个把手自由拖动（不锁比例），框内拖动整体平移；点「完成」或点图片以外的空白处即确认，记一条撤销并触发自动保存。
- 裁剪结果始终相对**原图**，所以重新裁剪能把之前裁掉的部分再框回来（非破坏式）。
- 动作条只在有图时出现；同一页只有一张图时不显示置顶/置底。

> 在加裁剪 UI 之前，**先单独修了 `srcRect` 渲染**：`SizedOverflowBox` 那条老路径在紧约束下是空操作（Flutter `RenderSizedOverflowBox.performLayout` 用父约束布局），不修的话会出现"拖了框、点了确认、界面毫无变化"的静默失效。

### 自由旋转 + 水平翻转

图片上边缘中点有一个蓝色圆形箭头，拖着它转（不是 90° 一档的按钮）。

- **旋转轴是图片中心，图、蓝色边框、8 个把手作为一个刚体一起转**，图片本身的绘制尺寸不随角度变化（外层"格子"按旋转后的外接矩形长大，四角才不会被裁）。
- 角度接近 90 的整数倍（±5°）时吸附并给一次轻微触感反馈；拖动时箭头**始终待在手指底下**（按"半径＝把手框高/2、角度＝当前角度"的轨道定位）。
- 顶部中间那个缩放把手让位给了旋转箭头，顶部仍可用两个上角拉伸。
- 「重置」把裁剪、旋转、翻转一起恢复；三项分别撤销。

## 踩过的坑（可能对做 Flutter 安卓的人有用）

1. **删掉一个 Flutter 插件依赖时，必须检查 `AndroidManifest.xml` 是否手工声明了它的组件**。上游为 `super_clipboard` 声明过一个 `<provider>`；插件删了、类不在 APK 里，但清单仍声明它 → Android 在进程启动、Activity 起来之前实例化 ContentProvider，抛 `ClassNotFoundException`，**点图标即闪退**。
2. **跨在父级边界上的交互元素，超出父级的那部分既画得出来也点不到**。Flutter 的 `hitTest` 一旦发现点落在自身尺寸之外就直接返回，所以压在图片边界上的把手只有朝内的四分之一能点。修法是把外层"格子"向外扩一圈专门放把手（视觉位置不变）。
3. **旋转相关几何要留容差**：`sin(π)` 是 `1.2e-16` 而不是 `0`；2:1 的矩形转 60° 后宽反而变窄，不变量是**面积**不减而不是每边都不减。
4. **网络**：TUN/fake-IP 代理会让 Java 的 HTTPS 长连接静默挂死在 0 字节而 Gradle 不会自己超时；解决办法是给 Gradle 配国内镜像（`mavenCentral()` 排在 `google()` 之前）。
5. **PowerShell 5.1 的 `-Encoding UTF8` 会写 BOM**，Groovy 构建文件带 BOM 会报 `startup failed: 1 error`。写文件用 `UTF8Encoding($false)`。
6. `flutter pub get` 提示 "requires symlink support / Please enable Developer Mode" **不阻塞 APK 构建**，可忽略。

## 已知限制

- **iOS/macOS 已不可构建**（刻意保留原样）：删掉 `workmanager` / `flutter_web_auth_2` 后，`ios/Runner/AppDelegate.swift` 仍 `import workmanager_apple`。安卓构建不受影响。
- 主页卡片与页面管理器里的**缩略图仍不显示裁剪**（缩略图坐标系与原图不同源，强行走裁剪会算错）；主画布与导出都是准的。
- `lib/i18n/` 未改动，新增 UI 文案为中文字面量；删掉的模块留下若干无用翻译键（无害）。
- 测试套件：改动后 **299 通过 / 12 跳过 / 3 失败**，3 个失败全部是 Windows/平台差异（缺 Apple 字体导致的 golden、上游 Windows 专属的路径处理、Windows 句柄占用导致临时文件删不掉），与本次改动无关。
- **这些补丁只做过 `flutter analyze` + 测试套件 + 构建产物的静态校验，真机手感请自行实测。**

## 关于署名

这 14 个提交的作者字段当时写的是占位名 `Saber Customizer <local@localhost>`。
若要把改动归到某个 GitHub 账号名下，需要重写提交作者——那会**改变提交哈希**，`patches/` 里的文件名与哈希对照随之失效，需重新导出。

## 构建环境（作者当时使用）

Flutter 3.47.4 ／ JDK 21 (Temurin) ／ Android SDK platform 37.0 ＋ build-tools 37.0.0 ／ NDK 28.2.13676358 ／ Gradle 9.5.0。