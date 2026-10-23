# RustDesk 独立分支构建（去除组织 / 官方依赖）

本仓库在官方 `rustdesk/rustdesk` 基础上做了最小化定制，目标是做一个**不连接任何官方 /
RustDesk 组织基础设施的独立分支**：

- 不再自动连接 21114 端口（旧版 API / admin 控制台）；
- 移除「地址簿 / Address book」和「可访问的设备 / Accessible devices」两个依赖 API 服务器的标签页；
- 彻底关闭软件更新检查（GUI 检查、macOS 后台静默自动更新、手动检查），永不连官方服务器比对版本。

其余 P2P 行为（rendezvous、relay、NAT 打洞、设备发现、文件 / 剪贴板传输等）保持官方原样。

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

### 1.1 本次追加的定制：移除两个标签页 + 关闭更新检查

在 21114 改动基础上，又做了两处定制（同样基于上游 `398364f`）：

**(a) 移除「地址簿 / Address book」和「可访问的设备 / Accessible devices」标签页**

- 这两个标签页依赖 API 服务器（即已移除的 21114）提供后端，没有它就没有意义。
- 改动位置：`src/flutter_ffi.rs` 的 `is_disable_ab()` 与 `is_disable_group_panel()`，
  现在**直接返回 `true`**（与官方「许可证禁用该功能」语义一致），从而让这两个标签页不显示。
  配套的 `ab_model.dart` / `group_model.dart` 的 early-return 逻辑也自然生效，无需改动。
- **未动账号系统**：登录按钮、账户设置页等仍保留（只是没有后端时会连不上，不影响其它功能）。

**(b) 彻底关闭软件更新检查**

- 桌面端更新检查有两条链路，最终都走到 `src/common.rs` 的 `do_check_software_update()`：
  1. GUI 启动检查：`main.dart → checkUpdate() → mainGetSoftwareUpdateUrl() → check_software_update()`；
  2. macOS 后台静默自动更新（以 root 运行的 LaunchDaemon 每天执行）：
     `start_auto_update_macos → check_update_as_root → do_check_software_update()`。
- `do_check_software_update()` 现已改为 **no-op（直接 `Ok(())`，不发任何网络请求）**；
  GUI 入口 `check_software_update()` 也置为 no-op；并从桌面设置里移除了
  「Check for software update on startup」和「Auto update」两个开关（避免留下死选项）。
- 效果：客户端**永远不会**连 `api.rustdesk.com/version/latest` 比对版本，也**不会**弹出
  「有新版本」提示或静默自更新。你可以一直停留在这个自定义分支，无需跟随上游升级。

```diff
  // src/flutter_ffi.rs
  pub fn is_disable_ab() -> SyncReturn<bool> {
-     SyncReturn(config::is_disable_ab())
+     SyncReturn(true)   // 无 API 服务器（21114 已移除）→ 地址簿无后端
  }
  pub fn is_disable_group_panel() -> SyncReturn<bool> {
-     SyncReturn(LocalConfig::get_option("disable-group-panel") == "Y")
+     SyncReturn(true)   // 无 API 服务器（21114 已移除）→ 可访问设备无后端
  }

  // src/common.rs —— 更新检查核心入口改为 no-op
  pub fn check_software_update() {
-     if is_custom_client() { return; }
-     let opt = LocalConfig::get_option(keys::OPTION_ENABLE_CHECK_UPDATE);
-     if config::option2bool(keys::OPTION_ENABLE_CHECK_UPDATE, &opt) {
-         std::thread::spawn(move || allow_err!(do_check_software_update()));
-     }
+     return;   // 独立分支：永不连官方服务器检查更新
  }
  pub async fn do_check_software_update() -> ... {
-     // 整段网络请求（连 api.rustdesk.com 比对版本）...
+     Ok(())    // no-op
  }

  // flutter/lib/desktop/pages/desktop_setting_page.dart —— 删除两个死开关
-     if (!isWeb && !bind.isCustomClient())
-       _OptionCheckBox(context, 'Check for software update on startup', ...);
-     if (showAutoUpdate)
-       _OptionCheckBox(context, 'Auto update', ...);
```

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

## 6. 提交 / 推送本仓库改动

所有定制已提交到本地 `main` 分支（基于上游 `398364f`），并推送到你的 fork `master`。
若本地又改了代码，提交并打 tag 触发 CI：

```bash
git add -A
git commit -m "build: standalone fork - drop 21114, hide org tabs, disable update check"
git tag 1.5.2-x        # 任意新 tag
git push fork 1.5.2-x  # fork = 你的 fork remote
```
