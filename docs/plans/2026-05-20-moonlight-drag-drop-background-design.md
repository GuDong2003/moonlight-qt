# 主界面拖入图片替换软件背景 — 设计文档

- 日期：2026-05-20
- 作者：Claude（与用户协作 brainstorming）
- 范围：moonlight-qt 客户端
- 状态：设计已确认，准备进入实施

## 目标

允许用户在 Moonlight 主界面通过 **拖入图片文件** 或 **右键菜单选择文件** 来替换软件全局背景，并能通过右键弹出的 Popup 面板独立调节 **主界面遮罩** 与 **设置页遮罩** 的透明度，提升个性化体验同时保留前景内容可读性。

## 非目标

- 不做多张背景图轮播 / 幻灯片
- 不做模糊 / 渐变 / 滤镜
- 不做 PC 卡片或 App 网格的局部背景替换
- 不修改 SettingsView 现有任何控件
- 不引入新的翻译流程（仍走 qsTr + lupdate）

## 架构与层次

根 `ApplicationWindow` 子节点（从底到顶）：

```
ApplicationWindow
├── Image          backgroundImage     // 全局背景，fillMode = PreserveAspectCrop
├── Rectangle      backgroundOverlay   // 全局遮罩，颜色黑，opacity 跟随当前页
├── StackView      stackView           // 原有视图栈（保持透明背景）
├── DropArea       imageDropArea       // 全屏接收图片拖入
├── MouseArea      rightClickArea      // 仅 RightButton，弹 Popup
└── Popup          bgSettingsPopup     // 背景设置面板
```

**关键设计点**：

- 背景图是 **全局** 的，所有页面共享一张图
- 遮罩 **按页区分**：根据 `stackView.currentItem.objectName` 选择 `backgroundOverlayMain` 或 `backgroundOverlaySettings`
- StackView 必须透明，原有 View 中如果有 `Rectangle { color: ... }` 充当背景需改为 `color: "transparent"`
- 入口收敛到 **主界面右键**，不在 SettingsView 添加任何控件（用户偏好 YAGNI）

## 数据流

```
拖入图片 / 右键 → 选择图片
      ↓
StreamingPreferences.setBackgroundImage(QUrl)
      ↓
SHA-1(前 12 位) + 原扩展名 → backgrounds/<hash>.<ext>
      ↓
QFile::copy 同步落盘 + 清理旧副本
      ↓
backgroundImagePath = 新副本绝对路径 (NOTIFY)
      ↓
main.qml 根 Image source 自动刷新
```

## 持久化

复用 `StreamingPreferences` 现有 QSettings 机制，新增 3 个 key：

| Key                          | 类型     | 默认值 | 说明                                |
|------------------------------|----------|--------|-------------------------------------|
| `backgroundImagePath`        | QString  | ""     | 副本绝对路径，空串表示未设置        |
| `backgroundOverlayMain`      | double   | 0.35   | 浏览页遮罩透明度（0–0.8）            |
| `backgroundOverlaySettings`  | double   | 0.65   | 表单页遮罩透明度（0–0.8）            |

## 拖入流程

- DropArea `keys: ["text/uri-list"]`，仅接受文件
- `onEntered`：校验单文件 + 扩展名白名单 `[jpg/jpeg/png/webp/bmp]` → `drop.accept(Qt.CopyAction)`
- 拖入悬停时显示半透明覆盖提示："释放以替换背景"，离开/释放后隐藏
- 多文件 → 提示 "请仅拖入一张图片" 并取消
- 复制失败（磁盘满/权限）→ 弹 `ErrorMessageDialog`
- 图片解码失败 → Image `onStatusChanged === Error` → 自动清空 path + 弹错误

## 落盘策略

1. 目标目录：`Path::getBackgroundsDir()` → 内部首次访问时 `QDir().mkpath()`
2. 命名：`SHA-1(file 前 1MB)` 前 12 位 + 原扩展名 → 例如 `a3f9e21b04c8.jpg`
3. 已存在同名 → 跳过复制（同图复用）
4. 切换图片时删除上一张副本（避免缓存膨胀）
5. 同步执行，图片小（一般 < 10MB），主线程毫秒级可接受

## 右键 Popup 面板

**触发**：根 `MouseArea`（acceptedButtons = RightButton），`onClicked` 在鼠标坐标处 `popup.open()`。

**面板结构**（约 280×220）：

```
┌──────────────────────────────┐
│  背景设置                     │
├──────────────────────────────┤
│  [缩略图]  选择图片...  清除  │
│                              │
│  主界面遮罩  [───●─────] 35% │
│  设置页遮罩  [──────●──] 65% │
└──────────────────────────────┘
```

- 顶部：缩略图（无图时灰块占位） + "选择图片..." 按钮（FileDialog） + "清除" 按钮
- 中部：两个 `Slider`（from 0, to 0.8, stepSize 0.05），右侧实时显示百分比
- 滑块拖动 → 双向绑定 preferences → 根遮罩 opacity 实时刷新（所见即所得）
- 未设置背景图时两个滑块灰显
- `closePolicy: CloseOnEscape | CloseOnPressOutside`
- 弹出位置：跟随右键点击坐标，超出窗口边缘自动收回

**事件冲突避免**：

- PC 卡片本身已有交互需要保持，根 MouseArea 设 `propagateComposedEvents: true`
- PC 卡片右键事件 `accepted = true` 即可拦住根菜单
- 仅空白区域右键弹此 Popup

## 改动文件清单

| 文件                                           | 改动                                                                          |
|------------------------------------------------|-------------------------------------------------------------------------------|
| `app/path.h` / `app/path.cpp`                  | 新增 `getBackgroundsDir()` 和 `deleteDataFile(QString)`                       |
| `app/settings/streamingpreferences.h` / `.cpp` | 新增 3 个 Q_PROPERTY、`setBackgroundImage(QUrl)`、`clearBackgroundImage()`     |
| `app/gui/main.qml`                             | 加 Image / Rectangle / DropArea / MouseArea / Popup                           |
| `app/gui/PcView.qml` 等浏览页                   | `objectName: "browseView"`，确保自身背景透明                                  |
| `app/gui/SettingsView.qml` 等表单页             | `objectName: "settingsView"`，确保自身背景透明                                |

## 验证步骤

1. `qmake && make -j` 编译通过
2. 启动 → 主界面右键 → Popup 弹出
3. Popup 选择 / 拖入图片 → 背景立即生效
4. 切到 SettingsView → 遮罩深度自动切换为更深
5. 拖动滑块 → 遮罩实时变化
6. 重启应用 → 背景与遮罩值保留
7. 清除 → 副本文件被删除，背景恢复默认 Material 颜色
8. 拖入非图片文件 / 多文件 → 正确拒绝
9. 拖入损坏图片 → 错误对话框 + 自动回滚

## YAGNI 决策记录

| 砍掉的功能            | 原因                                  |
|-----------------------|---------------------------------------|
| 设置页 Appearance 分区 | 用户明确要求入口收敛到主界面右键      |
| 模糊 / 滤镜           | 单层 Rectangle 遮罩已足够保证可读性    |
| 多张轮播              | 范围外，单图已满足核心需求            |
| 在线壁纸库            | 范围外                                |
| 异步落盘              | 图片小，同步毫秒级，不必引入额外复杂度 |

## 实施路由

- 前端 QML（main.qml / Popup / DropArea / 各 View 适配）→ GEMINI 协作
- C++ Preferences 改动（StreamingPreferences / Path）→ CODEX 复核
- 全栈联调 → CROSS_VALIDATION

## 后续

设计确认后用 `superpowers:using-git-worktrees` 隔离开发分支，再用 `superpowers:writing-plans` 拆 TDD 任务，进入 `subagent-driven-development` 流程。
