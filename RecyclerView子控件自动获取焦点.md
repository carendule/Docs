# RecyclerView子控件自动获取焦点


##### User:


Android的 RecyclerView如何在向上下移动时默认控制上焦到 item 内部的某个子item上



##### Assistant:


这是一个非常经典的 Android TV 或键盘交互场景下的开发问题。在 AOSP 或普通 Android应用开发中，默认的焦点查找算法（FocusFinder）通常会寻找距离最近的 focusable 视图，这往往导致焦点落在 Item 的边缘子 View 上，而不是你期望的核心操作按钮上。

要实现**“当焦点移动到 RecyclerView 的某个 Item 时，强制让特定的内部子 View 获取焦点”**，主要有三种层层递进的方案。

---

### 前置知识：焦点分发策略 (`descendantFocusability`)

在应用任何方案前，必须确保你的 Item 布局（根布局）允许子 View 获取焦点。
在 RecyclerView 的 Item 根布局（如 ConstraintLayout）中，必须设置：

```xml
android:descendantFocusability="afterDescendants"
```
或者 `blocksDescendants` (如果你想让根布局先拿焦点，但这通常不是你现在的需求)。通常我们设为 `afterDescendants`，意味着系统会优先尝试让子 View 获取焦点。

---

### 方案一：使用 `android:focusedByDefault` (推荐 API 26+)

这是最简单、最符合现代 Android 标准的做法。如果你的 `minSdkVersion` 是 26 (Android 8.0) 及以上，可以直接在 XML 中指定默认焦点。

**实现方式：**
在你希望默认获取焦点的那个**子 View** 上添加 `android:focusedByDefault="true"`。

```xml
<!-- Item 的布局文件 -->
<androidx.constraintlayout.widget.ConstraintLayout 
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:descendantFocusability="afterDescendants"> <!-- 关键点 1 -->

    <!-- 普通文本，不可聚焦 -->
    <TextView
        android:id="@+id/tv_title"
        ... />

    <!-- 次要按钮 -->
    <Button
        android:id="@+id/btn_secondary"
        ... />

    <!-- 核心按钮：我们希望焦点进入 Item 时主要落在这里 -->
    <Button
        android:id="@+id/btn_primary"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:focusable="true"
        android:focusedByDefault="true" /> <!-- 关键点 2 -->

</androidx.constraintlayout.widget.ConstraintLayout>
```

**原理：** 当 RecyclerView 滚动或焦点通过方向键进入这个 Item 容器时，系统会查找设置了 `focusedByDefault` 的子 View 并优先给它焦点。

---

### 方案二：自定义 ViewGroup 重写 `onRequestFocusInDescendants` (通用、强控)

如果需要兼容低版本，或者逻辑比较复杂（例如：根据业务状态，有时默认聚焦按钮 A，有时聚焦按钮 B），则需要继承 Item 的根布局容器，重写焦点分发逻辑。

**实现方式：**

1.  创建一个自定义布局（例如 `FocusControlConstraintLayout`）。
2.  重写 `onRequestFocusInDescendants`。

```kotlin
class FocusControlConstraintLayout @JvmOverloads constructor(
    context: Context, attrs: AttributeSet? = null
) : ConstraintLayout(context, attrs) {

    // 可以在 ViewHolder 中通过代码设置这个 ID，决定哪个子 View 优先
    var defaultFocusViewId: Int = R.id.btn_primary 

    override fun onRequestFocusInDescendants(
        direction: Int,
        previouslyFocusedRect: Rect?
    ): Boolean {
        // 1. 找到我们想要强制聚焦的子 View
        val targetView = findViewById<View>(defaultFocusViewId)
        
        // 2. 如果子 View 存在且可见、可聚焦，强制请求焦点
        if (targetView != null && targetView.isFocusable && targetView.visibility == View.VISIBLE) {
            return targetView.requestFocus(direction, previouslyFocusedRect)
        }
        
        // 3. 如果没找到目标，回退到默认逻辑
        return super.onRequestFocusInDescendants(direction, previouslyFocusedRect)
    }
}
```

**原理：** 当焦点从外部（上一个 Item）进入当前 Item 容器时，系统会调用容器的 `requestFocus`，进而触发 `onRequestFocusInDescendants`。在这里截断逻辑，引导焦点去往指定的地方。

---

### 方案三：使用 `TouchDelegate` 或 `nextFocus` (针对定点跳转)

如果你非常确定焦点是从“上方”下来的，可以通过 XML 指定 `nextFocus` 链，但由于 RecyclerView 的 Item 是复用的，ID 会重复，导致 `android:nextFocusDown="@+id/btn_target"` 往往失效（它可能跳到同一个 Item 内部的 View，也可能跳不出去）。

**在 Adapter 中手动控制 (不推荐，但有时有用)：**

在 `onBindViewHolder` 中，你可以通过代码连接焦点关系，但这对 RecyclerView 来说很难维护，因为你需要知道“下一个 Item”是谁。

**更好的变种：保留该 Item 上次离开时的焦点**

有时用户希望：我在 Item 1 选中了右边的按钮，向下移到 Item 2，再按上回到 Item 1 时，焦点应该**回到刚才选中的那个右边按钮**，而不是默认按钮。

这需要在 Adapter/ViewHolder 中通过 `DescendantFocusability` 配合状态保存来实现：

```kotlin
// 在 Custom ViewGroup 中
override fun addFocusables(views: ArrayList<View>?, direction: Int, focusableMode: Int) {
    if (hasFocus()) {
        // 如果当前 Item 已经有焦点，按标准逻辑处理
        super.addFocusables(views, direction, focusableMode)
        return
    }
    
    // 如果是外部正要进来，我们可以只把"即使获得焦点"的那个 View 加入查找列表
    // 这样 FocusFinder 除了选它别无选择
    val target = findViewById<View>(lastFocusedId) // lastFocusedId 需要自行维护保存
    if (target != null && target.isFocusable) {
        views?.add(target) 
        return
    }
    
    super.addFocusables(views, direction, focusableMode)
}
```

---

### 总结与建议

1.  **首选方案**：若 `minSdk >= 26` 且逻辑静态，使用 XML 属性 **`android:focusedByDefault="true"`**。
2.  **次选方案**：若需要兼容性或条件判断，**自定义 Item 根布局并重写 `onRequestFocusInDescendants`**。
3.  **核心配置**：无论哪种方案，务必在 Item 根布局设置 **`android:descendantFocusability="afterDescendants"`**。

### 常见坑点排查
*   **Item 根布局抢焦点**：确认根布局没有设置 `focusable="true"`，或者 `descendantFocusability` 设置成了 `blocksDescendants`。
*   **TouchMode 干扰**：如果是 Android TV，通常没有 Touch 干扰。如果是手机接键盘，确保没有进入 TouchMode（一旦触摸屏幕，View 可能会清除 focus 状态）。
*   **RecyclerView 自身**：确保 RecyclerView 本身 `android:descendantFocusability="afterDescendants"`（默认通常就是）。



##### User:


onRequestFocusInDescendants是什么时候触发的，我的item内部还有其他的View，这个方法会不会出现导致这个Item永远就上焦到targetView上了



##### Assistant:


这是一个非常敏锐且关键的问题！

**核心结论：**
**不会导致死循环，也不会导致焦点永远困在这个 `targetView` 上。**

只要你的 `onRequestFocusInDescendants` 逻辑写得标准（只处理“分发”逻辑，不处理“拦截”逻辑），它**只会在焦点从外部进入该 Item 时触发一次**。一旦焦点进入了 Item 内部，后续在子 Item 之间的移动（比如从左边的按钮移到右边的按钮）完全不会触发这个方法。

下面我帮你拆解一下它的触发时机和系统内部的运行机制。

---

### 1. 什么时候触发 `onRequestFocusInDescendants`？

**只有当 ViewGroup（你的 Item 根布局）本身“收到获取焦点的请求”时，才会触发。**

这通常发生在以下场景：
1.  **从外部进入**：用户按“下”键，焦点从 `Item 0` 离开，系统计算出下一个落脚点是 `Item 1` 的根布局。系统对 `Item 1` 调用 `requestFocus()`。
2.  **代码强制调用**：你在代码里手动调用了 `itemLayout.requestFocus()`。

**关键点：**
当 ViewGroup 的 `descendantFocusability` 设为 `afterDescendants`（默认或推荐设置）时，`ViewGroup.requestFocus()` 的源码逻辑大致是这样的：

```java
// ViewGroup.java (AOSP 伪代码简化)
public boolean requestFocus(int direction, Rect previouslyFocusedRect) {
    // 1. 如果策略是 BLOCK，自己尝试拿焦点...
    
    // 2. 如果策略是 AFTER_DESCENDANTS (也就是我们的情况)
    // 它不会自己先拿焦点，而是直接问子 View 谁要
    if (descendantFocusability == FOCUS_AFTER_DESCENDANTS) {
        // ===> 只有在这里，才会通过这个方法去问子 View <===
        if (onRequestFocusInDescendants(direction, previouslyFocusedRect)) {
            return true; // 子 View 接盘了，任务结束
        }
    }

    // 3. 如果没子 View 要，或者策略不同，自己再尝试兜底拿焦点...
    if (super.requestFocus(...)) {
        return true;
    }
    return false;
}
```

### 2. 为什么不会导致 Item 内部移动“卡死”？

假设你的 Item 布局如下：
`[ 根布局 (ConstraintLayout) ]`
   ├── `View A (默认 default)`
   └── `View B (其他按钮)`

**场景演示：**

*   **阶段一：焦点从外部进入（触发！）**
    1.  你按“下”键，焦点撞到了 `根布局`。
    2.  `根布局` 执行 `onRequestFocusInDescendants`。
    3.  你的代码生效：强制把焦点给了 `View A`。
    4.  **结果：View A 获得焦点。**

*   **阶段二：焦点在内部移动（不触发！）**
    1.  现在焦点在 `View A` 上。
    2.  你按“右”键（想去 View B）。
    3.  系统 FocusFinder 开始工作：它会在当前视图树中查找 `View A` 右边是谁？
    4.  它找到了 `View B`。
    5.  系统直接将焦点从 `View A` 转移到 `View B`。
    6.  **关键：** 这个过程是子 View 平级之间或者是直接的点对点传输，**不会再次调用根布局的 `requestFocus`**，因此也就**完全不会走 `onRequestFocusInDescendants`**。

### 3. 如何确保代码万无一失？

为了避免逻辑 BUG，你的重写代码应该遵循“仅仅是个引导者”的原则。

正确的写法范式：

```kotlin
override fun onRequestFocusInDescendants(
    direction: Int,
    previouslyFocusedRect: Rect?
): Boolean {
    // 1. 只有在这个 Item 且所有子 View 当前都没有焦点时，这个方法才会被调用。
    // 所以这里不需要判断 "if (hasFocus())"，因为如果有焦点，系统根本不会调这里。

    // 2. 获取默认目标
    val target = findViewById<View>(R.id.btn_primary)
    
    // 3. 只有当目标真的 有效、可见、启用 时才强行分配
    if (target != null && target.visibility == View.VISIBLE && target.isEnabled && target.isFocusable) {
        return target.requestFocus(direction, previouslyFocusedRect)
    }

    // 4. 【重要】如果目标 View 处于 GONE 状态，或者不可点击，
    // 必须调用 super，让系统去寻找除此之外最合适的子 View（比如 View B）。
    // 如果这里直接返回 false 或 true 而不调 target.requestFocus，可能会导致焦点丢失。
    return super.onRequestFocusInDescendants(direction, previouslyFocusedRect)
}
```

### 总结

可以将 `onRequestFocusInDescendants` 理解为 **“大堂迎宾员”**。

*   当你**第一次走进大楼**（焦点进入 Item）时，迎宾员会引导你去特定的房间（`targetView`）。
*   一旦你**已经在房间里**了，想去隔壁房间（从 Child A 移到 Child B），你直接走过去就行，不会再经过大堂，迎宾员也就不会拦着你把你拽回原来的房间。

所以，请放心使用此方法，它是处理 RecyclerView 复杂 Item 焦点层级导航的标准 AOSP 级解决方案。

