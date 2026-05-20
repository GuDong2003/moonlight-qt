# Drag-Drop Background Image — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** 让用户在 Moonlight 主界面通过拖入图片或右键 Popup 替换全局背景，并能独立调节浏览页/设置页的遮罩透明度。

**Architecture:** 全局背景图 + 全局遮罩挂在 main.qml 根 `ApplicationWindow`，遮罩 opacity 通过 `stackView.currentItem.isSettingsView` 切换；图片落盘到 `Path::getBackgroundsDir()`，hash 命名；右键弹 Popup 集中所有控制。

**Tech Stack:** Qt 5 / QtQuick / QtQuick.Controls 2 / QSettings / Qt FileDialog

**Source of truth:** `docs/plans/2026-05-20-moonlight-drag-drop-background-design.md`

**Workspace:** 所有改动必须在 worktree `/Volumes/Samsung980PRO/CODE/moonlight-qt/.worktrees/drag-drop-background` 上，分支 `feature/drag-drop-background`。

**Design 修正项（相对 design doc）：**
- `objectName` 在现有视图已被 `qsTr("Computers"/"Settings")` 占用作无障碍标签 → 不能复用作页面分类
- 改为给每个根视图新增 `property bool isSettingsView: false`，SettingsView 设 true，main.qml 据此切换遮罩

**测试策略：** Qt 项目本身无单元测试框架，本计划采用 **"编译 + 手动验证"** 替代 TDD：每个 task 后跑 `qmake -r && make -j` 确保编译通过，并按"手动验证"小节启动 app 确认行为。

---

## Task 0：环境准备

**Files:** （无改动，纯验证）

**Step 1:** 进入 worktree
```bash
cd /Volumes/Samsung980PRO/CODE/moonlight-qt/.worktrees/drag-drop-background
git status
```
Expected: `On branch feature/drag-drop-background`，工作区干净。

**Step 2:** 验证 qmake / make 工具可用
```bash
which qmake && qmake -v
```
Expected: 输出 Qt 5.x 的 qmake 路径与版本（项目要求 Qt 5.9+）。

**Step 3:** 第一次 baseline 编译（仅做一次，后续 task 跑增量）
```bash
qmake -r && make -j$(sysctl -n hw.ncpu) 2>&1 | tail -5
```
Expected: 无错误，产物 `app/Moonlight.app/Contents/MacOS/Moonlight` 存在。

**Step 4:** 启动 app 确认 baseline 行为
```bash
open app/Moonlight.app
```
Expected: 出现 PC 列表主界面，背景为深灰（Material #303030）。关闭。

不 commit（仅环境准备）。

---

## Task 1：Path 加 `getBackgroundsDir()` 与 `deleteDataFile()`

**Files:**
- Modify: `app/path.h`
- Modify: `app/path.cpp`

**Step 1:** 在 `app/path.h` `class Path` 公开区追加：
```cpp
    static QString getBackgroundsDir();
    static void deleteDataFile(QString fileName);
```
（放在 `getDataFilePath` 行下方）

**Step 2:** 在 `app/path.cpp` 末尾追加实现：
```cpp
QString Path::getBackgroundsDir()
{
    QString dir = QDir(s_CacheDir).absoluteFilePath("backgrounds");
    QDir().mkpath(dir);
    return dir;
}

void Path::deleteDataFile(QString fileName)
{
    QFile dataFile(getDataFilePath(fileName));
    if (dataFile.exists()) {
        dataFile.remove();
    }
}
```

**Step 3:** 增量编译
```bash
make -j$(sysctl -n hw.ncpu) 2>&1 | tail -5
```
Expected: 编译通过，无 warning/error 涉及 path.cpp。

**Step 4:** Commit
```bash
git add app/path.h app/path.cpp
git commit -m "feat(path): add backgrounds dir helper and deleteDataFile"
```

---

## Task 2：StreamingPreferences 加 3 个 Q_PROPERTY 与字段

**Files:**
- Modify: `app/settings/streamingpreferences.h`
- Modify: `app/settings/streamingpreferences.cpp`

**Step 1:** 在 `streamingpreferences.h` 中其它 `Q_PROPERTY` 区（约第 130 行附近，紧跟 `richPresence` 等同类条目）追加：
```cpp
    Q_PROPERTY(QString backgroundImagePath MEMBER backgroundImagePath NOTIFY backgroundImageChanged)
    Q_PROPERTY(double backgroundOverlayMain MEMBER backgroundOverlayMain NOTIFY backgroundOverlayChanged)
    Q_PROPERTY(double backgroundOverlaySettings MEMBER backgroundOverlaySettings NOTIFY backgroundOverlayChanged)
```

**Step 2:** 在成员变量区（与 `richPresence` 同类布尔/字符串字段附近）追加：
```cpp
    QString backgroundImagePath;
    double backgroundOverlayMain = 0.35;
    double backgroundOverlaySettings = 0.65;
```

**Step 3:** 在 `signals:` 段追加：
```cpp
    void backgroundImageChanged();
    void backgroundOverlayChanged();
```

**Step 4:** 在 `streamingpreferences.cpp` 顶部 `#define SER_*` 区追加：
```cpp
#define SER_BGIMAGE "backgroundimage"
#define SER_BGOVERLAYMAIN "bgoverlaymain"
#define SER_BGOVERLAYSETTINGS "bgoverlaysettings"
```

**Step 5:** 在 `reload()` 函数中（与其它 `settings.value(...)` 同样位置）追加：
```cpp
    backgroundImagePath = settings.value(SER_BGIMAGE, "").toString();
    backgroundOverlayMain = settings.value(SER_BGOVERLAYMAIN, 0.35).toDouble();
    backgroundOverlaySettings = settings.value(SER_BGOVERLAYSETTINGS, 0.65).toDouble();
```

**Step 6:** 在 `save()` 函数中追加：
```cpp
    settings.setValue(SER_BGIMAGE, backgroundImagePath);
    settings.setValue(SER_BGOVERLAYMAIN, backgroundOverlayMain);
    settings.setValue(SER_BGOVERLAYSETTINGS, backgroundOverlaySettings);
```

**Step 7:** 因为 Q_PROPERTY 触发 moc，需要重新跑 qmake
```bash
qmake -r && make -j$(sysctl -n hw.ncpu) 2>&1 | tail -5
```
Expected: 编译通过。如果 moc 报 "Cannot find class with id" 重新 `make clean && qmake -r && make -j`。

**Step 8:** Commit
```bash
git add app/settings/streamingpreferences.h app/settings/streamingpreferences.cpp
git commit -m "feat(prefs): add background image path & per-view overlay opacity props"
```

---

## Task 3：StreamingPreferences 加 `setBackgroundImage` / `clearBackgroundImage`

**Files:**
- Modify: `app/settings/streamingpreferences.h`
- Modify: `app/settings/streamingpreferences.cpp`

**Step 1:** 在 `streamingpreferences.h` `Q_INVOKABLE` 区（紧跟 `Q_INVOKABLE void save();`）追加：
```cpp
    Q_INVOKABLE void setBackgroundImage(const QUrl& sourceUrl);
    Q_INVOKABLE void clearBackgroundImage();
```

**Step 2:** 在 `streamingpreferences.cpp` 顶部 include 区追加：
```cpp
#include <QUrl>
#include <QFile>
#include <QFileInfo>
#include <QDir>
#include <QCryptographicHash>
#include "../path.h"
```
（部分头可能已存在，去重）

**Step 3:** 在 `streamingpreferences.cpp` `save()` 函数下方追加实现：
```cpp
void StreamingPreferences::setBackgroundImage(const QUrl& sourceUrl)
{
    QString sourcePath = sourceUrl.isLocalFile() ? sourceUrl.toLocalFile() : sourceUrl.toString();
    QFile sourceFile(sourcePath);
    if (!sourceFile.open(QIODevice::ReadOnly)) {
        qWarning() << "Failed to open background image:" << sourcePath;
        return;
    }

    // Hash first 1 MB for fast dedup
    QCryptographicHash hasher(QCryptographicHash::Sha1);
    hasher.addData(sourceFile.read(1024 * 1024));
    sourceFile.close();
    QString hashHex = QString(hasher.result().toHex()).left(12);
    QString ext = QFileInfo(sourcePath).suffix().toLower();
    if (ext.isEmpty()) ext = "img";
    QString destName = hashHex + "." + ext;
    QString destPath = QDir(Path::getBackgroundsDir()).absoluteFilePath(destName);

    if (!QFile::exists(destPath)) {
        if (!QFile::copy(sourcePath, destPath)) {
            qWarning() << "Failed to copy background image to" << destPath;
            return;
        }
    }

    // Clean up previous copy (if any and different)
    if (!backgroundImagePath.isEmpty() && backgroundImagePath != destPath) {
        QFile::remove(backgroundImagePath);
    }

    backgroundImagePath = destPath;
    emit backgroundImageChanged();
    save();
}

void StreamingPreferences::clearBackgroundImage()
{
    if (backgroundImagePath.isEmpty()) return;
    QFile::remove(backgroundImagePath);
    backgroundImagePath.clear();
    emit backgroundImageChanged();
    save();
}
```

**Step 4:** 编译
```bash
make -j$(sysctl -n hw.ncpu) 2>&1 | tail -5
```
Expected: 编译通过。

**Step 5:** Commit
```bash
git add app/settings/streamingpreferences.h app/settings/streamingpreferences.cpp
git commit -m "feat(prefs): add setBackgroundImage/clearBackgroundImage Q_INVOKABLE"
```

---

## Task 4：给视图加 `isSettingsView` 标记

**Files:**
- Modify: `app/gui/PcView.qml`
- Modify: `app/gui/AppView.qml`
- Modify: `app/gui/SettingsView.qml`

**Step 1:** `PcView.qml` 在 `CenteredGridView { ... }` 内紧跟 `id: pcGrid` 行追加：
```qml
    property bool isSettingsView: false
```

**Step 2:** `AppView.qml` 在 `id: appGrid` 行下追加：
```qml
    property bool isSettingsView: false
```

**Step 3:** `SettingsView.qml` 在 `id: settingsPage` 行下追加：
```qml
    property bool isSettingsView: true
```

**Step 4:** 重新生成 QML 资源并编译
```bash
qmake -r && make -j$(sysctl -n hw.ncpu) 2>&1 | tail -5
```
Expected: 编译通过。

**Step 5:** Commit
```bash
git add app/gui/PcView.qml app/gui/AppView.qml app/gui/SettingsView.qml
git commit -m "feat(gui): tag views with isSettingsView for overlay routing"
```

---

## Task 5：main.qml 加全局背景图 + 遮罩层

**Files:**
- Modify: `app/gui/main.qml`

**Step 1:** 在 `StackView { id: stackView ... }` 之前（即 `ApplicationWindow` 内 ToolTip 配置之后）插入：
```qml
    // ── Background layer ─────────────────────────────────────────
    Image {
        id: backgroundImage
        anchors.fill: parent
        source: StreamingPreferences.backgroundImagePath ? "file://" + StreamingPreferences.backgroundImagePath : ""
        fillMode: Image.PreserveAspectCrop
        asynchronous: true
        cache: true
        visible: source != ""
        z: -2

        onStatusChanged: {
            if (status === Image.Error) {
                StreamingPreferences.clearBackgroundImage()
            }
        }
    }

    Rectangle {
        id: backgroundOverlay
        anchors.fill: parent
        color: "black"
        visible: backgroundImage.visible
        opacity: {
            var item = stackView.currentItem
            if (item && item.isSettingsView === true) {
                return StreamingPreferences.backgroundOverlaySettings
            }
            return StreamingPreferences.backgroundOverlayMain
        }
        z: -1

        Behavior on opacity {
            NumberAnimation { duration: 200 }
        }
    }
```

**Step 2:** 把现有 `StackView { id: stackView; anchors.fill: parent; focus: true; ... }` 加上 `background: null`（保证 StackView 不绘制自己的灰底）。如果已是 null 可跳过。具体改动：
```qml
    StackView {
        id: stackView
        anchors.fill: parent
        focus: true
        background: null   // ← 加这行
```

**Step 3:** 编译
```bash
make -j$(sysctl -n hw.ncpu) 2>&1 | tail -5
```
Expected: 编译通过。

**Step 4:** 手动验证
```bash
open app/Moonlight.app
```
Expected: 由于此时 `backgroundImagePath` 为空，背景图与遮罩均隐藏，UI 与原版一致。关闭。

**Step 5:** Commit
```bash
git add app/gui/main.qml
git commit -m "feat(gui): add global background image and per-view overlay"
```

---

## Task 6：main.qml 加拖入图片 DropArea

**Files:**
- Modify: `app/gui/main.qml`

**Step 1:** 在 `StackView` 之后（与 StackView 同层）追加：
```qml
    DropArea {
        id: imageDropArea
        anchors.fill: parent
        z: 1000
        keys: ["text/uri-list"]

        property bool isValidImage: false

        function isImageUrl(url) {
            var s = url.toString().toLowerCase()
            return s.endsWith(".jpg") || s.endsWith(".jpeg") ||
                   s.endsWith(".png") || s.endsWith(".webp") || s.endsWith(".bmp")
        }

        onEntered: {
            isValidImage = false
            if (drag.hasUrls && drag.urls.length === 1 && isImageUrl(drag.urls[0])) {
                isValidImage = true
                drag.accept(Qt.CopyAction)
            } else {
                drag.accepted = false
            }
        }
        onExited: isValidImage = false
        onDropped: {
            if (isValidImage && drop.urls.length === 1) {
                StreamingPreferences.setBackgroundImage(drop.urls[0])
            }
            isValidImage = false
        }

        Rectangle {
            anchors.fill: parent
            color: "#88000000"
            visible: parent.isValidImage
            Text {
                anchors.centerIn: parent
                text: qsTr("Release to use as background")
                color: "white"
                font.pixelSize: 28
            }
        }
    }
```

**Step 2:** 编译
```bash
make -j$(sysctl -n hw.ncpu) 2>&1 | tail -5
```
Expected: 编译通过。

**Step 3:** 手动验证
```bash
open app/Moonlight.app
```
Expected: 
- 从 Finder 拖一张 .jpg 进主界面 → 看到 "Release to use as background" 覆盖层 → 释放 → 背景图出现 + 默认 35% 黑色遮罩
- 切到 SettingsView（齿轮按钮）→ 遮罩自动加深到 65%
- 重启 app → 背景仍在
- 关闭。

**Step 4:** Commit
```bash
git add app/gui/main.qml
git commit -m "feat(gui): accept dropped image files to set background"
```

---

## Task 7：main.qml 加右键 Popup 设置面板

**Files:**
- Modify: `app/gui/main.qml`

**Step 1:** 在 `imports` 顶部追加：
```qml
import QtQuick.Dialogs 1.3
```

**Step 2:** 在 `DropArea` 之后追加右键 MouseArea：
```qml
    MouseArea {
        id: rightClickArea
        anchors.fill: parent
        acceptedButtons: Qt.RightButton
        propagateComposedEvents: true
        z: 500
        onClicked: function(mouse) {
            bgSettingsPopup.x = mouse.x
            bgSettingsPopup.y = mouse.y
            // Clamp inside window
            if (bgSettingsPopup.x + bgSettingsPopup.width > window.width) {
                bgSettingsPopup.x = window.width - bgSettingsPopup.width - 8
            }
            if (bgSettingsPopup.y + bgSettingsPopup.height > window.height) {
                bgSettingsPopup.y = window.height - bgSettingsPopup.height - 8
            }
            bgSettingsPopup.open()
        }
    }
```

**Step 3:** 在文件 `ApplicationWindow` 末尾追加 Popup 和 FileDialog：
```qml
    Popup {
        id: bgSettingsPopup
        width: 320
        height: 240
        modal: false
        focus: true
        closePolicy: Popup.CloseOnEscape | Popup.CloseOnPressOutside

        background: Rectangle {
            color: "#dd1e1e1e"
            radius: 8
            border.color: "#444"
        }

        contentItem: ColumnLayout {
            spacing: 10

            Label {
                text: qsTr("Background Settings")
                color: "white"
                font.bold: true
                font.pixelSize: 16
            }

            RowLayout {
                spacing: 8
                Layout.fillWidth: true

                Rectangle {
                    width: 64; height: 36
                    color: "#333"
                    border.color: "#555"
                    Image {
                        anchors.fill: parent
                        anchors.margins: 1
                        source: StreamingPreferences.backgroundImagePath ? "file://" + StreamingPreferences.backgroundImagePath : ""
                        fillMode: Image.PreserveAspectCrop
                        visible: source != ""
                    }
                }

                Button {
                    text: qsTr("Choose...")
                    onClicked: bgFileDialog.open()
                }

                Button {
                    text: qsTr("Clear")
                    enabled: StreamingPreferences.backgroundImagePath !== ""
                    onClicked: StreamingPreferences.clearBackgroundImage()
                }
            }

            RowLayout {
                Layout.fillWidth: true
                Label { text: qsTr("Browse overlay"); color: "white"; Layout.preferredWidth: 110 }
                Slider {
                    Layout.fillWidth: true
                    from: 0; to: 0.8; stepSize: 0.05
                    enabled: StreamingPreferences.backgroundImagePath !== ""
                    value: StreamingPreferences.backgroundOverlayMain
                    onMoved: StreamingPreferences.backgroundOverlayMain = value
                }
                Label {
                    text: Math.round(StreamingPreferences.backgroundOverlayMain * 100) + "%"
                    color: "white"; Layout.preferredWidth: 40
                }
            }

            RowLayout {
                Layout.fillWidth: true
                Label { text: qsTr("Settings overlay"); color: "white"; Layout.preferredWidth: 110 }
                Slider {
                    Layout.fillWidth: true
                    from: 0; to: 0.8; stepSize: 0.05
                    enabled: StreamingPreferences.backgroundImagePath !== ""
                    value: StreamingPreferences.backgroundOverlaySettings
                    onMoved: StreamingPreferences.backgroundOverlaySettings = value
                }
                Label {
                    text: Math.round(StreamingPreferences.backgroundOverlaySettings * 100) + "%"
                    color: "white"; Layout.preferredWidth: 40
                }
            }
        }

        onClosed: StreamingPreferences.save()
    }

    FileDialog {
        id: bgFileDialog
        title: qsTr("Choose background image")
        nameFilters: [qsTr("Images (*.jpg *.jpeg *.png *.webp *.bmp)")]
        onAccepted: StreamingPreferences.setBackgroundImage(fileUrl)
    }
```

**Step 4:** 编译
```bash
make -j$(sysctl -n hw.ncpu) 2>&1 | tail -5
```
Expected: 编译通过。

**Step 5:** 手动验证
```bash
open app/Moonlight.app
```
Expected:
- 主界面空白区右键 → Popup 弹在鼠标位置
- 点击 "Choose..." → 系统 FileDialog → 选图 → 背景立即出现
- 拖动两个滑块 → 主界面遮罩跟随、切到设置页遮罩跟随
- 点击 "Clear" → 背景消失，滑块灰显
- 在 PC 卡片上右键 → PC 卡片右键菜单仍能弹（如果原本有），否则同空白区行为
- 关闭。

**Step 6:** Commit
```bash
git add app/gui/main.qml
git commit -m "feat(gui): add right-click background settings popup with sliders"
```

---

## Task 8：端到端回归验证 + 截图

**Files:** （仅验证，无代码改动）

**Step 1:** 全量清理 + 重编
```bash
make clean && qmake -r && make -j$(sysctl -n hw.ncpu) 2>&1 | tail -10
```
Expected: 编译完全通过。

**Step 2:** 启动 + 走完用户故事
```bash
open app/Moonlight.app
```
逐条验证：
1. ✅ 首次启动无背景图，默认 Material 灰底
2. ✅ 拖入 .png → 背景显示 + 35% 主遮罩
3. ✅ 拖入 .jpg 替换 → 旧副本被删（`ls ~/Library/Caches/Moonlight*/backgrounds/`）
4. ✅ 拖入 .pdf 或两个文件 → 不接受
5. ✅ 右键 → Popup → 拖动 "Browse overlay" 滑块 → 主界面遮罩实时变化
6. ✅ 进入设置页 → 遮罩切换为 settings 值
7. ✅ 右键 → 拖动 "Settings overlay" 滑块 → 设置页遮罩实时变化（要先回设置页观察）
8. ✅ Popup "Clear" → 背景与遮罩同时消失
9. ✅ 重启 app → 背景与两个遮罩值都保留
10. ✅ 故意删除磁盘上副本文件 → 重启 → 背景消失（Image.Error 触发 clear）

**Step 3:** 截一张主界面 + 一张设置页截图存档（备 PR 用）：
```bash
screencapture -x docs/plans/screenshots/bg-main.png
screencapture -x docs/plans/screenshots/bg-settings.png
```

**Step 4:** Commit 截图
```bash
mkdir -p docs/plans/screenshots
# 把上面截图放进去
git add docs/plans/screenshots
git commit -m "docs(plans): add screenshots of drag-drop background feature"
```

---

## Task 9（可选）：lupdate 注入新文案

**Files:**
- Modify: `app/languages/*.ts`（仅 en_US.ts 等用到的）

**Step 1:** 跑 lupdate 提取新 qsTr 字符串
```bash
lupdate app/app.pro
```
Expected: 新增条目 "Release to use as background"、"Background Settings"、"Choose..."、"Clear"、"Browse overlay"、"Settings overlay"、"Choose background image"、"Images (*.jpg *.jpeg *.png *.webp *.bmp)" 出现在 `.ts` 文件中。

**Step 2:** 仅 commit ts diff，不翻译（社区译者处理）
```bash
git add app/languages/*.ts
git commit -m "i18n: extract strings for background image feature"
```

---

## 收尾：开 PR

参考 `superpowers:finishing-a-development-branch`。建议 PR title：
> feat: drag/right-click to set custom background image with per-view overlay

PR body 包含：
- design doc 链接（`docs/plans/2026-05-20-...-design.md`）
- 截图（Task 8 产物）
- 验证清单（Task 8 的 10 条）

---

## 多模型路由策略（实施时）

| Task | 路由 | 说明 |
|------|------|------|
| Task 1, 2, 3 | CODEX | C++/Qt 后端逻辑，Codex 做原型 + Claude 重写 |
| Task 4, 5, 6, 7 | GEMINI | QML 前端 UI 与交互，Gemini 做原型 + Claude 重写 |
| Task 8 | CROSS_VALIDATION | 端到端联调，双模型 review |
| Task 9 | CLAUDE | lupdate 工具调用，无原型需求 |
