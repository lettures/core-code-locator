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
   - **🔍 核心代码定位** —— 点击即扫：内置 8 类 + 附带关键字
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
# 「核心代码定位」自动附带的关键字（用 | 分隔，忽略大小写 OR 匹配）
# 缺省=sign|token|Authorization|/api/；删掉本行可恢复内置默认；置空=关闭附带关键字（回纯 8 类）
keywordHits=sign|token|Authorization|/api/
```

- 想让某个三方库参与扫描：把它的前缀从 `excludePackagePrefixes` 里删掉即可；
- 想全部扫描（不排除任何库）：把该项设为空字符串；
- 改完保存即可，**无需重启 jadx-gui**（每次扫描/反查前自动热重载）；
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
│   │   ├── CoreCodeLocatorPlugin.java     # 插件入口：注册 4 个菜单动作（含 📚 第三方组件引用；核心定位点击即扫，自动并入 keywordHits）
│   │   ├── Category.java                  # 类别枚举（8 类 + 🧷 KEYWORD_HIT 关键字命中 + 📚 SDK_REFERENCE 组件引用，含图标）
│   │   ├── MatchHit.java                  # 单条命中（fallbackRef 类级降级 + needle 行级关键字）
│   │   ├── ScanResult.java                # 扫描结果聚合 + 多类别嫌疑 + 可达标记
│   │   ├── ScanConfig.java                # 外置配置（~/.core-code-locator/config.properties，热重载；含 keywordHits 附带关键字）
│   │   ├── CoreCodeScanner.java           # 扫描调度器（排除库/建调用图/可达性/加固提示/噪声统计；关键字规则并入主循环）
│   │   ├── SdkScanner.java                # 第三方组件扫描器（不跳过三方库，独立于 CoreCodeScanner）
│   │   ├── analysis/
│   │   │   ├── CallGraphIndex.java        # 方法级调用图（文本启发式，BFS 可达性）
│   │   │   ├── FoundLine.java             # 检索结果行（label/tooltip/跳转引用/needle 行级搜索关键字）
│   │   │   ├── ExtraSearcher.java         # 反查调用点 + 域名/URL 聚合（字符串搜索已迁 rule/KeywordRule）
│   │   │   └── NativeLibSearcher.java     # .so 证据扫描器（源码保留、不再被主流程引用）
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
│   │   │   ├── SdkSignatureRule.java      # SDK 引用规则（SDK_REFERENCE）
│   │   │   ├── SdkSignature.java          # SDK 签名（包前缀 + 类证据）
│   │   │   └── SdkSignatureLibrary.java   # 内置 SDK 签名库（10+ 常用第三方组件）
│   │   └── ui/
│   │       ├── ResultWindow.java          # 核心定位结果窗口（🎯 嫌疑 Tab / 📂 分类 Tab / 右键忽略 / 导出）
│   │       ├── FoundListWindow.java       # 检索结果列表（调用点/域名；顶部统计 + 底部片段预览 + 行级定位）
│   │       ├── LineLevelNavigator.java    # 行级定位（改用 JDK JTextComponent API，零反射）
│   │       ├── NativeTabPanel.java        # 原生库证据列表面板（源码保留、不再被主流程引用）
│   │       ├── SdkResultWindow.java       # SDK 引用结果窗口（按 SDK 聚合、双击跳转、片段预览）
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

### v0.6.0：第三方组件引用扫描（SDK 暴露面识别）

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

- 版本升至 `0.6.0`（`CclVersion.VALUE` 单一来源）。

## 许可

[MIT](LICENSE) © 2026 lettures
