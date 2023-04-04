# 第五十五节: 明智地的返回 Optional

在 Java 8 之前，在编写在某些情况下无法返回值的方法时，可以采用两种方法。要么抛出异常，要么返回 null（假设返回类型是对象引用类型）。这两种方法都不完美。应该为异常条件保留异常（[Item-69](/Chapter-10/Chapter-10-Item-69-Use-exceptions-only-for-exceptional-conditions.md)），并且抛出异常代价高昂，因为在创建异常时捕获整个堆栈跟踪。返回 null 没有这些缺点，但是它有自己的缺点。如果方法返回 null，客户端必须包含特殊情况代码来处理 null 返回的可能性，除非程序员能够证明 null 返回是不可能的。如果客户端忽略检查 null 返回并将 null 返回值存储在某个数据结构中，那么 NullPointerException 可能会在将来的某个时间，在代码中的某个与该问题无关的位置产生。

在 Java 8 中，还有第三种方法来编写可能无法返回值的方法。`Optional<T>` 类表示一个不可变的容器，它可以包含一个非空的 T 引用，也可以什么都不包含。不包含任何内容的 Optional 被称为空。一个值被认为存在于一个非空的 Optional 中。Optional 的本质上是一个不可变的集合，它最多可以容纳一个元素。`Optional<T>` 不实现 `Collection<T>`，但原则上可以。

理论上应返回 T，但在某些情况下可能无法返回 T 的方法可以将返回值声明为 `Optional<T>`。这允许该方法返回一个空结果来表明它不能返回有效的结果。具备 Optional 返回值的方法比抛出异常的方法更灵活、更容易使用，并且比返回 null 的方法更不容易出错。

在 [Item-30](/Chapter-5/Chapter-5-Item-30-Favor-generic-methods.md) 中，我们展示了根据集合元素的自然顺序计算集合最大值的方法。

```
// Returns maximum value in collection - throws exception if empty
public static <E extends Comparable<E>> E max(Collection<E> c) {
    if (c.isEmpty())
        throw new IllegalArgumentException("Empty collection");
    E result = null;
    for (E e : c)
        if (result == null || e.compareTo(result) > 0)
    result = Objects.requireNonNull(e);
    return result;
}
```

如果给定集合为空，此方法将抛出 IllegalArgumentException。我们在 [Item-30](/Chapter-5/Chapter-5-Item-30-Favor-generic-methods.md) 中提到，更好的替代方法是返回 `Optional<E>`。

```
// Returns maximum value in collection as an Optional<E>
public static <E extends Comparable<E>> Optional<E> max(Collection<E> c) {
    if (c.isEmpty())
        return Optional.empty();
    E result = null;
    for (E e : c)
        if (result == null || e.compareTo(result) > 0)
    result = Objects.requireNonNull(e);
    return Optional.of(result);
}
```

如你所见，返回一个 Optional 是很简单的。你所要做的就是使用适当的静态工厂创建。在这个程序中，我们使用了两个静态工厂：`Optional.empty()` 返回一个空的 Optional，`Optional.of(value)` 返回一个包含给定非空值的可选值。将 null 传递给 `Optional.of(value)` 是一个编程错误。如果你这样做，该方法将通过抛出 NullPointerException 来响应。`Optional.ofNullable(value)` 方法接受一个可能为空的值，如果传入 null，则返回一个空的 Optional。**永远不要从具备 Optional 返回值的方法返回空值:** 它违背了这个功能的设计初衷。

许多流上的 Terminal 操作返回 Optional。如果我们使用一个流来重写 max 方法，那么流版本的 max 操作会为我们生成一个 Optional（尽管我们必须传递一个显式的 comparator）：

```
// Returns max val in collection as Optional<E> - uses stream
public static <E extends Comparable<E>> Optional<E> max(Collection<E> c) {
    return c.stream().max(Comparator.naturalOrder());
}
```

那么，如何选择是返回 Optional 而不是返回 null 或抛出异常呢？Optional 在本质上类似于已检查异常（[Item-71](/Chapter-10/Chapter-10-Item-71-Avoid-unnecessary-use-of-checked-exceptions.md)），因为它们迫使 API 的用户面对可能没有返回值的事实。抛出未检查的异常或返回 null 允许用户忽略这种可能性，从而带来潜在的可怕后果。但是，抛出一个已检查的异常需要在客户端中添加额外的样板代码。

如果一个方法返回一个 Optional，客户端可以选择如果该方法不能返回值该采取什么操作。你可以指定一个默认值：

```
// Using an optional to provide a chosen default value
String lastWordInLexicon = max(words).orElse("No words...");
```

或者你可以抛出任何适当的异常。注意，我们传递的是异常工厂，而不是实际的异常。这避免了创建异常的开销，除非它实际被抛出：

```
// Using an optional to throw a chosen exception
Toy myToy = max(toys).orElseThrow(TemperTantrumException::new);
```

如果你能证明一个 Optional 非空，你可以从 Optional 获取值，而不需要指定一个操作来执行，如果 Optional 是空的，但是如果你错了，你的代码会抛出一个 NoSuchElementException：

```
// Using optional when you know there’s a return value
Element lastNobleGas = max(Elements.NOBLE_GASES).get();
```

有时候，你可能会遇到这样一种情况：获取默认值的代价很高，除非必要，否则你希望避免这种代价。对于这些情况，Optional 提供了一个方法，该方法接受 `Supplier<T>`，并仅在必要时调用它。这个方法被称为 orElseGet，但是它可能应该被称为 orElseCompute，因为它与以 compute 开头的三个 Map 方法密切相关。有几个 Optional 的方法来处理更特殊的用例：filter、map、flatMap 和 ifPresent。在 Java 9 中，又添加了两个这样的方法：or 和 ifPresentOrElse。如果上面描述的基本方法与你的实例不太匹配，请查看这些更高级方法的文档，确认它们是否能够完成任务。

如果这些方法都不能满足你的需要，Optional 提供 `isPresent()` 方法，可以将其视为安全阀。如果 Optional 包含值，则返回 true；如果为空，则返回 false。你可以使用此方法对 Optional 结果执行任何你希望进行的处理，但请确保明智地使用它。`isPresent()` 的许多用途都可以被上面提到的方法所替代，如此生成的代码可以更短、更清晰、更符合习惯。

例如，考虑这段代码，它打印一个进程的父进程的 ID，如果进程没有父进程，则打印 N/A。该代码段使用了在 Java 9 中引入的 ProcessHandle 类：

```
Optional<ProcessHandle> parentProcess = ph.parent();
System.out.println("Parent PID: " + (parentProcess.isPresent() ?
String.valueOf(parentProcess.get().pid()) : "N/A"));
```

上面的代码片段可以替换为如下形式，它使用了 Optional 的 map 函数：

```
System.out.println("Parent PID: " + ph.parent().map(h -> String.valueOf(h.pid())).orElse("N/A"));
```

当使用流进行编程时，通常会发现你经常使用 `Stream<Optional<T>>`，并且需要一个 `Stream<T>`，其中包含非空 Optional 中的所有元素，以便继续。如果你正在使用 Java 8，下面的语句演示了如何弥补这个不足：

```
streamOfOptionals.filter(Optional::isPresent).map(Optional::get)
```

在 Java 9 中，Optional 配备了一个 `stream()` 方法。这个方法是一个适配器，它将一个 Optional 元素转换成一个包含元素的流（如果一个元素出现在 Optional 元素中），如果一个元素是空的，则一个元素都没有。与 Stream 的 flatMap 方法（[Item-45](/Chapter-7/Chapter-7-Item-45-Use-streams-judiciously.md)）相结合，这个方法为上面的代码段提供了一个简洁的替换版本：

```
streamOfOptionals..flatMap(Optional::stream)
```

并不是所有的返回类型都能从 Optional 处理中获益。**容器类型，包括集合、Map、流、数组和 Optional，不应该封装在 Optional 中。** 你应该简单的返回一个空的 `List<T>`，而不是一个空的 `Optional<List<T>>`（[Item-54](/Chapter-8/Chapter-8-Item-54-Return-empty-collections-or-arrays-not-nulls.md)）。返回空容器将消除客户端代码处理 Optional 容器的需要。ProcessHandle 类确实有 arguments 方法，它返回 `Optional<String[]>`，但是这个方法应该被视为一种特例，不应该被仿效。

那么，什么时候应该声明一个方法来返回 `Optional<T>` 而不是 T 呢？作为规则，**你应该声明一个方法来返回 `Optional<T>`（如果它可能无法返回结果），如果没有返回结果，客户端将不得不执行特殊处理。** 也就是说，返回 `Optional<T>` 并不是没有代价的。Optional 对象必须分配和初始化，从 Optional 对象中读取值需要额外的间接操作。这使得 Optional 不适合在某些性能关键的情况下使用。某一特定方法是否属于这一情况只能通过仔细衡量来确定（[Item-67](/Chapter-9/Chapter-9-Item-67-Optimize-judiciously.md)）。

与返回基本数据类型相比，返回包含包装类的 Optional 类型的代价高得惊人，因为 Optional 类型有两个装箱级别，而不是零。因此，库设计人员认为应该为基本类型 int、long 和 double 提供类似的 `Optional<T>`。这些可选类型是 OptionalInt、OptionalLong 和 OptionalDouble。它们包含 `Optional<T>` 上的大多数方法，但不是所有方法。因此，**永远不应该返包装类的 Optional**，可能除了「次基本数据类型」，如 Boolean、Byte、Character、Short 和 Float 之外。

到目前为止，我们已经讨论了返回 Optional 并在返回后如何处理它们。我们还没有讨论其他可能的用法，这是因为大多数其他 Optional 用法都是值得疑的。例如，永远不要将 Optional 用作 Map 的值。如果这样做，则有两种方法可以表示键在 Map 中逻辑上的缺失：键可以不在 Map 中，也可以存在并映射到空的 Optional。这代表了不必要的复杂性，很有可能导致混淆和错误。更一般地说，**在集合或数组中使用 Optional 作为键、值或元素几乎都是不合适的。**

这留下了一个悬而未决的大问题。在实例字段中存储 Optional 字段是否合适？通常这是一种「代码中的不良习惯」：建议你可能应该有一个包含 Optional 字段的子类。但有时这可能是合理的。考虑 [Item-2](/Chapter-2/Chapter-2-Item-2-Consider-a-builder-when-faced-with-many-constructor-parameters.md) 中的 NutritionFacts 类的情况。NutritionFacts 实例包含许多不需要的字段。不能为这些字段的所有可能组合提供子类。此外，字段具有原始类型，这使得直接表示缺少非常困难。对于 NutritionFacts 最好的 API 将为每个可选字段从 getter 返回一个 Optional，因此将这些 Optional 作为字段存储在对象中是很有意义的。

总之，如果你发现自己编写的方法不能总是返回确定值，并且你认为该方法的用户在每次调用时应该考虑这种可能性，那么你可能应该让方法返回一个 Optional。但是，你应该意识到，返回 Optional 会带来实际的性能后果；对于性能关键的方法，最好返回 null 或抛出异常。最后，除了作为返回值之外，你几乎不应该以任何其他方式使用 Optional。