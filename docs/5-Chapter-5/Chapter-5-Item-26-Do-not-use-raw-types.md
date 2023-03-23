# 第二十六节: 不要使用原始类型

首先，介绍一些术语。声明中具有一个或多个类型参数的类或接口就是泛型类或泛型接口 [JLS, 8.1.2, 9.1.2]。例如，List 接口有一个类型参数 E，用于表示其元素类型。该接口的全名是 `List<E>`（读作「List of E」），但人们通常简称为 List。泛型类和泛型接口统称为泛型。

每个泛型定义了一组参数化类型，这些参数化类型包括类名或接口名，以及带尖括号的参数列表，参数列表是与泛型的形式类型参数相对应的实际类型 [JLS, 4.4, 4.5]。例如，`List<String>`（读作「List of String」）是一个参数化类型，表示元素类型为 String 类型的 List。（String 是与形式类型参数 E 对应的实际类型参数。）

最后，每个泛型都定义了一个原始类型，它是没有任何相关类型参数的泛型的名称 [JLS, 4.8]。例如，`List<E>` 对应的原始类型是 List。原始类型的行为就好像所有泛型信息都从类型声明中删除了一样。它们的存在主要是为了与之前的泛型代码兼容。

在将泛型添加到 Java 之前，这是一个典型的集合声明。就 Java 9 而言，它仍然是合法的，但不应效仿：

```
// Raw collection type - don't do this!
// My stamp collection. Contains only Stamp instances.
private final Collection stamps = ... ;
```

如果你今天使用这个声明，然后意外地将 coin 放入 stamp 集合中，这一错误的插入依然能够编译并没有错误地运行（尽管编译器确实发出了模糊的警告）：

```
// Erroneous insertion of coin into stamp collection
stamps.add(new Coin( ... )); // Emits "unchecked call" warning
```

直到从 stamp 集合中获取 coin 时才会收到错误提示：

```
// Raw iterator type - don't do this!
for (Iterator i = stamps.iterator(); i.hasNext(); )
    Stamp stamp = (Stamp) i.next(); // Throws ClassCastException
stamp.cancel();
```

正如在本书中提到的，在出现错误之后尽快发现错误是有价值的，最好是在编译时。在本例这种情况下，直到运行时（在错误发生很久之后）才发现错误，而且报错代码可能与包含错误的代码相距很远。一旦看到 ClassCastException，就必须在代码中搜索将 coin 放进 stamp 集合的方法调用。编译器不能帮助你，因为它不能理解注释「Contains only Stamp instances.」

对于泛型，类型声明应该包含类型信息，而不是注释：

```
// Parameterized collection type - typesafe
private final Collection<Stamp> stamps = ... ;
```

从这个声明看出，编译器应该知道 stamps 应该只包含 Stamp 实例，为保证它确实如此，假设你的整个代码库编译没有发出（或抑制；详见 [Item-27](/Chapter-5/Chapter-5-Item-27-Eliminate-unchecked-warnings.md)）任何警告。当 stamps 利用一个参数化的类型进行声明时，错误的插入将生成编译时错误消息，该消息将确切地告诉你哪里出了问题：

```
Test.java:9: error: incompatible types: Coin cannot be converted
to Stamp
c.add(new Coin());
^
```

当从集合中检索元素时，编译器会为你执行不可见的强制类型转换，并确保它们不会失败（再次假设你的所有代码没有产生或抑制任何编译器警告）。虽然不小心将 coin 插入 stamps 集合看起来有些牵强，但这类问题是真实存在的。例如，很容易想象将一个 BigInteger 放入一个只包含 BigDecimal 实例的集合中。

如前所述，使用原始类型（没有类型参数的泛型）是合法的，但是你永远不应该这样做。**如果使用原始类型，就会失去泛型的安全性和表现力。** 既然你不应该使用它们，那么为什么语言设计者一开始就允许原始类型呢？答案是：为了兼容性。Java 即将进入第二个十年，泛型被添加进来时，还存在大量不使用泛型的代码。保持所有这些代码合法并与使用泛型的新代码兼容被认为是关键的。将参数化类型的实例传递给设计用于原始类型的方法必须是合法的，反之亦然。这被称为迁移兼容性的需求，它促使原始类型得到支持并使用擦除实现泛型 （[Item-28](/Chapter-5/Chapter-5-Item-28-Prefer-lists-to-arrays.md)）。

虽然你不应该使用原始类型（如 List），但是可以使用参数化的类型来允许插入任意对象，如 `List<Object>`。原始类型 List 和参数化类型 `List<Object>` 之间的区别是什么？粗略地说，前者选择了不使用泛型系统，而后者明确地告诉编译器它能够保存任何类型的对象。虽然可以将 `List<String>` 传递给 List 类型的参数，但不能将其传递给类型 `List<Object>` 的参数。泛型有子类型规则，`List<String>` 是原始类型 List 的子类型，而不是参数化类型 `List<Object>` 的子类型（[Item-28](/Chapter-5/Chapter-5-Item-28-Prefer-lists-to-arrays.md)）。因此，**如果使用原始类型（如 List），就会失去类型安全性，但如果使用参数化类型（如 `List<Object>`）则不会。**

为了使这一点具体些，考虑下面的程序：

```
// Fails at runtime - unsafeAdd method uses a raw type (List)!

public static void main(String[] args) {
    List<String> strings = new ArrayList<>();
    unsafeAdd(strings, Integer.valueOf(42));
    String s = strings.get(0); // Has compiler-generated cast
}

private static void unsafeAdd(List list, Object o) {
    list.add(o);
}
```

该程序可以编译，但因为它使用原始类型 List，所以你会得到一个警告：

```
Test.java:10: warning: [unchecked] unchecked call to add(E) as a
member of the raw type List
list.add(o);
^
```
实际上，如果你运行程序，当程序试图将调用 `strings.get(0)` 的结果强制转换为字符串时，你会得到一个 ClassCastException。这是一个由编译器生成的强制类型转换，它通常都能成功，但在本例中，我们忽略了编译器的警告，并为此付出了代价。

如果将 unsafeAdd 声明中的原始类型 List 替换为参数化类型 `List<Object>`，并尝试重新编译程序，你会发现它不再编译，而是发出错误消息：

```
Test.java:5: error: incompatible types: List<String> cannot be
converted to List<Object>
unsafeAdd(strings, Integer.valueOf(42));
^
```

对于元素类型未知且无关紧要的集合，你可能会尝试使用原始类型。例如，假设你希望编写一个方法，该方法接受两个集合并返回它们共有的元素数量。如果你是使用泛型的新手，那么你可以这样编写一个方法：

```
// Use of raw type for unknown element type - don't do this!
static int numElementsInCommon(Set s1, Set s2) {
    int result = 0;
    for (Object o1 : s1)
        if (s2.contains(o1))
    result++;
    return result;
}
```

这种方法是可行的，但是它使用的是原始类型，这是很危险的。安全的替代方法是使用无界通配符类型。如果你想使用泛型，但不知道或不关心实际的类型参数是什么，那么可以使用问号代替。例如，泛型集 `Set<E>` 的无界通配符类型是 `Set<?>`（读作「set of some type」）。它是最通用的参数化集合类型，能够容纳任何集合：

```
// Uses unbounded wildcard type - typesafe and flexible
static int numElementsInCommon(Set<?> s1, Set<?> s2) { ... }
```

无界通配符类型 `Set<?>` 和原始类型 Set 之间的区别是什么？问号真的能起作用吗？我并不是在强调这一点，但是通配符类型是安全的，而原始类型则不是。将任何元素放入具有原始类型的集合中，很容易破坏集合的类型一致性（如上述的 unsafeAdd 方法所示）；你不能将任何元素（除了 null）放入 `Collection<?>`。尝试这样做将生成这样的编译时错误消息：

```
WildCard.java:13: error: incompatible types: String cannot be converted to CAP#1
c.add("verboten");
^ where CAP#1
is a fresh type-variable:
CAP#1 extends Object from capture of ?
```

无可否认，这个错误消息让人不满意，但是编译器已经完成了它的工作，防止你无视它的元素类型而破坏集合的类型一致性。你不仅不能将任何元素（除 null 之外）放入 `Collection<?>`，而且不能臆想你得到的对象的类型。如果这些限制是不可接受的，你可以使用泛型方法（[Item-30](/Chapter-5/Chapter-5-Item-30-Favor-generic-methods.md)）或有界通配符类型（[Item-31](/Chapter-5/Chapter-5-Item-31-Use-bounded-wildcards-to-increase-API-flexibility.md)）。

对于不应该使用原始类型的规则，有一些小的例外。**必须在类字面量中使用原始类型。** 该规范不允许使用参数化类型（尽管它允许数组类型和基本类型）[JLS, 15.8.2]。换句话说，`List.class`，`String[].class` 和 `int.class` 都是合法的，但是 `List<String>.class` 和 `List<?>.class` 不是。

规则的第二个例外是 instanceof 运算符。由于泛型信息在运行时被删除，因此在不是无界通配符类型之外的参数化类型上使用 instanceof 操作符是非法的。使用无界通配符类型代替原始类型不会以任何方式影响 instanceof 运算符的行为。在这种情况下，尖括号和问号只是多余的。**下面的例子是使用通用类型 instanceof 运算符的首选方法：**

```
// Legitimate use of raw type - instanceof operator
if (o instanceof Set) { // Raw type
    Set<?> s = (Set<?>) o; // Wildcard type
    ...
}
```

注意，一旦确定 o 是一个 Set，就必须将其强制转换为通配符类型 `Set<?>`，而不是原始类型 Set。这是一个经过检查的强制类型转换，所以不会引发编译器警告。

总之，使用原始类型可能会在运行时导致异常，所以不要轻易使用它们。它们仅用于与引入泛型之前的遗留代码进行兼容和互操作。快速回顾一下，`Set<Object>` 是一个参数化类型，表示可以包含任何类型的对象的集合，`Set<?>` 是一个通配符类型，表示只能包含某种未知类型的对象的集合，Set 是一个原始类型，它选择了泛型系统。前两个是安全的，后一个就不安全了。


为便于参考，本条目中介绍的术语（以及后面将要介绍的一些术语）总结如下：

|    Term    |       Example       |      Item     |
|:-------:|:-------:|:-------:|
|   Parameterized type  |     `List<String>`    |   [Item-26](/Chapter-5/Chapter-5-Item-26-Do-not-use-raw-types.md)   |
|   Actual type parameter  |     `String`    |   [Item-26](/Chapter-5/Chapter-5-Item-26-Do-not-use-raw-types.md)   |
|   Generic type  |     `List<E>`    |   [Item-26](/Chapter-5/Chapter-5-Item-26-Do-not-use-raw-types.md), [Item-29](/Chapter-5/Chapter-5-Item-29-Favor-generic-types.md)   |
|   Formal type parameter  |     `E`    |   [Item-26](/Chapter-5/Chapter-5-Item-26-Do-not-use-raw-types.md)   |
|   Unbounded wildcard type  |     `List<?>`    |   [Item-26](/Chapter-5/Chapter-5-Item-26-Do-not-use-raw-types.md)   |
|   Raw type  |     `List`    |   [Item-26](/Chapter-5/Chapter-5-Item-26-Do-not-use-raw-types.md)   |
|   Bounded type parameter  |     `<E extends Number>`    |   [Item-29](/Chapter-5/Chapter-5-Item-29-Favor-generic-types.md)   |
|   Recursive type bound  |     `<T extends Comparable<T>>`    |   [Item-30](/Chapter-5/Chapter-5-Item-30-Favor-generic-methods.md)   |
|   Bounded wildcard type  |     `List<? extends Number>`    |   [Item-31](/Chapter-5/Chapter-5-Item-31-Use-bounded-wildcards-to-increase-API-flexibility.md)   |
|   Generic method  |     `static <E> List<E> asList(E[] a)`   |   [Item-30](/Chapter-5/Chapter-5-Item-30-Favor-generic-methods.md)   |
|   Type token  |     `String.class`    |   [Item-33](/Chapter-5/Chapter-5-Item-33-Consider-typesafe-heterogeneous-containers.md)   |
