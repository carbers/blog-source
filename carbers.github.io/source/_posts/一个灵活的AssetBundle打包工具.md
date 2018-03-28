---
title: 一个灵活的AssetBundle打包工具
date: 2017-07-13 23:10:00
tags: ["Unity", "AssetBundle"]
---
上周介绍了[Unity项目中的资源配置](https://carbers.github.io/2017/07/09/Unity%E9%A1%B9%E7%9B%AE%E4%B8%AD%E7%9A%84%E8%B5%84%E6%BA%90%E7%AE%A1%E7%90%86/)，今天和大家分享一个AssetBundle打包工具。从资源处理开始，再到Bundle工具设计，最后谈谈项目中Bundle打包方案。

Unity提供了两种资源加载方式，一种是Resources，另外种就是AssetBundle。所有的资源只要放在Resources目录下，在打包的时候会自动打进去，并可以通过相应的接口加载。正常情况下Resources非常方便，可以满足日常的需求，但资源放Resources会带来资源更新上的问题。之前写过一篇文章[Unity资源目录及加载接口介绍](http://www.jianshu.com/p/5206c7dc88c4)可以了解些细节。

假设首包所有资源都放Resources，后续更新资源的走AssetBundle，会发现AssetBundle和Resources的资源互相不兼容。当调整一个模型的材质参数后，对模型进行打包仍需要把Mesh，Texture等资源都打进去。这会导致更新包过大，同时在加载这个模型时，这些资源是不共用的，相同的资源可能在内存中存在两份。所以正常情况下，项目发布时所有需要更新的资源要打成AssetBundle。

正常项目中资源的提交与变更非常频繁，手工对每个资源配置Bundle费时费力，基本不可取。所以一般项目中的Bundle都是程序自动创建的。同时为了避免有多余的资源被打包，通常需要配置哪些资源是发布资源（直接加载的），其他资源通过引用的形式获取。这个配置需要方便修改，来满足日常变更。

Bundle的打包规则对资源加载速度，更新大小，重复资源数量以及最终包数量等等都有较大影响。**一个可靠的Bundle打包方案应该是根据实际情况对Bundle打包规则做调整慢慢产生的。**

在Unity 4，只有最基础的几个打包接口可以用于打包。Unity 5简化了Bundle打包时候的依赖关系，但实际如何创建Bundle以及对依赖资源的配置都节省不了。远远不能满足项目对资源打包这块的需求。

这里实现的AssetBundle打包工具帮助简化这个繁琐的打包过程，同时方便做规则调整，得到更优的打包方案。目前工具[BundleBuildTool](https://github.com/carbers/UnityComponent/tree/master/Assets/BundleBuildTool)已经放在GitHub，可以作为一份打包实现的参考，也可以直接使用这工具来进行打包。

#### AssetBundle
>An AssetBundle is an archive file containing platform specific Assets (Models, Textures, Prefabs, Audio clips, and even entire Scenes) that can be loaded at runtime.

#### 资源类型
不同类型资源会有不同的打包方式，比如场景文件的打包接口和其他资源的打包接口就是不一样的。通过定义不同的资源类型，可以实现不同的打包方式，支持更多资源的打包。
```
public enum BundleType
{
    None = 0,
    Script,         // .cs
    Shader,         // .shader or build-in shader with name
    Font,           // .ttf
    Texture,        // .tga, .png, .jpg, .tif, .psd, .exr
    Material,       // .mat
    Animation,      // .anim
    Controller,     // .controller
    FBX,            // .fbx
    TextAsset,      // .txt, .bytes
    Prefab,         // .prefab
    UnityMap,       // .unity
}
```
对于特殊类型的资源，通过类型可以做一些定制化操作。比如把所有的Script配置在一个Bundle里面，然后在启动的时候对这个Bundle做预加载。通常情况下也会把所有的Shader配置到一个Bundle里面。

正常一个模型会有自己的Texture，Mesh & Animation，把资源按类型打成三个包，在加载的时候可以得到更高的加载速度。Unity异步加载接口会同时进行多个资源加载，资源配置在不同的包里，可以有较好的加载速度提升，所以一般是按资源类型来进行打包。不过要注意如果太分散的话，一样会影响加载速度。

资源加载速度这个是在文章[Asset Bundles vs. Resources: A Memory Showdown](https://blogs.unity3d.com/2017/04/12/asset-bundles-vs-resources-a-memory-showdown/)提及。
>These blocks sizes are optimized for loading multiple Assets and bundles in parallel. For example, you should be able to load objects from 4 to 5 Asset Bundles at the same time without the the allocators for Asset Bundle Async loading or Type Trees needing new blocks.

#### 资源依赖
处理资源依赖应该是打包过程最复杂的一块功能，这里把获取资源依赖文件列表单独设计一个类，做一些特殊情况处理。如果发现一些依赖关系上的错误，除了修改资源本身外，也可以在打包环节实现一些脚步做保障。

正常情况下，通过AssetDatabase.GetDependencies即可获取一个资源的所以依赖文件。但实际情况中，Unity内部是通过分析内部guid来生成依赖文件。有时候在文件里面会存在一些脏的guid这会产生多余的依赖。比如你修改一个材质贴图属性名，然后设置了一张新的贴图给这个新的属性名。打开材质文件会发现旧的属性名以及引用guid出现在材质文件，通过GetDependencies获取的最后结果也包含这个数据。实现自己获取依赖函数来处理这种多余依赖关系。同时提供带缓存接口，提高打包效率。

下面是对材质依赖贴图文件获取的代码实现。
```
...
MaterialProperty[] proTes = MaterialEditor.GetMaterialProperties(new Object[] {mat});
for (int i = 0; proTes != null && i < proTes.Length; ++i)
{
    if (proTes[i].type == MaterialProperty.PropType.Texture)
    {
        Texture tex = mat.GetTexture(proTes[i].name);
        string path = AssetDatabase.GetAssetPath(tex);
        if (!dict.ContainsKey(path))
        {
            dict.Add(path, path);
        }
        Resources.UnloadAsset(tex);
    }
}   
...
```
#### 资源剔除
处理完资源依赖后，还碰到一个问题就是最后打包Assets资源。通过AssetDatabase.LoadAllAssetAtPath获取这个文件依赖的所有的Assets资源。如果对所有的这些Assets资源都做打包的话，会发现一些编辑器用数据也会被打包进去。特别是对于FBX类型文件，通常会存在一个"__preview_Take 001"的动作资源使包体变大很多。对于这些不必要的数据，在打包环节中增加一个剔除规则，减少包体大小。
```
public static List<UnityEngine.Object> FilterObjectByType(UnityEngine.Object[] assets, BundleType bundleType)
{
    List<UnityEngine.Object> ret = new List<UnityEngine.Object>();
    foreach (UnityEngine.Object asset in assets)
    {
        switch (bundleType)
        {
        case BundleType.FBX:
            if (!(asset.GetType() == typeof(AnimationClip) && asset.name == "__preview_Take 001"))
            {
                ret.Add(asset);
            }
            break;
        default:
            ret.Add(asset);
            break;
        }
    }
    return ret;
}
```
Unity 5刚出的时候会把这个数据打进AssetBundle造成包体过大，后面版本观察已经修复这个问题。不过也可以发现这个环节的必要性，如果发现资源出问题在这个环节处理即可。

这个环节不仅可以剔除不必要的数据，还可以直接修改数据本身。就拿Mesh数据举例，美术在制作过程中会导出多余的顶点数据在文件里面(uv3,uv4...)。通常配置Optimize Mesh可以干掉这些无用数据，不过直接启用可能会出现删除了需要数据情况，比如color数据丢失。所以自己来做，通过把Mesh对象上不需要的对象数据置空，然后再打包即可。在之前分享的资源配置工具里已经做了对Mesh顶点数据的配置，基本上就是为这个打包环节服务，因为无法修改FBX文件，只能美术重新导出。

#### 资源大小
资源大小影响最后的包体大小，如果对包体大小以及更新量有关注的话，对资源大小做预估是一个非常有必要的环节。在资源大小计算环节，不能疏漏之前二个资源环节对资源的处理，同时不同类型的资源统计方式不一样。

通常通过下面两个方式预估资源大小
```
int resSize = UnityEngine.Profiling.Profiler.GetRuntimeMemorySize(asset);
FileInfo fileInfo = new FileInfo(assetPath);
int fileSize = fileInfo.Length;
```
如何对一个资源做一个大小估算，并不是一件非常方便的事情的。如果依赖资源已经在之前打包了，那这个资源的实际大小是要考虑减去依赖资源那部分的大小。如果不统计依赖资源的大小，那这个资源的包的大小也是不准确的。所以这里的实际逻辑较为复杂，但实际一个大致的值就可以了，然后观察最后的包大小做一些配置微调即可。

#### Bundle模型
讨论完资源上的一些细节，下面开始Bundle设计的介绍。一个Bundle模型用name做唯一标识，为了方便管理加入了parent与children数据。同时一个Bundle应该有一个固定资源类型。为了方便对包大小做限制加入了size属性，作为资源大小的预估。
```
public class BundleData
{
    public string name = string.Empty;
    public string parent = string.Empty;
    public BundleType type = BundleType.None;
    public BundleLoadState loadState = BundleLoadState.UnloadImmediately;
    public int size = 0;
    public List<string> includs = new List<string>();
    public List<string> children = new List<string>();
}
```
最后一个Bundle包含多个资源文件路径。尽管AssetBundle是按Assets打包的，但在正常环境下的资源是以文件存在的。一个资源文件可能包含多个资源，也可能引用到其他资源。资源文件可以用路径来标识，Unity内部通过GUID来标识资源文件，所以即使你挪动文件因为GUID不变，还是可以找到这个文件。这里决定直接用资源路径来标识资源而不是使用GUID，因为挪动资源目录有较多的风险，原则上禁止挪动资源。如果真挪动了资源，按最新的资源路径生成Bundle是一个不错的选择。

**如果有对Bundle有其他属性上的需求，在这个类扩展就好。**

#### Bundle创建规则
定义Bundle后，创建Bundle是很困扰的一个问题。在大型项目中，资源的量非常大，资源之间的互相引用也较为复杂。这里定义一个数据结构帮忙创建Bundle。
```
public class BundleImportData
{
    public string RootPath = "";
    public string FileNameMatch = "*.*";
    public int Index = -1;
    public int TotalCount = 0;
    public BundleType Type = BundleType.None;
    public BundleLoadState LoadState = BundleLoadState.OnUnloadAsset;
    public bool Publish = false;
    public int LimitCount = -1;
    public int LimitKBSize = -1;
    public bool PushDependice = false;
    public bool SkipData = false;
}
```

对于一个Bundle，可以约束它的大小，对象数量、类型、加载方式、打包方式。然后根据规则，自动给每个资源文件配置Bundle。

**资源分加载资源和被依赖引用到的资源，对于直接加载的资源，需要配置Publish为True。Bundle创建就是从这些配置了Publish的资源文件以及其依赖生成的。**

对所有可能被打包的资源配置打包规则，没有被配置资源文件，则会被一起打倒最后资源的包里面。这里会碰到一个问题，有些资源需要补分包，但是通用规则会包含不需要分包的资源。这里增加了一个**SkipData**属性，当为True时这些资源不单独创建Bundle。

然后讨论下PushDependice属性，正常情况下只有在打Prefab类型的资源的时候才会做这个操作。因为Prefab数据本身是不共享的，然后避免Prefab与Prefab之间的复杂依赖。

最后讨论下打包的顺序，因为资源之间有互相依赖，所以需要配置资源的打包顺序。这里资源的打包顺序就是BundleImportData创建的顺序。这里需要对资源之间的依赖以及资源类型有一定的认识。

**已经配置过Bundle的资源不会变更，新增的资源会按规则配置相应的Bundle。通常规则发生变更会影响非常多的资源，如果所有资源重新配置会导致更新包过大。**

#### Bundle构建
首次创建的Bundle，由于本地文件不存在，会触发构建。然后资源之间有互相依赖，所有被依赖的Bundle也需要参加构建。对于增量构建，这里做了一个简化设计，不自己去计算文件是否变更，而是由外部提供一个文件变化列表。通过这个列表工具自动生成Bundle构建列表，提高打包速度。

在配置打包参数为**BuildAssetBundleOptions.DeterministicAssetBundle**后，如果不对资源做修改，两次打包的文件是一样的。所以即使有很多资源因为依赖要重新打包，最后的文件未发生变化，就不会触发更新。

#### Bundle索引
Bundle构建完后只是一堆二进制文件，需要根据Bundle之间的依赖关系生成出一份数据。除了需要知道Bundle之间的依赖之外，同时还需要知道资源路径与Bundle之间的映射关系。最后还要把Bundle状态信息保存下来，用于Bundle更新、加载和卸载。
```
public class BundleState
{
    public string bundleID = string.Empty;
    public uint crc = 0;
    public uint compressCrc = 0;
    public int version = -1;
    public long size = -1;
    public BundleLoadState loadState = BundleLoadState.OnUnloadAsset;
    public BundleStorePos storePos = BundleStorePos.Building;
}
// like UnityEngine.AssetBundleManifest
public class BundleManifest { ... }
```
这个文件自己定义形式，可以使分散的多个文件，也可以统一放到一个文件里面，自己实现可以优化数据结构减少内存开销。

#### 通用的Bundle打包方案

下面是在Unity Standard Assets资源上做配置后的结果
![BuildBundleTool](https://upload-images.jianshu.io/upload_images/3503018-35ae7aca0795e1d6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)
按大小配置基础资源，然后对于Prefab和Unity文件限定下个数，避免过多的资源依赖。配置结束后点击CreateBundle就可以得到下面的结果。这里的21.3MB是由于一张贴图本身就有这么大，所以这里还是会出现很多大于2M的包，这就需要根据实际情况做微调。


[完  2017-07-13 Carber]

