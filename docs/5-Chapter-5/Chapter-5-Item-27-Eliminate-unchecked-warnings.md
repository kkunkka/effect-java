# 第二十七节: 消除 unchecked 警告

当你使用泛型编程时，你将看到许多编译器警告：unchecked 强制转换警告、unchecked 方法调用警告、unchecked 可变参数类型警告和 unchecked 自动转换警告。使用泛型的经验越丰富，遇到的警告就越少，但是不要期望新编写的代码能够完全正确地编译。

许多 unchecked 警告很容易消除。例如，假设你不小心写了这个声明：

```
Set<Lark> exaltation = new HashSet();
```

编译器会精确地提醒你做错了什么：

```
Venery.java:4: warning: [unchecked] unchecked conversion
Set&lt;Lark&gt; exaltation = new HashSet();
^ required: Set&lt;Lark&gt;
found: HashSet
```

你可以在指定位置进行更正，使警告消失。注意，你实际上不必指定类型参数，只需给出由 Java 7 中引入的 diamond 操作符（&lt;&gt;）。然后编译器将推断出正确的实际类型参数（在本例中为 Lark）：

```
Set&lt;Lark/&gt; exaltation = new HashSet&lt;&gt;();
```

一些警告会更难消除。这一章充满这类警告的例子。当你收到需要认真思考的警告时，坚持下去！**力求消除所有 unchecked 警告。** 如果你消除了所有警告，你就可以确信你的代码是类型安全的，这是一件非常好的事情。这意味着你在运行时不会得到 ClassCastException，它增加了你的信心，你的程序将按照预期的方式运行。

**如果不能消除警告，但是可以证明引发警告的代码是类型安全的，那么（并且只有在那时）使用 SuppressWarnings("unchecked") 注解来抑制警告。** 如果你在没有首先证明代码是类型安全的情况下禁止警告，那么你是在给自己一种错误的安全感。代码可以在不发出任何警告的情况下编译，但它仍然可以在运行时抛出 ClassCastException。但是，如果你忽略了你知道是安全的 unchecked 警告（而不是抑制它们），那么当出现一个代表真正问题的新警告时，你将不会注意到。新出现的警告就会淹设在所有的错误警告当中。

SuppressWarnings 注解可以用于任何声明中，从单个局部变量声明到整个类。**总是在尽可能小的范围上使用 SuppressWarnings 注解。** 通常用在一个变量声明或一个非常短的方法或构造函数。不要在整个类中使用 SuppressWarnings。这样做可能会掩盖关键警告。

如果你发现自己在一个超过一行的方法或构造函数上使用 SuppressWarnings 注解，那么你可以将其移动到局部变量声明中。你可能需要声明一个新的局部变量，但这是值得的。例如，考虑这个 toArray 方法，它来自 ArrayList：

```
public &lt;T&gt; T[] toArray(T[] a) {
    if (a.length &lt; size)
        return (T[]) Arrays.copyOf(elements, size, a.getClass());
    System.arraycopy(elements, 0, a, 0, size);
    if (a.length &gt; size)
        a[size] = null;
    return a;
}
```

如果你编译 ArrayList，这个方法会产生这样的警告：

```
ArrayList.java:305: warning: [unchecked] unchecked cast
return (T[]) Arrays.copyOf(elements, size, a.getClass());
^ required: T[]
found: Object[]
```

将 SuppressWarnings 注释放在 return 语句上是非法的，因为它不是声明 [JLS, 9.7]。你可能想把注释放在整个方法上，但是不要这样做。相反，应该声明一个局部变量来保存返回值并添加注解，如下所示：

```
// Adding local variable to reduce scope of @SuppressWarnings
public &lt;T&gt; T[] toArray(T[] a) {
    if (a.length &lt; size) {
        // This cast is correct because the array we're creating
        // is of the same type as the one passed in, which is T[].
        @SuppressWarnings("unchecked") T[] result = (T[]) Arrays.copyOf(elements, size, a.getClass());
        return result;
    }
    System.arraycopy(elements, 0, a, 0, size);
    if (a.length &gt; size)
        a[size] = null;
    return a;
}
```

生成的方法编译正确，并将抑制 unchecked 警告的范围减到最小。

**每次使用 SuppressWarnings("unchecked") 注解时，要添加一条注释，说明这样做是安全的。** 这将帮助他人理解代码，更重要的是，它将降低其他人修改代码而产生不安全事件的几率。如果你觉得写这样的注释很难，那就继续思考合适的方式。你最终可能会发现，unchecked 操作毕竟是不安全的。

总之，unchecked 警告很重要。不要忽视他们。每个 unchecked 警告都代表了在运行时发生 ClassCastException 的可能性。尽最大努力消除这些警告。如果不能消除 unchecked 警告，并且可以证明引发该警告的代码是类型安全的，那么可以在尽可能狭窄的范围内使用 @SuppressWarnings("unchecked") 注释来禁止警告。在注释中记录你决定隐藏警告的理由。