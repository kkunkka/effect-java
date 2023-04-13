# 第四十四节:  优先使用标准函数式接口

现在 Java 已经有了 lambda 表达式，编写 API 的最佳实践已经发生了很大的变化。例如，模板方法模式 [Gamma95]，其中子类覆盖基类方法以专门化其超类的行为，就没有那么有吸引力了。现代的替代方法是提供一个静态工厂或构造函数，它接受一个函数对象来实现相同的效果。更一般地，你将编写更多以函数对象为参数的构造函数和方法。选择正确的函数参数类型需要谨慎。

考虑 LinkedHashMap。你可以通过覆盖受保护的 removeEldestEntry 方法将该类用作缓存，每当向映射添加新键时，put 都会调用该方法。当该方法返回 true 时，映射将删除传递给该方法的最老条目。下面的覆盖允许映射增长到 100 个条目，然后在每次添加新键时删除最老的条目，维护 100 个最近的条目：

```
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return size() > 100;
}
```

这种技术工作得很好，但是使用 lambda 表达式可以做得更好。如果 LinkedHashMap 是现在编写的，它将有一个静态工厂或构造函数，它接受一个函数对象。看着 removeEldestEntry 的定义,你可能会认为这个函数对象应该 `Map.Entry<K,V>` 和返回一个布尔值，但不会完全做到：removeEldestEntry 方法调用 `size()` 地图中的条目的数量，这工作，因为 removeEldestEntry 在 Map 上是一个实例方法。传递给构造函数的函数对象不是 Map 上的实例方法，无法捕获它，因为在调用 Map 的工厂或构造函数时，Map 还不存在。因此，Map 必须将自身传递给函数对象，函数对象因此必须在输入端及其最老的条目上接受 Map。如果要声明这样一个函数式接口，它看起来是这样的：

```
// Unnecessary functional interface; use a standard one instead.
@FunctionalInterface interface EldestEntryRemovalFunction<K,V>{
    boolean remove(Map<K,V> map, Map.Entry<K,V> eldest);
}
```

这个接口可以很好地工作，但是你不应该使用它，因为你不需要为此声明一个新接口。`java.util.function` 包提供了大量的标准函数接口供你使用。**如果一个标准的函数式接口可以完成这项工作，那么你通常应该优先使用它，而不是使用专门构建的函数式接口。** 通过减少 API 的概念表面积，这将使你的 API 更容易学习，并将提供显著的互操作性优势，因为许多标准函数式接口提供了有用的默认方法。例如，Predicate 接口提供了组合谓词的方法。在我们的 LinkedHashMap 示例中，应该优先使用标准的 `BiPredicate<Map<K,V>`、`Map.Entry<K,V>>` 接口，而不是定制的 EldestEntryRemovalFunction 接口。

`java.util.function` 中有 43 个接口。不能期望你记住所有的接口，但是如果你记住了 6 个基本接口，那么你可以在需要时派生出其余的接口。基本接口操作对象引用类型。Operator 接口表示结果和参数类型相同的函数。Predicate 接口表示接受参数并返回布尔值的函数。Function 接口表示参数和返回类型不同的函数。Supplier 接口表示一个不接受参数并返回（或「供应」）值的函数。最后，Consumer 表示一个函数，该函数接受一个参数，但不返回任何内容，本质上是使用它的参数。六个基本的函数式接口总结如下：

|    Interface    |       Function Signature       |      Example     |
|:-------:|:-------:|:-------:|
|   `UnaryOperator<T>`  |     `T apply(T t)`    |   `String::toLowerCase`   |
|   `BinaryOperator<T>`  |     `T apply(T t1, T t2)`    |   `BigInteger::add`   |
|   `Predicate<T>`  |     `boolean test(T t)`    |   `Collection::isEmpty`   |
|   `Function<T,R>`  |     `R apply(T t)`    |   `Arrays::asList`   |
|   `Supplier<T>`  |     `T get()`    |   `Instant::now`   |
|   `Consumer<T>`  |     `void accept(T t)`    |   `System.out::println`   |

还有 6 个基本接口的 3 个变体，用于操作基本类型 int、long 和 double。它们的名称是通过在基本接口前面加上基本类型前缀而派生出来的。例如，一个接受 int 的 Predicate 就是一个 IntPredicate，一个接受两个 long 值并返回一个 long 的二元操作符就是一个 LongBinaryOperator。除了由返回类型参数化的函数变量外，这些变量类型都不是参数化的。例如，`LongFunction<int[]>` 使用 long 并返回一个 int[]。

Function 接口还有 9 个额外的变体，在结果类型为基本数据类型时使用。源类型和结果类型总是不同的，因为不同类型的函数本身都是 UnaryOperator。如果源类型和结果类型都是基本数据类型，则使用带有 SrcToResult 的前缀函数，例如 LongToIntFunction（六个变体）。如果源是一个基本数据类型，而结果是一个对象引用，则使用带前缀 `<Src>ToObj` 的 Function 接口，例如 DoubleToObjFunction（三个变体）。

三个基本函数式接口有两个参数版本，使用它们是有意义的：`BiPredicate<T,U>`、`BiFunction<T,U,R>`、`BiConsumer<T,U>`。也有 BiFunction 变体返回三个相关的基本类型：`ToIntBiFunction<T,U>`、 `ToLongBiFunction<T,U>`、`ToDoubleBiFunction<T,U>`。Consumer 有两个参数变体，它们接受一个对象引用和一个基本类型：`ObjDoubleConsumer<T>`、`ObjIntConsumer<T>`、`ObjLongConsumer<T>`。总共有9个基本接口的双参数版本。

最后是 BooleanSupplier 接口，它是 Supplier 的一个变体，返回布尔值。这是在任何标准函数接口名称中唯一显式提到布尔类型的地方，但是通过 Predicate 及其四种变体形式支持布尔返回值。前面描述的 BooleanSupplier 接口和 42 个接口占了全部 43 个标准函数式接口。不可否认，这有很多东西需要消化，而且不是非常直观。另一方面，你将需要的大部分函数式接口都是为你编写的，并且它们的名称足够常规，因此在需要时你应该不会遇到太多麻烦。

大多数标准函数式接口的存在只是为了提供对基本类型的支持。**不要尝试使用带有包装类的基本函数式接口，而不是使用基本类型函数式接口。** 当它工作时，它违反了 [Item-61](../Chapter-9/Chapter-9-Item-61-Prefer-primitive-types-to-boxed-primitives) 的建议，“与盒装原语相比，更喜欢原语类型”。在批量操作中使用装箱原语的性能后果可能是致命的。

现在你知道，与编写自己的接口相比，通常应该使用标准的函数式接口。但是你应该什么时候写你自己的呢？当然，如果标准的函数式接口都不能满足你的需要，那么你需要自行编写，例如，如果你需要一个接受三个参数的 Predicate，或者一个抛出已检查异常的 Predicate。但是有时候你应该编写自己的函数接口，即使其中一个标准接口在结构上是相同的。

考虑我们的老朋友 `Comparator<T>`，它在结构上与 `ToIntBiFunction<T,T>` 接口相同。即使后者接口在将前者添加到库时已经存在，使用它也是错误的。有几个原因说明比较器应该有自己的接口。首先，每次在 API 中使用 Comparator 时，它的名称都提供了优秀的文档，而且它的使用非常频繁。通过实现接口，你保证遵守其契约。第三，该接口大量配备了用于转换和组合比较器的有用默认方法。

如果你需要与 Comparator 共享以下一个或多个特性的函数式接口，那么你应该认真考虑编写一个专用的函数式接口，而不是使用标准接口：

- 它将被广泛使用，并且可以从描述性名称中获益。

- 它有一个强有力的约定。

- 它将受益于自定义默认方法。

如果你选择编写自己的函数式接口，请记住这是一个接口，因此应该非常小心地设计它（[Item-21](../Chapter-4/Chapter-4-Item-21-Design-interfaces-for-posterity)）。

注意 EldestEntryRemovalFunction 接口(第199页)使用 `@FunctionalInterface` 注释进行标记。这种注释类型在本质上类似于 `@Override`。它是程序员意图的声明，有三个目的：它告诉类及其文档的读者，接口的设计是为了启用 lambda 表达式；它使你保持诚实，因为接口不会编译，除非它只有一个抽象方法；它还可以防止维护者在接口发展过程中意外地向接口添加抽象方法。**总是用 `@FunctionalInterface` 注释你的函数接口。**

最后一点应该是关于 API 中函数式接口的使用。不要提供具有多个重载的方法，这些方法采用相同参数位置的不同函数式接口，否则会在客户机中造成可能的歧义。这不仅仅是一个理论问题。ExecutorService 的 submit 方法可以是 `Callable<T>` 级的，也可以是 Runnable 的，并且可以编写一个客户端程序，它需要一个类型转换来指示正确的重载(Item 52)。避免此问题的最简单方法是不要编写将不同函数式接口放在相同参数位置的重载。这是 [Item-52](../Chapter-8/Chapter-8-Item-52-Use-overloading-judiciously) 「明智地使用过载」建议的一个特例。

总之，既然 Java 已经有了 lambda 表达式，你必须在设计 API 时考虑 lambda 表达式。在输入时接受函数式接口类型，在输出时返回它们。一般情况下，最好使用 `java.util.function` 中提供的标准函数式接口，但请注意比较少见的一些情况，在这种情况下，你最好编写自己的函数式接口。
