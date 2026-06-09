# CodeStyle 恢复默认值时 Default.xml 未更新 Bug 分析报告

## 概述

**Bug 现象**：在 `Settings → Editor → Code Style → Java → Tabs and Indents` 中：
1. 先修改为非默认值 → Apply → Default.xml 正确更新
2. 再将值恢复为默认值 → Apply → Default.xml **未更新**
3. 重启 IDE 后，旧的非默认值仍然生效，默认值不生效

---

## 环境信息

- 项目：IntelliJ IDEA Community Edition (2026.1.1)
- 涉及源文件：

| 文件 | 路径 |
|------|------|
| `CodeStyleSchemeImpl.java` | `platform/lang-impl/src/.../codeStyle/` |
| `SchemeManagerImpl.kt` | `platform/configuration-store-impl/src/schemeManager/` |
| `CodeStyleSettings.java` | `platform/code-style-api/src/.../codeStyle/` |
| `CodeStyleSchemesModel.java` | `platform/lang-impl/src/.../codeStyle/` |
| `CodeStyleSchemesConfigurable.java` | `platform/lang-impl/src/.../codeStyle/` |
| `StoredOptionsContainer.java` | `platform/code-style-api/src/.../codeStyle/` |
| `DifferenceFilter.java` | `platform/util/src/.../util/` |
| `DefaultJDOMExternalizer.java` | `platform/util/src/.../util/` |

---

## 详细执行链路分析

### Phase 1: Apply 触发

**`CodeStyleSchemesConfigurable.java:99`**
```java
public void apply() throws ConfigurationException {
    super.apply();
    myModel.apply();  // ← 入口
}
```

### Phase 2: 克隆写回 Scheme — commitClonedSettings()

**`CodeStyleSchemesModel.java`** → `commitClonedSettings()`:
```java
private void commitClonedSettings() {
    for (CodeStyleScheme scheme : mySettingsToClone.keySet()) {
        if (!(scheme instanceof ProjectScheme)) {
            CodeStyleSettings settings = scheme.getCodeStyleSettings();  // ← 原 scheme 的 settings
            settings.copyFrom(mySettingsToClone.get(scheme));            // ← 从 clone 复制值（此时=默认值）
            settings.getModificationTracker().incModificationCount();    // ← 递增计数器 N→N+1，标记已修改
        }
    }
}
```

> **关键**: `copyFrom()` 使用 `CommonCodeStyleSettings.copyPublicFields(from, this)` 通过反射复制所有 `public` 字段的值，但**不复制** `private final` 的 `myStoredOptions`。

### Phase 3: 保存判定 — getSchemeState()

**`CodeStyleSchemeImpl.java:91`**:
```java
public @Nullable SchemeState getSchemeState() {
    synchronized (lock) {
        if (myDataHolder == null) {
            long curr = myCodeStyleSettings.getModificationTracker().getModificationCount(); // N+1
            if (myLastModificationCount != curr) {   // N != N+1 → true
                myLastModificationCount = curr;
                return SchemeState.POSSIBLY_CHANGED;  // ← 标记已修改
            }
        }
        return SchemeState.UNCHANGED;
    }
}
```

### Phase 4: 收集待保存 Scheme

**`SchemeManagerImpl.kt:95`** — `saveImpl()`:
```kotlin
for (scheme in schemes) {
    val state = (scheme as? SerializableScheme)?.schemeState  // POSSIBLY_CHANGED
    if (state != SchemeState.UNCHANGED) {
        changedSchemes.add(scheme as MUTABLE_SCHEME)
    }
}
writeWithEnsureWritable(project, changedSchemes,
    { scheme -> saveScheme(scheme, ...) },
    { scheme, e -> errorCollector.addError(...) })
```

### Phase 5: saveScheme() — BUG 触发点

**`SchemeManagerImpl.kt:134`**:
```kotlin
private fun saveScheme(scheme: MUTABLE_SCHEME, ...) {
    var externalInfo = schemeListManager.getExternalInfo(scheme)
    val element = processor.writeScheme(scheme)?.let {
        it as? Element ?: (it as Document).detachRootElement()
    }
    // ★★★ element = <code_scheme name="Default" /> — 只有属性，无子节点 ★★★

    if (element == null || element.isEmpty) {  // ← JDOM isEmpty() 不含属性检查！
        externalInfo?.scheduleDelete(filesToDelete, "empty")  // 标记删除，不写文件
        return
    }
    // ↓↓↓ 以下保存逻辑永远不执行 ↓↓↓
    val newDigest = hashElement(element)
    // ...
}
```

### Phase 6: writeScheme() 产生空 Element

**`CodeStyleSchemeImpl.java:124-141`**:
```java
public @NotNull Element writeScheme() {
    // ...
    if (dataHolder == null) {
        Element newElement = new Element(CODE_STYLE_TAG_NAME);
        newElement.setAttribute(CODE_STYLE_NAME_ATTR, getName());
        myCodeStyleSettings.writeExternal(newElement);  // ← DiffFilter 过滤掉所有默认值，无子节点
        return newElement;  // = <code_scheme name="Default" />
    }
    // ...
}
```

### Phase 7: writeExternal() 的 DiffFilter 过滤机制（核心原因）

**`CodeStyleSettings.java:612-619`**:
```java
public void writeExternal(Element element) throws WriteExternalException {
    setVersion(element, myVersion);
    CodeStyleSettings defaultSettings = new CodeStyleSettings(true, false);  // ← 全新全默认实例
    DefaultJDOMExternalizer.write(this, element,
        myStoredOptions.createDiffFilter(this, defaultSettings));  // ← 传入 DiffFilter
    // ...
}
```

#### 7.1 DefaultJDOMExternalizer.write() 逻辑

```java
// DefaultJDOMExternalizer.java
public static void write(Object data, Element parentNode,
                         Predicate<? super Field> filter) {
    for (Field field : fieldCache.get(data.getClass()).values()) {
        if (filter != null && !filter.test(field)) {
            continue;  // ← filter 返回 false 就跳过该字段
        }
        // 否则写入 <option name="字段名" value="值"/>
        Element element = new Element("option");
        parentNode.addContent(element);
        element.setAttribute("name", field.getName());
        element.setAttribute("value", value);
    }
}
```

遍历 `CodeStyleSettings` 的所有 `public` 字段，对**每个字段**调用 `filter.test(field)`。

#### 7.2 StoredOptionsDifferenceFilter 的双重判断

```java
// StoredOptionsContainer.java
private class StoredOptionsDifferenceFilter extends DifferenceFilter<CodeStyleSettings> {
    @Override
    public boolean test(@NotNull Field field) {
        return myOptionSet.contains(field.getName())  // ① 该字段之前被记录过？
            || super.test(field);                     // ② 与默认值不同？
    }
}
```

#### 7.3 DifferenceFilter 的核心比较

```java
// DifferenceFilter.java
public boolean test(@NotNull Field field) {
    Object thisValue = field.get(myThisSettings);       // 当前值（比如 INDENT_SIZE=4）
    Object parentValue = field.get(myParentSettings);   // 全新默认值（INDENT_SIZE=4）
    return !Comparing.equal(thisValue, parentValue);     // 相等? → false → 过滤掉
}
```

#### 7.4 myOptionSet 的生命周期 — 问题的根源

```java
// StoredOptionsContainer.java
class StoredOptionsContainer {
    private final Set<String> myOptionSet = new HashSet<>();

    void processOptions(@NotNull Element element) {
        // 解析 XML 中已有的 <option name="...">，存入 myOptionSet
        element.getChildren().forEach(child -> {
            Attribute childAttribute = child.getAttribute("name");
            if (childAttribute != null) {
                myOptionSet.add(childAttribute.getValue());
            }
        });
    }
}
```

| 时机 | 调用方 | 对 myOptionSet 的影响 |
|------|--------|----------------------|
| IDE 启动加载 Default.xml | `readExternal()` → `processOptions(element)` | 记录 XML 中已有的 option 名 |
| 用户 Apply（保存） | `writeExternal()` | **不调 processOptions，不更新 myOptionSet** |
| `copyFrom()`（commitClonedSettings） | `CodeStyleSettings.copyFrom()` | 只用反射复制 public 字段，不碰 myOptionSet |

> **`myOptionSet` 在 IDE 启动时初始化一次后就再也不会被更新！**

---

## 完整场景复现

### 场景设定

假设 Default.xml 初始状态为空：
```xml
<code_scheme name="Default" />
```

用户修改 INDENT_SIZE（默认值 4 → 8）。

### 第一次 Apply（INDENT_SIZE = 8，非默认值）

```
commitClonedSettings() 复制值: INDENT_SIZE = 8
  ↓
writeExternal():
  ↓
DefaultJDOMExternalizer 遍历字段，对 INDENT_SIZE 字段:
  → StoredOptionsDifferenceFilter.test("INDENT_SIZE"):
      → myOptionSet.contains("INDENT_SIZE") = false ← 原 XML 为空，未记录
      → super.test(): !equal(8, 4) = true            ← 8 ≠ 4，接受
  → 写入 <option name="INDENT_SIZE" value="8"/> ✅
  ↓
其他字段同理（只写非默认值）
  ↓
element 有子节点 → isEmpty() = false → 文件写入成功 ✅
```

保存后的 Default.xml:
```xml
<code_scheme name="Default">
  <option name="INDENT_SIZE" value="8"/>
  <!-- 其他非默认值... -->
</code_scheme>
```

### 第二次 Apply（INDENT_SIZE = 4，恢复为默认值）

```
commitClonedSettings() 复制值: INDENT_SIZE = 4（默认）
  ↓
writeExternal():
  ↓
DefaultJDOMExternalizer 遍历字段，对 INDENT_SIZE 字段:
  → StoredOptionsDifferenceFilter.test("INDENT_SIZE"):
      → myOptionSet.contains("INDENT_SIZE") = false ← 仍为空！writeExternal 不更新它！
      → super.test(): !equal(4, 4) = false           ← 4 = 4，拒绝 ❌
  → 不写入
  ↓
其他所有字段同理（如果也都=默认值）
  ↓
全部被过滤，element 无子节点
  ↓
element = <code_scheme name="Default" />  ← 仅有属性
```

### Phase 8: JDOM isEmpty() 的陷阱

```java
// org.jdom2.Element.java
public boolean isEmpty() {
    return content.isEmpty();  // content 只包含子元素/文本/注释 — 不包含属性！
}
```

`<code_scheme name="Default" />` 有属性但 content 为空 → **`isEmpty() == true`**。

### Phase 9: scheduleDelete — 既无更新，删除也失败

```kotlin
// SchemeManagerImpl.kt
if (element == null || element.isEmpty) {  // true！
    externalInfo?.scheduleDelete(filesToDelete, "empty")  // 标记删除
    return
}
```

```kotlin
fun scheduleDelete(filesToDelete: MutableSet<String>, reason: String) {
    if (fileNameWithoutExtension != null) {
        filesToDelete.add(fileNameWithoutExtension)
    }
}
```

删除存在多个故障点：

| # | 故障点 | 条件 | 后果 |
|---|--------|------|------|
| ① | `externalInfo == null` | Scheme 无外部文件信息 | `scheduleDelete` 不会被调用 |
| ② | `fileNameWithoutExtension == null` | 外部信息未初始化 | 文件不加入删除列表 |
| ③ | `deleteFiles` 执行失败 | 文件锁定/权限/VFS 未刷新 | 文件留在磁盘 |
| ④ | `scheduleDelete` 只在内存列表标记，依赖后续阶段实际删除 | 删除阶段未正确执行 | 文件残留 |

### Phase 10: 重启后 — 旧文件残留

```
旧 Default.xml 未被删除
    ↓
IDE 重启 → loadSchemes() → 扫描 codestyles/
    ↓
Default.xml 仍在 → 读取旧的 <option name="INDENT_SIZE" value="8"/>
    ↓
默认值不生效 ❌
```

---

## 根因总结

| 原因 | 说明 |
|------|------|
| **直接原因** | `saveScheme()` 中 `element.isEmpty()` 误判无子节点的 `<code_scheme name="Default" />` 为 empty |
| **深层原因 1** | JDOM 的 `isEmpty()` 只检查 `content`（子元素/文本），**不检查属性** |
| **深层原因 2** | `StoredOptionsContainer.myOptionSet` **只在 IDE 启动时 `readExternal()` → `processOptions()` 中初始化**，`writeExternal()` 不更新它。因此第二次 Apply 时 `myOptionSet` 为空，`StoredOptionsDifferenceFilter` 只能靠 `DifferenceFilter` 比较值与默认值是否相等；所有值=默认 → 全部过滤，不写任何子节点 |
| **触发条件** | `writeExternal()` 用 DiffFilter 过滤默认值，所有值匹配默认 → 不写任何子节点 |
| **最终后果** | `scheduleDelete` 而非写入，删除不可靠导致旧文件残留 |

### 完整触发链

```
第二次 Apply（恢复默认值）
  ↓
commitClonedSettings()  → copyFrom() 值变回默认, incModificationCount()
  ↓
getSchemeState()        → POSSIBLY_CHANGED（修改计数变化）
  ↓
writeScheme()
  ↓
writeExternal()
  ↓
DefaultJDOMExternalizer.write(this, element, myStoredOptions.createDiffFilter(this, defaultSettings))
  ↓
对于每个字段: StoredOptionsDifferenceFilter.test(field):
  myOptionSet.contains(name)?   ← false（未更新，始终为空）
  !equal(currVal, defaultVal)?  ← false（值=默认，与全新实例相等）
  ↓
所有字段过滤 → element 无子节点 → <code_scheme name="Default" />
  ↓
saveScheme() 中: element.isEmpty() == true（JDOM 不检查属性）
  ↓
scheduleDelete("empty")  → 标记删除，不写文件
  ↓
deleteFiles()  → ❌ 删除失败（锁/权限/VFS）
  ↓
Default.xml 残留旧值 → 重启后默认值不生效 ❌
```

---

## 修复方案

### 方案 1：修复 `CodeStyleSchemeImpl.writeScheme()`（推荐，改动最小）

```java
public @NotNull Element writeScheme() {
    Element newElement = new Element(CODE_STYLE_TAG_NAME);
    newElement.setAttribute(CODE_STYLE_NAME_ATTR, getName());
    myCodeStyleSettings.writeExternal(newElement);
    // 兜底：如果所有值都是默认导致无子节点，写入一个占位
    if (newElement.isEmpty()) {
        newElement.addContent(new Element("dummy").setText(""));
    }
    return newElement;
}
```

**优点**：改动极小，仅影响 CodeStyle 保存路径。

### 方案 2：修复 `SchemeManagerImpl.saveScheme()`

```kotlin
if (element == null || element.isEmpty) {
    // 放弃删除，改为写入一个仅有根元素的有效文件
    val root = Element("code_scheme").setAttribute("name",
        processor.getSchemeKey(scheme))
    Files.newBufferedWriter(ioDirectory.resolve(
        "$fileNameWithoutExtension$schemeExtension")).use {
        XMLOutputter().output(root, it)
    }
    return
}
```

**优点**：从根本上修复了 `isEmpty()` 误判导致删除而非写入的问题。

### 方案 3：修复 `StoredOptionsContainer`（最彻底）

在 `writeExternal()` 完成后更新 `myOptionSet`：

```java
// CodeStyleSettings.writeExternal() 末尾追加
myStoredOptions.processOptions(element);
```

这样第二次 Apply 时 `myOptionSet` 中已有字段名，`StoredOptionsDifferenceFilter` 即使值=默认也会接受该字段，写入默认值。

**优点**：保留 DiffFilter 机制，写出的 XML 包含明确等于默认值的 option，避免 isEmpty 误判。
**缺点**：Default.xml 会包含大量等于默认值的冗余 `<option>` 节点。

---
