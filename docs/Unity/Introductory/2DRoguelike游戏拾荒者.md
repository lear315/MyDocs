# 2D Roguelike游戏——拾荒者

> 本文仅供个人学习，并不是教程

拾荒者是 Untiy 官方的教学案例，想看官方源码的请转到——[2D Roguelike tutorial](https://unity3d.com/cn/learn/tutorials/s/2d-roguelike-tutorial)。Demo 我已经传到 GitHub 的练习项目中了，不过各位也就稍微看看就好，因为这是跟着某个视频教程写的，代码写得很乱，具体的还是参考官网放出来的源码吧。

## 前言

---------
相较于我之前写的 2D 乒乓球，这次的游戏更为完善，基本上做完这个案例后也能写出相当一部分的游戏了。这一次的笔记主要是为了介绍游戏用到的各种组件与功能，源码的话就只挑重要的讲了。

还是那句话，我比较推荐先做几个案例，然后再根据自己不懂的地方去查文档和资料。直接学习功能组件其实比较枯燥无味，也没有太多的必要。

## 处理图片

---------
如果你有过 2D 游戏开发经验，你应该会接触过类似`Texture Packer`的软件。为了节省空间和方便管理，我们一般把零散的图片汇集成一张大图，并用一个文件记录各个精灵元素在大图中的位置，使用时直接将整张大图导入即可。

如下图，我把一张大图导入编辑器。现在我们可以将大图展开，自由地选取我们需要的精灵图片：

![](http://cdn.fantasticmiao.cn/image/post/U3D/2Dproject-Roguelike/01.png)

当然，你也可以在属性栏中点击`Sprite Editor`来查看各个图片：

![](http://cdn.fantasticmiao.cn/image/post/U3D/2Dproject-Roguelike/02.png)

?> 为了能够直观地查看各项资源图片，建议各位把布局方式设置成`Defalut`。

## 游戏中的预制体

---------
由于这次的游戏里需要用到大量的预制体，所以我有必要解释一下预制体的基本操作：

* 把物体拖到资源窗口就能够制作一个预制体。
* 当你对某个物体进行修改时，如果想要将这个修改应用到它的预制体上，那么你就必须点击物体属性栏中的`Apply`。

!> 对于预制体的修改将直接应用到与之关联的物体上，但位置参数是不会改变的，建议各位多去试一试。

### 主角

游戏的主角一共有三个动作：闲置、攻击、受伤，每一个动作都是由多张图片组合而成。在 Unity 中，你只需要选中多张图片，并将它们拖动到`Hierarchy`中，便能够生成一个动画。这里我们先选中所有和闲置动作有关的图片，然后将它们拖入到`Hierarchy`中：

![](http://cdn.fantasticmiao.cn/image/post/U3D/2Dproject-Roguelike/04.png)

如果没有操作错误的话，你应该能在资源窗口中看到上面这两个文件（记得修改一下名字）。`Player`是一种动画状态机（Animator Controller），用于控制物体的动作逻辑，而`PlayerIdle`则是动画（Animation），包含了一个动作的所有动画帧。

点开`Player`，得到如下窗口：

![](http://cdn.fantasticmiao.cn/image/post/U3D/2Dproject-Roguelike/05.png)

图中的窗口便是动画状态机了，里面的四个方块则代表了角色的行为逻辑。这里我简单地介绍一下这四个部分：

* `Any State`：用于向任意状态过渡。例如角色死亡时跳转到死亡状态。
* `Entry`：状态机入口，也就是起始点。
* `PlayerIdle`：这是我们添加的闲置动作，包含了所有与其相关的动画帧。
* `Exit`：结束。

光有了一个动作还不行，我们还需要为主角添加攻击和受伤两个动作。由于我们已经创建了一个动画对象`Player`，所以我们可以直接把两个动作相关的图片拖动到`Hierarchy`中的对象上：

![](http://cdn.fantasticmiao.cn/image/post/U3D/2Dproject-Roguelike/07.png)

如上图，此时主角的动作就全部添加完成了。我们把主角制作成预制体，然后删除掉原本的对象。

!> 为已存在的物体添加动画时，只会生成一个动画文件，而且物体的状态机中会出现新的状态。如果你的资源目录中出现了一个新的状态机，说明你没有添加成功。

### 敌人

游戏中的敌人有两种，由于它们的行为逻辑都是相同的，因此在创建敌人时可以共用一个动画状态机。

与生成主角的步骤类似，先创建一个`Enemy1`，把它制作成预制体，之后再把场景中的`Enemy1`改名成`Enemy2`，并且导入`Enmey2`的两种动画（闲置和攻击）。此时，`Enemy2`的动画状态机中就会有四个动作（分别是Enemy1、Enemy2的闲置和攻击动作），把后加入的两个动作删除即可。

你可能会觉得奇怪，为什么我要把新加入的两个动作给删除？很简单，因为当前的状态机是 Enemy1 的，我们需要为 Enemy2 重写一个状态机，然后才能将 Enemy2 的动作添加进来。

![](http://cdn.fantasticmiao.cn/image/post/U3D/2Dproject-Roguelike/09.png)

如上图，创建一个重写的动画状态机（Animator Override Controller），将重写的对象指定为 Enemy1 的状态机，然后将 Enemy2 的两个动作赋值给`Enemy1Idle`和`Enemy1Attack`。这一步的操作就相当于把 Enemy1 的动作替换成了 Enemy2 的，但是行为逻辑是和 Enemy1 的一模一样，且会随着 Enemy1 的改变而改变。

Enemy2 的状态机写好了，但别忘了要修改一下 Enemy2 的属性：

![](http://cdn.fantasticmiao.cn/image/post/U3D/2Dproject-Roguelike/08.png)

因为 Enemy2 只是改了个名字，图片还是用的 Enemy1 的，所以要记得换一下图片：

![](http://cdn.fantasticmiao.cn/image/post/U3D/2Dproject-Roguelike/10.png)

这样，主角与敌人的预制体就创建好了：

![](http://cdn.fantasticmiao.cn/image/post/U3D/2Dproject-Roguelike/11.png)

### 墙壁、地板、食物

这些东西是不需要动画的，直接将它们制作成预制体即可：

![](http://cdn.fantasticmiao.cn/image/post/U3D/2Dproject-Roguelike/12.png)

## 渲染顺序

---------
2D 游戏尤其要注重精灵的层次，谁在上面谁在下面要弄清楚。这里我分了三个层：Background、Items、Role。
* Background：背景层。OutWall、Floor 可以放在这里。
* Items：物品层。包含 Food、Soda、Wall。
* Role：角色层。包含 Player、Enemy1、Enemy2。

![](http://cdn.fantasticmiao.cn/image/post/U3D/2Dproject-Roguelike/17.png)

如果不分层次结构的话，地板很有可能会把障碍物、食物等物体遮盖住。

## 角色的动作切换

---------
以主角为例，他会在最开始时重复播放闲置动画。当他攻击或者受伤时，都会播放对应的动画，然后马上切换回闲置状态。明白了这一点后，想必你大概知道怎么设计了。

![](http://cdn.fantasticmiao.cn/image/post/U3D/2Dproject-Roguelike/24.png)

主角的状态机基本上就是这样了。不过还没完，我们得为攻击动作和受伤动作添加`Trigger`，以便我们能在代码中调用它们。

?> 关于如何创建和调用 Trigger，请转到——[Animation Parameters](https://docs.unity3d.com/2017.4/Documentation/Manual/AnimationParameters.html)

![](http://cdn.fantasticmiao.cn/image/post/U3D/2Dproject-Roguelike/26.png)

如上图，我们需要对每个分支做修改。由于攻击和受伤状态的逻辑是一样的，所以我这里只演示对攻击状态的修改。

首先是从闲置转换到攻击。我把右边属性栏中的`Has Exit Time`取消了，这样主角就会立即切换动作，无需等待闲置动画播放完。除此之外，由于我们播放的是帧动画，为了保证两个状态的流畅切换，还需要把`Transition Duration`（转换间隔）设置为 0。

然后就是从攻击转换到闲置。`Transition Duration`我们依然设置为 0。由于攻击状态需要播放整个动画，所以我们要将`Exit Time`设置为 1。

?> `Exit Time`的值在 0~1 之间，值为 0.5 则相当于播放一半的动画。详情转到——[Animation transitions](https://docs.unity3d.com/2017.4/Documentation/Manual/class-Transition.html)。

受伤状态与攻击状态是类似的，只需要跟着上面的步骤来即可。顺带一提，如果你想要预览动画的切换效果，你可以点击某个分支，然后把`Player`拖动到编辑器右下角的窗口中：

![](http://cdn.fantasticmiao.cn/image/post/U3D/2Dproject-Roguelike/27.png)

或者你也可以直接运行游戏，然后点击两个动作来查看效果：

![](http://cdn.fantasticmiao.cn/image/post/U3D/2Dproject-Roguelike/28.png)

## 关卡结构

---------
从这一节开始，我们就要正式进入到编码环节了，我下面贴的代码都是官方的，完整代码的话请去官网看。

### BoardManager

> 管理关卡的生成，场景中各元素的摆放。

首先是各项预制体的集合，需要在编辑器中为其赋值。

```csharp
public GameObject exit;                // 出口
public GameObject[] floorTiles;        // 地板集合
public GameObject[] wallTiles;         // 障碍物集合
public GameObject[] foodTiles;         // 食物集合
public GameObject[] enemyTiles;        // 敌人集合
public GameObject[] outerWallTiles;    // 围墙集合
```

赋值就是将预制体直接拖到某一集合下（为了拖拽方便，记得先把属性栏锁上）。

![](http://cdn.fantasticmiao.cn/image/post/U3D/2Dproject-Roguelike/13.png)

预制体的实例化用的是下面这两行代码：

```csharp
/**
 * toInstantiate 是需要实例化的预制体
 * Vector3 是实例化对象的位置
 * Quaternion.identity = Quaternion(0,0,0,0)，无旋转角度
 * as GameObject 是将实例化对象设为 GameObject
 */
GameObject instance = Instantiate (toInstantiate, new Vector3 (x, y, 0f), Quaternion.identity) as GameObject;

// 将 boardHolder 设为实例化对象的父类，以免产生混乱的层次结构
instance.transform.SetParent (boardHolder);
```

我在这里要说明一下，`boardHolder`是一个空对象，实例化的对象都存储在这个空对象中，以免层次结构混乱。（不过我看了一下，官方的源码中似乎只把地板和围墙加入了 boardHolder）

![](http://cdn.fantasticmiao.cn/image/post/U3D/2Dproject-Roguelike/16.png)

官方源码为了减少重复的部分，将部分功能写成了方法，比如`RandomPosition()`是用来获取一个随机的位置；`LayoutObjectAtRandom()`是根据传入的集合随机选取一个预制体进行实例化，并且能够设置物体数量的范围，增加了关卡的随机性。

最后就是整个关卡的初始化方法`SetupScene()`，这个方法会在`GameManager`中调用。

```csharp
// SetupScene 用于初始化关卡，并且调用上面的方法来展示整个游戏场景
public void SetupScene (int level)
{
    // 创建地板和外墙
    BoardSetup ();
    
    // 将中心的6*6大小的网格加入到一个列表中，障碍物、食物、敌人只在这个区域中生成
    InitialiseList ();
    
    // 生成位置随机、数量随机、种类随机的障碍物，mininum、maximum 用于限制障碍物的数量
    LayoutObjectAtRandom (wallTiles, wallCount.minimum, wallCount.maximum);
    
    // 效果同上，用于随机创建食物
    LayoutObjectAtRandom (foodTiles, foodCount.minimum, foodCount.maximum);
    
    // 敌人的数量由当前关卡等级所决定。Mathf.Log 用于取对数，例如以2为底时，对8取对数等于3，所以第八关会出现3个敌人
    int enemyCount = (int)Mathf.Log(level, 2f);
    
    // 效果同上，用于随机创建敌人
    LayoutObjectAtRandom (enemyTiles, enemyCount, enemyCount);
    
    // 将出口实例化
    Instantiate (exit, new Vector3 (columns - 1, rows - 1, 0f), Quaternion.identity);
}
```

### GamaManager

> 初始化游戏关卡，管理游戏进程，记录部分数值

由于代码不多，我就全部贴上来了。

```csharp
public class GameManager : MonoBehaviour
{

    public static GameManager instance = null;              // 单例
    private BoardManager boardScript;
    private int level = 3;                                  // 当前关卡等级

    // Awake 用于初始化，比 Start 先调用
    void Awake()
    {
        // 若单例为空，则将当前对象赋值给单例
        if (instance == null)
            
            instance = this;
        
        // 若单例不等于当前对象，那么要将当前对象销毁，因为单例只能存在一个。
        else if (instance != this)
            
            Destroy(gameObject);    
        
        // 重新加载关卡时，我们任需要 GameManager 中的数据，所以不能够销货它
        DontDestroyOnLoad(gameObject);
        
        // 获取脚本
        boardScript = GetComponent();
        
        // 初始化游戏
        InitGame();
    }
    
    // 调用 BoardManager 中的 SetupScene() 方法来加载当前关卡
    void InitGame()
    {
        boardScript.SetupScene(level);
    }
}
```

单例的写法有很多种，比如：

```csharp
private static GameManager _instance;

public static GameManager Instance {
    get {
        return _instance;
    }
}
```

### Loader

> 实例化游戏关卡和音乐

官方的源码还额外写了一个`Loader`脚本用于加载游戏和音乐。

```csharp
public class Loader : MonoBehaviour 
{
    public GameObject gameManager;          // 实例化 GameManager 预制体
    public GameObject soundManager;         // 实例化 SoundManager 预制体
    
    
    void Awake ()
    {
        // GameManager 为空时对其进行实例化
        if (GameManager.instance == null)
            
            Instantiate(gameManager);
        
        // SoundManager 为空时对其进行实例化
        if (SoundManager.instance == null)
            
            Instantiate(soundManager);
    }
}
```

因为游戏是关卡形式的，为了方便加载，`GameManager`和`SoundManager`都被做成了预制体，且都是单例。

## 游戏角色逻辑

---------
由于我们之前已经把动画的部分做完了，那么现在就可以进行角色逻辑的脚本编写。

角色逻辑用到了 `Raycast` 方法，这个方法可以向某个位置发射一条射线，假如途中碰撞到了某个物体，那么它就会返回碰撞物体的相关信息。当然，在这个 Demo 中我们主要是用来检测移动的方向上有没有障碍物，如果有的话就不允许角色移动。

?> 在此之前，记得要为各类预制体加上标签，为碰撞检测做准备。

## 关卡重载

---

关卡重载的这部分我没有写，不过可以给各位提供一个思路。我们可以把 GameManager 做成一个预制体，然后根据游戏的进程实例化预制体。至于如何保存食物和关卡进度，官网的代码中其实说的很清楚，各位可以自己去看看。