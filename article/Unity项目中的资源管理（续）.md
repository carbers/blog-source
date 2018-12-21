---
title: Unity项目中资源管理（续）
date: 2017-08-04 23:10:00
tags: ["Unity"]
---
上次和大家分享了[Unity项目中的资源管理](https://carbers.github.io/2017/07/09/Unity%E9%A1%B9%E7%9B%AE%E4%B8%AD%E7%9A%84%E8%B5%84%E6%BA%90%E7%AE%A1%E7%90%86/)主要讲资源配置以及资源配置工具，Unity资源配置在资源管理中处于基础地位，影响资源的增长速率以及量级。通过合理的资源配置，可以承载更多的资源，丰富游戏的内容。今天主要分享运行时的资源管理，探讨如何妥善的管理资源以达到内存与性能兼顾。从资源介绍开始，分析加载接口与对象池设计，然后讨论资源内容分级，最后分享一款轻量级内存Profile工具。

#### 内存与程序稳定性
iPhone 6& iPhone 6P只有1G的内存，而这两个机型在iOS平台上的市场份额超过40%。如果使用超量的内存游戏将闪退，这会带来极差的游戏体验。想象在进行激烈的战斗的时候，由于加载了更多的特效和模型，游戏突然闪退了。或许游戏有一套不错的断线重连机制，你还能回到战场。但基本上来说你很难获得这场战斗的胜利，这一个非常差的游戏体验。

同时在iPhone 6S以上的机型又有2G的内存可以使用，只要性能没有问题，完全可以承载更多的内容(资源)。在制作了过量了资源的情况下，如何妥善的管理资源是一个较大的挑战。一个项目一百多号人参与制作，如何协调工作，规整制作内容是一个头疼的问题。

**合理的资源管理方案兼顾性能与内存，提供一个稳定流畅的游戏环境。**

### Unity资源介绍
在做资源管理之前，首先我们要对资源有足够的了解，这样可以方便展开之后的工作。Unity官方已经有一篇非常精彩的文章来介绍Unity资源[《Assets, Objects and serialization》](https://unity3d.com/cn/learn/tutorials/temas/best-practices/assets-objects-and-serialization?playlist=30089)。
>An Asset is a file on disk, stored in the Assets folder of a Unity project. For example, texture files, material files and FBX files are all Assets. Some Assets contain data in formats native to Unity, such as materials. Other Assets need to be processed into native formats, such as FBX files.

>A UnityEngine.Object, or Object with a capitalized 'O', is a set of serialized data collectively describing a specific instance of a resource. This can be any type of resource which the Unity Engine uses, such as a mesh, a sprite, an AudioClip or an AnimationClip. All Objects are subclasses of the UnityEngine.Object base class.

Asset是指在Assets目录下的所有文件，在工程里面每个Asset会有一个对应的Meta文件，Meta文件用于描述Asset在工程里面的格式，之前分享的贴图配置也是通过修改Meta文件来达成。一个Asset包含一个或多个Object，这里的Obejct可以直接包含数据，也可以表示引用了其他Asset文件下的Object。

GameObject是一个特殊类型的Obejct，通常我们通过把一系列的Assets组装成Prefab（GameObject）来制作资源，Unity通过依赖关系加载所有资源。在加载一个GameObject之后，我们通常需要实例化GameObject。大部分Asset资源是共用的，实例化过程中Unity并不会复制这些共用资源，而是复制那些可修改的不可复用的数据，比如MonoBehaviour上的数据。当然我们也可以直接加载Asset资源来使用，比如直接加载一张贴图，放在一个UI面板上展示。通过依赖加载的贴图和直接加载的贴图是同一份贴图，Unity内部帮我们解决了资源重复的问题，可以放心使用。

#### Resources
>The Assets and Objects in all folders named "Resources" are combined into a single serialized file when a project is built.

Resources目录下所有的资源，都会被打包且可以通过Resources接口加载，加载路径为Resources目录的相对路径。支持同步与异步两个加载接口，支持单对象的UnloadAsset，还有一个清理未被引用的资源的接口。这里UnloadAsset不能卸载GameObject和Component，而且是强制卸载，即使外部仍然在使用这个资源。UnloadUnusedAssets则是一个安全的接口，只清理那些不再被引用的资源，不过这个接口开销较大会引起卡顿。
```
public static T Load<T>(string path) where T : Object;
public static ResourceRequest LoadAsync<T>(string path) where T : Object;
public static void UnloadAsset(Object assetToUnload);
public static AsyncOperation UnloadUnusedAssets();
```

#### AssetBundles
>An AssetBundle is an archive file containing platform specific Assets (Models, Textures, Prefabs, Audio clips, and even entire Scenes) that can be loaded at runtime.

通常推荐使用AssetBundle来加载资源，使用AssetBundle可以按更小的包来管理资源、更新资源，同时还可以加快游戏启动速度。更深入的内容可以看看Unity官方的文章[《The Resources folder》](https://unity3d.com/learn/tutorials/temas/best-practices/resources-folder)。

加载AssetBundle需要我们自己去维护依赖关系，对比起Resources来说更加麻烦。通常在开发的时候使用Resources加载，而在发布版本使用AssetBundle。这里需要实现自己的加载器来满足两套资源的切换。

### 资源管理器
- 统一Resources和AssetBundles加载
- 类似的加载接口设计，包括同步与异步
- 强引用计数管理，Load与Unload匹配
- 支持按优先级加载资源
- 支持配置系统开销，异步加载开销

对外实现为静态接口，正常情况下支持Editor运行时与非运行时，运行时不管在PC还是手机都支持Resources与AssetBundles无缝切换。所有的加载路径参数统一为Resources目录相对路径且不包含扩展名，这里要求在同一目录不要有同名文件（仅扩展名不一样）。按类型匹配资源是较烦琐的工作，而且对于Object基类加载，无法匹配到正确的资源。

异步接口定义一个自己Request类返回，除了原有的ResourceRequest数据，这里新增一个打断属性。当不再持有这个对象的时候设置打断属性来中断加载。同时这里还支持配置回调接口，这样不需要每次更新去查询状态，资源管理器在异步加载完成后执行回调接口。

异步加载接口增加优先级参数，优先加载高优先级的对象。自己维护一个优先级列表，并发起一定数量的异步加载请求，对于在队列中被打断的资源则可以节省一次资源价值请求。

然后还要关注异步加载的开销，避免异步加载占用太多的主线程时间。Unity可以通过配置[Application.backgroundLoadingPriority](https://docs.unity3d.com/ScriptReference/Application-backgroundLoadingPriority.html)来约束开销。如果要求游戏跑30帧的话，建议配置为Normal即可，在过场景的情况下，则配置成High来提高加载速度。
- ThreadPriority.Low - 2ms;
- ThreadPriority.BelowNormal - 4ms;
- ThreadPriority.Normal - 10ms;
- ThreadPriority.High - 50ms.

由于实现了自己Request，所以这里也要实现自己的时间片管理器。实例化对象与回调接口的开销都是不可预期的，我们配置一个每帧最大执行时间做平滑。

最后讨论下资源卸载策略，实时卸载资源导致资源反复加载，引起游戏卡顿。通常会缓存一定数量的资源来改善体验，对于非GameObject和Component的Asset资源，可以使用UnloadAsset接口直接卸载。剩下的GameObject，通过取消资源管理器对对象的持有再调用Resources.UnloadUnusedAssets接口来卸载。由于我们使用了强引用计数管理，所以在清理的时候通过对引用计数的判断就可以正确的清理资源。特别对于使用AssetBundle加载资源的情况，错误的管理可能会导致资源重复加载，浪费内存。

#### 资源对象池
资源加载器负责加载、卸载资源，同时缓存资源，这里的资源对象池特指**GameObject资源池**。GameObject资源通常带有自己的数据，在加载的时候需要实例化一份以便使用。实例化GameObject是一个开销较大的操作，同时也会带来较高的GC Alloc（内存分配）。资源对象池就是一个GameObject对象池用于缓存实例化的GameObject对象。

资源对象池在使用上要注意GameObject对象的可复用性，开始的时候加载一个预制体（Prefab）是一个干净的数据。外部逻辑会修改GameObject上的数据、添加新的组件，之后这个对象会入池。设计上如果一个对象需要使用对象池的复用功能，逻辑需要保证这个GameObject是可复用的，这并不是一件容易的事情。把状态还原重置本身就有一定的开销，如果实例化一个对象的成本低于重置数据的开销，那就不需要对象池，每次重新实例化即可。

同时在对象入池的时候还需要做一项工作是让对象不可见，与销毁一个对象（对象入池）在这里保持行为一致。有两个常见的做法，一个是SetActive(false)，还有一个做法是把对象移出摄像机。对于对象数特别多的对象修改坐标的开销较大，对于组件较多的对象修改激活状态的开销可能会更大。这里提供了三种入池行为，InActive、InVisible、Destroy用于处理上面讨论的情况。

资源对象池封装实现自己的一个Spawn接口，表示生成一个对象，然后对应的一个Despawn接口用于销毁对象。这里还提供了异步的SpawnAsync接口用于异步加载以及错帧实例化，使游戏体验更加平滑。对于Spawn接口提供带初始坐标的实例化接口与Instantiate保持一致，提供初始坐标减少坐标次数，一般来说可以得到5%-10%的性能提升。对于一些拖尾特效，正确的坐标也可以避免特效拉一条从原点到当前位置的长线。

最后讨论下资源池的缓存策略，通常资源池里面存在两种情况的资源。一种外部还存在相同的对象在使用，另一种则是所有的对象都在资源池。对于所有对象都在资源池的对象，可以认为是不使用资源根据时间淘汰。对于外部存在引用的情况，增加其权重值但还是会按时间来淘汰。存在部分类型资源会有较多的实例而部分资源只有一两个实例，这里做资源池总上限的约束而不做单类型数量约束。在激烈的战斗场景下对象数量会远远高于平时，过小的资源池上限会导致卡顿，过大的资源池上限会导致内存过高。这里增加一个资源池下限，当资源池对象数高于这个数目的时候执行按时间清理操作，然后配置一个较高的资源池上限而不用当心资源池一直占用过高的内存，得到一个性能与内存兼顾的结果。

### 资源内容分级
当资源的使用上超标时，也可以通过简单的调整一些参数来开关这些对内存有较大影响的对象。
观察iPhone机型内存可以发现内存有较大的跨越，在2G机型可以承载游戏内容的情况下，1G机型承载不了这么多的内容。通过对资源内容进行分级，来稳定1G机型的内存使用，避免闪退。
|机型|内存|
|:--|:--:|
|iPhone 5 – 6P|1 GB LPDDR2/LPDDR3 DRAM|
|iPhone 6S – 7|2 GB LPDDR4 DRAM|
|iPhone 7P|3 GB LPDDR4 DRAM|

#### 常见的分级内容
- 屏幕后期效果
- 高低材质
- 贴图大小减半

```
void OnRenderImage(RenderTexture source, RenderTexture destination)
```
如果通过上面的OnRenderImage实现屏幕后期效果，这里的source和destination贴图都是Unity申请的与分辨率直接挂钩。在1080P的分辨率情况下，会消耗掉50M左右的内存。所以一个比较好的做法是在低内存机型上关掉这个效果。

高级的材质使用更多的顶点数据与贴图，比如法线贴图、通道贴图。低级材质使用更少的贴图，通过高低材质的切换可以减少贴图到达节省内存的效果。Mesh这里也是同理，如果不需要法线则不需要有法线的顶点数据。然后缩减贴图大小也是一个不错的方法，不过保存两份贴图会使包文件变大。

#### 配置资源管理参数
前面我们为了得到一个较好的性能做了较多的资源缓存工作，针对不同的内存配置不同的参数达到优化内存的目的。内存不够带来的体验是游戏直接闪退，所以这里认为游戏稳定性的优先级高于游戏卡顿。

这里主要配置Assets资源缓存数量，资源池的上限与下限，还有一些资源清理时间间隔的参数配置。这些数据可以方便的修改，通常经过一系列的压力测试可以得到一个安全配置参数。后期如果增长导致内存不够，则可以通过修改配置参数来达到稳定游戏的目的。

### 简洁的内存管理机制
想要解决内存闪退，了解闪退时的内存使用情况是很有必要的。在游戏运行过程中，我们可以记录内存使用情况。同时可以对资源类型进行分类，了解细节。Unity的Profile工具虽然非常方便，功能也足够强悍。但是没有数据落地，而且采样占用额外的内存。这里自己实现了一个简易的内存Profile工具，支持数据落地方便对比，同时不占用过多的额外内存。

在了解到闪退时的内存情况后，我们可以很容易就了解是在什么样的情况下内存会不够用。有哪些地方的内存使用超标，是否有可以优化的余地。极限情况下最低内存使用量。

#### 内存Profile工具
```
Resources.FindObjectsOfTypeAll(Type type);
Profiler.GetRuntimeMemorySize(UnityEngine.Object o);
Profiler.GetMonoHeapSize();
Profiler.GetTotalAllocatedMemory();
```
通过Resources.FindObjectsOfTypeAll获取当前所有的对象，通过Profiler.GetRuntimeMemorySize计算每个Object的内存大小，通过Object.name可以获取对象的名字。了解了这些信息可以实现一个简洁的内存Profile工具，对比起Unity提供的Profile工具，自己实现工具可以比较方便的做一些数据落地以及自动采样的过程。

同时这个Profile工具还和自己实现的资源管理器进行了整合，可以记录当前的Assets数量，GameObject数量、缓存数量，以及引用计数为零的对象数量。这些额外的数据有较大的参考价值也可以直接记录，方便做后续的对比以及观察数据趋势。同时Profile工具还支持导出资源列表，之前提到我们的加载接口是通过路径加载做强引用计数管理。这里可以输出每个资源的路径与引用计数，可以定位资源泄露，排查资源残留的情况。

数据以文本的形式记录，支持自动采集上报，之后可以对这些数据做图形化显示、分析。每次跑游戏都可以得到一份数据报告，对比数据报告可以对客户端内存使用趋势有一定的认识。避免出现内存不够导致游戏闪退的情况。

![内存Profile工具](https://upload-images.jianshu.io/upload_images/3503018-03d87ed0f242fed3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

上图是内存Profile工具在PC上采样的结果，这里对数据进行了分类，按类型和使用场景分类。通过这些数据可以得出一些指标，比如贴图不能超过50M，Mesh不能超过20M。不同场景下的资源使用情况是不同的，这里做的工作就是把50M分配给各个模块。这样做的一个好处是能找到一个人负责，同时这个人又是对这个模块最熟悉的。

#### 兼顾内存与性能
- Android 高内存，低CPU，低I/O
- IOS 低内存，高CPU，高I/O

资源加载是一项非常慢的操作，如果所有的资源都实时释放，那下次加载资源带来的卡顿也会带来较差的游戏体验。由于iPhone机型内存少，加载快，可以做实时释放策略。对于Android机型内存多，加载慢，可以做预加载策略。同时还可以做带权重的资源缓存策略，资源缓存是由资源的最后使用时间和加载时间得到一个权重，优先释放加载快不经常使用的资源。这样可以在内存和性能上得到一个较好的照顾。同时在IOS上会有内存的Warning警告，当触发警告的时可以做强制性清理，避免游戏闪退。

[完 Carber 2017-08-04]