# Core Code Locator — jadx-gui 核心代码快速定位插件

一个为 [jadx-gui](https://github.com/skylot/jadx) 开发的插件，导入 APK/DEX 后一键扫描，
从「应用入口/组件、接口请求/网络、加密/摘要算法、签名/鉴权校验、业务核心逻辑、native/so 调用、
反调试/root 检测、弱校验/风险逻辑」**八个维度**定位核心代码，双击结果节点直接跳转到对应反编译代码。

此外还提供 **抓包关键字并入扫描**、**第三方组件（SDK）引用识别**、调用点反查与域名聚合等能力。

![类别](docs/categories.svg)

## 功能

jadx-gui 自带的搜索只能按类名/方法名匹配关键字。逆向分析时常常需要先回答这些问题：

- 应用从哪里启动？`Application`、`attachBaseContext`、四大组件入口在哪？
- 登录/注册、数据存储、设备标识这些业务核心逻辑在哪些方法里？
- 接口请求走的是 OkHttp/Retrofit 还是 HttpURLConnection？`baseUrl`、请求头、参数组包在哪？
- 加密、摘要、签名、验签逻辑分别在哪些方法里？`sign / verify / token` 校验如何组织的？
- 是否调用了 so 库（`System.loadLibrary` / native 方法）、是否做了反调试与 root 检测？
- 有没有证书校验绕过、恒真校验、弱加密模式这类风险点？

`Core Code Locator` 把这些问题做成可点击的清单：按 8 类聚合命中结果，
每条命中都带有可读的"命中理由"，双击即可跳到对应方法。

| 类别 | 命中内容（节选） |
| --- | --- |
| 📂 应用入口/组件 | 继承 `Application / Activity / Service / BroadcastReceiver / ContentProvider` 的类；`onCreate / attachBaseContext / onStartCommand / onReceive` 等入口生命周期方法 |
| 🌐 接口请求/网络 | `OkHttpClient`、`Retrofit.Builder`、`Interceptor`、`HttpURLConnection`、`setRequestMethod("GET"/"POST")`、`addHeader`、`RequestBody`、`baseUrl`、`WebView.loadUrl`、SSL/证书相关（`SSLContext`、`TrustManager`） |
| 🔐 加密/摘要算法 | `Cipher`、`MessageDigest`、`SecretKeySpec`、`IvParameterSpec`、`Mac/HmacSHA256`、`KeyStore`、`Base64`；算法名 AES/RSA/DES/MD5/SHA1/SHA256/SM2/SM3/SM4；方法名含 encrypt/decrypt/digest 等 |
| ✍️ 签名/鉴权校验 | `Signature.getInstance/sign/verify`、`getPackageInfo` + 签名校验、`checkSign / getSign / genSign / makeSign`、`token / authorization / authCode / timestamp / nonce`、`verify` |
| 🧩 业务核心逻辑 | `login / logout / register / submit`；`SharedPreferences / Editor`、`SQLiteOpenHelper / SQLiteDatabase` 存储；`deviceId / imei / ANDROID_ID / getDeviceId` 设备标识 |
| 📦 native/so 调用 | `System.loadLibrary / System.load / Runtime.loadLibrary`、`ReLinker / SoLoader`、`.so` 文件引用、**native 方法声明**、`DexClassLoader / PathClassLoader / InMemoryDexClassLoader` |
| 🛡️ 反调试/root 检测 | `isRoot / checkRoot / RootBeer / Magisk / /system/bin/su`、`isDebuggerConnected / debuggable`、`ptrace / TracerPid / /proc/self/status`、`frida / Xposed / findAndHookMethod`、模拟器特征 |
| ⚠️ 弱校验/风险逻辑 | `TrustAllCerts / ALLOW_ALL_HOSTNAME_VERIFIER / checkServerTrusted / X509TrustManager` 证书绕过；恒真校验；`AES/ECB`、`DES/ECB`、`SSLv3 / TLSv1.1` 等弱配置 |

## 关键词 → 分类覆盖

插件对关键词采用**分层匹配**以控制误报：长复合词（`loadLibrary`、`MessageDigest` 等 API 名）宽松子串匹配；
短泛词（`sign / token / get / post / save` 等）按**标识符边界**或**具体 API 形态**（如
`setRequestMethod("POST")`、`addHeader`、`RequestBody`）匹配；方法名则按 **camelCase 子词**判断，
避免误伤 `design`（含 sign）、`signature`（sign 过长尾）、`capitalize`（含 api）等无关代码。

| 你关心的代码 | 插件如何找到它 |
| --- | --- |
| 入口 `Application / MainActivity / attachBaseContext` | 类继承关系 + 类名兜底 + 入口方法名 |
| `http / https / okhttp / retrofit / api / url` | 网络类实例化 + 注解路由 + `"https://"` 字面量 |
| `get / post / request / response / header / body / params` | **只按 API 形态**（`setRequestMethod`、`addHeader`、`RequestBody`…）匹配 |
| `sign / signature / verify / token / auth / checkSign` | 签名/鉴权分类中的方法名 + 调用形态 |
| `AES / RSA / MD5 / SHA / Base64 / Cipher / MessageDigest` | 加解密与摘要分类 |
| `login / register / submit / SharedPreferences / sqlite / save` | 业务分类（排除框架生命周期方法的误伤） |
| `native / System.loadLibrary / loadLibrary / .so` | 加载调用 + 类级 native 方法声明检测（不再用裸 `so` 短词） |
| `isRoot / frida / ptrace / debuggable / hook` | 反调试分类 |
| `TrustAllCerts / ALLOW_ALL_HOSTNAME_VERIFIER / checkServerTrusted` | 弱校验分类（第 8 类） |
| `ClipboardManager / addJavascriptInterface / getExternalStorageDirectory / ZipInputStream / ACTION_VIEW` | 弱校验分类补：敏感数据/隐私合规风险点 |

## 特性总览

| 能力 | 菜单入口 | 说明 |
| --- | --- | --- |
| 🔍 核心代码定位 | `Plugins → 🔍 核心代码定位` | 按 8 类规则扫描核心代码，自动附带抓包关键字；双击跳转 |
| 📚 第三方组件引用 | `Plugins → 📚 第三方组件引用（SDK 暴露面）` | 识别 App 引用的 OkHttp / Retrofit / Gson 等第三方 SDK |
| 🔎 查找调用点 | `Plugins → 🔎 查找调用点（谁调用了 X）…` | 反查调用位置，沿调用链向上追业务入口 |
| 🌐 聚合域名 / URL | `Plugins → 🌐 聚合域名 / URL` | 提取全部 http(s) 地址并按域名聚合计数 |

配套能力：多类别核心嫌疑聚合（≥2 类）· ⭐ 入口可达标记 · 一键噪声排除 · 右键忽略 ·
HTML / CSV 报告导出 · 配置热重载（改配置免重启）· 加固壳指纹提示

## 快速开始

```bash
# 1) 构建（需要 JDK 11+）
git clone https://github.com/lettures/core-code-locator.git
cd core-code-locator
./gradlew dist          # 产物：build/dist/core-code-locator-0.6.0.jar

# 2) 用 jadx 官方命令安装（⚠️ 不要手动 cp 到 ~/.jadx/plugins/，见「安装与调试」）
jadx plugins --install-jar build/dist/core-code-locator-0.6.0.jar
```

在 jadx-gui 中打开 APK / DEX / AAR / JAR，等待反编译完成后通过 **Plugins** 菜单使用。


## 安装

### 方式一：从 release jar 安装（推荐给普通使用者）

1. 在 [Releases](../../releases) 下载最新 `core-code-locator-x.y.z.jar`
2. 打开 jadx-gui → 顶部菜单 **Plugins → Install plugin** → 选择 jar
3. 重启 jadx-gui（或不重启，jadx 会即时加载）

### 方式二：本地源码编译

需要 JDK 11+。

```bash
cd core-code-locator
VERSION=0.6.0 ./gradlew dist     # 产物在 build/dist/core-code-locator-0.6.0.jar
```

把 `build/dist/` 下的 jar 拖到 jadx-gui 的 **Plugins → Install plugin** 即可。

### 方式三：jadx-cli 安装

```bash
jadx plugins --install-jar build/dist/core-code-locator-0.6.0.jar
```

CLI 模式下插件不注册菜单（jadx 的 GUI 上下文不存在），但插件 jar 仍可被加载。

## 使用

1. jadx-gui 打开 APK / DEX / AAR / JAR，等待反编译完成
2. 顶部菜单 **Plugins**，可执行：
   - **🔍 核心代码定位** —— 点击即扫（v0.5.16 起不再弹输入框）：内置 8 类 + 附带关键字
     一并定位。附带关键字取 `config.properties` 的 `keywordHits`（缺省 = `sign|token|
     Authorization|/api/`，可用 `|` 增删、空串=关闭）；命中以「🧷 自定义关键字命中」类别展示、
     双击行级定位到字符串所在行；随后弹出结果窗口（顶部可能出现「一键噪声排除」建议条）
   - **🔎 查找调用点（谁调用了 X）…** —— 输入 `类.方法` / 裸方法名，反查调用位置
   - **🌐 聚合域名 / URL** —— 列出全部 http(s) 域名与出现次数
   - **📚 第三方组件引用（SDK 暴露面）** —— 扫描当前 dex **全部类**（不跳过三方库），
     按内置签名库识别引用的 OkHttp / Retrofit / Jackson / Gson / Glide 等第三方组件，
     独立窗口按 SDK 聚合展示（双击跳转、AOSP 系统库单独标注为 `📱 Android 系统库`）
3. 结果窗口内：双击跳转、右键忽略、一键噪声排除、导出 HTML/CSV
4. 修改 `~/.core-code-locator/config.properties` 后**无需重启**，下次扫描/反查自动生效

大型 APK 扫描可能耗时数十秒，可通过进度对话框查看当前类；取消即中止扫描。

### 配置文件 `~/.core-code-locator/config.properties`

首次运行自动生成（内容可随时改）：

```properties
# 扫描时整类跳过的三方库/框架包前缀（用 | 分隔）。内置默认恒生效，此处只做【追加】排除。
excludePackagePrefixes=androidx.|okhttp3.|retrofit2.|kotlin.|java.|javax.|...
# 忽略的类 或 "类 :: 方法"（用 | 分隔），也可在结果窗口右键「忽略」自动写入
ignoreTargets=com.example.lib.UpgradeChecker|com.example.app.net.LegacyApi :: send
# 「核心代码定位」自动附带的关键字（v0.5.16，用 | 分隔，忽略大小写 OR 匹配）
# 缺省=sign|token|Authorization|/api/；删掉本行可恢复内置默认；置空=关闭附带关键字（回纯 8 类）
keywordHits=sign|token|Authorization|/api/
```

- 想让某个三方库参与扫描：把它的前缀从 `excludePackagePrefixes` 里删掉即可；
- 想全部扫描（不排除任何库）：把该项设为空字符串；
- 改完保存即可，**无需重启 jadx-gui**（v0.4.0 起每次扫描/反查前自动热重载）；
- 也可在结果窗口顶部点「一键追加排除」，把「类多但 0 命中」的噪声前缀批量写入。

## 截图

插件运行后会弹出独立窗口，顶部为命中统计 + 提示，中部两个 Tab：
「多类别核心嫌疑」与「分类结果」（类别 → 类 → 方法），每条命中都给出可读的"理由"：

```
🎯 多类别核心嫌疑 (3)
   📦 com.example.app.security.ApiSigner
      🚩 ⭐ signRequest   [3 类: 接口请求/网络 / 加密/摘要算法 / 签名/鉴权校验]
📂 分类结果
   📂 应用入口/组件 (4)
      📦 com.example.app.App (2)
         🎯 ⭐ attachBaseContext — Application 入口方法 attachBaseContext（常做初始化/壳逻辑）
         🎯 <类>                 — 继承自 Application
   🌐 接口请求/网络 (6)
      📦 com.example.app.net.ApiClient (3)
         🎯 ⭐ buildRetrofit — 方法体含 Retrofit.Builder（Retrofit 构建）
   ⚠️ 弱校验/风险逻辑 (1)
      📦 com.example.app.net.Tls (1)
         🎯 checkServerTrusted — 空实现证书校验（TrustManager 绕过风险）
```

> ⭐ = 位于入口可达调用链上；双击任意方法跳转；右键可忽略方法/整类。

## 项目结构

```
core-code-locator/
├── README.md                              # 本文档
├── LICENSE                                # MIT
├── build.gradle.kts                       # Gradle 构建脚本（shadowJar 产出 fat-jar）
├── settings.gradle.kts
├── gradle.properties
├── gradle/wrapper/                        # Gradle Wrapper（首次构建时自动下载 gradle-8.13）
├── gradlew, gradlew.bat
├── docs/categories.svg                    # README 用 8 类架构图
├── src/main/
│   ├── java/com/myplugin/ccl/
│   │   ├── CoreCodeLocatorPlugin.java     # 插件入口：注册 4 个菜单动作（v0.6.0 起含 📚 第三方组件引用；核心定位点击即扫，自动并入 keywordHits）
│   │   ├── Category.java                  # 类别枚举（8 类 + 🧷 KEYWORD_HIT 关键字命中 + 📚 SDK_REFERENCE 组件引用，含图标）
│   │   ├── MatchHit.java                  # 单条命中（fallbackRef 类级降级 + needle 行级关键字）
│   │   ├── ScanResult.java                # 扫描结果聚合 + 多类别嫌疑 + 可达标记
│   │   ├── ScanConfig.java                # 外置配置（~/.core-code-locator/config.properties，热重载；含 keywordHits 附带关键字）
│   │   ├── CoreCodeScanner.java           # 扫描调度器（排除库/建调用图/可达性/加固提示/噪声统计；关键字规则并入主循环）
│   │   ├── SdkScanner.java                # v0.6.0 第三方组件扫描器（不跳过三方库，独立于 CoreCodeScanner）
│   │   ├── analysis/
│   │   │   ├── CallGraphIndex.java        # 方法级调用图（文本启发式，BFS 可达性）
│   │   │   ├── FoundLine.java             # 检索结果行（label/tooltip/跳转引用/needle 行级搜索关键字）
│   │   │   ├── ExtraSearcher.java         # 反查调用点 + 域名/URL 聚合（字符串搜索已迁 rule/KeywordRule）
│   │   │   └── NativeLibSearcher.java     # .so 证据扫描器（v0.5.6 起源码保留、不再被主流程引用）
│   │   ├── rule/
│   │   │   ├── Rule.java                  # 规则接口（底层 SPI）
│   │   │   ├── BodyScanRule.java          # 抽象基类：方法体/方法名关键词扫描
│   │   │   ├── ApiPatterns.java           # 匹配工具：API 形态/边界/camelCase 子词
│   │   │   ├── ComponentEntryRule.java    # 规则1: 应用入口/组件
│   │   │   ├── NetworkRule.java           # 规则2: 接口请求/网络
│   │   │   ├── CryptoRule.java            # 规则3: 加密/摘要算法
│   │   │   ├── SignatureRule.java         # 规则4: 签名/鉴权校验
│   │   │   ├── BusinessRule.java          # 规则5: 业务核心逻辑
│   │   │   ├── NativeRule.java            # 规则6: native/so 调用
│   │   │   ├── AntiDebugRule.java         # 规则7: 反调试/root 检测
│   │   │   ├── WeakSecurityRule.java      # 规则8: 弱校验/风险逻辑
│   │   │   ├── KeywordRule.java           # 抓包/接口关键字规则（KEYWORD_HIT，关键字来自 config keywordHits）
│   │   │   ├── SdkSignatureRule.java      # v0.6.0 SDK 引用规则（SDK_REFERENCE）
│   │   │   ├── SdkSignature.java          # v0.6.0 SDK 签名（包前缀 + 类证据）
│   │   │   └── SdkSignatureLibrary.java   # v0.6.0 内置 SDK 签名库（10+ 常用第三方组件）
│   │   └── ui/
│   │       ├── ResultWindow.java          # 核心定位结果窗口（🎯 嫌疑 Tab / 📂 分类 Tab / 右键忽略 / 导出）
│   │       ├── FoundListWindow.java       # 检索结果列表（调用点/域名；顶部统计 + 底部片段预览 + 行级定位）
│   │       ├── LineLevelNavigator.java    # 行级定位（v0.5.13 起改用 JDK JTextComponent API，零反射）
│   │       ├── NativeTabPanel.java        # 原生库证据列表面板（v0.5.6 起源码保留、不再被主流程引用）
│   │       ├── SdkResultWindow.java       # v0.6.0 SDK 引用结果窗口（按 SDK 聚合、双击跳转、片段预览）
│   │       ├── ReportExporter.java        # HTML/CSV 报告导出
│   │       └── ScanProgressDialog.java    # 进度对话框
│   └── resources/META-INF/services/
│       └── jadx.api.plugins.JadxPlugin     # SPI 声明
└── src/test/java/com/myplugin/ccl/rule/  # JUnit 5 单元测试
    ├── ApiPatternsTest.java
    └── SdkSignatureLibraryTest.java
```

## 添加自定义规则

多数自定义规则只需继承 `BodyScanRule`，声明两组关键词即可：

```java
public class MyRule extends BodyScanRule {
    @Override protected String[][] bodyKeywords() {
        return new String[][] {
            { "CustomApi", "调用 CustomApi（业务特征）" },   // 方法体命中
        };
    }
    @Override protected String[][] nameKeywords() {
        return new String[][] {
            { "custom", "方法名含 custom（自定义逻辑）" },    // 方法名命中（camelCase 子词匹配）
        };
    }
    @Override public Category getCategory() { return Category.BUSINESS; }
}
```

在 `CoreCodeScanner` 构造器里注册即可：

```java
this.rules = List.of(/* 默认规则… */, new MyRule());
```

如需更底层的控制（类级匹配、自定义命中逻辑），可直接实现 `Rule` 接口，覆写
`inspect(JadxDecompiler, JavaClass, ScanResult)`，或覆写 `BodyScanRule#inspectClass`
做类级检测（参见 `NativeRule` 对 native 方法声明的处理）。

`MatchHit` 需要传入 `jadx.api.metadata.ICodeNodeRef` 才能在结果窗口里跳转。`JavaMethod#getCodeNodeRef()` 和 `JavaClass#getCodeNodeRef()` 都可直接拿到。

## 安装与调试（重要：不要手动 cp 到 `~/.jadx/plugins/`）

jadx 1.5.x 有**两套插件目录**，装错位置会导致同 pluginId 冲突 → **jadx-gui 启动时弹出
模态「设置」对话框，看起来像卡死**（实测踩坑）。

| 目录 | 性质 | jadx 是否加载 |
| --- | --- | --- |
| `~/.jadx/plugins/` | dropins（手动丢 jar 的旧式目录） | 会扫描 |
| `~/Library/Application Support/io.github.skylot.jadx/plugins/installed/` | **官方插件管理器目录**（配 `plugins.json` 记录） | ✅ 实际加载来源 |

### 正确安装流程

```bash
# 1) 构建
cd core-code-locator
VERSION=0.6.0 ./gradlew shadowJar dist
# 产物：build/dist/core-code-locator-0.6.0.jar

# 2) 用 jadx 官方命令安装（自动处理 installed/ + plugins.json）
/path/to/jadx-1.5.6/bin/jadx plugins --install-jar \
  build/dist/core-code-locator-0.6.0.jar
# → Plugin installed: core-code-locator:null
```

**不要**用 `cp build/dist/*.jar ~/.jadx/plugins/` —— 那是 dropins 目录，
会与插件管理器已记录的插件形成**同 ID 双份**，触发 GUI 弹窗卡死。

### 常用插件管理命令

```bash
jadx plugins --list                 # 已安装插件
jadx plugins --list-all             # 含 dropins 与 jadx 自带插件
jadx plugins --install-jar <path>   # 从 jar 安装
jadx plugins --uninstall <pluginId> # 卸载
jadx plugins --disable <pluginId>   # 禁用
jadx plugins --enable  <pluginId>   # 启用
```

### 排查 jadx-gui「启动卡住、无输出」

```bash
# 关键：用 debug 日志级别启动（TRACE 不是合法值）
jadx-gui --log-level debug
# 合法值：quiet / progress / error / warn / info / debug
```

日志里的**线程堆栈**直接指向卡住的代码行。本插件项目实测过的卡死根因（插件冲突）：
```
at jadx.gui.ui.MainWindow.openSettings(MainWindow.java:1542)
at jadx.gui.ui.MainWindow.lambda$resetPluginsMenu$43(MainWindow.java:1804)
at java.awt.Dialog.setVisible(...)
```
→ `resetPluginsMenu` 检测到插件列表变化 → 弹模态设置框。此时把 dropins 里的重复 jar
移走，改用 `--install-jar` 重装即可。

### jadx 配置目录速查

| 路径 | 内容 |
| --- | --- |
| `~/Library/Application Support/io.github.skylot.jadx/plugins/installed/` | 已安装插件 jar |
| `~/Library/Application Support/io.github.skylot.jadx/plugins/plugins.json` | 插件记录（pluginId / locationId / disabled） |
| `~/Library/Application Support/io.github.skylot.jadx/gui.json` | GUI 设置（窗口、主题等） |
| `~/Library/Application Support/io.github.skylot.jadx/caches.json` | 缓存记录 |
| `~/.jadx/plugins/` | dropins（旧式，慎用） |
| `~/.core-code-locator/config.properties` | **本插件**的扫描配置（热重载） |

### 纯 CLI 验证插件能否加载（不启 GUI）

```bash
java -cp <plugin.jar>:<jadx-1.5.6-all.jar> YourClassLoadTest
# 用 Class.forName(name, true, cl) 逐个加载插件类，可在无 GUI 环境快速排除插件自身问题
```

## 兼容性

| jadx 版本 | 状态 | 备注 |
| --- | --- | --- |
| 1.5.6 | ✓ | 主要开发 / 实测环境（行级定位按其实装的 RSyntaxTextArea 3.x 适配） |
| 1.5.3 (r2504) | ✓ | 插件元数据声明的最低版本（`requiredJadxVersion`） |
| 1.5.2 | ? | 未验证 |
| < 1.5.0 | ✗ | JadxGuiContext 接口差异较大 |

## 更新日志

## v0.6.0：第三方组件引用扫描（SDK 暴露面识别）

把 `apk-static-scan` 的 SDK 签名识别能力集成进 jadx 插件，作为独立菜单入口，
与现有 8 类核心代码扫描共存：

- **新菜单 `📚 第三方组件引用（SDK 暴露面）`**：点击后扫描当前 dex 中所有类（**不跳过三方库**），
  按内置 SDK 签名库（包前缀 + 类证据双维度）识别引用的第三方组件，结果用独立窗口展示，
  每条带可双击跳转的 codeRef。
- **新 Category `SDK_REFERENCE`（📚）**：每个被识别的 SDK 一条命中，
  `target = 类全名 :: <class>`（整类级），`reason` 注明 SDK 名 + 证据类数。
- **新签名库 `SdkSignatureLibrary`**：内置 10+ 常用 SDK（OkHttp / Retrofit / Jackson / Gson /
  Glide / Fastjson / BouncyCastle / Fresco / Picasso / Volley / Lombok），每个含
  `packagePrefixes` + `classEvidence`（≥2 个证据类才确认，防 R8 裁剪残留类误报）。
- **新独立 Scanner `SdkScanner`**：不走 `CoreCodeScanner` 的 `isExcludedClass` 过滤，
  专门为 SDK 扫描设计；保留进度回调 / 取消能力。
- **AOSP Framework 标识**：对 `Lcom/android/okhttp/*`、`Lorg/apache/{xalan,xerces}/*` 等
  AOSP 系统库做 `📱 Android 系统库` 标注，不算入「App 引入的第三方组件」（与
  `apk-static-scan` 的 Framework 判定口径一致）。
- **新 UI `ui/SdkResultWindow`**：套用 `FoundListWindow` 模式（双击跳转、片段预览），
  按 SDK 名聚合，每条 SDK 展示证据类数 + 命中类数。

**与现有 8 类扫描的关系**：

| 维度 | 8 类扫描 | SDK 引用扫描 |
|---|---|---|
| 扫描目标 | 仅 App 类（跳过 SDK） | 全部类（含 SDK） |
| 匹配策略 | 关键字 / API 形态 | 包前缀 + 证据类阈值 |
| 用途 | 找业务核心代码 | 找引用的第三方组件 |
| 入口 | 🔍 核心代码定位 | 📚 第三方组件引用 |

**为什么独立菜单而不是附加 checkbox**：
SDK 扫描会遍历全部类（包括被现有降噪排除的 okhttp3./retrofit2. 等），与现有 8 类扫描的
降噪策略**根本相反**；混入会破坏现有 8 类的稳定性。独立菜单 = 独立通道 = 互不干扰。

**后续版本路线**（分阶段集成策略）：
- v0.7.0：加固检测升级为独立 Category + 顶部加固横幅
- v0.8.0：版本指纹提取（PackageVersion 常量 / VERSION 字符串 / 结构指纹）
- v0.9.0：离线 CVE 比对（`X.Y.*` 通配 + AND/OR 范围）

- 版本升至 `0.6.0`（`CclVersion.VALUE` 单一来源）。

## v0.5.16 调整：关键字并入核心定位直接扫描（撤下弹窗询问）

GUI 实测反馈：v0.5.15 点「🔍 核心代码定位」先弹「可选关键字」输入框（留空确定=纯 8 类、
取消=放弃扫描）是多余一步。改为关键字与核心代码**一步结合、点击即扫**。

- **点「🔍 核心代码定位」直接开扫**：内置 8 类 + 附带关键字一次完成，不再弹输入框，
  不再有「留空/取消」分支。`promptCustomKeywords` 弹窗已移除。
- **关键字外置到 `config.properties`（v0.5.16 新键 `keywordHits`）**：
  - 键**缺省** = 内置常用抓包词 `sign|token|Authorization|/api/`；
  - **显式配置**整体覆盖（用 `|` 分隔，忽略大小写、OR 匹配）——抓包拿到新关键字直接改文件，
    沿用热重载，无需重启；
  - **空串** = 关闭附带关键字（回纯 8 类）。
  - 示例：`keywordHits=sign|token|Authorization|/api/`
- **命中表现不变**：关键字命中以「🧷 自定义关键字命中」（KEYWORD_HIT）类别并入同一结果窗，
  参与「🎯 多类别核心嫌疑」聚合，双击节点行级定位到字符串所在行。
- 版本升至 `0.5.16`（`CclVersion.VALUE` 单一来源）。

## v0.5.15 调整：抓包关键字并入「🔍 核心代码定位」（撤下 v0.5.14 独立搜索菜单）

按使用反馈把 v0.5.14 的独立「🧷 按字符串搜代码」收拢进主流程：关键字搜索本就是
「找核心代码」的一种输入，独立菜单让两种能力割裂，且独立结果不参与核心嫌疑聚合、
无法复用进度/取消/忽略/导出等主流程能力。

- **菜单回到 3 项**：撤下「🧷 按字符串搜代码（抓包关键字定位）…」独立菜单。
- **核心定位执行时可选附加关键字**：点「🔍 核心代码定位」先弹关键字输入框，
  可填抓包/接口关键字（Burp 抓包里的 `sign` / `token` / `Authorization` / 接口路径等），
  多个用 `|` 或 `,` 分隔（忽略大小写、OR 匹配）；**留空确定 = 仅按内置 8 类定位，
  点取消 = 放弃本次扫描**。
- **命中并入同一结果窗口**：关键字命中以新类别 **「🧷 自定义关键字命中」（KEYWORD_HIT）**
  出现在分类结果，label 为 `类全名 :: 方法名`、理由为「含自定义关键字 `kw×次数`」
  （多关键字命中 `、` 连接）；随主扫描共享进度框 / 取消 / 右键忽略 / 一键噪声排除 /
  HTML、CSV 导出。
- **参与多类别核心嫌疑聚合**：同一方法同时被内置类别与自定义关键字命中时，按 ≥2 类
  聚合到「🎯 多类别核心嫌疑」——抓包关键字（sign/token/接口路径）命中的方法通常正是
  签名/网络类核心逻辑，聚合后定位价值更高。
- **双击行级定位保留**：命中携带 `needle`（代码里实际大小写形态，如搜 `sign` 命中
  `Sign`），双击跳转所在方法后自动选中并滚动到关键字所在行（沿用 `LineLevelNavigator`
  的 JDK JTextComponent 方案，零反射）；`MatchHit` 同时新增 `fallbackRef`（方法级打开
  失败时降级到类级引用）。
- **匹配语义与上限**：忽略大小写字面量子串匹配（非正则）、单方法多关键字只产出一条
  命中、关键字须出现在本方法声明之后（过滤匿名类前缀窗口广播的系统性误报）；单次扫描
  命中总量上限保护，防过宽关键字撑爆结果树。实现由 `ExtraSearcher.searchString` 迁移为
  独立规则 `rule/KeywordRule`（随 `CoreCodeScanner` 构造传入、并入扫描主循环），
  `ExtraSearcher` 不再提供字符串搜索。
- 版本升至 `0.5.15`（`CclVersion.VALUE` 单一来源）。

## v0.5.14：按字符串搜代码（抓包关键字 → 行级定位）

依据 jadx 反混淆实战教程（公众号《安卓逆向系列 02》）的「技巧一：字符串搜索」工作流
（抓包发现 sign/token/Authorization → 全局搜索 → 定位代码）产品化：

- **新菜单 `🧷 按字符串搜代码（抓包关键字定位）…`**：输入任意字符串关键字（忽略大小写、
  字面量匹配），全量扫描所有方法体并列出命中位置——每个命中方法一行，
  label 为 `类全名 :: 方法名 (出现次数)`，tooltip/底部预览含首次出现的代码上下文片段。
- **行级定位直达**：结果行 `needle` 记录代码里实际出现的大小写形态（如搜 `sign` 命中
  `Sign`），双击后跳转到所在方法并**自动选中关键字所在行**（复用 v0.5.13 的
  `LineLevelNavigator` JDK JTextComponent 方案，零反射）。
- **对话建议词**：输入框默认值 `sign`，提示文案给出常用关键字（sign · token ·
  Authorization · /api/login · encrypt · AES/CBC/PKCS5Padding · SECRET_2024）。
- **规则库查缺**（教程「技巧五：Android API 特征表」）：`BusinessRule` 补
  `Settings.Secure`（设备标识采集）；其余特征（OkHttpClient/Retrofit/HttpURLConnection/
  MessageDigest/SecretKeySpec/Cipher/SharedPreferences/SQLiteDatabase/getDeviceId）
  经核对此前各版本已全覆盖。
- 版本升至 `0.5.14`（`CclVersion.VALUE` 单一来源）。

## v0.5.13 补丁：放弃 RSyntaxTextArea 反射，改用 JDK JTextComponent API（修复 NoSuchMethodException）

GUI 日志确认 v0.5.12 报 `NoSuchMethodException: jadx.gui.ui.codearea.CodeArea.search(String, int, boolean, boolean, boolean)`
——jadx 1.5.6 捆绑的 RSyntaxTextArea **3.x 已移除**这个老签名（它是 1.x/2.x 时代的 API，
3.x 改用 `SearchEngine.find()` 静态方法）。v0.5.10～0.5.12 都在赌这个不存在的实例方法。

- **彻底放弃 RSyntaxTextArea 专属反射**：代码区组件继承链
  `CodeArea → RSyntaxTextArea → RTextArea → JTextComponent`，其中 `JTextComponent`
  是 **JDK 自带类**，插件与 jadx-gui 共享 bootstrap ClassLoader —— 直接 `instanceof`
  强转后调用公开方法，**零反射、零版本风险**。
- **定位逻辑**：`getText()` 找 `needle` 偏移（优先从 `getCaretPosition()` 向后找，
  避免命中类里更早出现的位置；找不到再从头找）→ `setCaretPosition(end) +
  moveCaretPosition(idx)` 选中匹配串，caret 移动触发自动滚动到目标行。
- **折叠区兜底**：目标行位于折叠代码块内时 `setCaretPosition` 可能失败，降级为
  仅定位到匹配起点。
- 组件查找仍沿用 v0.5.12 的「沿父类链按类名匹配 RSyntaxTextArea」（已验证可命中
  CodeArea 子类），渲染重试（150ms 等待 + 10×60ms）保留。
- 版本升至 `0.5.13`（`CclVersion.VALUE` 单一来源）。

## v0.5.12 补丁：沿父类链匹配代码区（修复 v0.5.11 找不到组件）

GUI 日志确认 v0.5.11 报 `no RSyntaxTextArea found in jadx main frame`：jadx-gui 的代码区
**并不直接是** `org.fife.ui.rsyntaxtextarea.RSyntaxTextArea`，而是它自己的子类
`jadx.gui.ui.codearea.CodeArea`（继承 RSyntaxTextArea）。v0.5.11 按「本类名精确匹配」
自然找不到。v0.5.10 用 `isInstance` 本可匹配子类，但被 ClassLoader 隔离坑掉。

- **沿父类链匹配**：`isCodeArea` 现在遍历组件 `getClass()` 的继承链（自身 → 各级父类），
  任一层类名等于 RSyntaxTextArea 即命中——既能匹配 `CodeArea` 这类子类，
  又全程不做 `Class.forName`，不受 ClassLoader 隔离影响。
- 反射调用（`search / getText / setCaretPosition`）继续用组件实际 `getClass()` 获取 Method，
  `search` 定义在父类 RTextArea 上，`getMethod` 可直接找到继承的 public 方法。
- 版本升至 `0.5.12`（`CclVersion.VALUE` 单一来源）。

## v0.5.11 补丁：修复行级定位在部分 jadx 环境下不生效

GUI 实测 v0.5.10 双击 `192.168.1.20` 仍只能高亮到 `onCreate` 方法名，未落到字符串所在行。
根因：插件由独立 ClassLoader 加载，直接用 `Class.forName` 加载 jadx-gui 自带的
`org.fife.ui.rsyntaxtextarea.RSyntaxTextArea` 可能失败，或拿到的 Class 对象与组件实际
Class 不属同一 ClassLoader，导致 `isInstance` / 反射调用不匹配。

- **按类名匹配组件**：`LineLevelNavigator.findCodeArea` 不再用 `Class.forName + isInstance`，
  改为比较 `Component.getClass().getName()`，彻底绕过 ClassLoader 隔离。
- **优先用组件实际 Class 反射**：所有 `search / getText / getCaretPosition / setCaretPosition`
  均通过组件实例的 `getClass()` 获取 Method，避免不同 ClassLoader 的 Class 对象不兼容。
- **异步渲染重试**：`open()` 切到目标方法后，代码区反编译文本可能是异步加载；
  `LineLevelNavigator` 先等待 150ms，再对代码区文本做多达 10 次、每次 60ms 的重试，
  命中关键字后才执行 `search()`，大幅提升稳定性。
- **保留兜底**：行级定位失败时静默降级为方法级 + 底部片段预览，不影响原有跳转。
- 版本升至 `0.5.11`（`CclVersion.VALUE` 单一来源）。

## v0.5.10 补丁：方法级跳转后自动行级搜索并高亮（方案 B）

针对 v0.5.9 「高亮落在 onCreate 而非字符串所在行」的进一步加固：在 jadx 公开 API
只能定位到方法级的前提下，**跳转后用反射调用代码区组件的 `search()` 把光标直接落到目标字符串上**，
肉眼上等价于"行级定位"：

- **核心实现 `ui/LineLevelNavigator`**：纯反射加载 jadx-gui 自带的
  `org.fife.ui.rsyntaxtextarea.RSyntaxTextArea`（不引入新依赖，避免污染编译 classpath）；
  在 jadx 主窗口的 Swing 组件树里深度优先找到第一个 RSyntaxTextArea 实例，
  调用其 `search(String, int, forward=true, caseSensitive=true, wholeWord=false)` 找首个匹配，
  找到后再 `setCaretPosition` 触发 jadx 滚动到光标所在行。
- **数据层**：`FoundLine` 新增 `needle` 字段（行级搜索关键字）。
  `ExtraSearcher.collectUrlHostsReport` 给每行设 `needle = host`（如 `192.168.1.10` /
  `example.com`）；调用点行 `needle` 保持 null（无需行级定位）。
- **调用层**：`FoundListWindow.jumpTo` 在方法级/类级 `open()` 成功后，
  `SwingUtilities.invokeLater` 延迟一帧再调 `LineLevelNavigator.searchAndHighlight`（让代码区
  完成渲染再 search，结果更稳）。失败时一律静默吞掉并降级为方法级 + 底部预览。
- **UI 层**：底部片段预览面板默认提示文本更新为「v0.5.10 起双击跳转后会自动搜索并高亮首个匹配」；
  `buildDetailText` 新增「行级高亮」行，提示本次跳转会自动搜 `needle`。
- **兼容性**：RSyntaxTextArea 2.6.x 起的 `search` 签名稳定，jadx-gui 1.4.x ~ 1.5.x 一直用它作为
  代码区；jadx 大版本若换文本组件，本工具会**安全降级**为不抛异常地返回 false，调用方走方法级 +
  底部预览的兜底路径，**不会**阻塞跳转或破坏现有功能。
- 版本升至 `0.5.10`（`CclVersion.VALUE` 单一来源）。

## v0.5.9 改进：检索结果窗口底部片段预览面板

按使用反馈「双击域名/IP 后高亮在 onCreate 而非字符串所在行」改进。jadx-gui 公开 API
（`JadxGuiContext.open(ICodeNodeRef)`）只能定位到类/方法/字段节点，**无法精确到字符串常量所在行**，
这是 jadx 端的限制，不是插件 bug。本版采用「方法级跳转 + 底部预览」组合，让用户在跳转到的
方法里也能肉眼快速定位到具体 URL 语句：

- **底部片段预览面板**：「🔎 查找调用点」与「🌐 聚合域名 / URL」结果窗口新增底部只读片段预览区，
  选中列表某一行时自动展示该行：
  - 标题（图标 + label）
  - 详细（tooltip 全文：首个出现位置 `类全名 :: 方法名` + 完整 URL + **代码上下文片段**）
  - 跳转能力说明（方法级 / 类级降级 / 不可跳转说明）
- **默认提示文本**：首次打开时面板给出说明——jadx 只能精确到方法级，跳转后把鼠标停在 `onCreate` 等
  方法名上时，URL 所在的具体代码行已在右侧预览中给出，可直接肉眼定位。
- **无需悬停也能看到上下文**：之前 tooltip 里的「代码上下文片段」需要把鼠标悬停才能看到，
  现在固定在窗口底部，**双击跳转 + 看预览** 即可完成定位，不必再悬停或 `Ctrl+F` 搜索。
- 实现：`FoundListWindow` 新增 `JTextArea snippetArea`（等宽字体、灰底、不可编辑、5 行高），
  `ListSelectionListener` 在选中变化时调用 `buildDetailText(line)` 重写内容；SOUTH 面板
  重构为 `BorderLayout` 嵌套，状态条在上、片段预览在下。
- 版本升至 `0.5.9`（`CclVersion.VALUE` 单一来源）。

## v0.5.8 补丁：聚合域名双击跳转再增强 + 类级降级

针对「🌐 聚合域名 / URL」双击后 jadx 仍未展开到代码位置的问题进一步加固：

- **双击按鼠标位置取行**：`FoundListWindow` 不再依赖「当前选中项」，而是直接根据鼠标双击坐标
  定位到被击中的行，避免"双击的是 A 行、跳转的是 B 行"或选中项为空导致无反应。
- **方法级 → 类级 fallback**：每行除了保存首个方法引用，还额外保存所在类的引用；
  当 `guiContext.open(methodRef)` 打不开时，自动尝试 `open(classRef)` 跳转到类，
  至少让主代码区展开到相关类，再靠 tooltip 里的上下文片段肉眼定位 URL 语句。
- **失败提示更明确**：方法 + 类都失败时弹窗说明「请确认当前打开文件与扫描工程一致」，
  并提示可悬浮查看首个 URL 与代码上下文。
- 源码：`FoundLine` 新增可选 `fallbackRef` 字段；`ExtraSearcher` 给 URL 行与调用点行都附加类级 fallback；
  `FoundListWindow` 双击与回车跳转均使用坐标/索引精确定位。
- 版本升至 `0.5.8`（`CclVersion.VALUE` 单一来源）。

## v0.5.7 改进：域名 / URL 聚合窗口顶部统计 + 跳转不再静默

针对「🌐 聚合域名 / URL」的 GUI 实测反馈（聚合行双击无反应、看不到扫描量）改进：

- **顶部统计总览**：结果窗口列表上方新增摘要条——「共提取 N 个 http(s) 地址，去重聚合为 M 个域名
  （出现在 X 个方法 / Y 个类中，用时 Z ms）」。N 为聚合前 URL 总数（同地址多次出现计多次），
  每行域名后括号数字为改域出现次数（原有），一眼看清扫描总量与分布。
- **双击跳转不再静默**：
  - 行无对应代码（聚合/说明行）双击时给出说明弹窗（此前对无说明的行完全无反应）；
  - jadx 的 `open()` 若"静默返回 false（不抛异常）"，现在会弹提示告知
    「未能定位到代码，请确认当前打开文件与扫描工程一致」，不再无声无息；
  - 域名行本身携带首个出现位置的方法引用，双击跳转到该方法（jadx 反编译代码无法精确到
    字符串常量行，跳转目标为方法级）。
- **tooltip 增强**：域名行悬浮提示由「首个位置 — URL」升级为三行——出现次数、首个位置与完整 URL、
  **代码上下文片段**（截取 URL 所在语句前后文本），便于肉眼定位到具体语句；tooltip 支持换行显示。
- 底层：`ExtraSearcher` 新增 `collectUrlHostsReport`（返回带统计的 `UrlAggReport`），旧
  `collectUrlHosts` 保留委托；入口把模态忙碌框工具方法泛型化为 `runBusy(Supplier, Consumer)`。
- 版本升至 `0.5.7`（`CclVersion.VALUE` 单一来源）。

## v0.5.6 调整：移除 .so 原生库功能，聚焦 DEX 核心代码定位

按使用反馈最终收拢：**移除 .so 原生库相关功能**（撤下 v0.5.3 起的第 4 个菜单与 v0.5.5 起的
顺带扫描集成），功能聚焦于在 DEX/APK/JAR 中查找核心代码。

- **菜单回到 3 项**：移除「🧬 扫描原生库(.so)…」；「🔍 核心代码定位」不再顺带扫 .so，结果窗口
  不再出现「🧬 原生库(.so)证据」页签（回退 v0.5.5 的同窗合并，恢复为 🎯 嫌疑 + 📂 分类两个页签）。
- 核心代码扫描中的「📦 NATIVE_SO」类目**保留**——它检测的是 dex 里 Java 对 native 的**用法**
  （native 方法声明 / `System.loadLibrary` 等），属于 DEX 扫描范畴，与被移除的 .so 二进制证据
  扫描不是一回事。
- 源码保留：`analysis/NativeLibSearcher` 与 `ui/NativeTabPanel` 未删除但**不再被主流程引用**，
  日后如需恢复 .so 扫描，在菜单注册处重新挂接即可。
- 版本升至 `0.5.6`（`CclVersion.VALUE` 单一来源）。

## v0.5.5 迭代：核心定位顺带扫 .so（同窗合并）+ 原生库扫描体验修复

针对「选完 APK 后没反应」「每次都要重新选文件」两条反馈，以及「能否与核心代码扫描结合」的需求：

- **🧬 原生库结果并入核心代码定位（同窗查看）**：点「🔍 核心代码定位」扫 dex 时，若当前打开的
  工程内含 `.so`，会在同一结果窗口追加第 3 个页签 **「🧬 原生库(.so)证据 (N)」**——Java 层命中
  （多类别嫌疑 / 分类结果）与原生层证据一次扫描、同窗查看；进度框在 dex 扫完后自动切到
  “正在读取 lib/**/*.so 抽取原生库证据…”阶段，结果窗口顶部同步显示原生库统计（个数/跳过/耗时）。
  工程无 `.so` 则不显示该页签，核心扫描行为与耗时不改变（对 jadx 打不开的加壳 APK 仍走第 4 个
  菜单的独立文件模式）。
- **原生库扫描“选完没反应”修复**：扫描结束**必定弹出结果窗口**——有 .so → 证据行；工程无 .so →
  说明行；异常 / 内存不足 → 错误行（原因直接展示在窗口里，不再静默或只闪一下提示框）。
- **记住上次文件 + 一键重扫**：再次点「🧬 扫描原生库(.so)…」时若上次文件仍在，先问
  “是否直接重新扫描该文件”；选“重新扫描该文件”即跳过文件选择框。选择框默认定位到上次目录并预选
  上次文件，换文件也少翻一层目录。
- 版本升至 `0.5.5`（`CclVersion.VALUE` 单一来源）。

## v0.5.4 修复：so 扫描选 APK 后进程闪退（流式内存模型）

**现象**：在 jadx 已打开较大脱壳 dex 的情况下点「🧬 扫描原生库(.so)…」选 APK，jadx-gui 直接消失。

**诊断**（无 hs_err / 无 crash 报告 + jadx-gui 启动参数 `-XX:MaxRAMPercentage=70` → 疑似内存峰值被系统 OOM 杀掉）：
v0.5.3 收集阶段会把 APK 内所有 `.so` 的 `byte[]` **一次性全部驻留内存**，读完才逐库抽字符串；
与正在反编译的大 dex 叠加后，多 ABI/多 35MB 级库可多占数百 MB → 触发系统杀进程。

**修复**（`NativeLibSearcher` 重构为流式）：
- 每个 `.so` **读入 → 立即抽字符串分类 → 随即释放 `byte[]`**，只保留轻量分类结果（`LibData` → `LibEvidence`）；
  峰值内存由「所有 .so 字节和」降为「单个最大 .so」。
- 新增**累计读取预算** 512MB：超预算/超单库上限的库只登记体积并提示，不再读内容，保护 jadx-gui。
- 对外入口 `scanNativeLibraries / scanNativeLibrariesAt` 签名与展示语义不变。

**验证**：
- 回归一致：xop-norasp 直扫 4 so（131ms）、wuji 单 35.8MB 库 22 导出/12 URL/141 命令字、
  `loadSame=true` 时 .so 总行仍 **[JUMP] → `com.wuji_app.app.Ipc::<clinit>`**；
- 低堆压测：`-Xmx64m` 跑 xop-norasp 通过、`-Xmx128m` 跑 wuji 35.8MB 单库 290ms 通过
  （旧实现多库叠加在此类小堆下会 OOM）。
- 版本升至 `0.5.4`（`CclVersion.VALUE` 单一来源）。

## v0.5.3 调整：恢复「🧬 扫描原生库(.so)」→ 独立文件模式

按实际反馈「只上传 dex、部分 APK 需脱壳拖不进 jadx」恢复 .so 证据扫描，并把入口改成
**不依赖反编译**的独立文件模式（本版起 v0.5.2 的「移除入口」决定被取代）：

- **菜单恢复为 4 项**：第 4 项「🧬 扫描原生库(.so)…」点击弹文件选择框，可选 **APK/AAB/ZIP/XAPK
  文件、解包目录、单个 .so、dex/jar**（dex/jar 自动回退其父目录找平级 `lib/`）。
- **加壳到打不开的 APK 也能直扫**：扫描器只把所选文件按 zip 打开、枚举 `lib/**/*.so` 抽字符串，
  全程不调用 jadx 反编译。实测连 jadx 都无法加载的 TargetLab-xop-norasp.apk（嵌套 zip 壳）
  ~150ms 直扫出 4 个 .so：`libprotector.so`（壳）、`libshadowhook.so`（Hook 库）、`libtargetlab.so`
  （业务库，内嵌 `TargetLab_Native_Secret_Key_7f3a` / `targetlab_check_password`）、`libc++_shared.so`。
- **跳转规则更稳**：仅当「所选文件与 jadx 当前打开工程为同一输入（canonical 比较）」时才建立
  dex 加载点索引，.so 总行双击跳 Java 加载处；不一致或纯文件模式时所有行走说明弹窗——
  不会把别的工程的加载类误挂到当前库上。
- 版本升至 `0.5.3`（`CclVersion.VALUE` 单一来源）。

## v0.5.2 调整：聚焦 DEX/JAR 核心代码扫描（移除原生库证据入口）

按使用反馈把能力收拢到「核心代码定位」：**移除「🧬 扫描原生库(.so)」菜单入口**。菜单收敛为 3 项，
全部作用于当前打开的 **DEX / APK / JAR** 反编译视图（`analysis/NativeLibSearcher` 源码保留，恢复时
在菜单注册处加回一行即可）：

- 版本统一升至 `0.5.2`（`CclVersion.VALUE`，窗口标题/状态栏/插件元数据同源）。
- **JAR 输入兼容确认**：jadx 会把 `.class` / `.jar` 经 java-input 统一转 dex 再反编译，规则对纯 Java
  类同样生效。用纯 Java demo jar（Socket / AES-GCM / SHA256withRSA / HMAC / Zip 信号）实测：
  扫描 1 类 11 方法 → **9 条命中**（NETWORK 3 / CRYPTO 4 / SIGNATURE 1 / WEAK_SECURITY 1），与 Android
  样本共用同一套分类与命中理由；无 AndroidManifest 时「入口可达⭐」自动为 0（不误报、不报错）。
- 说明：核心代码扫描里的「📦 NATIVE_SO」类目仍在——它检测的是 dex 中 Java 对 native 的**用法**
  （声明 native 方法 / loadLibrary），与被移除的 .so 二进制证据扫描不是一回事。

## v0.5.1 修复：.so 证据双击可跳转 + 窗口版本号显示

GUI 实测「🧬 扫描原生库(.so)」结果窗口双击无反应。根因：该窗口所有行都是**二进制证据**，
`FoundLine.codeRef` 全为 null，旧代码遇 null 直接静默返回。v0.5.1：

- **.so 总行 → 可跳转到 Java 加载处**：扫描前先在 dex 里做一次全量索引（`System.loadLibrary("x")`
  / `System.load("…/libx.so")` 字符串字面量，按「库核心名」匹配，兼容 `lib` 前缀与多 ABI），
  找到后把该方法的 code ref 挂到对应 .so 总行。实测 wuji：
  `lib/x86_64/libwuji_app_lib.so` 总行双击 → 跳 `com.wuji_app.app.Ipc::<clinit>`（加载入口）。
- **找不到加载方时给说明而非静默**：无对应 Java 加载方法的 .so 总行与分类行（导出符号/URL/路径/
  命令字聚合行）双击会弹提示，说明该证据在二进制库内、Java 层无源码行可跳、可用 rabin2/strings/
  Ghidra 复核。`FoundLine` 新增 `noJumpNote`，仅 `codeRef == null` 的行受影响。
- **窗口版本号显示**：新增 `CclVersion.VALUE` 单一版本源（当前 `0.5.1`）；「核心代码定位」结果窗口
  与全部检索/证据窗口标题与状态栏均显示版本，jar 文件名与构建版本同源同步。
- 状态栏会标注 `N 行为聚合/二进制证据行，双击查看说明`，避免"能不能跳"全靠试。

## v0.5.0 规则优化：7 样本语料回归驱动（误报治理 + 风险面补全）

用 TargetLab-debug/v1only/packed/xop(-norasp)、wuji(Tauri)、xcsckh 共 7 个 APK 做统一口径
基线（隔离 HOME、仅内置排除）后实施两类规则优化：

- **误报治理（治 FP）**：
  - `BodyScanRule` 新增「调用锚点」通道（`bodyCallKeywords()`）：关键字必须以紧邻 `(` 的
    调用形态出现才算命中，且须在本方法声明之后。`Socket` 由宽松子串改为调用锚点——
    修复 JSON 反序列化器解析 `SocketAddress/InetSocketAddress` 时被裸 `"Socket"` 误报
    的问题（实测 wuji 仅 jackson 一个库就贡献 52 条这类网络噪声）。
  - `NetworkRule` 排除 `onRequest*` 系统回调（`onRequestPermissionsResult` /
    `onRequestSendAccessibilityEvent` 等），避免方法名含 request 被误当接口请求。
  - 内置排除增补通用框架 `com.fasterxml.`(jackson) 与 `app.tauri.`(Tauri 壳框架)。
- **风险面补全（查全率）**：`WeakSecurityRule` 新增 14 个敏感/隐私合规关键词——剪贴板
  （ClipboardManager/getPrimaryClip/setPrimaryClip/ClipData）、WebView JS 注入与 file://
  访问开关（addJavascriptInterface/setJavaScriptEnabled/setAllowFileAccess/
  setAllowUniversalAccessFromFileURLs/setAllowFileAccessFromFileURLs）、外部存储公共目录、
  Zip 解压（ZipInputStream/ZipFile/ZipEntry）、DeepLink 跳转（ACTION_VIEW）。
  补上 TargetLab 中 Clipboard/DeepLink/ZipSlip 等此前只有入口命中、风险规则零覆盖的漏检。

回归结果与基线对比如下表（详见 harness/corpus-baseline.txt → corpus-v050.txt）：

| 样本 | v0.4.1 命中 | v0.5.0 命中 | 变化 |
| --- | --- | --- | --- |
| TargetLab-debug / v1only | 148 | 157 | 查全率 ↑ 9 条（剪贴板/Zip/JS 桥等风险面命中真实业务类），两版仍镜像一致 |
| TargetLab-packed | 5 | 5 | 壳样本不变（真 dex 仅 1 类 stub，业务代码被抽取） |
| TargetLab-xop / -norasp | 25 | 28 | +3 均为 ZipFile 命中，落在壳加载链（DexMerger.extractDexes / ProxyApplication.ensureDexesOnDisk / d.a）——静态可见的核心"dex 落盘解压"行为，属合理命中 |
| wuji | 333 | 243 | 噪声 ↓：扫描类数 2055→1698（jackson/tauri 共 357 个框架类免扫），NETWORK 129→87、SIGNATURE 56→30 假阳性集中下降；WEAK_SECURITY 0→8 补齐风险面 |
| xcsckh | 7 | 7 | 壳样本不变（6 类 JNI stub + 15 个 so，真逻辑在 native） |

关键佐证（治理的是噪声而非漏检）：
- **多分类核心嫌疑（≥2 类）**：wuji 8→8 不降反稳；TargetLab-debug/v1only 31→35（随风险面
  补全自然上浮）；xop/-norasp 4→4。
- **入口命中**：wuji COMPONENT_ENTRY 58→52，减少的 6 条全部是 `app.tauri.*` 系统广播接收器
  （框架），真实 Activity/入口类命中保持。
- **方法级入口可达**：TargetLab-debug/v1only 112→121⭐（新增风险面方法多为可达路径），
  wuji 命中总量下降但核心嫌疑不变，说明 Socket 锚点与 onRequest* 排除只清掉了框架噪声。

## v0.4.1 补丁：跳转错误可见化 + 类行双击降级

GUI 实测发现双击/回车/右键「跳转到代码」在个别样本/加载方式下无反应（异常被静默吞掉）：

- **错误可见化**：结果窗口与检索结果窗口的跳转失败不再静默，改为完整错误日志 + 弹窗显示
  真实异常（类名 + 信息），便于定位"扫描工程与当前视图不一致"等问题。
- **类行双击降级**：双击 📦 类名行时，自动跳到该类下第一个命中方法（此前类行双击无反应）。

## v0.4.0 新增：免重启 + 一键降噪 + 原生库(.so) 证据（Tauri 原生型样本实测驱动）

用「无极 wuji」(Tauri/Rust + 单 dex) 等原生核心型样本实测 v0.3.2 后新增三项能力：

- **配置热重载（免重启）**：`~/.core-code-locator/config.properties` 在每次扫描/反查前自动检测
  修改时间与大小，外部改动后**无需重启 jadx-gui** 即自动生效（v0.3.2 及以前必须重启才能读到
  新排除前缀，是本版本解决的主要痛点）。
- **一键噪声排除**：结果窗口顶部新增「🗑 噪声排除建议」条——自动统计「扫描类数 ≥ 20 且
  **整包 0 命中**」的包前缀（通常是遗漏的混淆前缀 / 三方库子包），点「一键追加排除」即写入
  配置并热重载。由于候选前缀 0 命中，排除**不会误伤任何真实命中**。
  实测：某大样本在仅内置排除基线时 2046 类 / 332 命中 / 6 条建议 → 一键追加后重扫
  1818 类 / **332 命中不变** / 建议归零（228 个纯噪声类被跳过）。
- **🧬 扫描原生库(.so)**：新菜单从输入目录 / APK 压缩包 / 单 dex 父目录三种来源定位样本携带的
  `lib/**/*.so`，抽取可读字符串按 **导出符号 Java_\* / URL 端点 / 路径 / 命令字敏感词** 分类展示。
  适用 Tauri / Flutter / Unity 等「Java 薄壳 + 原生核心」型 App——其 URL 与命令字编译在 .so 里，
  Java 层「🌐 聚合域名/URL」得到 0 是**真阴性**而非缺陷；本扫描补上该盲区。
  实测 wuji：识别出 22 个 `Java_*` JNI 导出（含 `Java_com_wuji_1app_app_Ipc_ipc` 等）、
  Rust 插件命令（fetch / proxy / m3u8 / sniff 相关）与更新端点
  `https://wuji.moshangwangluo.com/wuji/updater_win.json`。

## v0.3.2 补丁：框架类排除与大样本降噪（脱壳 dump 实测驱动）

用一份 **1.8 万类 / 13.5 万方法的脱壳 dump（37 个 dex）** 实测后发现并修复三个问题：

- **排除清单补全框架命名空间**：新增 `android.` / `com.android.` / `dalvik.` / `libcore.` / `com.sun.` /
  `org.xml.` 等系统运行时前缀。常规 APK 不会包含这些类所以此前未暴露，但脱壳 dump 会把
  Android 框架类一并带出，不排除会直接污染结果（实测 7688 条命中里约 7100 条是框架类）。
- **排除配置语义修正**：原 `config.properties` 一旦存在就**完全覆盖**内置默认清单，导致升级插件后
  新增的默认排除不生效。现改为**内置默认恒生效（安全基线）+ 用户配置在其上追加排除**，
  升级即自动获得新默认项。
- **ContentProvider 方法收窄**：`query / insert / update / delete / getType` 仅当类继承自
  `ContentProvider` 时才作为入口生命周期命中，消除普通类同名方法（如 `BluetoothDevice.getType`、
  `StringBuilder.insert`）的误报。

修复后同一样本命中从 7688 → **173 条（100% 落在业务包）**，扫描耗时 22.7s → **1.0s**：
三方库排除在 1.8 万类规模上验证有效（排除 18026 个类，仅扫 217 个业务类）。

## v0.3.1 补丁：匿名类误报修复（真 APK 实测驱动）

用靶标 APK 实测 v0.3.0 后修复两个影响命中质量的问题：

- **修复组件入口漏标**：`Service / BroadcastReceiver / ContentProvider` 的 extends 正则缺少捕获组，
  命中时抛出 `No group 1` 异常，导致这类组件（如 ContentProvider 子类）从未被标为入口。现已补全。
- **消除匿名类关键词"广播"误报**：jadx 对方法体内联匿名类（`$1..$N`）返回的代码文本是从宿主方法
  开头截取的累计前缀窗口，宿主方法里出现过的关键词会被复制到每个匿名方法上（实测 7 个按钮回调
  全部误命中同一关键词）。现要求方法体命中必须落在"本方法声明之后"，前缀窗口噪声被自然滤除。
  实测：TargetLab 靶标 App 命中 190 → 148，IPC 组件入口补齐，真实风险点（空实现 TrustManager、
  DES/ECB、TLSv1.1、签名逻辑）零丢失。

## v0.3.0 新增：更快、更准、可追溯

- **降噪**：扫描自动跳过 `android / androidx / okhttp3 / retrofit2 / kotlin / java.*` 等框架与三方库类，
  大 APK 命中量与耗时明显下降；可在 `~/.core-code-locator/config.properties` 追加排除前缀
  （内置基线恒生效，见 v0.3.2 说明）。
- **多类别核心嫌疑 Tab**：同一方法命中 ≥ 2 个类别（如「网络 + 加密 + 签名」）会聚合到
  「🎯 多类别核心嫌疑」，按命中类别数排序——这种方法几乎必是真核心逻辑。
- **⭐ 入口可达标记**：从 `Application / 组件入口` 方法出发做 BFS（深度 3）调用图追踪，
  命中若位于入口可达调用链上，方法节点前会带 ⭐（提示"这是被入口调起来的关键路径"）。
- **右键忽略**：对某个方法或整个类右键 →「忽略」，立即从结果消失，并持久化到配置文件，
  下次扫描不再出现。
- **导出报告**：结果窗口可直接导出 HTML / CSV 审计报告，供团队协作或留档。
- **反查调用点**：菜单「🔎 查找调用点」输入 `类.方法` 或裸方法名，列出所有调用它的位置——
  沿调用链向上追业务入口。
- **域名 / URL 聚合**：菜单「🌐 聚合域名/URL」提取全部 http(s) 地址，按域名聚合计数，
  快速看出 App 连接了哪些服务器（含首次出现位置，可跳转）。
- **加固/混淆提示**：扫描到 360 / 乐固 / 娜迦 / 梆梆 / 爱加密等壳指纹或类名高度混淆时，
  结果顶部给出提示，避免在"被加壳"状态下误判"没有核心代码"。

## 许可

[MIT](LICENSE) © 2026 lettures
