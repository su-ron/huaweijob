# IntelliJ Platform 插件 Action 改造方案

## 问题场景

从 IntelliJ IDEA 2024.3.3 底座升级到 2026.1.1 底座后，插件中引用的部分 IDE 内部类/构造器不再可访问。

---

## 问题 1：CollectZippedLogs Action 位置移动

### 变更说明

2024.3.3 中 "Compress Logs and Show in Explorer" 在 2026.1.1 中已更名为 **Collect Zipped Logs**。

| 项目 | 2024.3.3 | 2026.1.1 |
|------|----------|----------|
| Action ID | `CompressLogs`（旧） | `CollectZippedLogs` |
| 实现类 | 旧类名 | `CollectZippedLogsAction.kt` |
| 菜单文本 | Compress Logs and Show in Explorer | Collect Zipped Logs |
| 位置 | `HelpMenu` 中 | `HelpMenu` 中（未变） |

Action 注册代码（`PlatformActions.xml:884`）：
```xml
<action id="ShowLog" class="com.intellij.ide.actions.ShowLogAction"/>
<action id="CollectZippedLogs" class="com.intellij.ide.actions.CollectZippedLogsAction"/>
```

### 外部插件移动方案

在 `plugin.xml` 中使用 `remove` + 重新 `add-to-group`：

```xml
<idea-plugin>
  <actions>
    <group id="HelpMenu">
      <remove id="CollectZippedLogs"/>
    </group>
    <action id="CollectZippedLogs"
            class="com.intellij.ide.actions.CollectZippedLogsAction">
      <add-to-group group-id="HelpMenu"
                    anchor="after"
                    relative-to-action="ShowLog"/>
    </action>
  </actions>
</idea-plugin>
```

---

## 问题 2：ShowMemoryDialogAction 改造

### 编译错误

```
错误: 无法将类 EditMemorySettingsDialog中的构造器 EditMemorySettingsDialog应用到给定类型
  new EditMemorySettingsDialog().show();
  原因: EditMemorySettingsDialog()在EditMemorySettingsDialog中不是公共的;
        无法从外部程序包中对其进行访问
```

### 根因

**`EditMemorySettingsDialog.java`**（`platform/platform-impl/src/.../diagnostic/`）：

```java
class EditMemorySettingsDialog extends DialogWrapper {  // ← 包级私有！无 public
    EditMemorySettingsDialog() {                        // ← 包级私有！无 public
        this(VMOptions.MemoryKind.HEAP, false);
    }
}
```

- 类本身是 **package-private**（无 `public` 修饰）
- 构造器也是 **package-private**
- 只有同包（`com.intellij.diagnostic`）的类才能调用
- 插件在 `com.huawei.deveco.projectmgmt.ohos.actions` 包中，无法访问

原始的 `ShowMemoryDialogAction` 能调用是因为它在同一个包下：
```java
// com.intellij.diagnostic.ShowMemoryDialogAction.java
@ApiStatus.Internal
public final class ShowMemoryDialogAction extends AnAction implements DumbAware {
    @Override
    public void actionPerformed(@NotNull AnActionEvent e) {
        new EditMemorySettingsDialog().show();  // 同包，可访问
    }
}
```

### 修复方案

#### 方案 A（推荐）：Wrapper 类 + ActionManager 委派

```java
package com.huawei.deveco.projectmgmt.ohos.actions;

import com.intellij.openapi.actionSystem.*;

public class OhosShowMemoryDialogAction extends AnAction implements DumbAware {

    @Override
    public void actionPerformed(@NotNull AnActionEvent e) {
        // ① 委派给系统 action 执行实际逻辑
        AnAction action = ActionManager.getInstance()
            .getAction("performancePlugin.ShowMemoryDialogAction");
        if (action != null) {
            action.actionPerformed(e);
        }
    }

    @Override
    public void update(@NotNull AnActionEvent e) {
        // ② 先让系统 action 判断 enabled/visible
        AnAction action = ActionManager.getInstance()
            .getAction("performancePlugin.ShowMemoryDialogAction");
        if (action != null) {
            action.update(e);
        } else {
            e.getPresentation().setEnabledAndVisible(false);
        }

        // ③ 再设自定义文本（覆盖系统默认文本）
        e.getPresentation().setText(
            ProjectMgmtBundle.message("action.ShowMemoryDialogActionAction.text")
        );
    }

    @Override
    public @NotNull ActionUpdateThread getActionUpdateThread() {
        return ActionUpdateThread.BGT;
    }
}
```

**关键技巧**：先委托 `action.update(e)` 获取正确的 `enabledAndVisible` 状态，再 `setText(...)` 覆盖为你自己的文本。

---

#### 方案 B：在 plugin.xml 中引用 Wrapper 类并定位

Wrapper 类的注册和定位语法与普通 action **完全一致**：

```xml
<idea-plugin>
  <actions>
    <action id="ohos.ShowMemoryDialog"
            class="com.huawei.deveco.projectmgmt.ohos.actions.OhosShowMemoryDialogAction"
            text="Show Memory Dialog"
            icon="/icons/memory.svg">
      <!-- ★ 标准定位语法，和普通 action 完全一样 -->
      <add-to-group group-id="HelpMenu"
                    anchor="after"
                    relative-to-action="ShowLog"/>
    </action>

    <!-- 也可以把 CollectZippedLogs 也一并移过来 -->
    <action id="ohos.CollectZippedLogs"
            class="com.intellij.ide.actions.CollectZippedLogsAction"
            text="Collect Logs">
      <add-to-group group-id="HelpMenu"
                    anchor="after"
                    relative-to-action="ohos.ShowMemoryDialog"/>
    </action>
  </actions>
</idea-plugin>
```

**注意**：XML 的 `text` 属性会被 Wrapper 的 `update()` 中 `setText(...)` 覆盖，所以最终的文本以代码为准。

---

## 方案对比总结

| 方案 | 文本控制 | enabled/visible 逻辑 | 内部 API 依赖 | 推荐度 |
|------|---------|---------------------|--------------|--------|
| A: Wrapper + ActionManager | ✅ 完全控制 | ✅ 复用系统逻辑 | ✅ 只通过 `ActionManager` | ⭐⭐⭐⭐⭐ |
| B: XML 直接引用系统类 | ✅ 可设 text | ✅ 复用系统 update | ⚠️ `@ApiStatus.Internal` | ⭐⭐⭐ |

---

## 完整 plugin.xml 示例

```xml
<idea-plugin>
  <actions>
    <!-- ShowLog 保持原位 -->

    <!-- Wrapper：自定义文本 + 定位到 ShowLog 下方 -->
    <action id="ohos.ShowMemoryDialog"
            class="com.huawei.deveco.projectmgmt.ohos.actions.OhosShowMemoryDialogAction">
      <add-to-group group-id="HelpMenu"
                    anchor="after"
                    relative-to-action="ShowLog"/>
    </action>

    <!-- CollectZippedLogs 定位到 ShowMemoryDialog 下方 -->
    <action id="ohos.CollectZippedLogs"
            class="com.intellij.ide.actions.CollectZippedLogsAction">
      <add-to-group group-id="HelpMenu"
                    anchor="after"
                    relative-to-action="ohos.ShowMemoryDialog"/>
    </action>
  </actions>
</idea-plugin>
```

---

## 关键文件位置

| 文件 | 路径 |
|------|------|
| `PlatformActions.xml` | `platform/platform-resources/src/idea/` |
| `ShowMemoryDialogAction.java` | `platform/platform-impl/src/.../diagnostic/` |
| `EditMemorySettingsDialog.java` | `platform/platform-impl/src/.../diagnostic/` |
| `CollectZippedLogsAction.kt` | `platform/platform-impl/src/.../ide/actions/` |
| `ShowLogAction.java` | `platform/platform-impl/src/.../ide/actions/` |