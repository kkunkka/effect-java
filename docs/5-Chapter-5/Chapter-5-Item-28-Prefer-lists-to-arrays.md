# 第二十八节: list 优于数组

数组与泛型有两个重要区别。首先，数组是协变的。这个听起来很吓人的单词的意思很简单，如果 Sub 是 Super 的一个子类型，那么数组类型 Sub[] 就是数组类型 Super[] 的一个子类型。相比之下，泛型是不变的：对于任何两个不同类型 Type1 和 Type2，`List<Type1>` 既不是 `List<Type2>` 的子类型，也不是 `List<Type2>` 的超类型 [JLS, 4.10; Naftalin07, 2.5]。你可能认为这意味着泛型是有缺陷的，但可以说数组才是有缺陷的。这段代码是合法的：

```
// Fails at runtime!
Object[] objectArray = new Long[1];
objectArray[0] = "I don't fit in"; // Throws ArrayStoreException
```

但这一段代码就不是：

```
// Won't compile!
List<Object> ol = new ArrayList<Long>(); // Incompatible types
ol.add("I don't fit in");
```

两种方法都不能将 String 放入 Long 容器，但使用数组，你会得到一个运行时错误；使用 list，你可以在编译时发现问题。当然，你更希望在编译时找到问题。

数组和泛型之间的第二个主要区别：数组是具体化的 [JLS, 4.7]。这意味着数组在运行时知道并强制执行他们的元素类型。如前所述，如果试图将 String 元素放入一个 Long 类型的数组中，就会得到 ArrayStoreException。相比之下，泛型是通过擦除来实现的 [JLS, 4.6]。这意味着它们只在编译时执行类型约束，并在运行时丢弃（或擦除）元素类型信息。擦除允许泛型与不使用泛型的遗留代码自由交互操作（[Item-26](../Chapter-5/Chapter-5-Item-26-Do-not-use-raw-types)），确保在 Java 5 中平稳地过渡。

由于这些基本差异，数组和泛型不能很好地混合。例如，创建泛型、参数化类型或类型参数的数组是非法的。因此，这些数组创建表达式都不是合法的：`new List<E>[]`、`new List<String>[]`、`new E[]`。所有这些都会在编译时导致泛型数组创建错误。

为什么创建泛型数组是非法的？因为这不是类型安全的。如果合法，编译器在其他正确的程序中生成的强制转换在运行时可能会失败，并导致 ClassCastException。这将违反泛型系统提供的基本保证。


为了更具体，请考虑以下代码片段：

```
// Why generic array creation is illegal - won't compile!
List<String>[] stringLists = new List<String>[1]; // (1)
List<Integer> intList = List.of(42); // (2)
Object[] objects = stringLists; // (3)
objects[0] = intList; // (4)
String s = stringLists[0].get(0); // (5)
```

假设创建泛型数组的第 1 行是合法的。第 2 行创建并初始化一个包含单个元素的 `List<Integer>`。第 3 行将 `List<String>` 数组存储到 Object 类型的数组变量中，这是合法的，因为数组是协变的。第 4 行将 `List<Integer>` 存储到 Object 类型的数组的唯一元素中，这是成功的，因为泛型是由擦除实现的：`List<Integer>` 实例的运行时类型是 List，`List<String>`[] 实例的运行时类型是 List[]，因此这个赋值不会生成 ArrayStoreException。现在我们有麻烦了。我们将一个 `List<Integer>` 实例存储到一个数组中，该数组声明只保存 `List<String>` 实例。在第 5 行，我们从这个数组的唯一列表中检索唯一元素。编译器自动将检索到的元素转换为 String 类型，但它是一个 Integer 类型的元素，因此我们在运行时得到一个 ClassCastException。为了防止这种情况发生，第 1 行（创建泛型数组）必须生成编译时错误。

E、`List<E>` 和 `List<string>` 等类型在技术上称为不可具体化类型 [JLS, 4.7]。直观地说，非具体化类型的运行时表示包含的信息少于其编译时表示。由于擦除，唯一可具体化的参数化类型是无限制通配符类型，如 `List<?>` 和 `Map<?,?>`（[Item-26](../Chapter-5/Chapter-5-Item-26-Do-not-use-raw-types)）。创建无边界通配符类型数组是合法的，但不怎么有用。

禁止创建泛型数组可能很烦人。例如，这意味着泛型集合通常不可能返回其元素类型的数组（部分解决方案请参见 [Item-33](../Chapter-5/Chapter-5-Item-33-Consider-typesafe-heterogeneous-containers)）。这也意味着在使用 varargs 方法（[Item-53](../Chapter-8/Chapter-8-Item-53-Use-varargs-judiciously)）与泛型组合时，你会得到令人困惑的警告。这是因为每次调用 varargs 方法时，都会创建一个数组来保存 varargs 参数。如果该数组的元素类型不可具体化，则会得到警告。SafeVarargs 注解可以用来解决这个问题（[Item-32](../Chapter-5/Chapter-5-Item-32-Combine-generics-and-varargs-judiciously)）。

当你在转换为数组类型时遇到泛型数组创建错误或 unchecked 强制转换警告时，通常最好的解决方案是使用集合类型 `List<E>`，而不是数组类型 E[]。你可能会牺牲一些简洁性或性能，但作为交换，你可以获得更好的类型安全性和互操作性。

例如，假设你希望编写一个 Chooser 类，该类的构造函数接受一个集合，而单个方法返回随机选择的集合元素。根据传递给构造函数的集合，可以将选择器用作游戏骰子、魔术 8 球或蒙特卡洛模拟的数据源。下面是一个没有泛型的简单实现：

```
// Chooser - a class badly in need of generics!
public class Chooser {
  private final Object[] choiceArray;

  public Chooser(Collection choices) {
    choiceArray = choices.toArray();
}

  public Object choose() {
    Random rnd = ThreadLocalRandom.current();
    return choiceArray[rnd.nextInt(choiceArray.length)];
  }
}
```

要使用这个类，每次使用方法调用时，必须将 choose 方法的返回值从对象转换为所需的类型，如果类型错误，转换将在运行时失败。我们认真考虑了 [Item-29](../Chapter-5/Chapter-5-Item-29-Favor-generic-types) 的建议，试图对 Chooser 进行修改，使其具有通用性。变化以粗体显示：

```
// A first cut at making Chooser generic - won't compile
public class Chooser<T> {
  private final T[] choiceArray;

  public Chooser(Collection<T> choices) {
    choiceArray = choices.toArray();
  }

  // choose method unchanged
}
```

如果你尝试编译这个类，你将得到这样的错误消息：

```
Chooser.java:9: error: incompatible types: Object[] cannot be converted to T[]
choiceArray = choices.toArray();
^ where T is a type-variable:
T extends Object declared in class Chooser
```

没什么大不了的，你会说，我把对象数组转换成 T 数组：

```
choiceArray = (T[]) choices.toArray();
```

这样就消除了错误，但你得到一个警告：

```
Chooser.java:9: warning: [unchecked] unchecked cast choiceArray = (T[]) choices.toArray();
^ required: T[], found: Object[]
where T is a type-variable:
T extends Object declared in class Chooser
```

编译器告诉你，它不能保证在运行时转换的安全性，因为程序不知道类型 T 代表什么。记住，元素类型信息在运行时从泛型中删除。这个计划会奏效吗？是的，但是编译器不能证明它。你可以向自己证明这一点，但是你最好将证据放在注释中，指出消除警告的原因（[Item-27](../Chapter-5/Chapter-5-Item-27-Eliminate-unchecked-warnings)），并使用注解隐藏警告。

若要消除 unchecked 强制转换警告，请使用 list 而不是数组。下面是编译时没有错误或警告的 Chooser 类的一个版本：

```
// List-based Chooser - typesafe
public class Chooser<T> {
    private final List<T> choiceList;

    public Chooser(Collection<T> choices) {
        choiceList = new ArrayList<>(choices);
    }

    public T choose() {
        Random rnd = ThreadLocalRandom.current();
        return choiceList.get(rnd.nextInt(choiceList.size()));
    }
}
```

这个版本稍微有点冗长，可能稍微慢一些，但是为了让你安心，在运行时不会得到 ClassCastException 是值得的。

总之，数组和泛型有非常不同的类型规则。数组是协变的、具体化的；泛型是不变的和可被擦除的。因此，数组提供了运行时类型安全，而不是编译时类型安全，对于泛型反之亦然。一般来说，数组和泛型不能很好地混合。如果你发现将它们混合在一起并得到编译时错误或警告，那么你的第一个反应该是将数组替换为 list。
