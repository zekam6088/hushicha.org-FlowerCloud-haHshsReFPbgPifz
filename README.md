# 给旧版 .NET 也开一扇"私有之门" —— ILAccess.Fody 实现原理与设计

作者：**huoshan12345**
项目地址：[ILAccess.Fody](https://github.com)

---

## 前言：从 UnsafeAccessor 说起

在 .NET 8 中, 微软引入了一个让底层开发者非常心动的新特性 —— [`UnsafeAccessor`](https://github.com)
它允许我们在不使用反射的情况下访问类的私有字段、方法或构造函数, 而且是**强类型**、**零开销**的.

举个例子：

```
class Dog
{
    private string _name = "Puppy";
}

static class DogAccessors
{
    [UnsafeAccessor(UnsafeAccessorKind.Field, Name = "_name")]
    public static extern ref string GetName(Dog d);
}

var dog = new Dog();
ref var name = ref DogAccessors.GetName(dog);
Console.WriteLine(name); // Puppy
```

CLR 会在运行时将 `GetName` 绑定到 `_name` 字段, 生成直接访问的 IL 指令.性能几乎和直接访问公有字段相当.

> 🧩 Benchmark（来自 [Sharmila Malar 的 Medium 文章](https://github.com)）
>
> * Reflection: ~10.9 ns
> * UnsafeAccessor: ~1.99 ns
> * Direct Access: ~1.81 ns

但是, 这个特性**只在 .NET 8+ 可用**.

---

## ILAccess.Fody 诞生

我希望旧版 .NET 平台（例如 .NET Framework、.NET Standard、.NET 6）也能享受这种“反射级灵活 + 原生级性能”的能力.于是我编写了 **[ILAccess.Fody](https://github.com)**. 它实现了和 `UnsafeAccessor` 几乎一致的语法和体验, 但通过 [Fody](https://github.com):[veee加速器](https://blog.liuyunzhuge.com) + [Mono.Cecil](https://github.com) 在编译期修改 IL 来实现.

* `Mono.Cecil` 是一个用于分析和修改 .NET 程序集的库, 它提供了强大的对象模型, 让我们能在不加载程序集的情况下读取、编辑、甚至生成新的 IL 代码.
* `Fody` 是一个基于 `Mono.Cecil` 可扩展的编译期织入工具, 它让开发者能在构建过程中直接修改程序集的 IL, 而无需手动处理 MSBuild 或 Visual Studio 的复杂管线.

---

## 使用方法 (和UnsafeAccessor几乎一样)

```
static class DogILAccessors
{
    // UnsafeAccessorAttribute -> ILAccessorAttribute
    // UnsafeAccessorKind -> ILAccessorKind
    [ILAccessor(ILAccessorKind.Field, Name = "_name")]
    public static extern ref string GetName(Dog d);
}

var dog = new Dog();
ref var name = ref DogILAccessors.GetName(dog);
Console.WriteLine(name); // Puppy
```

编译后, `ILAccess.Fody` 会将这些标记了 `ILAccessor` 的桩方法体替换为直接访问私有成员的 IL :

```
.method public hidebysig static string& GetName(class ILAccess.Dog d) cil managed
{
    IL_0000: ldarg.0      // d
    IL_0001: ldflda       string ILAccess.Dog::_name
    IL_0006: ret
}
```

没有反射、没有委托、没有运行时查找.

---

## 核心实现: 编译期注入 IL

* 遍历所有方法, 找到带 `[ILAccessor]` 的桩方法.
* 根据不同的 `ILAccessorKind` 生成对应的 IL 指令.
* 确定目标类型, 对于构造函数目标类型就是桩方法的返回类型, 其余的就是桩方法的第一个参数的类型.
* 对于字段, 根据字段名匹配, 在目标类型的声明字段里找到目标字段, 然后根据是否为静态选择 `Ldsfld` 或 `Ldfld` 指令, 如果是 `ref`访问, 则需选择 `Ldsflda` 或 `Ldfld`
* 对于方法, 则需要根据方法名+泛型参数个数+参数类型列表来匹配, 在目标类型的声明方法里找到目标方法, 然后选择 `callvirt` 或 `call` 指令
* 对于构造函数, 其实就是一个名为 `.ctor` 的方法, 处理方式和普通方法类似, 不过指令是 `newobj`

---

## 额外处理

* 当想访问的私有成员的类型是定义在一个 [引用程序集(Reference Assemblies)](https://github.com) 中

比如想访问基础库中的 `List` 的 `_items`, 而基础库在编译时通常会以 `引用程序集` 的形式进行引用, 此时这个程序集只包含这个类型的公共成员的定义, 不包括其实现已经所A有的私有成员. 那么就无法根据名称找到私有成员的用于生成 IL 的元数据.
解决方法: 先通过引用程序集的路径找到其对应的 `实现程序集(Implementation Assemblies)` 的路径, 并从其中读取所需的元数据.

* 当想访问的私有成员的类型并不定义在当前被编译的程序集中

这种情况如果直接使用 IL 指令访问其私有成员会触发 `MethodAccessException` 或者 `FieldAccessException` 之类的异常.
解决方法: 使用 `IgnoresAccessChecksToAttribute` 跳过对想访问的程序集的权限检查. 例如

```
[assembly: IgnoresAccessChecksTo("System.Private.CoreLib")] // 跳过对基础库的访问检查
```

不过在使用 `ILAccess.Fody` 的时候无需手动添加这些代码, 它会自动生成并注入到被编译的程序集中.

---

## 访问私有成员方法的对比

| 特性 | 反射 | UnsafeAccessor | ILAccess.Fody |
| --- | --- | --- | --- |
| 支持平台 | 所有 .NET 平台 | 仅 .NET 8+ | 全部 .NET 平台 |
| 实现方式 | 运行时查找 | CLR 运行时注入 | Fody 编译期注入 |
| 性能 | 慢 | 几乎接近直接访问 | 几乎接近直接访问 |
| 编译期验证 | ❌ | ❌ | ✅ |
| AOT支持 | ⚠️ 受限支持 | ✅ | ✅ |

---

## 结语

ILAccess.Fody 的目标很纯粹:

> 让旧版 .NET 也能拥有 UnsafeAccessor 的力量.

它用编译期 IL 注入的方式, 让私有访问变得强大而安全.
如果你想了解更多, 欢迎访问: 👉 [ILAccess.Fody](https://github.com)
