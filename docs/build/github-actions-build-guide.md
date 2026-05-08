# GitHub Actions 编译配置指南

本文档介绍如何通过 GitHub Actions 自动编译 RustDesk 全平台产物。

---

## 1. 工作流总览

项目编译由 `.github/workflows/flutter-build.yml` 驱动（约 2000 行），通过两个入口触发：

| 触发方式 | 工作流文件 | 说明 |
|---------|-----------|------|
| 推送 tag（如 `v1.4.6`） | `flutter-tag.yml` | 自动发布 Release |
| 每日定时 / 手动触发 | `flutter-nightly.yml` | 每天 UTC 0:00 构建 nightly 版 |
| 手动触发 | `flutter-nightly.yml` → workflow_dispatch | 在 Actions 页面手动 Run |

### 编译目标矩阵

| Job | 目标产物 | 运行环境 | 产出格式 |
|-----|---------|---------|---------|
| `build-for-windows-flutter` | Windows x86_64 | windows-2022 | exe 安装包 |
| `build-for-macOS` | macOS x86_64 | macos-15-intel | dmg |
| `build-for-macOS` | macOS aarch64 | macos-14 (M1) | dmg |
| `build-rustdesk-ios` | iOS aarch64 | macos-latest | ipa |
| `build-rustdesk-android` | Android arm64/arm/x86_64 | ubuntu-24.04 | apk |
| `build-rustdesk-android-universal` | Android 统一包 | ubuntu-24.04 | apk |
| `build-rustdesk-linux` | Linux x86_64 | ubuntu-22.04 | deb/rpm |
| `build-rustdesk-linux` | Linux aarch64 | ubuntu-22.04-arm | deb/rpm |
| `build-appimage` | Linux AppImage | ubuntu-22.04 | AppImage |
| `build-flatpak` | Linux Flatpak | ubuntu-22.04 | Flatpak |
| `build-rustdesk-web` | Web | ubuntu-22.04 | 静态文件 |

### 依赖关系

```
generate-bridge ──┬──> build-for-windows-flutter (还依赖 build-RustDeskTempTopMostWindow)
                  ├──> build-for-macOS
                  ├──> build-rustdesk-ios
                  ├──> build-rustdesk-android ──> build-rustdesk-android-universal
                  ├──> build-rustdesk-linux ──> build-appimage / build-flatpak
                  └──> build-rustdesk-web
```

---

## 2. Fork 后配置步骤

### 2.1 Fork 仓库

```bash
# 方式一：GitHub 页面上点 Fork
# 方式二：gh CLI
gh repo fork rustdesk/rustdesk --clone
```

### 2.2 配置 Secrets

进入你 fork 的仓库 → Settings → Secrets and variables → Actions，添加以下 secrets：

#### 必需（否则对应平台编译会跳过签名步骤）

| Secret | 用途 | 不配置的后果 |
|--------|------|------------|
| `ANDROID_SIGNING_KEY` | Android APK 签名密钥（Base64） | APK 未签名，无法直接安装 |

#### 可选（代码签名 & 公证）

| Secret | 用途 |
|--------|------|
| `MACOS_P12_BASE64` | macOS 签名证书 .p12（Base64 编码） |
| `MACOS_P12_PASSWORD` | .p12 证书密码 |
| `MACOS_CODESIGN_IDENTITY` | 签名身份标识（如 `Developer ID Application: ...`） |
| `MACOS_NOTARIZE_JSON` | Apple 公证 API Key JSON（Base64） |
| `SIGN_BASE_URL` | Windows 远程签名服务地址 |

> 不配置签名 secrets 也能编译成功，只是产出物未签名。macOS/Windows 未签名包会触发安全警告。

### 2.3 生成 Android 签名密钥

```bash
# 生成 keystore
keytool -genkey -v -keystore rustdesk-release.jks \
  -keyalg RSA -keysize 2048 -validity 10000 \
  -alias rustdesk -storepass YOUR_PASSWORD

# 转 Base64 写入 secret
base64 -w 0 rustdesk-release.jks
# 将输出内容设置为 ANDROID_SIGNING_KEY
```

同时需要在 `flutter/android/key.properties` 或通过环境变量配置 keystore 密码和别名。

### 2.4 启用 GitHub Actions

Fork 后 Actions 默认禁用，需要：

1. 进入仓库 → Actions 标签页
2. 点击 "I understand my workflows, go ahead and enable them"

---

## 3. 触发编译

### 方式一：推送 Tag 触发正式构建

```bash
git tag v1.4.6
git push origin v1.4.6
```

自动触发 `flutter-tag.yml`，产物上传到 GitHub Releases（tag 名为版本号）。

### 方式二：手动触发 Nightly 构建

1. 进入仓库 → Actions → "Flutter Nightly Build"
2. 点击 "Run workflow" → 选择分支 → Run

或用 CLI：

```bash
gh workflow run flutter-nightly.yml --ref main
```

### 方式三：日常开发只编译单平台

如果只需要编译某个平台，可以临时注释 `flutter-build.yml` 中不需要的 job，或创建一个简化版工作流。

---

## 4. 关键版本号配置

编译前需要确认 `flutter-build.yml` 中的版本号常量：

```yaml
env:
  RUST_VERSION: "1.75"           # Rust 编译器版本
  MAC_RUST_VERSION: "1.81"       # macOS 专用 Rust 版本（cidre 要求）
  FLUTTER_VERSION: "3.24.5"      # Flutter SDK 版本
  ANDROID_FLUTTER_VERSION: "3.24.5"
  CARGO_NDK_VERSION: "3.1.2"     # cargo-ndk 版本
  NDK_VERSION: "r28c"            # Android NDK 版本
  LLVM_VERSION: "15.0.6"         # Windows LLVM 版本
  VERSION: "1.4.6"               # RustDesk 应用版本号
  VCPKG_COMMIT_ID: "120deac..."  # vcpkg 依赖版本锁定
```

修改应用版本号时，同时更新 `VERSION` 和 `Cargo.toml` 中的 `version` 字段。

---

## 5. 各平台编译细节

### 5.1 Windows (x86_64)

- **运行环境**：windows-2022（GitHub 托管）
- **工具链**：MSVC + LLVM 15 + Flutter 3.24.5
- **依赖管理**：vcpkg（triplet: `x64-windows-static`）
- **特殊依赖**：`RustDeskTempTopMostWindow`（C# 组件，单独编译）
- **使用自定义 Flutter 引擎**：从 `rustdesk/engine` 下载替换
- **产物**：exe 安装包 + MSI（通过 WiX/MSBuild）

### 5.2 macOS (x86_64 + aarch64)

- **运行环境**：x86_64 用 `macos-15-intel`，aarch64 用 `macos-14`（M1 runner）
- **工具链**：Rust 1.81 + Flutter 3.24.5 + LLVM (brew)
- **特殊处理**：
  - NASM 必须用 2.16.x（3.x 不兼容）
  - aarch64 设置 `MACOSX_DEPLOYMENT_TARGET=12.3`
  - aarch64 启用 `--screencapturekit` 参数
- **签名流程**：codesign → create-dmg → rcodesign notarize
- **产物**：dmg

### 5.3 iOS (aarch64)

- **运行环境**：macos-latest
- **工具链**：Rust 1.75 + Flutter 3.24.5
- **流程**：先编译 `liblibrustdesk.a`（静态库），再 `flutter build ipa --no-codesign`
- **产物**：未签名 ipa（需要自行签名分发）

### 5.4 Android (arm64 / arm / x86_64)

- **运行环境**：ubuntu-24.04
- **工具链**：Rust 1.75 + Flutter 3.24.5 + NDK r28c + Java 17
- **流程**：
  1. `cargo-ndk` 编译 Rust 库（`liblibrustdesk.so`）
  2. 复制 .so 到 `flutter/android/app/src/main/jniLibs/`
  3. `flutter build apk`
- **统一包**：`build-rustdesk-android-universal` 收集三个架构的 .so，生成一个通用 APK
- **产物**：apk（单架构 + 通用包）

### 5.5 Linux (x86_64 + aarch64)

- **运行环境**：ubuntu-22.04（x86_64），ubuntu-22.04-arm（aarch64 原生 ARM runner）
- **构建方式**：在 Docker 容器（debian:bullseye / ubuntu18.04）中编译，确保 glibc 兼容性
- **工具链**：通过 `rustdesk-org/run-on-arch-action` 在容器中安装 Rust + 依赖
- **aarch64 特殊处理**：
  - 使用 `flutter-elinux` 代替标准 Flutter
  - 限制并行编译 `--jobs 3` 避免内存溢出
- **产物**：deb + rpm + AppImage + Flatpak

---

## 6. 编译产物获取

编译完成后，产物在两个地方：

### GitHub Releases

tag 触发的构建自动上传到 Releases 页面：

```bash
# 查看最新 release
gh release list --limit 5

# 下载全部产物
gh release download v1.4.6
```

### GitHub Actions Artifacts

所有构建（包括 nightly）都会上传 Artifacts：

1. 进入 Actions → 选择对应的 workflow run
2. 底部 Artifacts 区域下载

或用 CLI：

```bash
# 列出最近的 workflow run
gh run list --workflow=flutter-nightly.yml --limit 5

# 下载某次 run 的全部产物
gh run download <run-id>
```

---

## 7. 自定义修改建议

### 7.1 修改应用名称

需要同时改以下位置：

- `Cargo.toml` → `name`
- `flutter/pubspec.yaml` → `name`
- `flutter/android/app/build.gradle` → `applicationId`
- `flutter/macos/Runner.xcodeproj/project.pbxproj` → `PRODUCT_BUNDLE_IDENTIFIER`
- `flutter/windows/runner/Runner.rc` → `ProductName`

### 7.2 只编译部分平台

在 `flutter-build.yml` 中注释不需要的 job，或在 matrix 中删除不需要的 target。例如只保留 Windows + Android：

```yaml
jobs:
  # 保留
  build-for-windows-flutter: ...
  build-rustdesk-android: ...

  # 注释掉
  # build-for-macOS: ...
  # build-rustdesk-linux: ...
  # build-rustdesk-ios: ...
```

### 7.3 加速编译

- **vcpkg 缓存**：已配置 `VCPKG_BINARY_SOURCES: "clear;x-gha,readwrite"`，利用 GitHub Actions Cache
- **Rust 缓存**：已配置 `Swatinem/rust-cache@v2`
- **减少目标**：只编译需要的平台和架构

---

## 8. 常见问题

### Q: 编译失败提示磁盘空间不足

A: GitHub 托管 runner 约 14GB 可用空间。工作流已包含清理步骤（删除 dotnet/haskell/docker），如果仍不够，可在 step 开头增加：

```yaml
- name: Free disk space
  run: |
    sudo rm -rf /opt/ghc /usr/local/lib/android /usr/share/dotnet
```

### Q: vcpkg 依赖编译失败

A: 确保 `VCPKG_COMMIT_ID` 与 `vcpkg.json` 中的 baseline 一致。修改 commit ID 后需运行：

```bash
$VCPKG_ROOT/vcpkg x-update-baseline
```

### Q: Flutter bridge 文件缺失

A: `generate-bridge` job 使用 `flutter_rust_bridge` 生成 Dart-Rust FFI 桥接代码。确保 `bridge.yml` 正常运行，其产物被后续 job 下载。

### Q: macOS 未签名 dmg 无法打开

A: 用户需执行：

```bash
xattr -cr /Applications/RustDesk.app
```

或在系统偏好设置 → 安全性与隐私中允许。正式分发建议配置签名证书。

### Q: Android 构建 Gradle 内存不足

A: 工作流已配置增加 JVM 内存。如仍不足，在 `flutter/android/gradle.properties` 添加：

```properties
org.gradle.jvmargs=-Xmx4096m
```

---

## 9. 版本号与 Secrets 速查

```
flutter-build.yml env:
  VERSION          = 1.4.6          # 改版本号时同步修改
  RUST_VERSION     = 1.75
  FLUTTER_VERSION  = 3.24.5
  NDK_VERSION      = r28c
  VCPKG_COMMIT_ID  = 120deac...

Secrets (仓库 Settings → Secrets):
  ANDROID_SIGNING_KEY              # Android 签名（必需）
  MACOS_P12_BASE64                 # macOS 签名（可选）
  MACOS_P12_PASSWORD               # macOS 证书密码（可选）
  MACOS_CODESIGN_IDENTITY          # macOS 签名身份（可选）
  MACOS_NOTARIZE_JSON              # macOS 公证（可选）
  SIGN_BASE_URL                    # Windows 签名（可选）
```
