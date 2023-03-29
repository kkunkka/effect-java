# 第一节：以静态工厂方法代替构造函数

---

客户端获得实例的传统方式是由类提供一个公共构造函数。还有一种技术应该成为每个程序员技能树的一部分。一个类可以提供公共静态工厂方法，它只是一个返回类实例的静态方法。下面是一个来自 Boolean （boolean 的包装类）的简单示例。该方法将 boolean 基本类型转换为 Boolean 对象的引用：

```Java
public static Boolean valueOf(boolean b) {
    return b ? Boolean.TRUE : Boolean.FALSE;
}
```

:::tip
静态工厂方法与来自设计模式的工厂方法模式不同 。本条目中描述的静态工厂方法在设计模式中没有直接等价的方法。
:::

除了公共构造函数，一个类还可以通过静态工厂方法提供它的客户端。使用静态工厂方法而不是公共构造函数的方式既有优点也有缺点。

## 静态工厂方法的优点

### 1. 静态工厂方法有确切名称

如果构造函数的参数本身并不能描述返回的对象，那么具有确切名称的静态工厂则更容易使用，生成的客户端代码也更容易阅读。例如，返回可能为素数的 BigInteger 类的构造函数 `BigInteger(int, int, Random)` 最好表示为名为 `BigInteger.probablePrime` 的静态工厂方法。（这个方法是在 Java 4 中添加的）

一个类只能有一个具有给定签名的构造函数。众所周知，程序员可以通过提供多个构造函数来绕过这个限制，这些构造函数的参数列表仅在参数类型、个数或顺序上有所不同。这真是个坏主意。面对这样一个 API，用户将永远无法记住该用哪个构造函数，并且最终会错误地调用不适合的构造函数。如果不参考类文档，阅读使用这些构造函数代码的人就不会知道代码的作用。

因为静态工厂方法有确切名称，所以它们没有前一段讨论的局限。如果一个类需要具有相同签名的多个构造函数，那么用静态工厂方法替换构造函数，并仔细选择名称以突出它们的区别。

### 2. 静态工厂方法不需要在每次调用时创建新对象

这允许不可变类（[Item-17](../Chapter-4/Chapter-4-Item-17-Minimize-mutability)）使用预先构造的实例，或在构造实例时缓存实例，并重复分配它们以避免创建不必要的重复对象。`Boolean.valueOf(boolean)` 方法说明了这种技术：它从不创建对象。这种技术类似于享元模式。如果经常请求相同的对象，特别是在创建对象的代价很高时，它可以极大地提高性能。

静态工厂方法在重复调用中能够返回相同对象，这样的能力允许类在任何时候都能严格控制存在的实例。这样的类被称为实例受控的类。编写实例受控的类有几个原因。实例控制允许一个类来保证它是一个单例（[第三节](../Chapter-2/Chapter-2-Item-3-Enforce-the-singleton-property-with-a-private-constructor-or-an-enum-type)）或不可实例化的（[第四节](..Chapter-2/Chapter-2-Item-4-Enforce-noninstantiability-with-a-private-constructor)）。同时，它允许一个不可变的值类（[第十七节](../Chapter-4/Chapter-4-Item-17-Minimize-mutability)）保证不存在两个相同的实例：`a.equals(b)` 当且仅当 `a==b` 为 true。这是享元模式的基础。枚举类型（[第三十四节](../Chapter-6/Chapter-6-Item-34-Use-enums-instead-of-int-constants)）提供了这种保证。

### 3. 可以通过静态工厂方法获取返回类型的任何子类的对象

这为选择返回对象的类提供了很大的灵活性。
这种灵活性的一个应用是 API 可以在不公开其类的情况下返回对象。以这种方式隐藏实现类会形成一个非常紧凑的 API。这种技术适用于基于接口的框架（[第二十节](../Chapter-4/Chapter-4-Item-20-Prefer-interfaces-to-abstract-classes)），其中接口为静态工厂方法提供了自然的返回类型。

在 Java 8 之前，接口不能有静态方法。按照惯例，一个名为 Type 的接口的静态工厂方法被放在一个名为 Types 的不可实例化的伴随类（[第四节](../Chapter-2/Chapter-2-Item-4-Enforce-noninstantiability-with-a-private-constructor)）中。例如，Java 的 Collections 框架有 45 个接口实用工具实现，提供了不可修改的集合、同步集合等。几乎所有这些实现都是通过一个非实例化类（`java.util.Collections`）中的静态工厂方法导出的。返回对象的类都是非公共的。

Collections 框架 API 比它导出 45 个独立的公共类要小得多，每个公共类对应一个方便的实现。减少的不仅仅是 API 的数量，还有概念上的减少：程序员为了使用 API 必须掌握的概念的数量和难度。程序员知道返回的对象是由相关的接口精确地指定的，因此不需要为实现类阅读额外的类文档。此外，使用这种静态工厂方法需要客户端通过接口而不是实现类引用返回的对象，这通常是很好的做法（[第六十四节](../Chapter-9/Chapter-9-Item-64-Refer-to-objects-by-their-interfaces)）。

自 Java 8 起，消除了接口不能包含静态方法的限制，因此通常没有理由为接口提供不可实例化的伴随类。许多公共静态成员应该放在接口本身中，而不是放在类中。但是，请注意，仍然有必要将这些静态方法背后的大部分实现代码放到单独的包私有类中。这是因为 Java 8 要求接口的所有静态成员都是公共的。Java 9 允许私有静态方法，但是静态字段和静态成员类仍然需要是公共的。

### 4. 返回对象的类可以随调用的不同而变化，作为输入参数的函数

声明的返回类型的任何子类型都是允许的。返回对象的类也可以因版本而异。

EnumSet 类[第三十六项](../Chapter-6/Chapter-6-Item-36-Use-EnumSet-instead-of-bit-fields)）没有公共构造函数，只有静态工厂。在 OpenJDK 实现中，它们返回两个子类中的一个实例，这取决于底层 enum 类型的大小：如果它有 64 个或更少的元素，就像大多数 enum 类型一样，静态工厂返回一个 long 类型的 RegularEnumSet 实例；如果 enum 类型有 65 个或更多的元素，工厂将返回一个由 `long[]` 类型的 JumboEnumSet 实例。

客户端看不到这两个实现类的存在。如果 RegularEnumSet 不再为小型 enum 类型提供性能优势，它可能会在未来的版本中被消除，而不会产生不良影响。类似地，如果事实证明 EnumSet 有益于性能，未来的版本可以添加第三或第四个 EnumSet 实现。客户端既不知道也不关心从工厂返回的对象的类；它们只关心它是 EnumSet 的某个子类。

### 5. 当编写包含方法的类时，返回对象的类不需要存在

这种灵活的静态工厂方法构成了服务提供者框架的基础，比如 Java 数据库连接 API（JDBC）。服务提供者框架是一个系统，其中提供者实现一个服务，系统使客户端可以使用这些实现，从而将客户端与实现分离。

服务提供者框架中有三个基本组件：代表实现的服务接口；提供者注册 API，提供者使用它来注册实现，以及服务访问 API，客户端使用它来获取服务的实例。服务访问 API 允许客户端指定选择实现的标准。在没有这些条件的情况下，API 返回一个默认实现的实例，或者允许客户端循环使用所有可用的实现。服务访问 API 是灵活的静态工厂，它构成了服务提供者框架的基础。

服务提供者框架的第四个可选组件是服务提供者接口，它描述了产生服务接口实例的工厂对象。在没有服务提供者接口的情况下，必须以反射的方式实例化实现（[第六十五节](../Chapter-9/Chapter-9-Item-65-Prefer-interfaces-to-reflection)）。在 JDBC 中，连接扮演服务接口 DriverManager 的角色。`DriverManager.registerDriver` 是提供商注册的 API，`DriverManager.getConnection` 是服务访问 API，驱动程序是服务提供者接口。

服务提供者框架模式有许多变体。例如，服务访问 API 可以向客户端返回比提供者提供的更丰富的服务接口。这是桥接模式 。依赖注入框架（[第五节](../Chapter-2/Chapter-2-Item-5-Prefer-dependency-injection-to-hardwiring-resources)）可以看作是强大的服务提供者。由于是 Java 6，该平台包括一个通用服务提供者框架 `Java.util.ServiceLoader`，所以你不需要，通常也不应该自己写（[第五十九节](../Chapter-9/Chapter-9-Item-59-Know-and-use-the-libraries)）。JDBC 不使用 ServiceLoader，因为前者比后者要早。

## 静态工厂方法的缺点

### 1. 仅提供静态工厂方法的主要局限是，没有公共或受保护构造函数的类不能被子类化

例如，不可能在集合框架中子类化任何方便的实现类。这可能是一种因祸得福的做法，因为它鼓励程序员使用组合而不是继承（[第十八节](../Chapter-4/Chapter-4-Item-18-Favor-composition-over-inheritance)），并且对于不可变的类型（[第十七节](../Chapter-4/Chapter-4-Item-17-Minimize-mutability)）是必需的。

### 2. 静态工厂方法的第二个缺点是程序员很难找到它们

它们在 API 文档中不像构造函数那样引人注目，因此很难弄清楚如何实例化一个只提供静态工厂方法而没有构造函数的类。Javadoc 工具总有一天会关注到静态工厂方法。与此同时，你可以通过在类或接口文档中对静态工厂方法多加留意，以及遵守通用命名约定的方式来减少这个困扰。下面是一些静态工厂方法的常用名称。这个列表还远不够详尽：

- from-A，一种型转换方法，该方法接受单个参数并返回该类型的相应实例，例如：
  
  ```Java
  Date d = Date.from(instant);
  ```

- of-An，一个聚合方法，它接受多个参数并返回一个包含这些参数的实例，例如：
  
  ```Java
  Set<Rank> faceCards = EnumSet.of(JACK, QUEEN, KING);
  ```

- valueOf-A，一种替代 from 和 of 但更冗长的方法，例如：
  
  ```Java
  BigInteger prime = BigInteger.valueOf(Integer.MAX_VALUE);
  ```

- instance 或 getInstance，返回一个实例，该实例由其参数（如果有的话）描述，但不具有相同的值，例如：
  
  ```Java
  StackWalker luke = StackWalker.getInstance(options);
  ```

- create 或 newInstance，与 instance 或 getInstance 类似，只是该方法保证每个调用都返回一个新实例，例如：
  
  ```Java
  Object newArray = Array.newInstance(classObject, arrayLen);
  ```

- getType，类似于 getInstance，但如果工厂方法位于不同的类中，则使用此方法。其类型是工厂方法返回的对象类型，例如：
  
  ```Java
  FileStore fs = Files.getFileStore(path);
  ```

- newType，与 newInstance 类似，但是如果工厂方法在不同的类中使用。类型是工厂方法返回的对象类型，例如：
  
  ```Java
  BufferedReader br = Files.newBufferedReader(path);
  ```

- type，一个用来替代 getType 和 newType 的比较简单的方式，例如：
  
  ```Java
  List<Complaint> litany = Collections.list(legacyLitany);
  ```

## 总结

总之，静态工厂方法和公共构造器都有各自的用途，理解它们相比而言的优点是值得的。通常静态工厂的方式更可取，因此应避免在没有考虑静态工厂的情况下就提供公共构造函数。