# 第四十三节: 方法引用优于 λ 表达式

lambda 表达式与匿名类相比，主要优势是更简洁。Java 提供了一种方法来生成比 lambda 表达式更简洁的函数对象：方法引用。下面是一个程序的代码片段，该程序维护从任意键到 Integer 类型值的映射。如果该值被解释为键实例数的计数，那么该程序就是一个多集实现。该代码段的功能是，如果数字 1 不在映射中，则将其与键关联，如果键已经存在，则将关联值递增：

```
map.merge(key, 1, (count, incr) -> count + incr);
```

注意，这段代码使用了 merge 方法，它是在 Java 8 中添加到 Map 接口的。如果给定键没有映射，则该方法只插入给定的值；如果已经存在映射，则 merge 将给定的函数应用于当前值和给定值，并用结果覆盖当前值。这段代码代表了 merge 方法的一个典型用例。

代码读起来不错，但是仍然有一些刻板。参数计数和 incr 不会增加太多的价值，而且它们会占用相当大的空间。实际上，lambda 表达式告诉你的是函数返回两个参数的和。在 Java 8 中，Integer（和所有其他基本类型的包装类）提供了一个静态方法 sum，它的作用完全相同。我们可以简单地传递一个引用到这个方法，并得到相同的结果，同时减少视觉混乱：

```
map.merge(key, 1, Integer::sum);
```

一个方法的参数越多，就可以通过一个方法引用消除越多的刻板模式。然而，在某些 lambda 表达式中，您选择的参数名提供了有用的文档，使得 lambda 表达式比方法引用更易于阅读和维护，即使 lambda 表达式更长。

对于方法引用，没有什么是你不能对 lambda 表达式做的（只有一个模糊的例外，如果你好奇的话可参见 [JLS, 9.9-2]）。也就是说，方法引用通常会产生更短、更清晰的代码。如果 lambda 表达式太长或太复杂，它们还会给出一个输出：可以将代码从 lambda 表达式提取到一个新方法中，并以对该方法的引用替换 lambda 表达式。可以为该方法起一个好名字，并将其文档化以满足需要。

如果你使用 IDE 编程，它将在任何可能的地方建议用方法引用替换 lambda 表达式。通常应该（但不总是）接受 IDE 的建议。有时候，lambda 表达式会比方法引用更简洁。当方法与 lambda 表达式在同一个类中时，这种情况最常见。例如，考虑这段代码片段，它假定发生在一个名为 GoshThisClassNameIsHumongous 的类中：

```
service.execute(GoshThisClassNameIsHumongous::action);
```

The lambda equivalent looks like this:

使用 lambda 表达式是这样的：

```
service.execute(() -> action());
```

使用方法引用的代码片段并不比使用 lambda 表达式的代码片段短，也不清楚，所以选择后者。类似地，函数接口提供了一个通用静态工厂方法来返回标识函数 `Function.identity()`。不使用这个方法，而是一行中编写等价的 lambda 表达式：`x -> x`，通常更短，也更简洁。

许多方法引用引用静态方法，但是有四种方法不引用静态方法。其中两个是绑定和非绑定实例方法引用。在绑定引用中，接收对象在方法引用中指定。绑定引用在本质上与静态引用相似：函数对象接受与引用方法相同的参数。在未绑定引用中，在应用函数对象时通过方法声明参数之前的附加参数指定接收对象。在流管道中，未绑定引用通常用作映射和筛选函数（[Item-45](../Chapter-7/Chapter-7-Item-45-Use-streams-judiciously)）。最后，对于类和数组，有两种构造函数引用。构造函数引用用作工厂对象。五种方法参考文献汇总如下表：

|    Method Ref Type    |       Example       |      Lambda Equivalent     |
|:-------:|:-------:|:-------:|
|   Static  |     `Integer::parseInt`    |   `str ->`   |
|   Bound  |     `Instant.now()::isAfter`    |   `Instant then =Instant.now(); t ->then.isAfter(t)`   |
|   Unbound  |     `String::toLowerCase`    |   `str ->str.toLowerCase()`   |
|   Class Constructor  |     `TreeMap<K,V>::new`    |   `() -> new TreeMap<K,V>`   |
|   Array Constructor  |     `int[]::new`    |   `len -> new int[len]`   |

总之，方法引用通常为 lambda 表达式提供了一种更简洁的选择。**如果方法引用更短、更清晰，则使用它们；如果没有，仍然使用 lambda 表达式。**
