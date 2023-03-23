# 第三十一节: 使用有界通配符增加 API 的灵活性

如 [Item-28](/Chapter-5/Chapter-5-Item-28-Prefer-lists-to-arrays.md) 所示，参数化类型是不可变的。换句话说，对于任意两种不同类型 `Type1` 和 `Type2`，`List<Type1>` 既不是 `List<Type>` 的子类型，也不是它的父类。虽然 `List<String>` 不是 `List<Object>` 的子类型，这和习惯的直觉不符，但它确实有意义。你可以将任何对象放入 `List<Object>`，但只能将字符串放入 `List<String>`。因为 `List<String>` 不能做 `List<Object>` 能做的所有事情，所以它不是子类型（可通过 Liskov 替换原则来理解这一点，[Item-10](/Chapter-3/Chapter-3-Item-10-Obey-the-general-contract-when-overriding-equals.md)）。

**译注：里氏替换原则（Liskov Substitution Principle，LSP）面向对象设计的基本原则之一。里氏替换原则指出：任何父类可以出现的地方，子类一定可以出现。LSP 是继承复用的基石，只有当衍生类可以替换掉父类，软件单位的功能不受到影响时，父类才能真正被复用，而衍生类也能够在父类的基础上增加新的行为。**

有时你需要获得比不可变类型更多的灵活性。考虑 [Item-29](/Chapter-5/Chapter-5-Item-29-Favor-generic-types.md) 中的堆栈类。以下是它的公共 API：

```
public class Stack<E> {
    public Stack();
    public void push(E e);
    public E pop();
    public boolean isEmpty();
}
```

假设我们想添加一个方法，该方法接受一系列元素并将它们全部推入堆栈。这是第一次尝试：

```
// pushAll method without wildcard type - deficient!
public void pushAll(Iterable<E> src) {
    for (E e : src)
        push(e);
}
```

该方法能够正确编译，但并不完全令人满意。如果 `Iterable src` 的元素类型与堆栈的元素类型完全匹配，那么它正常工作。但是假设你有一个 `Stack<Number>`，并且调用 `push(intVal)`，其中 `intVal` 的类型是 `Integer`。这是可行的，因为 `Integer` 是 `Number` 的子类型。因此，从逻辑上讲，这似乎也应该奏效：

```
Stack<Number> numberStack = new Stack<>();
Iterable<Integer> integers = ... ;
numberStack.pushAll(integers);
```

但是，如果你尝试一下，将会得到这个错误消息，因为参数化类型是不可变的：

```
StackTest.java:7: error: incompatible types: Iterable<Integer>
cannot be converted to Iterable<Number>
        numberStack.pushAll(integers);
                    ^
```

幸运的是，有一种解决方法。Java 提供了一种特殊的参数化类型，`有界通配符类型`来处理这种情况。pushAll 的输入参数的类型不应该是「E 的 Iterable 接口」，而应该是「E 的某个子类型的 Iterable 接口」，并且有一个通配符类型，它的确切含义是：`Iterable<? extends E>`（关键字 extends 的使用稍微有些误导：回想一下 [Item-29](/Chapter-5/Chapter-5-Item-29-Favor-generic-types.md)，定义了子类型，以便每个类型都是其本身的子类型，即使它没有扩展自己。）让我们修改 pushAll 来使用这种类型：

```
// Wildcard type for a parameter that serves as an E producer
public void pushAll(Iterable<? extends E> src) {
    for (E e : src)
        push(e);
}
```

更改之后，不仅 Stack 可以正确编译，而且不能用原始 pushAll 声明编译的客户端代码也可以正确编译。因为 Stack 和它的客户端可以正确编译，所以你知道所有东西都是类型安全的。现在假设你想编写一个与 pushAll 一起使用的 popAll 方法。popAll 方法将每个元素从堆栈中弹出，并将这些元素添加到给定的集合中。下面是编写 popAll 方法的第一次尝试：

```
// popAll method without wildcard type - deficient!
public void popAll(Collection<E> dst) {
    while (!isEmpty())
        dst.add(pop());
}
```

同样，如果目标集合的元素类型与堆栈的元素类型完全匹配，那么这种方法可以很好地编译。但这也不是完全令人满意。假设你有一个 `Stack<Number>` 和 Object 类型的变量。如果从堆栈中取出一个元素并将其存储在变量中，那么它将编译并运行，不会出错。所以你不能也这样做吗？

```
Stack<Number> numberStack = new Stack<Number>();
Collection<Object> objects = ... ;
numberStack.popAll(objects);
```

如果你尝试根据前面显示的 popAll 版本编译此客户端代码，你将得到一个与第一个版本的 pushAll 非常相似的错误：`Collection<Object>`不是 `Collection<Number>` 的子类型。同样，通配符类型提供解决方法。popAll 的输入参数的类型不应该是「E 的集合」，而应该是「E 的某个超类型的集合」（其中的超类型定义为 E 本身是一个超类型[JLS, 4.10]）。同样，有一个通配符类型，它的确切含义是：`Collection<? super E>`。让我们修改 popAll 来使用它：

```
// Wildcard type for parameter that serves as an E consumer
public void popAll(Collection<? super E> dst) {
  while (!isEmpty())
    dst.add(pop());
}
```

通过此更改，Stack 类和客户端代码都可以正确编译。

教训是清楚的。为了获得最大的灵活性，应在表示生产者或消费者的输入参数上使用通配符类型。如果输入参数既是生产者又是消费者，那么通配符类型对你没有任何好处：你需要一个精确的类型匹配，这就是在没有通配符的情况下得到的结果。这里有一个助记符帮助你记住使用哪种通配符类型：

**PECS 表示生产者应使用 extends，消费者应使用 super。**

换句话说，如果参数化类型表示 T 生成器，则使用 `<? extends T>`；如果它表示一个 T 消费者，则使用 `<? super T>`。在我们的 Stack 示例中，pushAll 的 src 参数生成 E 的实例供 Stack 使用，因此 src 的适当类型是 `Iterable<? extends E>`；popAll 的 dst 参数使用 Stack 中的 E 实例，因此适合 dst 的类型是 `Collection<? super E>`。PECS 助记符捕获了指导通配符类型使用的基本原则。Naftalin 和 Wadler 称之为 Get and Put 原则[Naftalin07, 2.4]。

记住这个助记符后，再让我们看一看本章前面提及的一些方法和构造函数声明。[Item-28](/Chapter-5/Chapter-5-Item-28-Prefer-lists-to-arrays.md) 中的 Chooser 构造函数有如下声明：

```
public Chooser(Collection<T> choices)
```

这个构造函数只使用集合选项来生成类型 T 的值（并存储它们以供以后使用），因此它的声明应该使用扩展 T 的通配符类型 **extends T**。下面是生成的构造函数声明：

```
// Wildcard type for parameter that serves as an T producer
public Chooser(Collection<? extends T> choices)
```

这种改变在实践中会有什么不同吗？是的，它会。假设你有一个 `List<Integer>`，并且希望将其传递给 `Chooser<Number>` 的构造函数。这不会与原始声明一起编译，但是一旦你将有界通配符类型添加到声明中，它就会编译。

现在让我们看看 [Item-30](/Chapter-5/Chapter-5-Item-30-Favor-generic-methods.md) 中的 union 方法。以下是声明：

```
public static <E> Set<E> union(Set<E> s1, Set<E> s2)
```

参数 s1 和 s2 都是 E 的生产者，因此 PECS 助记符告诉我们声明应该如下：

```
public static <E> Set<E> union(Set<? extends E> s1,Set<? extends E> s2)
```

注意，返回类型仍然设置为 `Set<E>`。**不要使用有界通配符类型作为返回类型。** 它将强制用户在客户端代码中使用通配符类型，而不是为用户提供额外的灵活性。经修订后的声明可正确编译以下代码：

```
Set<Integer> integers = Set.of(1, 3, 5);
Set<Double> doubles = Set.of(2.0, 4.0, 6.0);
Set<Number> numbers = union(integers, doubles);
```

如果使用得当，通配符类型对于类的用户几乎是不可见的。它们让方法接受它们应该接受的参数，拒绝应该拒绝的参数。**如果类的用户必须考虑通配符类型，那么它的 API 可能有问题。**

在 Java 8 之前，类型推断规则还不足以处理前面的代码片段，这要求编译器使用上下文指定的返回类型（或目标类型）来推断 E 的类型。前面显示的 union 调用的目标类型设置为 `Set<Number>` 如果你尝试在 Java 的早期版本中编译该片段（使用 Set.of factory 的适当替代），你将得到一条长而复杂的错误消息，如下所示：

```
Union.java:14: error: incompatible types
Set<Number> numbers = union(integers, doubles);
^ required: Set<Number>
found: Set<INT#1>
where INT#1,INT#2 are intersection types:
INT#1 extends Number,Comparable<? extends INT#2>
INT#2 extends Number,Comparable<?>
```

幸运的是，有一种方法可以处理这种错误。如果编译器没有推断出正确的类型，你总是可以告诉它使用显式类型参数[JLS, 15.12]使用什么类型。即使在 Java 8 中引入目标类型之前，这也不是必须经常做的事情，这很好，因为显式类型参数不是很漂亮。通过添加显式类型参数，如下所示，代码片段可以在 Java 8 之前的版本中正确编译：

```
// Explicit type parameter - required prior to Java 8
Set<Number> numbers = Union.<Number>union(integers, doubles);
```

接下来让我们将注意力转到 [Item-30](/Chapter-5/Chapter-5-Item-30-Favor-generic-methods.md) 中的 max 方法。以下是原始声明：

```
public static <T extends Comparable<T>> T max(List<T> list)
```

下面是使用通配符类型的修正声明：

```
public static <T extends Comparable<? super T>> T max(List<? extends T> list)
```

为了从原始声明中得到修改后的声明，我们两次应用了 PECS 启发式。直接的应用程序是参数列表。它生成 T 的实例，所以我们将类型从 `List<T>` 更改为 `List<? extends T>`。复杂的应用是类型参数 T。这是我们第一次看到通配符应用于类型参数。最初，T 被指定为扩展 `Comparable<T>`，但是 T 的 Comparable 消费 T 实例（并生成指示顺序关系的整数）。因此，将参数化类型 `Comparable<T>` 替换为有界通配符类型 `Comparable<? super T>`，Comparables 始终是消费者，所以一般应**优先使用 `Comparable<? super T>` 而不是 `Comparable<T>`**，比较器也是如此；因此，通常应该**优先使用 `Comparator<? super T>` 而不是 `Comparator<T>`。**

修订后的 max 声明可能是本书中最复杂的方法声明。增加的复杂性真的能给你带来什么好处吗？是的，它再次生效。下面是一个简单的列表案例，它在原来的声明中不允许使用，但经订正的声明允许：

```
List<ScheduledFuture<?>> scheduledFutures = ... ;
```

不能将原始方法声明应用于此列表的原因是 ScheduledFuture 没有实现 `Comparable<ScheduledFuture>`。相反，它是 Delayed 的一个子接口，扩展了 `Comparable<Delayed>`。换句话说，ScheduledFuture 的实例不仅仅可以与其他 ScheduledFuture 实例进行比较；它可以与任何 Delayed 实例相比较，这足以导致初始声明时被拒绝。更通俗来说，通配符用于支持不直接实现 Comparable（或 Comparator）但扩展了实现 Comparable（或 Comparator）的类型的类型。

还有一个与通配符相关的主题值得讨论。类型参数和通配符之间存在对偶性，可以使用其中一种方法声明许多方法。例如，下面是静态方法的两种可能声明，用于交换列表中的两个索引项。第一个使用无界类型参数（[Item-30](/Chapter-5/Chapter-5-Item-30-Favor-generic-methods.md)），第二个使用无界通配符：

```
// Two possible declarations for the swap method
public static <E> void swap(List<E> list, int i, int j);
public static void swap(List<?> list, int i, int j);
```

这两个声明中哪个更好，为什么？在公共 API 中第二个更好，因为它更简单。传入一个列表（任意列表），该方法交换索引元素。不需要担心类型参数。通常，如果类型参数在方法声明中只出现一次，则用通配符替换它。**如果它是一个无界类型参数，用一个无界通配符替换它；** 如果它是有界类型参数，则用有界通配符替换它。

交换的第二个声明有一个问题。这个简单的实现无法编译：

```
public static void swap(List<?> list, int i, int j) {
  list.set(i, list.set(j, list.get(i)));
}
```

试图编译它会产生一个不太有用的错误消息：

```
Swap.java:5: error: incompatible types: Object cannot be
converted to CAP#1
list.set(i, list.set(j, list.get(i)));
^ where CAP#1
is a fresh type-variable: CAP#1 extends Object from capture of ?
```

我们不能把一个元素放回刚刚取出的列表中，这看起来是不正确的。问题是 list 的类型是 `List<?>`，你不能在 `List<?>` 中放入除 null 以外的任何值。幸运的是，有一种方法可以实现，而无需求助于不安全的强制转换或原始类型。其思想是编写一个私有助手方法来捕获通配符类型。为了捕获类型，helper 方法必须是泛型方法。它看起来是这样的：

```
public static void swap(List<?> list, int i, int j) {
  swapHelper(list, i, j);
}
// Private helper method for wildcard capture
private static <E> void swapHelper(List<E> list, int i, int j) {
  list.set(i, list.set(j, list.get(i)));
}
```

swapHelper 方法知道 list 是一个 `List<E>`。因此，它知道它从这个列表中得到的任何值都是 E 类型的，并且将 E 类型的任何值放入这个列表中都是安全的。这个稍微复杂的实现能够正确编译。它允许我们导出基于 通配符的声明，同时在内部利用更复杂的泛型方法。swap 方法的客户端不必面对更复杂的 swapHelper 声明，但它们确实从中受益。值得注意的是，helper 方法具有我们认为对于公共方法过于复杂而忽略的签名。

总之，在 API 中使用通配符类型虽然很棘手，但可以使其更加灵活。如果你编写的库将被广泛使用，则必须考虑通配符类型的正确使用。记住基本规则：生产者使用 extends，消费者使用 super（PECS）。还要记住，所有的 comparable 和 comparator 都是消费者。
