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
    <!-- 从原位置移除 -->
    <group id="HelpMenu">
      <remove id="CollectZippedLogs"/>
    </group>

    <!-- 注册到 ShowLog 下方 -->
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

#### 方案 A（推荐）：通过 ActionManager 委派调用

不直接实例化 `EditMemorySettingsDialog`，而是调用系统已有的 action：

```java
package com.huawei.deveco.projectmgmt.ohos.actions;

import com.intellij.openapi.actionSystem.*;

public class OhosShowMemoryDialogAction extends AnAction implements DumbAware {

    @Override
    public void actionPerformed(@NotNull AnActionEvent e) {
        AnAction action = ActionManager.getInstance()
            .getAction("performancePlugin.ShowMemoryDialogAction");
        if (action != null) {
            action.actionPerformed(e);
        }
    }

    @Override
    public void update(@NotNull AnActionEvent e) {
        AnAction action = ActionManager.getInstance()
            .getAction("performancePlugin.ShowMemoryDialogAction");
        if (action != null) {
            action.update(e);
        } else {
            e.getPresentation().setEnabledAndVisible(false);
        }
    }

    @Override
    public @NotNull ActionUpdateThread getActionUpdateThread() {
        return ActionUpdateThread.BGT;
    }
}
```

#### 方案 B：直接在 plugin.xml 中引用系统 Action 类

如果只是改变位置和图标，不需要自己写类：

```xml
<action id="ohos.ShowMemoryDialog"
        class="com.intellij.diagnostic.ShowMemoryDialogAction"
        text="Show Memory Dialog"
        icon="/icons/myIcon.svg">
  <add-to-group group-id="YourGroupId" anchor="first"/>
</action>
```

#### 方案 C（不推荐）：反射调用

```java
Class<?> clazz = Class.forName("com.intellij.diagnostic.EditMemorySettingsDialog");
Object dialog = clazz.getDeclaredConstructor().newInstance();
clazz.getMethod("show").invoke(dialog);
```

**缺点**：脆弱、不类型安全、可能在 future 版本中无声崩溃。

---

## 方案对比总结

| 方案 | 代码量 | 稳定性 | 推荐度 |
|------|--------|--------|--------|
| A: ActionManager 委派 | 约 20 行 | 高（使用公开 API） | ⭐⭐⭐⭐⭐ |
| B: XML 直接引用 | 0 行 Java | 高 | ⭐⭐⭐⭐⭐ |
| C: 反射调用 | 3 行 | 低（无编译检查） | ⭐ |

---

## 完整 plugin.xml 示例

将两个 action 一起移动到 `ShowLog` 下方：

```xml
<idea-plugin>
  <actions>
    <!-- ShowMemoryDialog → ShowLog 下方 -->
    <group id="HelpMenu">
      <remove id="performancePlugin.ShowMemoryDialogAction"/>
    </group>
    <action id="performancePlugin.ShowMemoryDialogAction"
            class="com.intellij.diagnostic.ShowMemoryDialogAction">
      <add-to-group group-id="HelpMenu"
                    anchor="after"
                    relative-to-action="ShowLog"/>
    </action>

    <!-- CollectZippedLogs → ShowMemoryDialog 下方 -->
    <group id="HelpMenu">
      <remove id="CollectZippedLogs"/>
    </group>
    <action id="CollectZippedLogs"
            class="com.intellij.ide.actions.CollectZippedLogsAction">
      <add-to-group group-id="HelpMenu"
                    anchor="after"
                    relative-to-action="performancePlugin.ShowMemoryDialogAction"/>
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