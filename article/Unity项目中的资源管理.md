---
title: Unity项目中的资源管理
date: 2017-07-09 23:10:00
tags: ["Unity"]
---

 这是我在2017金山技术开放日分享的部分内容。从贴图资源格式配置的介绍开始，引申出资源配置工具，最后再谈谈一整套项目资源管理方案。在GitHub上可以获取到资源配置工具的代码，是基于下面理念的一份简单实现。

一个3D游戏项目中的资源主要由贴图、模型、动作、声音等组成。模型设计师设计模型，场景由模型组成，角色不仅是一个精致的模型还带了各类的动作，包括走路、跑步、待机、攻击等等。各类基础资源构成了游戏的大部分元素。

在我做的剑侠世界手游项目里面有上万的材质，主城更是有7000+的模型。每个角色可以有不同的外装、发型、武器、挂件、坐骑等等。承载这么多资源对内存以及性能都有非常高的要求。一个优秀的资源管理方案可以承载更多内容，满足更多的需求。

#### 贴图资源配置
对于这资源管理，Unity提供非常丰厚的支持。以贴图为例子，Unity支持直接把原始贴图直接放进工程，不需要做任何额外处理。Unity根据贴图配置会自动生成最后的贴图数据。不同平台（IOS、Android、PC）支持的贴图格式不一样，通过配置文件的形式，最后方便的生成不同格式的贴图。这是一个非常好的做法，同时可以直观的在开发环境看到不同格式的贴图表现。

![贴图的配置界面](http://upload-images.jianshu.io/upload_images/3503018-24bec1d02b17eda6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


可以看到上图，贴图的配置属性非常多。不同的贴图会有不同的配置，有些属性的配置对内存和性能会有较大的影响。下图可以看到现在这是一张 512 X 512的RGB贴图，内存大小为1M。如果限定使用50M的贴图内存的话，那只能加载50张贴图，这明显是不够用的。

![贴图初始大小](http://upload-images.jianshu.io/upload_images/3503018-e4fd86184bd38619.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

三维计算机图形渲染中一个常用的技术被称为Mipmap。为了加快渲染速度和减少图形锯齿，贴图被处理成由一系列被预先计算和优化过的图片组成的文件。对于UI使用的2D贴图，我们需要关闭Mipmap避免精度损失。同时在测试中我们也发现开启Mipmap并没有带来多少性能提升，在手游的场景中，我们的视距非常近，并没有过多这方面的需求。同时关闭Mipmap我们能节约33%内存，这是一个比较不错的提升。关闭Mipmap后，我们看到贴图变成了0.8M，这里应该是768KB。

![贴图关闭Mipmap](http://upload-images.jianshu.io/upload_images/3503018-93727ab9a884df5b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

一个不小的提升，不过还是要注意，这是测试之后做的决定，不同情况下的决策是不一样的。

接下来讨论贴图压缩，ETC是所有Android都接受的格式，这个格式的压缩质量较差。但在大部分情况下，很难在手机那么小的屏幕上看到差异，渲染本身就是一件视觉欺骗，如果觉得满意那就没问题。ETC 2对图片质量有较大的提升，但是需要 OPGL ES 3.0 以上才支持，这个要根据支持的机型做决定。这里不对压缩格式做过多的讨论，详情见[Unity官方文档](https://docs.unity3d.com/Manual/class-TextureImporterOverride.html)。

设置压缩格式为ETC后贴图的大小为128KB，减小了6倍，这是一个非常大的提升。

![贴图压缩后](http://upload-images.jianshu.io/upload_images/3503018-2b93c83e7ad74d15.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

再来说说Max Size这个属性，真的需要512X512这么大的一张贴图么，256X256是不是足够了。做出这样的决定，然后在真机上看看效果，贴图大小限制成256后，最后的贴图大小为32KB。总计减少了32倍，现在我们可以加载1600张贴图了，相信这个数字能满足大部分项目的需求了。

![贴图设置最大尺寸后](http://upload-images.jianshu.io/upload_images/3503018-e1e760884541ab0a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

最后来说说Read/Write Enable这个属性，如果打开这个属性，会使运行时贴图大小翻倍，内存中会额外存储一份贴图数据。在大部分情况下这是不必要的，所以默认关闭这个属性，在需要的时候开启。

在这种最优的配置下我们使贴图大小缩小了64倍，从开始的25张到最后的1600张，令人惊奇的优化。资源格式配置在整体资源管理中属于最基础的模块，但同时也是最重要的模块，他决定了你能显示多少资源。

#### 资源配置工具
正常情况下贴图资源使由美术提交，不同种类的贴图有不一样的配置文件，Ios的配置格式和Android也会有区别。很难要求美术对资源配置这块有深入的理解，同时每次都要配置一些属性也显的非常麻烦。这时候就需要程序来帮忙做一些工作了。

首先根据前面的认识，我们抽离出部分属性设置。
```
public class TextureImportData : ImportData 
{
    public TextureAlphaMode AlphaMode = TextureAlphaMode.FormTexture;
    public TextureImporterType TexType = TextureImporterType.Default;
    public TextureImporterShape ShapeType = TextureImporterShape.Texture2D;
    public TextureImporterFormat AndroidFormat = TextureImporterFormat.ETC2_RGB4;
    public TextureImporterFormat IosFormat = TextureImporterFormat.PVRTC_RGB4;
    public bool ReadWriteEnable = false;
    public bool MipmapEnable = false;
    public int MaxSize = -1;
}
```
然后可以给一个贴图应用这个配置
```
public static void FormatTexture(string path, TextureImportData data)
{
    TextureImporter tImporter = AssetImporter.GetAtPath(path) as TextureImporter;
    if (tImporter == null) return;

    if (tImporter.textureType != data.TexType)
    {
        tImporter.textureType = data.TexType;
    }
    tImporter.isReadable = data.ReadWriteEnable;
    tImporter.mipmapEnabled = data.MipmapEnable;

    if (data.MaxSize > 0)
    {
        tImporter.maxTextureSize = data.MaxSize;
    }

    TextureImporterPlatformSettings settingAndroid = tImporter.GetPlatformTextureSettings(EditorConst.PlatformAndroid);
    settingAndroid.overridden = true;
    settingAndroid.format = data.GetFormatByAlphaMode(data.AndroidFormat, tImporter);
    settingAndroid.maxTextureSize = tImporter.maxTextureSize;
    tImporter.SetPlatformTextureSettings(settingAndroid);

    TextureImporterPlatformSettings settingIos = tImporter.GetPlatformTextureSettings(EditorConst.PlatformIos);
    settingIos.overridden = true;
    settingIos.format = data.GetFormatByAlphaMode(data.IosFormat, tImporter);
    settingIos.maxTextureSize = tImporter.maxTextureSize;
    tImporter.SetPlatformTextureSettings(settingIos);

    tImporter.SaveAndReimport();
}
```
最后和美术做了一个约定或者说规范，可以按目录和一个正则来批量设置贴图格式。

一般来说十几条规则就能覆盖所有情况，如果有疏漏，增加一个规则即可。这里还要需要考虑一个细节问题就是如果一个文件被多个规则覆盖怎么办。设想一开始添加的是一些通用规则，后面添加的则是一些特殊规则。而且按照一条条规则去设置属性，后面的规则也会覆盖前面的规则。所以这里只用引入一个规则配置的顺序属性即可，顺序可以修改，以顺序靠后的规则为准。然后为了方便查看，还可以记录当前规则下贴图的数量以及大小，这样有一个更加直观的认识。
```
public class ImportData {
    public string RootPath = "ArtWorks";
    public string FileNameMatch = "*.*";
    public int  Index = -1;
    public int  TotalCount = 0;
    public int  TotalMemuse = 0;
    public bool PreBuild = true;
}
```
设计完了数据，最后就是实现一个窗口界面方便来配置这些属性，同时可以把需要的信息都显示窗口上。窗口是直接用Unity的API实现了，还是非常方便的。

![贴图格式化工具](http://upload-images.jianshu.io/upload_images/3503018-ebbb0bc8ed96ba75.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

最后我把资源配置工具交给技术美术(Tech Artist)来使用，美术这边有需求有调整，直接配置即可，不需要继续来参与。

#### 项目中的资源管理
然后考虑是否有资源漏配，资源配置是否合理，目前我们资源处于一个什么样的情况？需要一个方案来获取更多信息。上面的工具已经看到能统计贴图的资源大小，实现一个函数可以导出资源使用情况的统计，以及按大小排序的资源列表。然后把每次统计结果保存下来，对比每天的统计结果可以观察到项目中的一个资源增长情况。

![贴图统计列表](http://upload-images.jianshu.io/upload_images/3503018-1c02982b171c2cd1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我特意选了一张有问题的贴图列表，可以观察到如果配置有问题，很容易在列表中暴露出来。

还有一些想法没有实践，比如不仅显示贴图大小，还显示贴图的压缩格式。知道这张贴图的大小是2M，但是不知道是压缩前2M还是压缩之后的。同时项目中不可避免的还是存在重复贴图，可以考虑程序实现一个功能做重复贴图对比等等。

不过要考虑投入成本与收益，关注Top10与每日的资源变化已经对项目有一定程度上的把控。可以把精力放到更关键的产品开发上，这里实际做了产品质量的保障。

#### 总结
从关注资源本身配置的影响，到如何设置资源配置，到最后一个落地的总体预览数据。所有的东西都非常简单，主要分享的是思考的过程与方式。虽然这里只是介绍了如何在Unity项目下的工作，但很多思路和方法都是共用。不过这只是我在当前项目经验下的一下总结，不同的项目有不同背景与问题。还是需要针对问题本身深入思考，提出切实可行的方案。

同时这里的很多工作是展示了更多的信息，这些信息是整个项目都是可以观察到的。越多的人了解一个事物，那修改这个事物就会显得更加的容易。比如美术也能观察到我们项目资源量爆炸了，然后就下意识的控制资源量的增长。

#### 附
[资源配置工具](https://github.com/carbers/UnityComponent/tree/master/Assets/ResourceFormat)