---
title: Unity中资源依赖关系获取效率优化
date: 2018-08-11 20:30:00
tags: ["Unity"]
---

今天和大家分享一个优化经验，主要关于获取一个资源的依赖资源列表即对AssetDatabase.GetDependencies这个接口的调用效率优化。通过一步步优化最后在对工程中所有资源获取依赖资源的执行上提升了近100倍的效率。

在对AssetBundle进行打包时候，需要获取资源的依赖关系，并生成最后所有资源的BundleName。这里主要的瓶颈就是对资源的依赖关系数据获取上。在工程实践中发现整个构建环节20分钟，16分钟是BuildAssetBundles开销，3分钟是GetDependencies开销。在增量构建中，BuildAssetBundles可降为1-3分钟，而GetDependencies则仍需要3分钟开销。当然对于资源数量较小的工程，这个优化就是一个可有可无的选项对构建速度影响不大。

还有一个常见的应用场景就是快速查找资源资源的依赖数据以及被依赖数据，也可以通过这次的优化带来体验上提升。

## 一
首先从分析AssetDatabase.GetDependencies这个接口的行为开始，简单的编写一个测试函数：

```
public static void Test()
{
    long timeStamp = Stopwatch.GetTimestamp();
    string[] dir = Directory.GetFiles("Assets/", "*.*", SearchOption.AllDirectories);
    for (int i = 0; i < dir.Length; ++i)
    {
        if (dir[i].EndsWith(".meta", System.StringComparison.OrdinalIgnoreCase))
        {
            continue;
        }
        AssetDatabase.GetDependencies(dir[i], true);
    }
    UnityEngine.Debug.LogFormat(
        "GetDependencies cost {0} ms.", 
        (Stopwatch.GetTimestamp() - timeStamp) * 1000 / Stopwatch.Frequency
    );
}
```
通过执行这个函数可以了解这个函数的开销，以及获得数据为之后的优化做对比。

第一次执行的时候较慢，有较高的硬盘读写，总共花费6.3mins。

第二次执行的时候快了近一倍，基本无硬盘读写，总共花费3.2mins。

这里硬盘是使用SSD，如果使用机械键盘则这里性能堪忧，第二次基本所有内容都进了内存，操作系统做了缓存，所以快了很多。所以换更好的硬盘可以提高这里的效率，不过现在的执行时间还是太长了。

如果对所有的资源进行扫描通过GUID去查询并获取依赖关系，那应该不止这么点时间。这里猜测Unity做过一些数据预处理与缓存来优化这个接口效率。

这时候第一个优化思路是Cache，通过缓存每次结果下次查询时可以立即返回结果。不过由于资源会修改，依赖文件会发生变化，所以缓存可能会出错。如果不能判断当前缓存是否有效，则只能在确保资源部修改的情况下使用缓存数据。

这里使用**AssetDatabase.GetAssetDependencyHash**来验证缓存是否有效，这个接口返回Asset的一个Hash值（包括文件名以及meta文件），如果Hash值不变，我们可以认为这个Asset直接依赖的资源文件不变，由于直接依赖是通过Asset文件内部的GUID索引的，所以Hash不变即表示GUID不变，即依赖关系不变。这里缓存Hash值以及这个Asset的直接依赖。通过所有的直接依赖，可以快速的计算出这个Asset的全部依赖。


AssetDatabase.GetAssetDependencyHash接口非常高效，这里简单讨论下。

这部分数据在Import Asset的时候计算并缓存，所以可以高效获取。每个Asset都有自己的AssetDependencyHash，Reimport的时候重新计算。这里判断文件是否修改的依据是文件最后修改时间是否发生变化。获取目录下所有文件信息由于有操作系统文件系统做了索引是非常高效的。

由于Refresh是一个必要项，这项开销已经花费出去了，所以这里可以直接享受接口的高效率。

最后只要把每次的数据保存在本地，下次使用的时候先从本地加载，即可使这部分逻辑时间优化到2800ms左右，优化了近100倍，这里的执行效率已经非常优异了，主要开销在GetFiles上。


## 二
上面只讨论了有缓存数据情况下的优化情况，但实际缓存数据的加载和保存时间却被忽略了。实际结果是这部分的数据量较大，加载和保存开销也比较大，如果使用Json来存储的话，这里大概要花费1.5mins来读写这数据。

这里讨论下对这个数据存储效率的优化，首先来看看数据结构：

```
public class DepenData
{
    public string assetPath;
    public Hash128 assetDependencyHash;
    public string[] dependsPath;
}
// save data
Dictionary<string, DependData> m_data;
```

基本都是字符串数据，存储出来的文件都有300M左右（大概，具体忘了），把Json存储改为二进制以后，文件大小缩减为75M左右，加载时间从1.5mins变成了18.3S。较大的改进，不过还可以在改进我想。

这里依赖数据是递归即 A依赖B，B依赖C，在A的DependData里面就会有ABC，而B的依赖数据里面有BC。这里可以发现BC出现了两次，如果能把消除重复字符串，则可以近一步较少文件大小，提高读写速度。

修改后的结构如下：
```
public class DepenData
{
    public int assetPathIndex;
    public Hash128 assetDependencyHash;
    public int[] dependsPathIndex;
    public string[] dependsPath; // 用于返回查询结果，不保存
}
// save data
Dictionary<string, DependData> m_data;
List<string> m_strList;
// temp data
Dictionary<string, int> m_strIndex;
```

改进后文件大小变为28M，加载时间从18.3S优化到3.3S。6倍的改进，挺棒的，这时候又在思考是否有改进的余地。

第一个改进，把FileStream改为MemoryStream，数据则通过File.ReadAllBytes()读取。这个改造可以把3.3S改进为3S，主要是由于FileStream API调用的效率并不高，这里是通过减少调用频率来改进效率。对于FileStream每次ReadByte(2)和每次ReadByte(1024)，可能有接近100倍的性能差异。

第二个改进，分析发现3S里面BinaryReader占用了2.7S，剩下数据结构组织，填充Dictionary占用了0.3S。C#的BinaryReader实现并不高效，可以通过更高效的序列化数据方式来优化。这里尝试使用了FlatBuffers来替换BinaryReader，保存的开销从880ms增长到1200ms，读取的时间从3000ms优化到1200ms。又是一次大幅度的优化，虽然现在收益时间已经无关紧要了，不过实践和验证想法也是不错的收获。这里开启FlatBuffers Unsafe模式应该会有更高的收益，接近C++的性能，如果直接用C++写性能果然会好很多吧。

## 三
Unity所有路径都是Assets开头，大量路径字符串里面前缀包含重复数据，数据结构还可以再改进......

把二进制文件压缩后从28M变成5M，确实很多冗余数据，不过再改进可能付出太多时间而受益太低。这次的优化就到此为止了嘛。

这里上最后一个优化思路**异步化**。

异步加载在游戏中是很常见的做法，所以这里其实再实现两个异步化接口即可把这部分时间优化为0，由于还有其他很多任务可以并行执行，所以这部分时间在调整到适当的时机后可以忽略不计。

由于Unity的接口不能在多线程调用，所以一开始就不会往这个方面思考，后面问题转化后异步是一个非常优异的做法，FlatBuffers的改造非常繁琐，浪费了我大量测试时间。最后我把代码回滚到二进制版本，FlatBuffers在运行时确实能带来巨大的效率提升，不过这里可能并不需要上这个利器了。

一、二的优化是基于专注性思维的思考结果，而三则是发散性思维的思考结果。专注性思维容易陷入思维定式，这时候可以起来喝杯茶，出去散散步。

[完 Carber 2018-08-12]
