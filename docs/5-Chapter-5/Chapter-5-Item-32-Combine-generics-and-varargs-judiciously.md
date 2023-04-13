# 第三十二节: 明智地合用泛型和可变参数

可变参数方法（[Item-53](../Chapter-8/Chapter-8-Item-53-Use-varargs-judiciously)）和泛型都是在 Java 5 中添加的，因此你可能认为它们能够优雅地交互；可悲的是，他们并不能。可变参数的目的是允许客户端向方法传递可变数量的参数，但这是一个抽象泄漏：当你调用可变参数方法时，将创建一个数组来保存参数；该数组本应是实现细节，却是可见的。因此，当可变参数具有泛型或参数化类型时，会出现令人困惑的编译器警告。

**译注：有关「抽象泄漏」（Leaky Abstractions）的概念可参考 [奇舞精选 2021-07-06 的文章](https://mp.weixin.qq.com/s/KneomYX_7yQ78RzAmvIoHg)**

回想一下 [Item-28](../Chapter-5/Chapter-5-Item-28-Prefer-lists-to-arrays)，非具体化类型是指其运行时表示的信息少于其编译时表示的信息，并且几乎所有泛型和参数化类型都是不可具体化的。如果方法声明其可变参数为不可具体化类型，编译器将在声明上生成警告。如果方法是在其推断类型不可具体化的可变参数上调用的，编译器也会在调用时生成警告。生成的警告就像这样：

```
warning: [unchecked] Possible heap pollution from parameterized vararg type List<String>
```

当参数化类型的变量引用不属于该类型的对象时，就会发生堆污染[JLS, 4.12.2]。它会导致编译器自动生成的强制类型转换失败，违反泛型类型系统的基本保证。

例如，考虑这个方法，它摘自 127 页（[Item-26](../Chapter-5/Chapter-5-Item-26-Do-not-use-raw-types)）的代码片段，但做了些修改：

```
// Mixing generics and varargs can violate type safety!
// 泛型和可变参数混合使用可能违反类型安全原则！
static void dangerous(List<String>... stringLists) {
    List<Integer> intList = List.of(42);
    Object[] objects = stringLists;
    objects[0] = intList; // Heap pollution
    String s = stringLists[0].get(0); // ClassCastException
}
```

此方法没有显式的强制类型转换，但在使用一个或多个参数调用时抛出 ClassCastException。它的最后一行有一个由编译器生成的隐式强制转换。此转换失败，表明类型安全性受到了影响，并且**在泛型可变参数数组中存储值是不安全的。**

这个例子提出了一个有趣的问题：为什么使用泛型可变参数声明方法是合法的，而显式创建泛型数组是非法的？换句话说，为什么前面显示的方法只生成警告，而 127 页上的代码片段发生错误？答案是，带有泛型或参数化类型的可变参数的方法在实际开发中非常有用，因此语言设计人员选择忍受这种不一致性。事实上，Java 库导出了几个这样的方法，包括 `Arrays.asList(T... a)`、`Collections.addAll(Collection<? super T> c, T... elements)` 以及 `EnumSet.of(E first, E... rest)`。它们与前面显示的危险方法不同，这些库方法是类型安全的。

在 Java 7 之前，使用泛型可变参数的方法的作者对调用点上产生的警告无能为力。使得这些 API 难以使用。用户必须忍受这些警告，或者在每个调用点（[Item-27](../Chapter-5/Chapter-5-Item-27-Eliminate-unchecked-warnings)）使用 @SuppressWarnings("unchecked") 注释消除这些警告。这种做法乏善可陈，既损害了可读性，也忽略了标记实际问题的警告。

在 Java 7 中添加了 SafeVarargs 注释，以允许使用泛型可变参数的方法的作者自动抑制客户端警告。本质上，**SafeVarargs 注释构成了方法作者的一个承诺，即该方法是类型安全的。** 作为这个承诺的交换条件，编译器同意不对调用可能不安全的方法的用户发出警告。

关键问题是，使用 @SafeVarargs 注释方法，该方法实际上应该是安全的。那么怎样才能确保这一点呢？回想一下，在调用该方法时创建了一个泛型数组来保存可变参数。如果方法没有将任何内容存储到数组中（这会覆盖参数），并且不允许对数组的引用进行转义（这会使不受信任的代码能够访问数组），那么它就是安全的。换句话说，如果可变参数数组仅用于将可变数量的参数从调用方传输到方法（毕竟这是可变参数的目的），那么该方法是安全的。

值得注意的是，在可变参数数组中不存储任何东西就可能违反类型安全性。考虑下面的通用可变参数方法，它返回一个包含参数的数组。乍一看，它似乎是一个方便的小实用程序：

```
// UNSAFE - Exposes a reference to its generic parameter array!
static <T> T[] toArray(T... args) {
  return args;
}
```

这个方法只是返回它的可变参数数组。这种方法看起来并不危险，但确实危险！这个数组的类型由传递给方法的参数的编译时类型决定，编译器可能没有足够的信息来做出准确的决定。因为这个方法返回它的可变参数数组，所以它可以将堆污染传播到调用堆栈上。

为了使其具体化，请考虑下面的泛型方法，该方法接受三个类型为 T 的参数，并返回一个包含随机选择的两个参数的数组：

```
static <T> T[] pickTwo(T a, T b, T c) {
  switch(ThreadLocalRandom.current().nextInt(3)) {
    case 0: return toArray(a, b);
    case 1: return toArray(a, c);
    case 2: return toArray(b, c);
  }
  throw new AssertionError(); // Can't get here
}
```

这个方法本身并不危险，并且不会生成警告，除非它调用 toArray 方法，该方法有一个通用的可变参数。

编译此方法时，编译器生成代码来创建一个可变参数数组，在该数组中向 toArray 传递两个 T 实例。这段代码分配了 type Object[] 的一个数组，这是保证保存这些实例的最特定的类型，无论调用站点上传递给 pickTwo 的是什么类型的对象。toArray 方法只是将这个数组返回给 pickTwo，而 pickTwo 又将这个数组返回给它的调用者，所以 pickTwo 总是返回一个 Object[] 类型的数组。

现在考虑这个主要方法，练习 pickTwo：

```
public static void main(String[] args) {
  String[] attributes = pickTwo("Good", "Fast", "Cheap");
}
```

这个方法没有任何错误，因此它在编译时不会生成任何警告。但是当你运行它时，它会抛出 ClassCastException，尽管它不包含可见的强制类型转换。你没有看到的是，编译器在 pickTwo 返回的值上生成了一个隐藏的 String[] 转换，这样它就可以存储在属性中。转换失败，因为 Object[] 不是 String[] 的子类型。这个失败非常令人不安，因为它是从方法中删除了两个导致堆污染的级别（toArray），并且可变参数数组在实际参数存储在其中之后不会被修改。

这个示例的目的是让人明白，**让另一个方法访问泛型可变参数数组是不安全的**，只有两个例外：将数组传递给另一个使用 @SafeVarargs 正确注释的可变参数方法是安全的，将数组传递给仅计算数组内容的某个函数的非可变方法也是安全的。

下面是一个安全使用泛型可变参数的典型示例。该方法接受任意数量的列表作为参数，并返回一个包含所有输入列表的元素的序列列表。因为该方法是用 @SafeVarargs 注释的，所以它不会在声明或调用点上生成任何警告：

```
// Safe method with a generic varargs parameter
@SafeVarargs
static <T> List<T> flatten(List<? extends T>... lists) {
  List<T> result = new ArrayList<>();
  for (List<? extends T> list : lists)
    result.addAll(list);
  return result;
}
```

决定何时使用 SafeVarargs 注释的规则很简单：**在每个带有泛型或参数化类型的可变参数的方法上使用 @SafeVarargs**，这样它的用户就不会被不必要的和令人困惑的编译器警告所困扰。这意味着你永远不应该编写像 dangerous 或 toArray 这样不安全的可变参数方法。每当编译器警告你控制的方法中的泛型可变参数可能造成堆污染时，请检查该方法是否安全。提醒一下，一个通用的可变参数方法是安全的，如果：

1. 它没有在可变参数数组中存储任何东西，并且
2. 它不会让数组（或者其副本）出现在不可信的代码中。如果违反了这些禁令中的任何一条，就纠正它。

请注意，SafeVarargs 注释仅对不能覆盖的方法合法，因为不可能保证所有可能覆盖的方法都是安全的。在 Java 8 中，注释仅对静态方法和最终实例方法合法；在 Java 9 中，它在私有实例方法上也成为合法的。

使用 SafeVarargs 注释的另一种选择是接受 [Item-28](../Chapter-5/Chapter-5-Item-28-Prefer-lists-to-arrays) 的建议，并用 List 参数替换可变参数（它是一个伪装的数组）。下面是将这种方法应用到我们的 flatten 方法时的效果。注意，只有参数声明发生了更改：

```
// List as a typesafe alternative to a generic varargs parameter
static <T> List<T> flatten(List<List<? extends T>> lists) {
  List<T> result = new ArrayList<>();
  for (List<? extends T> list : lists)
    result.addAll(list);
  return result;
}
```

然后可以将此方法与静态工厂方法 List.of 一起使用，以允许可变数量的参数。注意，这种方法依赖于 List.of 声明是用 @SafeVarargs 注释的：

```
audience = flatten(List.of(friends, romans, countrymen));
```

这种方法的优点是编译器可以证明该方法是类型安全的。你不必使用 SafeVarargs 注释来保证它的安全性，也不必担心在确定它的安全性时可能出错。主要的缺点是客户端代码比较冗长，可能会比较慢。

这种技巧也可用于无法编写安全的可变参数方法的情况，如第 147 页中的 toArray 方法。它的列表类似于 List.of 方法，我们甚至不用写；Java 库的作者为我们做了这些工作。pickTwo 方法变成这样：

```
static <T> List<T> pickTwo(T a, T b, T c) {
  switch(rnd.nextInt(3)) {
    case 0: return List.of(a, b);
    case 1: return List.of(a, c);
    case 2: return List.of(b, c);
  }
  throw new AssertionError();
}
```

main 方法是这样的：

```
public static void main(String[] args) {
  List<String> attributes = pickTwo("Good", "Fast", "Cheap");
}
```

生成的代码是类型安全的，因为它只使用泛型，而不使用数组。

总之，可变参数方法和泛型不能很好地交互，因为可变参数工具是构建在数组之上的漏洞抽象，并且数组具有与泛型不同的类型规则。虽然泛型可变参数不是类型安全的，但它们是合法的。如果选择使用泛型（或参数化）可变参数编写方法，首先要确保该方法是类型安全的，然后使用 @SafeVarargs 对其进行注释，这样使用起来就不会令人不愉快。
