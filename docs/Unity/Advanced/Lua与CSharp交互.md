# Lua与C#交互

在之前的笔记中，我已经对常用的 Lua 解决方案进行了一些介绍。不过说实话，这些大多都只是非常浅显的知识，各位随便搜索一下就能够找到大把的资料。我们在开发一款实际的游戏项目时，为了能够让游戏的各功能模块尽可能的进行热更新，都会将大部分的逻辑写到 Lua 部分，然后由 Lua 调用 C# 的接口。不过这样一来，我们就不得不想办法优化 Lua 与 C# 之间的调用，以避免造成游戏的性能低下。

!> 注意，本篇笔记着重讨论重度使用 Lua 的情况，例如商业常见的 ToLua 项目基本都是把逻辑全写在了 Lua 端。如果你的项目是主要采用 C# 编写逻辑，并使用 XLua 进行热补丁修复，那么这类情况就不在此次的讨论范围内。

## 前言

---

从 Lua 纯反射调用 C#，到 C# 实现 Lua 虚拟机，再到现在的 LuaJit + C# 静态导出 Lua 文件，Lua 与 C# 的交互方案一直都在不断地改进中。不过说到底，用 Lua 调用 C# 本身就存在着效率问题，如果频繁在代码中调用 Unity 的接口，那么游戏的帧数将会非常难看。

我曾在实习期间参与过一款基于 ToLua 实现的三消游戏，按道理来说这种游戏类型对于机器配置的要求并不高，如果用 C# 来写的话都不怎么需要注意优化的问题。但遗憾的是，效率低下的 Lua 实现方式让游戏表现得十分糟糕，最后不得不开始考虑起 Lua 代码的优化方案。为了解决这一问题，我们就需要先对 Lua 与 C# 的交互方式进行一些分析。

## Lua与C#的交互优化

---

### 对象引用的性能分析

首先我们来看一下最简单的一句 `gameobj.transform.position = pos` 需要调用哪些 Lua 的 API（这里用的是 ToLua，可能根据版本的不同有不同的调用步骤）。下图借用是 [用好Lua+Unity，让性能飞起来——Lua与C#交互篇](https://blog.uwa4d.com/archives/USparkle_Lua.html) 中的步骤图：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Lua%E4%B8%8EC%23%E4%BA%A4%E4%BA%92%E4%BC%98%E5%8C%96/Blog_Sparkle_Lua_2.png)

相信各位在写逻辑时经常需要修改游戏物体的位置，但如果你要在 Lua 中这么写的话其实是一种非常糟糕的行为。这类直接在 Lua 中获取 C# 对象的写法效率很低，不太可能直接用来编写类型较为复杂的游戏（甚至于连三消这种游戏类型都表现得很差劲），因为每一行看似简单的代码都包含着大量的 GC 以及堆栈调用。

在上面的例子中，我们为了拿到一个 transform 就付出了相当多的代价，更别说你还需要调用 transform 的 API 去修改物体的位置。由于 C# 的 Object 类型无法作为指针传递给 C，所以一般的 Lua 解决方案都是用一个 Dictionary 存储 ID 和 Object 的关系，并且该字典对于 C# 对象的引用也避免了 C# 的自动 GC。

步骤图中也明确地表示出了 Lua 调用 C# 的过程。如果方法的参数中存在 Object 类型，那么就必须要从 Lua 持有的对象 ID 转换回 C# 的 Object，并且需要进行一次字典的查找。另外，每当调用一个 Object 的成员方法时，也需要先获取到这个 Object。

如果该对象之前有用过并且没有被回收，那么分配给 Lua 的 userdata 以及字典的 ID 索引都是存在的，所以只需要进行一次字典查找即可。但试想一下，如果你只是临时使用一下这个对象，那么之后你又得按照步骤图进行一系列准备工作，非常浪费时间。比如例子提到的 transform 就是一个大坑，因为你只是临时使用一下 transform 改变位置而已，下次使用时又得重新分配。

简而言之，在 Lua 中持有 C# 引用的代价是十分昂贵的，我们得换其他的思路来解决这一问题。

### 减少参数传递的复杂度

在 Lua 中进行参数传递时，我们需要尽量让参数变得简单，否则复杂的类型转换势必会导致性能变低。比如我们在修改物体位置时，需要传递一个 Vector3 参数。由于直接调用 C# 的 Vector3 过于缓慢，所以一般的做法都是在 Lua 中实现一个简单的 Vector3，包含 x、y、z 三个值即可。但实际上，当我们需要把 C# 中拿到的 Vector3 的值赋给 Lua 时，需要从 Vector3 中取出三个值，然后构造一个 Lua 表并为表中的三个值赋值，并最终把这个表返回。这其中经历的步骤依旧是不少的。

相比之下，如果你直接传递 x、y、z 的值，那么效率又会要高上一些。这一点对于其它的类型来说也是适用的，比如 Quaternion、Object、string、bool 等等。另外，由于 Lua 和 C# 的交互是经由 C 实现的，而不同语言的数据类型又不太一样，在传递的过程中经常会发生类型转换。因此，在进行参数传递时最好传递 int、float、double 这种简单类型，而各类型的 Object 则尽量不要进行传递。

?> Lua 与 C# 的交互是通过堆栈压入与弹出实现的，参数越多越复杂，性能开销就会越大。

### 使用自定义的对象ID以避免直接的对象引用

我们在 Lua 中获取 C# 的组件一般使用的是 `GetComponent` 方法，比如在与 UGUI 或者 NGUI 交互时就需要把 UI 界面上可能用到的组件全部获取下来。之前也讲过，在 Lua 保存 C# 引用以及调用 Object 的成员方法是非常消耗性能的，并且我们在获取组件时需要传递路径的字符串，这样又会产生类型转换造成的消耗。那么，我们有没有一种办法能够降低这种效率低下的调用方式呢？

答案当然是有的。既然在 Lua 中调用 C# 对象的方法十分消耗性能，那我们干脆就在 C# 中执行方法逻辑，而 Lua 则只用于调用 C# 的接口以及接受运算的结果。为了达成这一目的，我们就不能够在 Lua 中获取 C# 的引用，而是应该使用自定义的整型数 ID 来管理 C# 的对象，在方法调用时将对象的 ID 传递给 C#，然后由 C# 获取该对象并直接调用该对象的成员方法。

具体的做法是这样的。我们首先需要为对象进行 ID 编号，然后在 C# 中自定义一个字典用来管理 ID 与对象的关系。当然，你可能会说这种方式不就和 ToLua 一样了吗？其实这是不一样，ToLua 的管理方式则是用一个字典将 C# 的引用和 Lua 的 userdata 对应起来，获取对象时需要经过上面展示的一长串步骤。如果你是持续引用的那还好，但如果只是临时用一下（比如 `gameObject.transform.position`），那么每次都需要从头来一遍（因为对应 ID 和 userdata 会被 GC 掉）。综合考虑之下，我们最好使用自定义的对象管理，并且在 Lua 端编写相对应的类进行管理，比如下面这样：

```lua
local class = require "class"
local _M = {}
LuaGameObject = _M

---@param self LuaGameObject
function _M._init_(self)
    local LuaTransform = require "LuaTransform"
    if (_M._super_._init_) then
        _M._super_._init_(self)
    end

    ---@type LuaTransform
    self.transform = LuaTransform.new()
end

--// 下面是调用GameObject的相关接口
--// ...
```

除开自定义的 Lua 类，我们还需要为 C# 和 Lua 编写中间桥接类，以支持这种传递 ID 来调用成员方法的使用方式。具体的做法也是很简单，比如对于 NGUI 中的 `UIScrollView` 组件，你可以编写一个名为 `UIScrollViewBridge` 的 C# 桥接类，并在该类中编写所有可能用到的 ScrollView 的组件方法，例如 `ResetPosition`、`MoveRelative` 等等。具体的写法类似于下面这样：

```csharp
// 接口方法，调用NGUI原有的API
public static void MoveRelative(int componentID, int x, int y, int z)
{
    // 先根据ID获取到UIScrollView对象，然后调用其成员方法
    UIScrollView scrollView = PrefabBinder.GetObjectByID(componentID) as UIScrollView;
    scrollView.MoveRelative(new Vector3(x, y, z));
}
```

**使用静态方法编写的 C# 代码将有更快的调用效率**。将所有可能用到的 UIScrollView 的成员方法写成接口后，只需要把桥接类生成静态的 Wrap 文件，然后在 Lua 端也编写一个桥接类调用即可：

```lua
function UIScrollView.MoveRelative(id, data)
    UIScrollViewBridge.MoveRelative(id, data.x, data.y, data.z)
end
```

这样一来，我们就可以传入对应的 ID 与参数，并通过桥接类来直接执行 C# 的接口，从而避免在 Lua 中使用 C# 引用来调用其成员方法。当然，为了达成这一目的需要我们自行编写一套对象 ID 的管理方法，而且还要给所有要用到的类编写桥接类（可以编写工具自动生成类似的“胶水”代码）。

?> 将持有 `Transform` 等复杂类型的 userdata 改成传 `id` 会有一定的性能提升，但由于 Lua 与 C# 的交互用的还是虚拟堆栈，性能瓶颈还是存在的。这样做的主要优点是能够自行管理对象引用，减少了不必要的 GC 消耗，并且也在一定程度上避免了内存泄漏。

### 更高效的交互方式

使用自定义的 ID 管理 Object 确实能够减少临时引用的创建和销毁，从一定程度上提高了游戏帧数。不过究其根本，Lua 与其他语言的交互用的是虚拟堆栈，每次进行交互时都要进行参数的入栈和出栈，因而在函数调用频率和函数参数增多时，交互所带来的消耗几乎是致命的。部分商业项目为了提高核心逻辑的运行效率，通常是把这一部分代码放到了 C# 中（比如寻路、模型、特效等等），而 Lua 端只负责 UI 等功能的编写。但随着项目的一步步推进，游戏的核心逻辑可能会变得比较复杂，导致你很难在性能与热更新之间找到一个平衡点。因此，我们需要一种更好更直接的数据交互方式。

新方案的核心思想是直接将数据存储到 Lua 的数组中，然后将 Lua 的底层内存结构直接暴露给 C# 端，让 C# 不经过 Lua API 直接以 `unsafe` 的方式操作 Lua 内存。这样一来，Lua 端可以直接读写 Lua 数组，C# 端则以 unsafe 方式读写 Lua 内存，从而避免了堆栈交互所带来的消耗（腾讯的 UnLua 也是用的类似的交互方式，不过是给 UE4 用的）。

具体的做法可以看我另外一篇笔记。

## Lua与C#的一些交互问题

---

### IOS端的接口兼容问题

在正式发布版本之后，如果想要修改 C# 的接口，那么就必须要进行 DLL 文件替换。遗憾的是，IOS 禁止应用程序创建可写可执行的内存页，而这是 JIT 所必须的，因此我们得考虑到 IOS 旧包的兼容问题。

做法其实并没有多难，这里用到的是 Lua 的 `pcall` 方法，它会在执行函数期间进行异常捕获，并具有多个返回值：是否执行成功、执行函数的返回值。

```lua
if (IsIOS()) then
    --// 将所要执行的方法包在匿名函数中，交由pcall执行
    local ret, value = pcall(function()
        CommonBridge.NewFunc()
    end)
    --// 如果方法执行失败，那么证明当前是IOS的旧包（当然也有可能是方法本身抛出了异常，所以C#端要做一下异常处理）
    if (not ret) then
        --// ios的旧包需要执行原来的逻辑
        --// ...
    end
else
    --// 安卓端直接调用新的接口
    CommonBridge.NewFunc()
end
```

IOS 端由于没有更新 C# 接口，所以在 Lua 端调用新接口肯定是会报错的，这时候就可以走原来的处理逻辑。

!> 需要注意的是，如果调用的方法抛出了异常，那么 `pcall` 的返回结果也是失败的。最好的做法是在调用的方法中进行异常处理，并考虑是否要在 C# 中回调 Lua，以保证逻辑不会被卡住。