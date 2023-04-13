# 第四十二节:  λ 表达式优于匿名类

在历史上，带有单个抽象方法的接口（或者抽象类，但这种情况很少）被用作函数类型。它们的实例（称为函数对象）表示函数或操作。自从 JDK 1.1 在 1997 年发布以来，创建函数对象的主要方法就是匿名类（[Item-24](../Chapter-4/Chapter-4-Item-24-Favor-static-member-classes-over-nonstatic)）。下面是一个按长度对字符串列表进行排序的代码片段，使用一个匿名类来创建排序的比较函数（它强制执行排序顺序）：

```
// Anonymous class instance as a function object - obsolete!
Collections.sort(words, new Comparator<String>() {
    public int compare(String s1, String s2) {
        return Integer.compare(s1.length(), s2.length());
    }
});
```

匿名类对于需要函数对象的典型面向对象设计模式来说已经足够了，尤其是策略模式 [Gamma95]。Comparator 接口表示排序的抽象策略；上述匿名类是对字符串排序的一种具体策略。然而，匿名类的冗长使函数式编程在 Java 中变得毫无吸引力。

在 Java 8 中官方化了一个概念，即具有单个抽象方法的接口是特殊的，应该得到特殊处理。这些接口现在被称为函数式接口，允许使用 lambda 表达式创建这些接口的实例。Lambda 表达式在功能上类似于匿名类，但是更加简洁。下面的代码片段，匿名类被 lambda 表达式替换。已经没有了原有刻板的样子，意图非常明显：

```
// Lambda expression as function object (replaces anonymous class)
Collections.sort(words,(s1, s2) -> Integer.compare(s1.length(), s2.length()));
```

注意，lambda 表达式（`Comparator<String>`）、它的参数（s1 和 s2，都是字符串）及其返回值（int）的类型在代码中不存在。编译器使用称为类型推断的过程从上下文中推断这些类型。在某些情况下，编译器无法确定类型，你必须显式指定它们。类型推断的规则很复杂：它们在 JLS 中占了整整一章 [JLS, 18]。很少有程序员能详细理解这些规则，但这没有关系。**省略所有 lambda 表达式参数的类型，除非它们的存在使你的程序更清晰。** 如果编译器生成一个错误，告诉你它不能推断 lambda 表达式参数的类型，那么就显式指定它。有时你可能必须强制转换返回值或整个 lambda 表达式，但这种情况很少见。

关于类型推断，有些警告应该被提及。[Item-26](../Chapter-5/Chapter-5-Item-26-Do-not-use-raw-types) 告诉你不要使用原始类型，[Item-29](../Chapter-5/Chapter-5-Item-29-Favor-generic-types) 告诉你要优先使用泛型，[Item-30](../Chapter-5/Chapter-5-Item-30-Favor-generic-methods) 告诉你要优先使用泛型方法。在使用 lambda 表达式时，这些建议尤其重要，因为编译器获得了允许它从泛型中执行类型推断的大部分类型信息。如果不提供此信息，编译器将无法进行类型推断，并且必须在 lambda 表达式中手动指定类型，这将大大增加它们的冗长。举例来说，如果变量声明为原始类型 List 而不是参数化类型 `List<String>`，那么上面的代码片段将无法编译。

顺便说一下，如果使用 comparator 构造方法代替 lambda 表达式（[Item-14](../Chapter-3/Chapter-3-Item-14-Consider-implementing-Comparable)），那么代码片段可以变得更加简洁：

```
Collections.sort(words, comparingInt(String::length));
```

事实上，通过 Java 8 中添加到 List 接口的 sort 方法，可以使代码片段变得更短：

```
words.sort(comparingInt(String::length));
```

在语言中添加 lambda 表达式使得在以前没有意义的地方使用函数对象变得实际。例如，考虑 [Item-34](../Chapter-6/Chapter-6-Item-34-Use-enums-instead-of-int-constants) 中的操作枚举类型。因为每个枚举的 apply 方法需要不同的行为，所以我们使用特定于常量的类体并覆盖每个枚举常量中的 apply 方法。为了唤醒你的记忆，以下是代码：

```
// Enum type with constant-specific class bodies & data (Item 34)
public enum Operation {
    PLUS("+") {
        public double apply(double x, double y) { return x + y; }
    },
    MINUS("-") {
        public double apply(double x, double y) { return x - y; }
    },
    TIMES("*") {
        public double apply(double x, double y) { return x * y; }
    },
    DIVIDE("/") {
        public double apply(double x, double y) { return x / y; }
    };

    private final String symbol;

    Operation(String symbol) { this.symbol = symbol; }

    @Override
    public String toString() { return symbol; }

    public abstract double apply(double x, double y);
}
```

[Item-34](../Chapter-6/Chapter-6-Item-34-Use-enums-instead-of-int-constants) 指出，枚举实例字段比特定于常量的类体更可取。Lambda 表达式使得使用前者取代后者来实现特定于常量的行为变得容易。只需将实现每个枚举常量的行为的 lambda 表达式传递给它的构造函数。构造函数将 lambda 表达式存储在实例字段中，apply 方法将调用转发给 lambda 表达式。生成的代码比原始版本更简单、更清晰：

```
// Enum with function object fields & constant-specific behavior
public enum Operation {
    PLUS ("+", (x, y) -> x + y),
    MINUS ("-", (x, y) -> x - y),
    TIMES ("*", (x, y) -> x * y),
    DIVIDE("/", (x, y) -> x / y);

    private final String symbol;

    private final DoubleBinaryOperator op;

    Operation(String symbol, DoubleBinaryOperator op) {
        this.symbol = symbol;
        this.op = op;
    }

    @Override public String toString() { return symbol; }

    public double apply(double x, double y) {
        return op.applyAsDouble(x, y);
    }
}
```

注意，我们对表示枚举常量行为的 lambda 表达式使用了 DoubleBinaryOperator 接口。这是 `java.util.function` （[Item-44](../Chapter-7/Chapter-7-Item-44-Favor-the-use-of-standard-functional-interfaces)）中许多预定义的函数式接口之一。它表示一个函数，该函数接收两个 double 类型参数，并且返回值也为 double 类型。

查看基于 lambda 表达式的操作 enum，你可能会认为特定于常量的方法体已经过时了，但事实并非如此。与方法和类不同，**lambda 表达式缺少名称和文档；如果一个算法并非不言自明，或者有很多行代码，不要把它放在 lambda 表达式中。** 一行是理想的，三行是合理的最大值。如果你违反了这一规则，就会严重损害程序的可读性。如果 lambda 表达式很长或者很难读，要么找到一种方法来简化它，要么重构你的程序。此外，传递给 enum 构造函数的参数在静态上下文中计算。因此，enum 构造函数中的 lambda 表达式不能访问枚举的实例成员。如果枚举类型具有难以理解的特定于常量的行为，无法在几行代码中实现，或者需要访问实例字段或方法，则仍然需要特定于常量的类。

同样，你可能认为匿名类在 lambda 表达式时代已经过时了。这更接近事实，但是有一些匿名类可以做的事情是 lambda 表达式不能做的。Lambda 表达式仅限于函数式接口。如果想创建抽象类的实例，可以使用匿名类，但不能使用 lambda 表达式。类似地，你可以使用匿名类来创建具有多个抽象方法的接口实例。最后，lambda 表达式无法获得对自身的引用。在 lambda 表达式中，this 关键字指的是封闭实例，这通常是你想要的。在匿名类中，this 关键字引用匿名类实例。如果你需要从函数对象的内部访问它，那么你必须使用一个匿名类。

Lambda 表达式与匿名类共享无法通过实现可靠地序列化和反序列化它们的属性。因此，**很少（如果有的话）序列化 lambda**（或匿名类实例）。如果你有一个想要序列化的函数对象，比如比较器，那么使用私有静态嵌套类的实例（[Item-24](../Chapter-4/Chapter-4-Item-24-Favor-static-member-classes-over-nonstatic)）。

总之，在 Java 8 中，lambda 表达式是迄今为止表示小函数对象的最佳方式。**不要对函数对象使用匿名类，除非你必须创建非函数式接口类型的实例。** 另外，请记住，lambda 表达式使表示小函数对象变得非常容易，从而为 Java 以前不实用的函数式编程技术打开了大门。
