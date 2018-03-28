---
title: C# 内存与性能优化
date: 2017-08-04 23:10:00
tags: ["Unity", "C#"]
---
前两周分享了资源配置与资源管理，今天分享一种特殊的资源脚本数据。在Unity项目中，我们通常使用C#编写脚本，所以主要谈C#内存与性能优化。
> 一切都（代码）应该尽可能简单，但不能过于简单。

#### 字符串String
首先要关注String，String没有看起来那么简单，那么String是什么呢？
- String是一个UTF-16编码的文本
- String是一个引用类型
- String是不可变的

在C#里面，字符串是一个引用类型而不是一个值类型，即使看起来像是持有一个值类型对象并可以方便的修改。这里修改字符串会创建一个新的字符串，通常建议使用StringBuild来拼接字符串，下面看看不同行为的拼接字符串带来的性能差异。
```
public class MonoTest : MonoBehaviour {
    const int SIZE = 1024;
	void Update () {
        _UpdateStringAppend();
        _UpdateStringFormat();
        _UpdateStringBuild();
	}
    string _UpdateStringAppend() {
        string str = string.Empty;
        for (int i = 0; i < SIZE; ++i) {
            str += i;
        }
        return str;
    }
    string _UpdateStringFormat() {
        string str = string.Empty;
        for (int i = 0; i < SIZE; ++i) {
            str += string.Format("{0}", i);
        }
        return str;
    }
    string _UpdateStringBuild() {
        StringBuilder strBuilder = new StringBuilder();
        for (int i = 0; i < SIZE; ++i) {
            strBuilder.Append(i);
        }
        return strBuilder.ToString();
    }
}
```
|Func|Time ms|GC Alloc|
|--|--|--|
|StringAppend|9.09ms|2.9M|
|StringFormat|20.97ms|3.0M|
|StringBuilder|4.76ms|48.0KB|
观察数据可以发现StringBuilder在性能上和GC上都有极大的提升。Appendc操作每次创建一个新的字符串，字符串长度从0增长到n，这是一个O(n^2)的操作。而StringBuilder则只会在长度不够的时做一次O(n)的Resize操作，如果内部的长度是按2递增的话，这里的复杂度是O(nlogn)。开始时就设定一个足够大的初始值，则这里的复杂度可以降低到O(n)。

这里的内存申请的量级和运算复杂度也是一样的，关注大小与分配次数。过多的分配次数会导致堆碎片变多，过多的内存分配则会导致触发内存清理，这些额外的开销都是可以避免。推荐尽可能的使用StringBuild来优化这个操作，然后StringBuild本身也是可以复用的。

再来看看之前讨论里面被忽略的Format拼接，Format表现最差有点出人意外又在情理之中。我们在平时被建议使用Format来拼接字符串，但在有些情况下Format的表现非常差。这里就是一个不恰当的Format使用案例，这是一个冗余的Format操作，多了一次int转object的GC。如果还难以理解，则可以看看下面的Format操作，这里得到了一个更差的结果。
```
string _UpdateStringFormatEx() {
    string str = string.Empty;
    for (int i = 0; i < SIZE; ++i) {
        str = string.Format("{0}{1}", str, i);
    }
    return str;
}
```
|Func|Time ms|GC Alloc|
|--|--|--|
|StringAppend|9.09ms|2.9M|
|StringFormat|20.97ms|3.0M|
|StringBuilder|4.76ms|48.0KB|
|StringFormatEx|40.13ms|8.6M|
正确的Format用法是下面这样的
```
string str = string.Format("{0}{1}....{n}", 0, 1, ..., n);
```
通过把多次拼接操作合并成一次来达到减少GC提高效率，实际Format的内部使用了StringBuild来拼接字符串。N次使用StringBuild来拼接字符串的性能与1次的操作性能有较大差异，这也是平时使用中需要注意的。

#### 优化字符串数量
字符串是不可变，每次修改字符串都会生成一个新的字符串，那创建的字符串呢？经过实验了解到每次创建字符串都会得到一个新串，即使已经存在一个相同的字符串。这里有一篇顾露分享的[《Unity 游戏的 string interning 优化》](https://zhuanlan.zhihu.com/p/23877335)已经做了这块内容详细描述。**对于不可变的相同对象，完全是可以复用的，**这里通过string.Intern来完成这个操作。减少字符串数量达到优化内存的效果，同时让我发现了项目中存在着大量的字符串使用。如何更进一步的减少字符串数量是个有趣的问题。

通过顾露的自制工具[PA_ResourceTracker](https://github.com/PerfAssist/PA_ResourceTracker)采集的数据，分析数据发现字符串数据里面存在较多的资源加载路径。这些路径数据非常的长，而且数量也非常的多。字符串路径的作用是标识资源，考虑使用Hash来标识资源也可以做到相同的事情。
```
Resources.Load(string path, Type type);

Resources.Load(ulong pathHash, Type type);
Resources.PathToHash(string path);
```
在资源管理上实现两个新增的接口，支持按Hash加载资源，然后提供一个字符串路径转Hash的接口，来实现这一目标。
```
public class Template
{
   public int id = -1;
   public string name;
   public string path;
}
public class Template
{
   public int id = -1;
   public string name;
   public ulong pathHash;
}
```
然后替换结构体里面的变量为Hash，在第一次得到这个字符串后立刻调用Resources.PathToHash计算Hash值并存储。

计算路径的Hash还需要考虑路径的大小写、斜杠与放斜杠。
```
public ulong PathToHash(string str) {
    ulong hashCode = 0;
    for (int i = 0; i < str.Length; ++i) {
        char ch = Char.ToUpperInvariant(str[i]);
        if (str[i] == '\\')  { ch = '/'; }
        hashCode = (hashCode << 7) + (hashCode << 1) + hashCode + ch;
    }
    return hashCode;
}
```
使用ulong降低Hash的冲突，由于存在冲突的可能，这里在日常构建的时候对所有的资源路径计算Hash判断是否有冲突。这里路径Hash不仅减少了对象数量，也减少了一些字符串修改操作导致的GC。下面举个降低的例子，获取唯一字符串路径。
```
public string GetUniString(string str) {
    return str.Replace('\\', '/').ToUpperInvariant();
}
```
这实在是一个低效的行为，所以即使你不需要缩减字符串个数，还是强烈推荐使用Hash来做唯一标识符。

由于Unity提供的Resources接口需要使用路径字符串来加载资源，所以之前说了那么多还没有解释为什么可以减少字符串对象这个问题。这里我们项目能使用主要是由于使用了AssetBundle。只需要先存Hash对应的AssetBundle ID然后加载这个AssetBundle的时候加载Hash对应Name即可。

AssetBundle支持直接使用Name加载，也可以使用Asset Path加载。这里的AssetPath是相对于Assets目录的路径与Resources的相对于Resources目录还是有差异的，所以使用Name来加载。AssetBundle本身就有一个接口AssetBundle.GetAllAssetNames()获取所有资源路径。不过这里会包含被依赖的所有资源路径，所以一般自己存这个数据。

细心的人也注意到了上面提到的AssetBundle ID，由于AssetBundle打包是可以完全控制的。所以给AssetBundle命名一个数字ID，也是有效的减少字符串数量的方法。这对使用AssetBundle打包加载资源的项目是一个不错的参考。我们实现自己的AssetBundleManifest维护AssetBundle之间的依赖关系。

Unity的Animator类提供了StringToHash接口来帮助消除字符串，同时配套提供两套接口可以调用，和这里消除字符串路径的思路是一致的。相信还有其他地方也可以通过这个思路来消除字符串优化性能。

最后这里做字符串转路径这个实现是由于游戏在开始的时候就加载了大部分配置表，表现表里面有着大量路径字符串。在工具里面发现路径字符串的比重大概在20%，所以做这项工作优化字符串数量的工作，并带来了不错的性能收益。在Debug版本可以保留字符串实际路径便于差错。

#### 优化字符串比较
默认的字符串比较操作是非常低效的，[《Best Practices for Using Strings in .NET》](https://docs.microsoft.com/en-us/dotnet/standard/base-types/best-practices-strings)这篇文章讲了这方面的大部分细节。这里主要展示一些测试数据，让我们对性能有一个认识。
```
StringBuilder sBuilder = new StringBuilder();
System.Random random = new System.Random();
for (int i = 0; i < 100; ++i)
{
   sBuilder.Append((char)(random.Next() % 256));
}
string str = sBuilder.ToString();
string preStr = str.Substring(0, 16);
string lastStr = str.Substring(str.Length - 16, 16);
int cnt = 0;
for (int i = 0; i < 100 * 1024; ++i)
{
    if (str.StartsWith(preStr)) ++cnt;
    if (str.EndsWith(lastStr)) ++cnt;
}
```
测试结果

|Method|Time(ms) 100k compares|
|--------------------------------|--------|
|String.StartsWith,default culture| 360ms|
|String.EndsWith,default culture| 12465ms|
|String.StartsWith,Ordinal| 357ms|
|String.EndsWith,Ordinal| 174ms|
|CustomStartsWith|18ms|
|CustomEndsWith|17ms|

字符串比较接口默认行为
| Func Name| Default interpretation|
|-----------------|---------------------------------|
|String.Compare|StringComparison.CurrentCulture|
|String.CompareTo|StringComparison.CurrentCulture|
|String.Equals|StringComparison.Ordinal|
|String.ToUpper|StringComparison.CurrentCulture|
|Char.ToUpper|StringComparison.CurrentCulture|
|String.StartsWith|StringComparison.CurrentCulture|
|String.IndexOf|StringComparison.CurrentCulture|
正常情况下使用Ordinal比较即可，自己实现Ordinal行为的比较还可以提高10倍的性能。

#### 从容器谈Boxing
泛型容器内部实现会调用一些System.Object接口，如果我们不实现对应的泛型接口，在调用接口的时候就会找到基类Object的接口。而由于Struct是一个值类型，value type转class type会触发内存分配，定义这种行为为Boxing。[《c-performance-tips-for-unity-part-2-structs-and-enums》](http://www.somasim.com/blog/2015/08/c-performance-tips-for-unity-part-2-structs-and-enums/)这篇文章已经对这块做了详细描述与举例。我自己也做了一些数据测试，分享给大家做参考。
```
public struct SmallStruct
{   // 2 int fields. Total size: 2 * 4B + 16B = 24B
    public int a, b;
}
public struct LargeStruct
{   // 20 int fields. Total size: 20 * 4B + 16B = 96B
    public int a, b,  /* … */;
}
// Dictionary<Struct, bool> dict
// 1024 calls dict. ContainsKey
```
|Struct|GC Alloc|Time ms|
|-|-|-|
|SmallStruct|72.0KB|2.50ms|
|LargeStruct|288.0KB|11.05ms|
实现了不同接口之后
|SmallStruct|GC Alloc|Time ms|
|-|-|-|
|None|72.0KB|2.50ms|
|IEquatable<&T>|24.0KB|1.77ms|
|GetHashCode|48.0KB|2.57ms|
|GetHashCode,IEquatable<&T>|0.0KB|1.81ms|
观察发现Dictionary内部使用 EqualityComparer
```
public abstract class EqualityComparer<T>
{
    protected EqualityComparer();
    public static EqualityComparer<T> Default { get; }
    public abstract bool Equals(T x, T y);
    public abstract int GetHashCode(T obj);
}
```
如果没有实现GetHashCode触发一次boxing，而Equals则触发两次。实现IEquatable泛型接口，以及override int GetHashCode则可避免触发GC。非泛型的HashTable实现和泛型Dictionary基本一致，推荐使用Dictionary泛型版本，提高性能。

GetHashCode的实现并不需要返回一个唯一结果，而是尽可能的减少冲突即可，在HashCode相同的情况下才会执行Equals比较。实现好的GetHashCode减少冲突，提高性能，更多内容可以参考[《Why is it important to override GetHashCode when Equals method is overridden?
》](https://stackoverflow.com/questions/371328/why-is-it-important-to-override-gethashcode-when-equals-method-is-overridden)。

#### 其他Tips
```
void DispatchEvent(string str, params object[] data);

static object[] _default = new object[] {};
void DispatchEvent(string str)  {
    _DispatchEvent(str, _default);
}![Alt text](./04.【1】磯村由纪子 - 风の住む街.jpg)

void _DispatchEvent(string str, object[] data)；
```
params object每次调用会申请一个object数组，对于无参数的行为，实现一个默认接口减少GC。

一般情况下使用Profile Windows排查不必要的GC Alloc。
![Profiler window](./1502416403334.png)

这个工具能帮助我们定位发生GC Alloc行为的代码，通常第一步优化那些每帧都存在的GC，之后优化那些峰值很高的GC。优化GC能带来什么好处呢，假设当前使用了30M内存，申请了50M内存。这里有20M的空间可以用于日常的GC Alloc。假设我们每帧的GC Alloc=100K，则20 * 1024 / 100 = 204帧。如果每帧的执行时间为33ms（30帧），则6.76S触发一次GC.Collect()。这个函数开销在100ms以上，当前帧的开销从33ms变成133ms，这会有明显的卡顿感。更多的GC优化可以参考[《Structing out code to minimize the impact of garbage collection》](https://unity3d.com/cn/learn/tutorials/temas/performance-optimization/optimizing-garbage-collection-unity-games?playlist=44069)。

#### 从Struct再谈优化对象数量
从Rich Geldreich的[《Lessons Learned While Fixing Memory Leaks in our First Unity Title》](http://www.gamasutra.com/blogs/RichGeldreich/20150731/250071/Lessons_Learned_While_Fixing_Memory_Leaks_in_our_First_Unity_Title.php)了解到对象数量过大造成额外的内存使用。这里再次谈对象数量优化，优化内存使用。
> The Boehm collector grows its OS memory allocation so it has enough internal heap headroom to avoid collecting too frequently. You must factor this headroom into account when budgeting your C# memory, i.e. if your budget calls for 25MB of C# memory then the actual amount of memory consumed at the OS level will be significantly larger (approximately 40-50MB in our experience).

这里主要讨论配置表，配置表一般是一种Key-Value结构，同时在运行时我们不需要修改内存，最后配置表的总量和数量会非常多。
```
public class Dicitonary<TKey, TValue> { 
    private int[] m_buckets;
    private int[] m_entryNext;
    private int[] m_entryHash;
    private TKey[] m_entryKey;
    private TValue[] m_entryValue ;
}
public class PlayerTemplate {
    public int id;
    public ulong pathHash;
    public float height;
    /* ... more data */
} // assume size = 128B
Dictionary<int, PlayerTemplate> dict;
```
一般使用Dictionary存储配置表数据，上面定义的配置表数据类型为class，则可以得到下面的数据。
```
Set PlayerTemplate Count = 5000;
// 第一个大于Count * 2的素数
Dictionary ArraySize = 10103; 

ObjectCount = 5000 + 5 + 1 = 5006;
MemorySize = 5000 * 128B + 10103 * 24 = 882472B = 861.8KB 
```
之后我们把class改struct	
```
public struct PlayerTemplate {/* … */}

ObjectCount = 5 + 1 = 6;
MemorySize = 10103 * 128B + 10103 * 16 = 1454832B = 1420.7KB
```
对象数量减少后的代价是内存使用的增长，下面来看怎么优化内存使用。
```
public interface ITableType<TKey, TValue> {
    TKey GetKey();
}
public class TableOrderList<Tkey, TValue> {
    private bool m_sorted;
    private TValue[] m_data;
    private int m_size;
}
public struct PlayerTemplate : ITableType<int, PlayerTemplate> {
    public int GetKey() {
        return id;
    }
    public int id;
    public ulong pathHash;
    public float height;
    /* ... more data */
}
```
自定义容器与接口实现线性内存空间存储数据。
```
public int LowerBounder(TKey key) {
    int low = 0, high = m_size;
    while (low < high) {
        int mid = (low + high) >> 1;
        if (m_list[mid].GetKey().CompareTo(key) < 0) {
            low = mid + 1;
        } else { 
            high = mid;
        }
    }
    return low;
}
```
在数据加载结束后进行一次排序，之后就通过二分查找数据。

最后的数据对比如下。
|Type|Object Count|Memory Use|Complexity|
|-|-|-|-|
|Class,Dictionary|5006|861.8KB|O(1)|
|Struct,Dictionary|6|1420.7KB|O(1)|
|Struct,TableOrderList|1|625KB|O(logn)|
新实现的容器再对象数量与内存使用上都有着较大优势，由于一般游戏很难有超过1W以上的数据量，这里O(logn)与O(1)的差距较小可以接受，而且一般这里也不是性能瓶颈。

Struct只能整存整取，Class则可以简易的修改成员变量。但是对于只读的数据来说，使用Struct来存储数据有极大的优势。更多Struct与Class的讨论可以参考[《What's the difference between struct and class in .NET》](http://stackoverflow.com/questions/13049/whats-the-difference-between-struct-and-class-in-net)。