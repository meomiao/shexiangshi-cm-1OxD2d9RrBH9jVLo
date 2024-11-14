
## 前言


本篇文章前面客观评估了 .NET 创建动态方方案多个方面的优劣，后半部分是 [Natasha V9](https://github.com) 的新版特性。


## .NET 中创建动态方法的方案


### 创建动态方法的不同选择


以下陈列了几种创建动态方法的方案：以下示例输入为 value, 输出为 Math.Floor(value/0\.3\)：


#### emit 版本



```
 DynamicMethod dynamicMethod = new DynamicMethod("FloorDivMethod", typeof(double), new Type[] { typeof(double) }, typeof(Program).Module);
 ILGenerator ilGenerator = dynamicMethod.GetILGenerator();

 ilGenerator.Emit(OpCodes.Ldarg_0);  
 ilGenerator.Emit(OpCodes.Ldc_R8, 0.3);  
 ilGenerator.Emit(OpCodes.Div);  
 ilGenerator.Emit(OpCodes.Call, typeof(Math).GetMethod("Floor", new Type[] { typeof(double) }));  
 ilGenerator.Emit(OpCodes.Ret); 

 Func<double, double> floorDivMethod = (Func<double, double>)dynamicMethod.CreateDelegate(typeof(Func<double, double>));

```

#### 表达式树版本



```
 ParameterExpression valueParameter = Expression.Parameter(typeof(double), "value");

 Expression divisionExpression = Expression.Divide(valueParameter, Expression.Constant(0.3));
 Expression floorExpression = Expression.Call(typeof(Math), "Floor", null, divisionExpression);
 Expressiondouble, double>> expression = Expression.Lambdadouble, double>>(floorExpression, valueParameter);

 Func<double, double> floorDivMethod = expression.Compile();

```

#### Natasha 版本



```
AssemblyCSharpBuilder builder = new();
var func = builder
    .UseRandomLoadContext()
    .UseSimpleMode()
    .ConfigLoadContext(ctx => ctx
        .AddReferenceAndUsingCode(typeof(Math))
        .AddReferenceAndUsingCode(typeof(double)))
    .Add("public static class A{ public static double Invoke(double value){ return Math.Floor(value/0.3);  }}")
    .GetAssembly()
    .GetDelegateFromShortNamedouble, double>>("A", "Invoke");

```

#### Natasha 方法模板封装版



> 该扩展库 `DotNetCore.Natasha.CSharp.Extension.MethodCreator` 在原 Natasha 基础上封装，并在 Natasha v9\.0 版本后发布。


1. 轻量化构建方案：



```
var simpleFunc = "return Math.Floor(arg1/0.3);"
    .WithMetadata(typeof(Math))
    .WithMetadata(typeof(Console)) //如果报 object 未定义, 加上
    .ToFunc<double, double>();

```

2. 智能构建方案：



```
var smartFunc = "return Math.Floor(arg1/0.3);".ToFunc<double, double>();

```

## 方案对比与分析


### 时由此可以看出，无论哪种动态构建，都无法挣脱 `typeof(Math)` 的束缚，甚至需要反射更多的元数据。


元数据在动态方法创建中是必不可少的结构，既然大家都依赖元数据，不妨做一个对比；




| 方案名称 | 编码形式 | Using 管理 | 内存占用 | 卸载功能 | 构建速度 | 执行性能 | 断点调试 | 学习成本 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Emit | 繁琐 | 不需要 | 低 | .NET9 可卸载 | 快 | 高 | .NET9 支持 | 高 |
| Expression | 一般 | 不需要 | 低 | .NET9 可卸载 | 快 | 高 | .NET9 支持 | 中 |
| Natasha | 一般 | 需要 | 中 | 可卸载 | 首次慢 | 高 | 支持 | 中 |
| Natasha Method | 简单 | 需要 | 中 | 可卸载 | 首次慢 | 高 | 支持 | 低 |


### 场景


首先从场景角度来说，Emit / Expression 方案在 .NET 生态建设中扮演了非常重要的角色，也是不少高性能类库的主要核心技术栈。而 Roslyn 的动态编译技术从初期到完成，走的是一个全面的编译流程，Natasha 是基于 Roslyn 的。虽然我是 Natasha 作者，但我还是建议，小打小闹，使用表达式树。而那些比较复杂的动态业务、动态框架、动态类库使用 Natasha 更加舒适。举几个例子，比如规则引擎, Natasha 不是规则引擎，但你可以用 Natasha 来定制一个符合你操作习惯的规则引擎。再比如对象映射, Natasha 不是对象映射库，但你可以用 Natasha 来定制一个符合你操作习惯的对象映射库; 如果你觉得市面上的 ORM 用着都不顺手，当然可以用 Natasha 来定制一款自己喜欢的 ORM.


### 编码形式程与 Using 管理


从编码过程来说，Emit 是比较繁琐的，从编码思维上来讲，Emit 属于“栈式编程”，数据的操作顺序后进先出，与平常使用的 C\# 关键字不同，它更趋近于底层的指令，你不会看到 if/switch/while 等操作，而是 Label 和一些跳转指令，为此你也无法享受到正常编译所带来的便捷，甚至需要还原一些操作，比如 "str1" \=\= "str2" 实际上要换成 Equal() 方法。
而表达式树相比 Emit 就要舒服一点了，然而这也不是正儿八经的 C\# 编程思维, 仍然还是需要经过加工处理的，如果你非常享受这种转换过程，并能从中获得成就感或者其他感觉，那无可厚非，它适合你。我想大多数人是因为没有办法才选择动态方案，无论哪种，能从中获得愉快感觉的开发者并不会很多。
相比前两者，Natasha 需要注意的是 “域” 操作和 Using 引用，有了 “域” 更加方便隔离程序集和卸载，如果你的动态代码创建的程序集永远都是有用的，使用默认域即可。反之那些需要卸载或更新的动态功能，就得选择一个非默认域了。
除卸载之外，另一个参与动态构建过程的就是 Using ，因为 Using 代码是 C\# 脚本必不可少的一环，因此以 C\# 脚本方式构建动态功能需要考虑到 `using System;` 这类代码. 而 Using 中遇到的最大的问题是二义性引用 (CS0104\) 问题：
假设有 `namespace MyFile{ public static class File{} }` 在 VS 里开启隐式 `enable` 后引用它你会发现错误，表面原因是 `MyFile` 和 `System.IO` 命名空间冲突了，实际原因是这两个命名空间都有 File 相关的元数据，而语义分析不会对后续代码功能进行推断你到底想使用哪个 File, 这种情况在复杂的编程环境下或许会出现，不可避免只能发现和改正。
![](https://img2024.cnblogs.com/blog/1119790/202407/1119790-20240712201556690-1854600153.png)


一旦发生这种情况，您需要排除不想引用的 Using.



```
//排除 System.IO
builder.AppendExceptUsings("System.IO");

```

我们继续看第四种，基于 Natasha 封装的动态方法构建，非常的简单，其中：


* 轻量化构建写法是按需引用元数据编译成委托。
* 智能构建写法是直接在元数据和 using 全覆盖的情况下编译成委托，该写法的前提是预热了元数据和 Using 集合，详情可以看 Natasha 预热相关的方法。


### 内存占用


前二者的编译占用系统内存很少，毕竟是直接一步到位的做法，少了很多分析转换缓存的过程。可以这么理解，前二者是 Roslyn 编译中的一环。


### 卸载功能



> 后文 Natasha 的"域"均用 AssemblyLoadContext(ALC) 代替。


限定在本文4种编码案例范围内，目前我还没看到过关于 直接卸载 Emit/表达式树 创建的动态方法相关的文章。
但 .NET9 推出的 PersistedAssemblyBuilder 将允许编译 Emit 并输出流到 ALC 中。



```
PersistedAssemblyBuilder ab = ....;
using var stream = new MemoryStream();
ab.Save(stream);

NatashaDomain domain = new("MyDomain");
var newAssembly = domain.LoadFromStream(stream);

```

虽然 .NET9 支持保存程序集了，但 ALC 的卸载功能有点难搞，.NET 官方对 ALC 的卸载操作几乎是无能为力的，从理论上来讲只要你的类在使用中，这个类就无法被卸载，这种情况很多，一个静态泛型类，或一个全局事件，或被编译到某个不被清理的方法中，又不提供清理方法，他们就会成为程序的僵尸类型。官方没有办法强制对正在使用的数据做些什么强制处理。假如我的程序有 60 个依赖项，我需要找到这些依赖项的作者，也许作者不止 60 个，然后一一询问他们：请问您的库如何能够清除保存在您这里的 ALC 创建的元数据？然后附上一些调试证据，告诉他，在 2 代 GC 中发现了大量无法被释放的元数据，这些元数据与你的 XXX 有关。读到这里你觉得非常难办，甚至有点荒谬，没错，就是这样，也只能这样。所以我在制作 HotExector 的过程中也不断的思考和实验整域代理以屏蔽域外引用的干扰。
话说回来如果是自己封装的框架，那么这个卸载的问题是会很好解决，因为你知道什么东西该清理，什么字段该置空。


### 构建速度


与内存占用说明类似，一个完整的编译过程肯定要比其中一环占用的时间长，况且 Roslyn 内部还会缓存和预热一些数据。首次编译后，编译速度就会非常的快。


### 执行性能


如果被编译的 Emit 代码逻辑能够和 Roslyn 内部优化后的脚本逻辑一样，那么执行性能理论上是持平的。例如在多条 if 或 switch 的数值查找逻辑中，Roslyn 可能会对这些查找分支进行优化，比如使用二分查找树代替原有的查找逻辑，如果你想不到这些优化点， Emit 代码的性能只能依靠后续 JIT 的持续优化来提高了，因为考虑到 JIT 后续的优化可能会让它们都达到最优结果，因此都给了 “高”。而二者的区别开发者应该了解，相比 Emit 原生编程，Roslyn 编译的 Emit 逻辑更加优秀和高性能。


### 断点调试


Natasha 自 V8 版本起开始支持脚本断点调试，V9 版本升级了不同平台、不同 PDB 输出方式的兼容性优化, Natasha 编译框架支持 .netstd2\.0。
而 Emit 也迎来的自己的调试方案，上文提到 .NET9 的 PersistedAssemblyBuilder，通过使用该类的 `GenerateMetadata` 方法来生成元数据流，进而创建 `PortablePdbBuilder` 调试数据实例，然后转化为 `Blob` (BlobBuilder)，最后写入 PDB 流. 有了 PDB 文件， Debug 断点调试将变得可行。


## Natasha 以及动态方法模板的优势


### 套娃编译


使用 Natasha 可以进行套娃编译，使用动态编译进行动态编译，这种逻辑很复杂，举个简单的例子，假如需求是生成动态类型 A，在 A 的静态初始化方法中生成一个动态委托 B， 而且 B 的部分逻辑是根据动态类型 C 和 D 所接受到的数据决定的。这在表达式树和 Emit 的编码思维中，可能对数据和类型做一个中转处理，而且在编译过程中要用到`还没有被编译的 A 的成员元数据`，这里很绕。这种情况我更推荐使用 Natasha ,因为是考虑到学习成本和时间成本，按照正常思维 5 分钟编写完脚本，为啥还要用 20 分钟甚至 1 小时去找解决方案，设计缓存，定制运行时强编码规则呢？


### 私有成员


很多开发者应该都有读源码的习惯，甚至在高性能场景会对源码进行魔改和定制，部分开发者有 200% 的信心保证他们获取的私有实例和方法不会在程序中被乱用，这里就会遇到一些问题，重新定制源码，或者使用未开放的实例方法会遇到访问权限的问题。节省时间和篇幅直接上例子：


已知在支持热重载的 MVC 框架中，有 `IControllerPropertyActivator / IModelMetadataProvider` 两个服务实例，它们提供了私有方法 `ClearCache` 方法清除元数据缓存的接口，但 `IControllerPropertyActivator` 接口由于访问限制，写代码时 IDE 会报错，事已至此，运行时要拿到 `IControllerPropertyActivator` 接口类型只能先获取它的实例然后通过反射获取类型，而这里的操作就是矛盾的操作，如果我不知道哪个类型实现了该接口，或者说实现接口的类型也是个 private 级别，那么我又该如何获取到实例.


Natasha V9 版以前需要自己定制开放私有操作，我们看一下 V9 更新后的操作：



```
//打开私有编译开关
builder.WithPrivateAccess();
//改写脚本
builder.Add(script.ToAccessPrivateTree("Microsoft.AspNetCore.Mvc.ModelBinding","Microsoft.AspNetCore.Mvc.ModelBinding.Metadata"));
//或
builder.Add(script.ToAccessPrivateTree(typeof(IModelMetadataProvider),"Microsoft.AspNetCore.Mvc.ModelBinding.Metadata"));
//或
builder.Add(script.ToAccessPrivateTree(instance1,instance2...));

```

开启以上选项，Natasha 将重写语法树以支持私有操作。最终脚本无需任何处理，一气呵成，以下是使用 Natasha 绕过了访问检查的脚本案例：



```
//该脚本如果在 IDE 中会有访问权限问题
var modelMetadataProvider = app.Services.GetService();
var controllerActivatorProvider = app.Services.GetService();
((DefaultModelMetadataProvider)modelMetadataProvider).ClearCache();
((DefaultControllerPropertyActivator)controllerActivatorProvider).ClearCache();

```

### 安全相关


有人说使用脚本会导致安全问题，我觉得这种说法太过片面，不应该把人为因素强加到某个类库中，Natasha 不会自行的为应用程序提供后门漏洞，任何上传的文本和图片都需要有严格的审核，包括上传的脚本，即便没有非法的网络请求代码，占用资源，数据安全等问题的代码也要进行严格排查。对于需要大量动态脚本的支持的服务，服务应该严格限制元数据和规范功能脚本粒度。


## Natasha V9 新版变化



> 项目主页：[https://github.com/dotnetcore/Natasha](https://github.com):[FlowerCloud机场](https://hushicha.org)


### 链式初始化 API


为了让初始化更容易懂，在新版本中增加了一组链式操作的 API.
此组 API 更容易控制 Natasha 的初始化行为。



```
NatashaManagement
    .GetInitializer()
    .WithMemoryUsing() //不写这句, Natasha 将不会扫描和预存内存中的 UsingCode. 
    .WithMemoryReference()
    .Preheating();

```


> 注：在使用智能模式时，预热 Using 和 Reference 是必要的，除非你能很好的管理这些。


### 更灵活的元数据管理


* 自 v9 版本起，简单模式（自管理元数据模式）支持单独添加元数据，单独添加 using code, 而不是 引用和using 一起添加。
* 编译单元允许添加排除 using code 集合，`builder.AppendExceptUsings("System.IO","MyNamespace",....)`, 该方法将防止指定的 using 被添加到语法树中.


### 外部异常获取


* 增强错误提示，引发编译异常时，将首先抛出错误级别的异常，而不是警告。
* 增加 “GetException” API, 在 Natasha 编译周期外，获取异常错误。


### 重复编译


V9 版本在重复编译方面做了一些增强，进一步增加复用性和灵活性。


#### 1\. 重复的文件输出


* WithForceCleanOutput 该 API 使用后将开启 强制清除文件 开关，以避免在重复编译时产生 IO 方面的错误。
* WithoutForceCleanOutput 是默认使用的 API, 此时遇到重复文件，Natasha 将自动改名到新文件中， oldname 被替换成 repeate.guid.oldname.


#### 2\. 重复的编译选项


* WithPreCompilationOptions 该 API 开启后，将复用上一次的生成的编译选项（若没有则生成新的），该编译选项对应着 CSharpCompilationOptions 相关参数， 如果第二次编译需要切换 debug/release，unsafe/nullable 等选项，请关闭该选项。
* WithoutPreCompilationOptions 是默认使用的 API, 该 API 不会锁定 CompilationOptions，保证每次编译都是最新的。


#### 3\. 重复的引用


* WithPreCompilationReferences 该 API 开启后，将复用上一次的元数据引用集。
* WithoutPreCompilationReferences 是默认使用的 API。
* 新版本中增强了对 “引用API” 的注释，让其行为更加容易被看懂。321，[上链接](https://github.com)。
![](https://img2024.cnblogs.com/blog/1119790/202411/1119790-20241111210016269-749844098.png)


#### 4\. 私有脚本支持


在使用前需要在工程中加上 IgnoresAccessChecksToAttribute.cs 文件



```
namespace System.Runtime.CompilerServices
{
    [AttributeUsage(AttributeTargets.Assembly, AllowMultiple = true)]
    public class IgnoresAccessChecksToAttribute : Attribute
    {
        public IgnoresAccessChecksToAttribute(string assemblyName)
        {
            AssemblyName = assemblyName;
        }
        public string AssemblyName { get; }
    }
}

```


```
//给当前脚本增加私有注解(标签)
//privateObjects 可以是私有实例，可以是私有类型，也可以是命名空间字符串
classScript = classScript.ToAccessPrivateTree(privateObjects)

builder
.WithPrivateAccess() //编译单元开启私有元数据支持
.Add(classScript );

```

### 5\.编译优化级别



> 注意：使用动态调试前，请先在工具\-选项\-调试中关闭\[地址级调试]。


Natasha v9 对编译优化级别做了细化:



```
//普通 Debug 模式
WithDebugCompile(item=>item.ForCore()/ForStandard()/ForAssembly())
//加强 Debug 模式
WithDebugPlusCompile(item=>item.ForCore()/ForStandard()/ForAssembly())

```


```
//普通 Release 模式
WithReleaseCompile()
//加强 Release 模式
WithReleasePlusCompile()

```

理论上的加强模式可以理解为“刨根问底，全部显现”模式，虽然普通的模式就已经足够用，但这个模式可以更细粒度的输出调试信息，包括一些隐式的转换。
注：实验中没有看到更为细致的调试结果，有经验的同志可以告知我哪些代码可以呈现出更细腻的结果。


### 6\. 其他 API


* 新版本对 API 的注释进行了大量中文重写，小伙伴们可以看到更接地气，容易懂的注释，由于编译单元 (AssemblyCSharpBuilder) 多采用状态方式保存用户配置，因此在 API 上还简单增加了复用方面的说明。
* 熟知的 `UseDefaultDomain()` 已过时，更符合 API 本意的 `UseDefaultLoadContext()` 名称更为合适, Domain 系列已经不能成为编译单元的前沿 API, 从 V9 版本起 LoadContext 系列将取而代之。
* 增加 `CompileWithoutAssembly` API, 允许开发者在编译后不向域中注入程序集，程序将不实装编译后的程序集。


## 结尾


之前以为自己入了 Roslyn 的冰山一角，没想到只是浮冰一块。


#### 谢谢俞佬在文档上的支持。


#### 码字不易，感谢看完，多谢点赞。


