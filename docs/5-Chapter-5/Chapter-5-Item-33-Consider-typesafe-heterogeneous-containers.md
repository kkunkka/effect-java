# 第三十三节: 考虑类型安全的异构容器

集合是泛型的常见应用之一，如 `Set<E>` 和 `Map<K,V>`，以及单元素容器，如 `ThreadLocal<T>` 和 `AtomicReference<T>`。在所有这些应用中，都是参数化的容器。这将每个容器的类型参数限制为固定数量。通常这正是你想要的。Set 只有一个类型参数，表示其元素类型；Map 有两个，表示`键`和`值`的类型；如此等等。

然而，有时你需要更大的灵活性。例如，一个数据库行可以有任意多列，能够以类型安全的方式访问所有这些列是很好的。幸运的是，有一种简单的方法可以达到这种效果。其思想是参数化`键`而不是容器。然后向容器提供参数化`键`以插入或检索`值`。泛型类型系统用于确保`值`的类型与`键`一致。

作为这种方法的一个简单示例，考虑一个 Favorites 类，它允许客户端存储和检索任意多种类型的 Favorites 实例。Class 类的对象将扮演参数化`键`的角色。这样做的原因是 Class 类是泛型的。Class 对象的类型不仅仅是 Class，而是 `Class<T>`。例如，String.class 的类型为 `Class<String>`、Integer.class 的类型为 `Class<Integer>`。在传递编译时和运行时类型信息的方法之间传递类 Class 对象时，它被称为类型标记[Bracha04]。

Favorites 类的 API 很简单。它看起来就像一个简单的 Map，只不过`键`是参数化的，而不是整个 Map。客户端在设置和获取 Favorites 实例时显示一个 Class 对象。以下是 API:

```
// Typesafe heterogeneous container pattern - API
public class Favorites {
    public <T> void putFavorite(Class<T> type, T instance);
    public <T> T getFavorite(Class<T> type);
}
```

下面是一个示例程序，它演示了 Favorites 类、存储、检索和打印 Favorites 字符串、整数和 Class 实例：

```
// Typesafe heterogeneous container pattern - client
public static void main(String[] args) {
    Favorites f = new Favorites();
    f.putFavorite(String.class, "Java");
    f.putFavorite(Integer.class, 0xcafebabe);
    f.putFavorite(Class.class, Favorites.class);
    String favoriteString = f.getFavorite(String.class);
    int favoriteInteger = f.getFavorite(Integer.class);
    Class<?> favoriteClass = f.getFavorite(Class.class);
    System.out.printf("%s %x %s%n", favoriteString,favoriteInteger, favoriteClass.getName());
}
```

如你所料，这个程序打印 Java cafebabe Favorites。顺便提醒一下，Java 的 printf 方法与 C 的不同之处在于，你应该在 C 中使用 \n 的地方改用 %n。

**译注：`favoriteClass.getName()` 的打印结果与 Favorites 类所在包名有关，结果应为：包名.Favorites**

Favorites 的实例是类型安全的：当你向它请求一个 String 类型时，它永远不会返回一个 Integer 类型。它也是异构的：与普通 Map 不同，所有`键`都是不同类型的。因此，我们将 Favorites 称为一个类型安全异构容器。

Favorites 的实现非常简短。下面是全部内容：

```
// Typesafe heterogeneous container pattern - implementation
public class Favorites {
  private Map<Class<?>, Object> favorites = new HashMap<>();

  public <T> void putFavorite(Class<T> type, T instance) {
    favorites.put(Objects.requireNonNull(type), instance);
  }

  public <T> T getFavorite(Class<T> type) {
    return type.cast(favorites.get(type));
  }
}
```

这里发生了一些微妙的事情。每个 Favorites 实例都由一个名为 favorites 的私有 `Map<Class<?>, Object>` 支持。你可能认为由于通配符类型是无界的，所以无法将任何内容放入此映射中，但事实恰恰相反。需要注意的是，通配符类型是嵌套的：通配符类型不是 Map 的类型，而是`键`的类型。这意味着每个`键`都可以有不同的参数化类型：一个可以是 `Class<String>`，下一个是 `Class<Integer>`，等等。这就是异构的原理。

接下来要注意的是 favorites 的`值`类型仅仅是 Object。换句话说，Map 不保证`键`和`值`之间的类型关系，即每个`值`都是其`键`所表示的类型。实际上，Java 的类型系统还没有强大到足以表达这一点。但是我们知道这是事实，当需要检索一个 favorite 时，我们会利用它。

putFavorite 的实现很简单：它只是将从给定 Class 对象到给定 Favorites 实例的放入 favorites 中。如前所述，这将丢弃`键`和`值`之间的「类型关联」；将无法确定`值`是`键`的实例。但这没关系，因为 getFavorites 方法可以重新建立这个关联。

getFavorite 的实现比 putFavorite 的实现更复杂。首先，它从 favorites 中获取与给定 Class 对象对应的`值`。这是正确的对象引用返回，但它有错误的编译时类型：它是 Object（favorites 的`值`类型），我们需要返回一个 T。因此，getFavorite 的实现通过使用 Class 的 cast 方法，将对象引用类型动态转化为所代表的 Class 对象。

cast 方法是 Java 的 cast 运算符的动态模拟。它只是检查它的参数是否是类对象表示的类型的实例。如果是，则返回参数；否则它将抛出 ClassCastException。我们知道 getFavorite 中的强制转换调用不会抛出 ClassCastException，假设客户端代码已正确地编译。也就是说，我们知道 favorites 中的`值`总是与其`键`的类型匹配。

如果 cast 方法只是返回它的参数，那么它会为我们做什么呢？cast 方法的签名充分利用了 Class 类是泛型的这一事实。其返回类型为 Class 对象的类型参数：

```
public class Class<T> {
    T cast(Object obj);
}
```

这正是 getFavorite 方法所需要的。它使我们能够使 Favorites 类型安全，而不需要对 T 进行 unchecked 的转换。

Favorites 类有两个`值`得注意的限制。首先，恶意客户端很容易通过使用原始形式的类对象破坏 Favorites 实例的类型安全。但是生成的客户端代码在编译时将生成一个 unchecked 警告。这与普通的集合实现（如 HashSet 和 HashMap）没有什么不同。通过使用原始类型 HashSet（[Item-26](../Chapter-5/Chapter-5-Item-26-Do-not-use-raw-types)），可以轻松地将 String 类型放入 `HashSet<Integer>` 中。也就是说，如果你愿意付出代价的话，你可以拥有运行时类型安全。确保 Favorites 不会违反其类型不变量的方法是让 putFavorite 方法检查实例是否是 type 表示的类型的实例，我们已经知道如何做到这一点。只需使用动态转换：

```
// Achieving runtime type safety with a dynamic cast
public <T> void putFavorite(Class<T> type, T instance) {
    favorites.put(type, type.cast(instance));
}
```

java.util.Collections 中的集合包装器也具有相同的功能。它们被称为 checkedSet、checkedList、checkedMap，等等。除了集合（或 Map）外，它们的静态工厂还接受一个（或两个）Class 对象。静态工厂是通用方法，确保 Class 对象和集合的编译时类型匹配。包装器将具体化添加到它们包装的集合中。例如，如果有人试图将 Coin 放入 `Collection<Stamp>` 中，包装器将在运行时抛出 ClassCastException。在混合了泛型类型和原始类型的应用程序中，这些包装器对跟踪将类型错误的元素添加到集合中的客户端代码非常有用。

Favorites 类的第二个限制是它不能用于不可具体化的类型（[Item-28](../Chapter-5/Chapter-5-Item-28-Prefer-lists-to-arrays)）。换句话说，你可以存储的 Favorites 实例类型为 String 类型或 String[]，但不能存储 `List<String>`。原因是你不能为 `List<String>` 获取 Class 对象，`List<String>.class` 是一个语法错误，这也是一件好事。`List<String>` 和 `List<Integer>` 共享一个 Class 对象，即 List.class。如果「字面类型」`List<String>.class` 和 `List<Integer>.class` 是合法的，并且返回相同的对象引用，那么它将严重破坏 Favorites 对象的内部结构。对于这个限制，没有完全令人满意的解决方案。

Favorites 使用的类型标记是无界的：getFavorite 和 put-Favorite 接受任何 Class 对象。有时你可能需要限制可以传递给方法的类型。这可以通过有界类型标记来实现，它只是一个类型标记，使用有界类型参数（[Item-30](../Chapter-5/Chapter-5-Item-30-Favor-generic-methods)）或有界通配符（[Item-31](../Chapter-5/Chapter-5-Item-31-Use-bounded-wildcards-to-increase-API-flexibility)）对可以表示的类型进行绑定。

annotation API（[Item-39](../Chapter-6/Chapter-6-Item-39-Prefer-annotations-to-naming-patterns)）广泛使用了有界类型标记。例如，下面是在运行时读取注释的方法。这个方法来自 AnnotatedElement 接口，它是由表示类、方法、字段和其他程序元素的反射类型实现的：

```
public <T extends Annotation>
    T getAnnotation(Class<T> annotationType);
```

参数 annotationType 是表示注释类型的有界类型标记。该方法返回该类型的元素注释（如果有的话），或者返回 null（如果没有的话）。本质上，带注释的元素是一个类型安全的异构容器，其`键`是注释类型。

假设你有一个 `Class<?>` 类型的对象，并且希望将其传递给一个需要有界类型令牌（例如 getAnnotation）的方法。你可以将对象强制转换为 `Class<? extends Annotation>`，但是这个强制转换是未选中的，因此它将生成一个编译时警告（[Item-27](../Chapter-5/Chapter-5-Item-27-Eliminate-unchecked-warnings)）。幸运的是，class 类提供了一个实例方法，可以安全地（动态地）执行这种类型的强制转换。该方法称为 asSubclass，它将类对象强制转换为它所调用的类对象，以表示由其参数表示的类的子类。如果转换成功，则该方法返回其参数；如果失败，则抛出 ClassCastException。

下面是如何使用 asSubclass 方法读取在编译时类型未知的注释。这个方法编译没有错误或警告：

```
// Use of asSubclass to safely cast to a bounded type token
static Annotation getAnnotation(AnnotatedElement element,String annotationTypeName) {
    Class<?> annotationType = null; // Unbounded type token
    try {
        annotationType = Class.forName(annotationTypeName);
    } catch (Exception ex) {
        throw new IllegalArgumentException(ex);
    }
    return element.getAnnotation(annotationType.asSubclass(Annotation.class));
}
```

总之，以集合的 API 为例的泛型在正常使用时将每个容器的类型参数限制为固定数量。你可以通过将类型参数放置在`键`上而不是容器上来绕过这个限制。你可以使用 Class 对象作为此类类型安全异构容器的`键`。以这种方式使用的 Class 对象称为类型标记。还可以使用自定义`键`类型。例如，可以使用 DatabaseRow 类型表示数据库行（容器），并使用泛型类型 `Column<T>` 作为它的`键`。
