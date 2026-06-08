# CodeStyle 恢复默认值时 Default.xml 未更新 Bug 分析报告

## 概述

**Bug 现象**：在 `Settings → Editor → Code Style → Java → Tabs and Indents` 中：
1. 先修改为非默认值 → Apply → Default.xml 正确更新
2. 再将值恢复为默认值 → Apply → Default.xml **未更新**
3. 重启 IDE 后，旧的非默认值仍然生效，默认值不生效

---

## 环境信息

- 项目：IntelliJ IDEA Community Edition
- 涉及模块：
  - `platform/code-style-api`
  - `platform/lang-impl`
  - `platform/configuration-store-impl`
  - `java/java-frontback-impl`

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

### Phase 2: 克隆写回 Scheme

**`CodeStyleSchemesModel.java:239`** → `commitClonedSettings()`:
```java
private void commitClonedSettings() {
    for (CodeStyleScheme scheme : mySettingsToClone.keySet()) {
        if (!(scheme instanceof ProjectScheme)) {
            CodeStyleSettings settings = scheme.getCodeStyleSettings();
            settings.copyFrom(mySettingsToClone.get(scheme));           // ① 覆盖为默认值
            settings.getModificationTracker().incModificationCount();   // ② 递增计数器 N→N+1
        }
    }
}
```

### Phase 3: 保存判定

**`CodeStyleSchemeImpl.java:91`** — `getSchemeState()`:
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

### Phase 5: `saveScheme()` — **BUG 触发点**

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
    if (externalInfo != null && externalInfo.isDigestEquals(newDigest)) { return }
    else if (processor is LazySchemeProcessor && processor.isSchemeDefault(scheme, newDigest)) {
        externalInfo?.scheduleDelete(filesToDelete, "equals to default")
        return
    }
}
```

### Phase 6: `writeScheme()` 产生空 Element

**`CodeStyleSchemeImpl.java:107-109`**:
```java
public @NotNull Element writeScheme() {
    Element newElement = new Element(CODE_STYLE_TAG_NAME);
    newElement.setAttribute(CODE_STYLE_NAME_ATTR, getName());
    myCodeStyleSettings.writeExternal(newElement);  // ← DiffFilter 过滤掉所有默认值，无子节点
    return newElement;  // = <code_scheme name="Default" />
}
```

### Phase 7: JDOM `isEmpty()` 的陷阱

```java
// org.jdom2.Element.java
public boolean isEmpty() {
    return content.isEmpty();  // content 只包含子元素/文本/注释 — 不包含属性！
}
```

`<code_scheme name="Default" />` 有属性但 content 为空 → **`isEmpty() == true`**。

### Phase 8: `scheduleDelete` — 删除不可靠

```kotlin
fun scheduleDelete(filesToDelete: MutableSet<String>, reason: String) {
    if (fileNameWithoutExtension != null) {
        filesToDelete.add(fileNameWithoutExtension)
    }
}
```

删除存在 3 个故障点：

| # | 故障点 | 条件 | 后果 |
|---|--------|------|------|
| ① | `externalInfo == null` | Scheme 无外部文件信息 | `scheduleDelete` 空调用 |
| ② | `fileNameWithoutExtension == null` | 外部信息未初始化 | 文件不加入删除列表 |
| ③ | `deleteFiles` 失败 | 文件锁定/权限/VFS 未刷新 | 文件留在磁盘 |

### Phase 9: 重启后 — 旧文件残留

```
旧 Default.xml 未被删除
    ↓
IDE 重启 → loadSchemes() → 扫描 codestyles/
    ↓
Default.xml 仍在 → 读取旧非默认值
    ↓
默认值不生效 ❌
```

---

## 根因总结

| 原因 | 说明 |
|------|------|
| **直接原因** | `saveScheme()` 中 `element.isEmpty()` 误判无子节点的 `<code_scheme name="Default" />` 为 empty |
| **深层原因** | JDOM 的 `isEmpty()` 只检查 content（子元素/文本），不检查属性 |
| **触发条件** | `writeExternal()` 用 DiffFilter 过滤默认值，所有值匹配 → 不写任何子节点 |
| **最终后果** | `scheduleDelete` 而非写入，删除不可靠导致旧文件残留 |

### 完整触发链

```
第二次 Apply
  ↓
commitClonedSettings()  → copyFrom() 值变回默认, incModificationCount()
  ↓
getSchemeState()        → POSSIBLY_CHANGED
  ↓
saveScheme()            → writeScheme() → element.isEmpty() == true
  ↓                          ↓
scheduleDelete("empty")  → 标记删除，不写文件
  ↓
deleteFiles()           → ❌ 删除失败（锁/权限/VFS）
  ↓
Default.xml 残留旧值 → 重启后默认值不生效
```

---

## 推荐修复方案

### 方案 1：修复 `CodeStyleSchemeImpl.writeScheme()`（推荐）

```java
public @NotNull Element writeScheme() {
    Element newElement = new Element(CODE_STYLE_TAG_NAME);
    newElement.setAttribute(CODE_STYLE_NAME_ATTR, getName());
    myCodeStyleSettings.writeExternal(newElement);
    if (newElement.isEmpty()) {
        newElement.addContent(new Element("dummy").setText(""));  // 保底填充
    }
    return newElement;
}
```

**优点**：改动最小，仅影响 CodeStyle 保存

### 方案 2：修复 `SchemeManagerImpl.saveScheme()`

```kotlin
if (element == null || element.isEmpty) {
    val root = Element("code_scheme").setAttribute("name", processor.getSchemeKey(scheme))
    Files.newBufferedWriter(ioDirectory.resolve("$fileNameWithoutExtension$schemeExtension")).use {
        XMLOutputter().output(root, it)
    }
    return
}
```

**优点**：从根本上修复 isEmpty 误判

---

## 涉及源文件

| 文件 | 路径 | 作用 |
|------|------|------|
| `CodeStyleSchemeImpl.java` | `platform/lang-impl/src/.../codeStyle/` | `writeScheme()` 产生空 element |
| `SchemeManagerImpl.kt` | `platform/configuration-store-impl/src/schemeManager/` | `saveScheme()` 中 `isEmpty` 误判 |
| `CodeStyleSettings.java` | `platform/code-style-api/src/.../codeStyle/` | `writeExternal()` 用 DiffFilter |
| `CodeStyleSchemesModel.java` | `platform/lang-impl/src/.../codeStyle/` | `commitClonedSettings()` 回写 |
| `CodeStyleSchemesConfigurable.java` | `platform/lang-impl/src/` | Apply 触发入口 |