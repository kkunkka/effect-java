# 第五节: 依赖注入优于硬连接资源

许多类依赖于一个或多个底层资源。例如，拼写检查程序依赖于字典。常见做法是，将这种类实现为静态实用工具类（[Item-4](../Chapter-2/Chapter-2-Item-4-Enforce-noninstantiability-with-a-private-constructor)）：

```
// Inappropriate use of static utility - inflexible & untestable!
public class SpellChecker {
    private static final Lexicon dictionary = ...;
    private SpellChecker() {} // Noninstantiable
    public static boolean isValid(String word) { ... }
    public static List<String> suggestions(String typo) { ... }
}
```

类似地，我们也经常看到它们的单例实现（[Item-3](../Chapter-2/Chapter-2-Item-3-Enforce-the-singleton-property-with-a-private-constructor-or-an-enum-type)）:

```
// Inappropriate use of singleton - inflexible & untestable!
public class SpellChecker {
    private final Lexicon dictionary = ...;
    private SpellChecker(...) {}
    public static INSTANCE = new SpellChecker(...);
    public boolean isValid(String word) { ... }
    public List<String> suggestions(String typo) { ... }
}
```

这两种方法都不令人满意，因为它们假设只使用一个字典。在实际应用中，每种语言都有自己的字典，特殊的字典用于特殊的词汇表。另外，最好使用一个特殊的字典进行测试。认为一本字典就足够了，是一厢情愿的想法。

你可以尝试让 SpellChecker 支持多个字典：首先取消 dictionary 字段的 final 修饰，并在现有的拼写检查器中添加更改 dictionary 的方法。但是在并发环境中这种做法是笨拙的、容易出错的和不可行的。**静态实用工具类和单例不适用于由底层资源参数化的类。**

所需要的是支持类的多个实例的能力（在我们的示例中是 SpellChecker），每个实例都使用客户端需要的资源（在我们的示例中是 dictionary）。满足此要求的一个简单模式是在**创建新实例时将资源传递给构造函数。** 这是依赖注入的一种形式：字典是拼写检查器的依赖项，在创建它时被注入到拼写检查器中。

```
// Dependency injection provides flexibility and testability
public class SpellChecker {
    private final Lexicon dictionary;
    public SpellChecker(Lexicon dictionary) {
        this.dictionary = Objects.requireNonNull(dictionary);
    }
    public boolean isValid(String word) { ... }
    public List<String> suggestions(String typo) { ... }
}
```

依赖注入模式非常简单，许多程序员在不知道其名称的情况下使用了多年。虽然拼写检查器示例只有一个资源（字典），但是依赖注入可以处理任意数量的资源和任意依赖路径。它保持了不可变性（[Item-17](../Chapter-4/Chapter-4-Item-17-Minimize-mutability)），因此多个客户端可以共享依赖对象（假设客户端需要相同的底层资源）。依赖注入同样适用于构造函数、静态工厂（[Item-1](../Chapter-2/Chapter-2-Item-1-Consider-static-factory-methods-instead-of-constructors)）和构建器（[Item-2](../Chapter-2/Chapter-2-Item-2-Consider-a-builder-when-faced-with-many-constructor-parameters)）。

这种模式的一个有用变体是将资源工厂传递给构造函数。工厂是一个对象，可以反复调用它来创建类型的实例。这样的工厂体现了工厂方法模式 [Gamma95]。Java 8 中引入的 `Supplier<T>` 非常适合表示工厂。在输入中接受 `Supplier<T>` 的方法通常应该使用有界通配符类型（[Item-31](../Chapter-5/Chapter-5-Item-31-Use-bounded-wildcards-to-increase-API-flexibility)）来约束工厂的类型参数，以允许客户端传入创建指定类型的任何子类型的工厂。例如，这里有一个生产瓷砖方法，每块瓷砖都使用客户提供的工厂来制作马赛克：

```
Mosaic create(Supplier<? extends Tile> tileFactory) { ... }
```

尽管依赖注入极大地提高了灵活性和可测试性，但它可能会使大型项目变得混乱，这些项目通常包含数千个依赖项。通过使用依赖注入框架（如 Dagger、Guice 或 Spring），几乎可以消除这种混乱。这些框架的使用超出了本书的范围，但是请注意，为手动依赖注入而设计的 API 很容易被这些框架所使用。

总之，不要使用单例或静态实用工具类来实现依赖于一个或多个底层资源的类，这些资源的行为会影响类的行为，也不要让类直接创建这些资源。相反，将创建它们的资源或工厂传递给构造函数（或静态工厂或构建器）。这种操作称为依赖注入，它将大大增强类的灵活性、可复用性和可测试性。
