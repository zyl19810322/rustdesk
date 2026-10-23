# RustDesk 自定义构建（去除 21114 依赖）

本仓库在官方 `rustdesk/rustdesk` 基础上做了一处最小改动：**让客户端不再自动连接 21114 端口**，
其余行为（rendezvous、relay、NAT 打洞、设备发现等）完全不变。

## 1. 改动说明

- **改了什么**：`src/common.rs` 中的 `get_api_server_()` 原本会在你配置了自定义
  rendezvous 服务器、且未显式指定 API 服务器时，自动推导出
  `http://<host>:21114` 这个**旧版 API 服务器 / admin 控制台**地址（21114 即旧 admin 端口）。
  现在该分支直接返回空字符串，即「不连接 21114」。
- **没改什么**：rendezvous（21115/21116）、relay（21117）、WebSocket（21118/21119）、
  NAT 打洞、剪贴板/文件传输等全部保持官方原样。
- **如果你自有 API 服务器**：仍可像官方一样**显式配置** API 地址（Windows 许可文件名里的
  `api` 字段，或通过服务器配置里的 `api` 参数），该路径不受影响。
- 同步更新了对应的单元测试 `test_api_server_from_bare_ipv6_rendezvous_server`。

改动 diff（基于上游 `398364f`）：

```diff
     let s0 = get_custom_rendezvous_server(custom);
     if !s0.is_empty() {
-        let s = crate::increase_port(&s0, -2);
-        if s == s0 {
-            let s = if hbb_common::is_ipv6_str(&s) && !s.starts_with('[') {
-                format!("[{s}]")
-            } else {
-                s
-            };
-            return format!("http://{}:{}", s, config::RENDEZVOUS_PORT - 2);
-        } else {
-            return format!("http://{}", s);
-        }
+        // Do not auto-derive the legacy API server on port 21114.
+        return "".to_owned();
     }
     "https://admin.rustdesk.com".to_owned()
```

## 2. 前置依赖（与官方 `flutter-build.yml` 一致）

| 依赖 | 版本 / 说明 |
| --- | --- |
| Rust | macOS 用 **1.81**；Windows/Linux 用 **1.75**（sciter i128 ABI 限制） |
| Flutter | **3.24.5** |
| vcpkg | commit `9e593bb18ea69cc5095e012465dcd675a822ed0d`（用于 ffmpeg / obs 等） |
| macOS 额外 | Xcode、 `brew install llvm create-dmg nasm yasm`、vcpkg triplet `x64-osx` / `arm64-osx` |
| Windows 额外 | Visual Studio 2022（含“C++ 桌面开发”）、vcpkg triplet `x64-windows-static` / `arm64-windows-static` |

> ⚠️ **Windows 客户端无法在 macOS 上交叉编译**，必须在 Windows 机器上构建，或通过下面的 GitHub Actions 方案。

## 3. 本地构建 macOS（在本机）

```bash
# 1. 安装依赖：Rust 1.81、Flutter 3.24.5、Xcode、vcpkg、brew 包
brew install llvm create-dmg nasm yasm

# 2. 指向 vcpkg（mac 用对应 triplet）
export VCPKG_ROOT=/opt/vcpkg        # 你的 vcpkg 路径
export MACOSX_DEPLOYMENT_TARGET=10.14

# 3. 构建（会自动 cargo build + flutter build macos）
python3 build.py --flutter

# 4. 产物
#    RustDesk.app : build/macos/Build/Products/Release/RustDesk.app
#    如需 dmg     : create-dmg rustdesk.dmg ./build/macos/Build/Products/Release/RustDesk.app
```

## 4. 本地构建 Windows（必须在 Windows 上）

```bat
:: 1. 安装：Rust 1.75、Flutter 3.24.5、Visual Studio 2022、vcpkg
:: 2. 指向 vcpkg（Windows 用对应 triplet）
set VCPKG_ROOT=C:\vcpkg

:: 3. 构建（会自动 cargo build + flutter build windows + 打包 install/portable）
python3 build.py --flutter

:: 4. 产物
::    rustdesk-<version>-install.exe   （安装版）
::    rustdesk_portable.exe            （便携版）
```

## 5. 推荐：用 GitHub Actions 一次出 Mac + Windows（无需 Windows 机器）

1. 把本仓库推到你**自己的 fork**（不要推到上游 `rustdesk/rustdesk`）。
2. 打一个版本 tag 触发官方 tag 构建流程：
   ```bash
   git tag 1.5.1-no21114
   git push origin 1.5.1-no21114
   ```
3. 仓库自带的 `.github/workflows/flutter-tag.yml` 会被触发，复用官方
   `.github/workflows/flutter-build.yml`，在 `macos-latest` 与 `windows-2022`
   上自动构建并上传 **macOS（dmg）** 与 **Windows（exe / msi）** 等产物到
   Actions Artifacts / Release。该流程已包含本仓库的 21114 改动。

> 提示：官方 tag 流程会同时构建 iOS / Android / Linux。如果只想跑 mac + windows，
> 可以按需精简 `flutter-build.yml` 的 matrix（非必须，默认全平台也能用）。

## 6. 提交本仓库改动（可选）

当前已切到 `main` 分支（基于上游 `398364f`），21114 改动尚未提交。若要走第 5 步的 CI：

```bash
git add src/common.rs
git commit -m "build: drop auto 21114 (legacy API server) derivation"
```
