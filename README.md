# HonorLocalOTA

荣耀 MagicOS「软件更新」（`com.hihonor.ouc`）的 Xposed 模块。可按需屏蔽新版本提示，并强制向服务器请求全量包。

## 功能

| 功能 | 说明 |
| --- | --- |
| 屏蔽新版本提示 | 检查更新时压成「无新版本」，不再提示升级 |
| 强制全量包 | 向服务器请求完整包，而不是增量包 |
| 隐藏尝鲜更新 | 「屏蔽更新」生效时，升级尝鲜页显示「暂无尝鲜活动」 |

两个开关的相互关系：

- 「屏蔽更新」与「强制全量包」互斥。打开「强制全量包」时「屏蔽更新」不生效，设置页会在旁边提示「已被强制全量包覆盖」。
- 设计上不在程序里偷偷改写另一个开关，避免出现「是谁把开关关掉的」这种难以追溯的状态。

## 要求

- **LSPosed**，且框架需支持 libxposed API 102（模块声明 `minApiVersion=101`、`targetApiVersion=102`）
- **root**（原因见下一节）
- 作用域：`com.hihonor.ouc`（「软件更新」）

> 关于框架版本：`LSPosed/LSPosed` 官方仓库的安装包停留在较早的版本，未必支持 API 102。若模块在 LSPosed 中不显示或无法启用，请先确认框架是否支持 libxposed 102。

## 为什么需要 root

模块的开关配置存在 `/data/local_ota/module_switch.conf`，这个目录的 SELinux 标签是 `local_ota_file`。

从系统策略里能查到，允许访问该标签的域只有四个：`adbd`、`platform_app_all`、`shell`、`update_engine`。

- 「软件更新」自身跑在 `platform_app_all` 域，本来就能读写，所以 hook 侧不需要额外权限
- 模块的设置页跑在 `untrusted_app` 域，**不在放行名单里，连读取属性都不允许**，因此必须通过 `su` 访问

没有 root 时，设置页会显示「root 不可用，开关无法保存」，而不会静默失败。

## 使用方法

1. 安装 APK
2. 在 LSPosed 中启用本模块
3. 作用域勾选「软件更新」
4. 重启手机，或强制停止「软件更新」
5. 打开模块设置页，确认显示「root 可用」以及两个开关的状态

## 兼容性

在荣耀 AAK-AN00（MagicOS 11 / Android 17）上测试通过。对应「软件更新」：
预装 **18.0.3.227**（versionCode 180003227），升级后 **18.0.3.230**
（versionCode 180003230）——两个版本上都验证过目标定位。

目标定位不依赖固定版本号：模块在运行时读取设备上实际生效的 APK，用语义
特征（方法签名、协议字符串、未混淆的接口方法名）查找 hook 目标。换版本
后若某个目标找不到，设置页的来源会显示「写死的名字（换版本会失效）」，
而不会静默失效。

不同机型、不同系统版本的「软件更新」类名与结构可能不同，模块按上述版本实现，未覆盖的情况可能不生效。

## 风险提示

- 修改系统更新行为可能导致错过安全补丁，请自行权衡
- 强制全量包会增加流量与存储占用

## AI 生成内容声明

**本模块的代码与文档绝大部分由大语言模型（LLM）生成，不是人工逐行编写的。**

这不是免责套话，它直接影响你该怎么使用这个模块：

| 项目 | 实际情况 |
| --- | --- |
| 模块代码 | 由 LLM 阅读 `com.hihonor.ouc` 的反编译结果（smali）后编写，再由人工审阅 |
| README、Release 说明、代码注释 | 由 LLM 撰写 |
| hook 目标（类名 / 方法名） | 由 LLM 从 smali/jadx 反编译结果提取，配合 DexKit 语义特征在运行时重新定位 |
| 真机验证范围 | 仅荣耀 AAK-AN00 一台设备（「软件更新」18.0.3.227 与 18.0.3.230 两个版本） |

具体意味着：

- hook 目标不全靠写死。类名方法名来自对特定版本的反编译；但模块在运行时用 DexKit 按语义特征（方法签名、协议字符串、未混淆的接口方法名）重新查找，所以换版本后通常还能找到。写死的那份只在语义查询失败时兜底。
- 兜底是可见的，不是静默的。设置页会逐个目标显示「来源」：`语义定位（跨版本可用）` 表示是运行时查出来的，`写死的名字（换版本会失效）` 表示回落到了硬编码。后者出现就说明该目标的语义特征在这个版本上失效了。
- LLM 可能给出听起来很确定、实际是错的结论。本项目开发过程中就出现过误判，被人工当场纠正后才修正。
- `isMinifyEnabled = true`，代码经过 R8 混淆。你在 APK 里搜不到源码中的类名，这是正常的，不代表功能缺失。

完整的说明、已知误判记录与二次开发建议，见源码仓库的 README。

## 许可
本项目自身采用 **AGPL-3.0**，详见 [LICENSE](LICENSE)。

### 第三方组件

模块不是「纯自己写的」——它依赖并**随 APK 分发**了别人的代码。完整声明见
[THIRD_PARTY_NOTICES.txt](THIRD_PARTY_NOTICES.txt)，这里只列结论：

| 组件 | 版本 | 许可 | 是否随 APK 分发 |
| --- | --- | --- | --- |
| [DexKit](https://github.com/LuckyPray/DexKit) | 2.3.0 | LGPL-3.0 与 Apache-2.0 双许可 | 是（dex + 4 个 ABI 的 `libdexkit.so`） |
| [FlatBuffers Java](https://github.com/google/flatbuffers) | 23.5.26 | Apache-2.0 | 是（DexKit 的传递依赖） |
| [kotlin-stdlib](https://github.com/JetBrains/kotlin) | 见依赖调解 | Apache-2.0 | 是（DexKit 的传递依赖） |
| [libcxx Prefab](https://github.com/RikkaW/libcxx-prefab) | 1.2.0 | Apache-2.0 | 仅构建期（代码已链接进 `libdexkit.so`） |
| [libxposed API](https://github.com/libxposed/api) | 102.0.0 | Apache-2.0 | 否（`compileOnly`，运行时由 LSPosed 提供） |

**DexKit 是目标定位的核心**：模块之所以能在换版本后仍然找到 hook 目标，
靠的就是它读 dex 做静态语义查询。它不是可选工具，所以它的许可义务必须
认真对待。

关于 LGPL-3.0 的静态链接：dex 里的类无法在运行时替换，所以本模块属于
静态链接。LGPL-3.0 第 4 条 d 项要求提供「能重新链接」的条件，本模块的
满足方式是**完整公开源码与构建脚本**——fork
[源码仓库](https://github.com/oopnv70-lab/HonorLocalOTA)、改
`gradle/libs.versions.toml` 里的 `dexkit` 版本，就能构建一个替换掉
DexKit 的版本。

许可全文随 APK 一起分发，在 APK 里可以查到：

```
META-INF/LICENSE                  本项目 AGPL-3.0
META-INF/THIRD_PARTY_NOTICES.txt  第三方组件声明
META-INF/licenses/LGPL-3.0.txt    DexKit
META-INF/licenses/GPL-3.0.txt     LGPL-3.0 的补充条款
META-INF/licenses/Apache-2.0.txt  FlatBuffers / kotlin-stdlib / libcxx
```

本仓库根目录也放了同样的 `THIRD_PARTY_NOTICES.txt` 与 `licenses/`，
内容与 APK 内一致，便于不下载 APK 直接查阅。

### 关于「没有分发荣耀的代码」

本模块不打包、不分发 `com.hihonor.ouc` 的任何字节。它只在运行时读取设备上
已存在的 APK 做静态分析，不复制其代码，也不把它嵌进自己的产物里。
