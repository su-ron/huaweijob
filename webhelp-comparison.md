# ApplicationInfoEx.getWebHelpUrl() vs ExternalProductResourceUrls.helpPageUrl 对比分析

> **版本**: IntelliJ Platform 2026.1.1  
> **本文档对比** `getWebHelpUrl()`（旧 API）和 `helpPageUrl`（新 API）的差异、设计思路与迁移指南

---

## 一、基本信息对比

| 对比项 | `ApplicationInfoEx.getWebHelpUrl()` | `ExternalProductResourceUrls.helpPageUrl` |
|--------|--------------------------------------|-------------------------------------------|
| **所在包** | `com.intellij.openapi.application.ex` | `com.intellij.platform.ide.customization` |
| **所属类** | `ApplicationInfoEx`（抽象类） | `ExternalProductResourceUrls`（接口） |
| **引入时间** | 远古 API（IntelliJ 早期版本） | 2023+（新资源定位架构） |
| **当前状态** | `@Deprecated` + `@ApiStatus.ScheduledForRemoval` | ✅ 当前主力 API |
| **获取方式** | `ApplicationInfoEx.getInstanceEx().getWebHelpUrl()` | `ExternalProductResourceUrls.getInstance().helpPageUrl` |
| **返回类型** | `String` | `((topicId: String) -> Url)?`（可为 null 的函数） |
| **null 安全性** | 未标注 @Nullable，默认有 fallback URL | 显式可为 null，IDE 可以关闭帮助功能 |

---

## 二、功能差异

### 2.1 数据来源

**旧 API** — 读取 `ApplicationInfoImpl.java`:
```java
// ApplicationInfoImpl.java:85
private String myWebHelpUrl = "https://www.jetbrains.com/idea/webhelp/";

// ApplicationInfoImpl.java:168-170 — 从 XML 解析
case "help": {
    String webHelpUrl = getAttributeValue(child, "webhelp-url");
    if (webHelpUrl != null) {
        myWebHelpUrl = webHelpUrl;
    }
}
```

对应的 `ApplicationInfo.xml` 配置：
```xml
<component name="HelpResources">
  <help webhelp-url="https://www.jetbrains.com/idea/webhelp/"/>
</component>
```

**新 API** — 通过 Application Service 注册实现：
```kotlin
// ExternalProductResourceUrls.kt
interface ExternalProductResourceUrls {
    companion object {
        @JvmStatic
        fun getInstance(): ExternalProductResourceUrls =
            ApplicationManager.getApplication().getService(ExternalProductResourceUrls::class.java)
    }

    val helpPageUrl: ((topicId: String) -> Url)?
}
```

默认实现 `BaseJetBrainsExternalProductResourceUrls`：
```kotlin
abstract val baseWebHelpUrl: Url?

override val helpPageUrl: ((topicId: String) -> Url)?
    get() = baseWebHelpUrl?.let { baseUrl ->
        { topicId ->
            baseUrl.resolve("${ApplicationInfo.getInstance().shortVersion}/")
                .addParameters(mapOf(topicId to ""))
        }
    }
```

### 2.2 返回值差异

| 特性 | `getWebHelpUrl()` | `helpPageUrl` |
|------|-------------------|---------------|
| 返回值示例 | `"https://www.jetbrains.com/idea/webhelp/"` | `(topicId) -> Url` 函数 |
| 是否支持 topic | ❌ 固定 URL | ✅ 支持按 topicId 构建帮助页 URL |
| 版本感知 | ❌ 不包含版本号 | ✅ 自动追加当前版本号 |
| 可覆盖性 | 通过 ApplicationInfo.xml 的 `webhelp-url` 属性 | ✅ 通过 `plugin.xml` 注册 `overrides="true"` 的 Service 实现 |

### 2.3 生成 URL 对比

假设 IDE 版本为 `2026.1`：

```
// 旧 API
info.getWebHelpUrl()
→ "https://www.jetbrains.com/idea/webhelp/"

// 新 API
ExternalProductResourceUrls.getInstance().helpPageUrl?.invoke("gettingStarted")
→ "https://www.jetbrains.com/idea/webhelp/2026.1/?gettingStarted"
```

---

## 三、迁移路径

### 3.1 旧代码 -> 新代码

```java
// ❌ 旧写法（已废弃）
ApplicationInfoEx info = ApplicationInfoEx.getInstanceEx();
String helpUrl = info.getWebHelpUrl();

// ✅ 新写法（推荐）
import com.intellij.platform.ide.customization.ExternalProductResourceUrls;
import com.intellij.util.Url;

ExternalProductResourceUrls urls = ExternalProductResourceUrls.getInstance();
Function<String, Url> helpPageUrl = urls.helpPageUrl;
if (helpPageUrl != null) {
    Url url = helpPageUrl.apply("myHelpTopic");
    // 或者用 invoke（Kotlin -> Java 桥接）
}
```

### 3.2 简化：如果你只需要固定帮助 URL

```java
// 不需要 topic 时，可以这么做
ExternalProductResourceUrls urls = ExternalProductResourceUrls.getInstance();
Url helpUrl = urls.helpPageUrl != null ? urls.helpPageUrl.apply("") : null;
```

---

## 四、外部插件如何自定义帮助 URL

如果第三方 IDE 或企业定制版需要覆盖帮助 URL，需要注册 Service：

### 4.1 实现接口

```kotlin
// Kotlin
class MyExternalProductResourceUrls : BaseJetBrainsExternalProductResourceUrls() {
    override val baseWebHelpUrl: Url? =
        Urls.newFromEncoded("https://docs.mycompany.com/help")
    override val productPageUrl: Url =
        Urls.newFromEncoded("https://mycompany.com/products/ide")
    override val basePatchDownloadUrl: Url =
        Urls.newFromEncoded("https://downloads.mycompany.com/patches")
    override val youtrackProjectId: String = "MYPROJ"
    override val shortProductNameUsedInForms: String? = "MyIDE"
}
```

### 4.2 注册 Service

```xml
<!-- plugin.xml -->
<applicationService
    serviceInterface="com.intellij.platform.ide.customization.ExternalProductResourceUrls"
    serviceImplementation="com.mycompany.MyExternalProductResourceUrls"
    overrides="true"/>
```

> **注意**: `overrides="true"` 是必须的，因为平台已经有默认实现（`BaseJetBrainsExternalProductResourceUrls` 的子类），外部插件必须覆盖它。

---

## 五、设计思路对比

| 维度 | 旧 API (`ApplicationInfoEx`) | 新 API (`ExternalProductResourceUrls`) |
|------|-------------------------------|----------------------------------------|
| **关注点** | IDE 全局应用信息（版本号、构建号、图标、URL 等） | **单一职责**：只负责定位外部资源 URL |
| **聚合程度** | 高耦合，所有 URL 在同一个类里 | 低耦合，一个接口管理多种资源 URL |
| **扩展方式** | 重写 `ApplicationInfoImpl`（侵入性强） | 注册 `overrides="true"` Service（标准扩展点） |
| **topic 支持** | 无 | ✅ 通过 `((topicId: String) -> Url)?` 支持上下文帮助 |
| **可配置性** | 仅通过 ApplicationInfo.xml 硬编码 | 支持 XML 配置 + 编程式覆盖 |

旧 API 把所有信息塞进 `ApplicationInfoEx`，成了一个上帝类；新 API 遵循 **单一职责原则**，将"外部资源 URL 定位"抽成独立的服务接口，且通过 Kotlin 函数类型天然支持参数化 URL 构建。

---

## 六、相关废弃方法对照

| 旧方法（ApplicationInfoEx） | 新接口属性（ExternalProductResourceUrls） |
|-----------------------------|-------------------------------------------|
| `getWebHelpUrl()` | `helpPageUrl` |
| `getDownloadUrl()` | `downloadPageUrl` |
| `getDocumentationUrl()` | `gettingStartedPageUrl` |
| `getSupportUrl()` | `technicalSupportUrl` |
| `getYoutrackUrl()` | `bugReportUrl` |
| `getFeedbackUrl()` | `feedbackReporter` |
| `getWhatsNewUrl()` | `whatIsNewPageUrl` |
| `getWinKeymapUrl()` / `getMacKeymapUrl()` | `keyboardShortcutsPdfUrl` |

---

## 七、总结

1. **`getWebHelpUrl()` 在 2026.1.1 仍可用**，但已标注 `@Deprecated` + `@ApiStatus.ScheduledForRemoval`，未来版本将移除。
2. **推荐立即迁移**到 `ExternalProductResourceUrls.getInstance().helpPageUrl`，迁移成本低且能获得 topic 支持。
3. **新 API 的核心优势**：
   - Kotlin 函数类型 `((topicId: String) -> Url)?` 支持按 topic 构建参数化 URL
   - `@OverrideOnly` 接口 + `overrides="true"` Service 注册，可扩展性更强
   - 遵循单一职责，不再污染 `ApplicationInfoEx` 上帝类
4. **对外部插件的影响**：
   - 如果只是**读取**帮助 URL，只需将 `info.getWebHelpUrl()` 换成 `ExternalProductResourceUrls.getInstance().helpPageUrl?.invoke(topicId)`
   - 如果是**自定义 IDE 发行版**需要修改帮助 URL，注册 `overrides="true"` 的 Service 实现

