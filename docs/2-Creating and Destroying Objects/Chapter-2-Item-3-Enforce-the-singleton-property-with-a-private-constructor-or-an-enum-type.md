# 第三节: 使用私有构造函数或枚举类型实施单例属性

### 单例
单例是一个只实例化一次的类 [Gamma95]。单例通常表示无状态对象，比如函数（[Item-24](/Chapter-4/Chapter-4-Item-24-Favor-static-member-classes-over-nonstatic.md)）或系统组件，它们在本质上是唯一的。**将一个类设计为单例会使它的客户端测试时变得困难，** 除非它实现了作为其类型的接口，否则无法用模拟实现来代替单例。

### 1. 公共属性
实现单例有两种常见的方法。两者都基于保持构造函数私有和导出公共静态成员以提供对唯一实例的访问。在第一种方法中，成员是一个 final 字段：

```
// Singleton with public final field
public class Elvis {
    public static final Elvis INSTANCE = new Elvis();
    private Elvis() { ... }
    public void leaveTheBuilding() { ... }
}
```

私有构造函数只调用一次，用于初始化 public static final 修饰的 Elvis 类型字段 INSTANCE。不使用 public 或 protected 的构造函数保证了「独一无二」的空间：一旦初始化了 Elvis 类，就只会存在一个 Elvis 实例，不多也不少。客户端所做的任何事情都不能改变这一点，但有一点需要注意：拥有特殊权限的客户端可以借助 AccessibleObject.setAccessible 方法利用反射调用私有构造函数（[Item-65](/Chapter-9/Chapter-9-Item-65-Prefer-interfaces-to-reflection.md)）如果需要防范这种攻击，请修改构造函数，使其在请求创建第二个实例时抛出异常。

**译注：使用 `AccessibleObject.setAccessible` 方法调用私有构造函数示例：**
```
Constructor<?>[] constructors = Elvis.class.getDeclaredConstructors();
AccessibleObject.setAccessible(constructors, true);

Arrays.stream(constructors).forEach(name -> {
    if (name.toString().contains("Elvis")) {
        Elvis instance = (Elvis) name.newInstance();
        instance.leaveTheBuilding();
    }
});
```

### 2.私有属性，提供公共静态方法
在实现单例的第二种方法中，公共成员是一种静态工厂方法：

```
// Singleton with static factory
public class Elvis {
    private static final Elvis INSTANCE = new Elvis();
    private Elvis() { ... }
    public static Elvis getInstance() { return INSTANCE; }
    public void leaveTheBuilding() { ... }
}
```

所有对 `getInstance()` 方法的调用都返回相同的对象引用，并且不会创建其他 Elvis 实例（与前面提到的警告相同）。

**译注：这里的警告指拥有特殊权限的客户端可以借助 `AccessibleObject.setAccessible` 方法利用反射调用私有构造函数**

公共字段方法的主要优点是 API 明确了类是单例的：public static 修饰的字段是 final 的，因此它总是包含相同的对象引用。第二个优点是更简单。

**译注：static factory approach 等同于 static factory method**

静态工厂方法的一个优点是，它可以在不更改 API 的情况下决定类是否是单例。工厂方法返回唯一的实例，但是可以对其进行修改，为调用它的每个线程返回一个单独的实例。第二个优点是，如果应用程序需要的话，可以编写泛型的单例工厂（[Item-30](/Chapter-5/Chapter-5-Item-30-Favor-generic-methods.md)）。使用静态工厂的最后一个优点是方法引用能够作为一个提供者，例如 `Elvis::getInstance` 是 `Supplier<Elvis>` 的提供者。除非能够与这些优点沾边，否则使用 public 字段的方式更可取。

**译注 1：原文方法引用可能是笔误，修改为 `Elvis::getInstance`**

**译注 2：方法引用作为提供者的例子：**
```
Supplier<Elvis> sup = Elvis::getInstance;
Elvis obj = sup.get();
obj.leaveTheBuilding();
```

要使单例类使用这两种方法中的任何一种实现可序列化（Chapter 12），仅仅在其声明中添加实现 serializable 是不够的。要维护单例保证，应声明所有实例字段为 transient，并提供 readResolve 方法（[Item-89](/Chapter-12/Chapter-12-Item-89-For-instance-control-prefer-enum-types-to-readResolve.md)）。否则，每次反序列化实例时，都会创建一个新实例，在我们的示例中，这会导致出现虚假的 Elvis。为了防止这种情况发生，将这个 readResolve 方法添加到 Elvis 类中：

```
// readResolve method to preserve singleton property
private Object readResolve() {
    // Return the one true Elvis and let the garbage collector
    // take care of the Elvis impersonator.
    return INSTANCE;
}
```

### 3.枚举
实现单例的第三种方法是声明一个单元素枚举：

```
// Enum singleton - the preferred approach
public enum Elvis {
    INSTANCE;
    public void leaveTheBuilding() { ... }
}
```

这种方法类似于 public 字段方法，但是它更简洁，默认提供了序列化机制，提供了对多个实例化的严格保证，即使面对复杂的序列化或反射攻击也是如此。这种方法可能有点不自然，但是**单元素枚举类型通常是实现单例的最佳方法。** 注意，如果你的单例必须扩展一个超类而不是 Enum（尽管你可以声明一个 Enum 来实现接口），你就不能使用这种方法。
