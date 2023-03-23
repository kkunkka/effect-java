# 第二十九节: 优先使用泛型

通常，对声明进行参数化并使用 JDK 提供的泛型和方法并不太难。编写自己的泛型有点困难，但是值得努力学习。

考虑 [Item-7](/Chapter-2/Chapter-2-Item-7-Eliminate-obsolete-object-references.md) 中简单的堆栈实现：

```
// Object-based collection - a prime candidate for generics
public class Stack {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(Object e) {
        ensureCapacity();
        elements[size++] = e;
    }

    public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        Object result = elements[--size];
        elements[size] = null; // Eliminate obsolete reference
        return result;
    }

    public boolean isEmpty() {
        return size == 0;
    }

    private void ensureCapacity() {
        if (elements.length == size)
            elements = Arrays.copyOf(elements, 2 * size + 1);
    }
}
```

这个类一开始就应该是参数化的，但是因为它不是参数化的，所以我们可以在事后对它进行泛化。换句话说，我们可以对它进行参数化，而不会损害原始非参数化版本的客户端。按照目前的情况，客户端必须转换从堆栈中弹出的对象，而这些转换可能在运行时失败。生成类的第一步是向其声明中添加一个或多个类型参数。在这种情况下，有一个类型参数，表示堆栈的元素类型，这个类型参数的常规名称是 E（[Item-68](/Chapter-9/Chapter-9-Item-68-Adhere-to-generally-accepted-naming-conventions.md)）。

下一步是用适当的类型参数替换所有的 Object 类型，然后尝试编译修改后的程序：

```
// Initial attempt to generify Stack - won't compile!
public class Stack<E> {
    private E[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        elements = new E[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(E e) {
        ensureCapacity();
        elements[size++] = e;
    }

    public E pop() {
        if (size == 0)
            throw new EmptyStackException();
        E result = elements[--size];
        elements[size] = null; // Eliminate obsolete reference
        return result;
    } ... // no changes in isEmpty or ensureCapacity
}
```

通常至少会得到一个错误或警告，这个类也不例外。幸运的是，这个类只生成一个错误：

```
Stack.java:8: generic array creation
elements = new E[DEFAULT_INITIAL_CAPACITY];
^
```

正如 [Item-28](/Chapter-5/Chapter-5-Item-28-Prefer-lists-to-arrays.md) 中所解释的，你不能创建非具体化类型的数组，例如 E。每当你编写由数组支持的泛型时，就会出现这个问题。有两种合理的方法来解决它。第一个解决方案直接绕过了创建泛型数组的禁令：创建对象数组并将其强制转换为泛型数组类型。现在，编译器将发出一个警告来代替错误。这种用法是合法的，但（一般而言）它不是类型安全的：

```
Stack.java:8: warning: [unchecked] unchecked cast
found: Object[], required: E[]
elements = (E[]) new Object[DEFAULT_INITIAL_CAPACITY];
^
```

编译器可能无法证明你的程序是类型安全的，但你可以。你必须说服自己，unchecked 的转换不会损害程序的类型安全性。所涉及的数组（元素）存储在私有字段中，从未返回给客户端或传递给任何其他方法。数组中存储的惟一元素是传递给 push 方法的元素，它们属于 E 类型，因此 unchecked 的转换不会造成任何损害。

一旦你证明了 unchecked 的转换是安全的，就将警告限制在尽可能小的范围内（[Item-27](/Chapter-5/Chapter-5-Item-27-Eliminate-unchecked-warnings.md)）。在这种情况下，构造函数只包含 unchecked 的数组创建，因此在整个构造函数中取消警告是合适的。通过添加注解来实现这一点，Stack 可以干净地编译，而且你可以使用它而无需显式强制转换或担心 ClassCastException：

```
// The elements array will contain only E instances from push(E).
// This is sufficient to ensure type safety, but the runtime
// type of the array won't be E[]; it will always be Object[]!
@SuppressWarnings("unchecked")
public Stack() {
    elements = (E[]) new Object[DEFAULT_INITIAL_CAPACITY];
}
```

消除 Stack 中泛型数组创建错误的第二种方法是将字段元素的类型从 E[] 更改为 Object[]。如果你这样做，你会得到一个不同的错误：

```
Stack.java:19: incompatible types
found: Object, required: E
E result = elements[--size];
^
```

通过将从数组中检索到的元素转换为 E，可以将此错误转换为警告，但你将得到警告：

```
Stack.java:19: warning: [unchecked] unchecked cast
found: Object, required: E
E result = (E) elements[--size];
^
```

因为 E 是不可具体化的类型，编译器无法在运行时检查强制转换。同样，你可以很容易地向自己证明 unchecked 的强制转换是安全的，因此可以适当地抑制警告。根据 [Item-27](/Chapter-5/Chapter-5-Item-27-Eliminate-unchecked-warnings.md) 的建议，我们仅对包含 unchecked 强制转换的赋值禁用警告，而不是对整个 pop 方法禁用警告：

```
// Appropriate suppression of unchecked warning
public E pop() {
    if (size == 0)
        throw new EmptyStackException();
    // push requires elements to be of type E, so cast is correct
    @SuppressWarnings("unchecked")
    E result =(E) elements[--size];
    elements[size] = null; // Eliminate obsolete reference
    return result;
}
```

消除泛型数组创建的两种技术都有其追随者。第一个更容易读：数组声明为 E[] 类型，这清楚地表明它只包含 E 的实例。它也更简洁：在一个典型的泛型类中，从数组中读取代码中的许多点；第一种技术只需要一次转换（在创建数组的地方），而第二种技术在每次读取数组元素时都需要单独的转换。因此，第一种技术是可取的，在实践中更常用。但是，它确实会造成堆污染（[Item-32](/Chapter-5/Chapter-5-Item-32-Combine-generics-and-varargs-judiciously.md)）：数组的运行时类型与其编译时类型不匹配（除非 E 恰好是 Object）。尽管堆污染在这种情况下是无害的，但这使得一些程序员感到非常不安，因此他们选择了第二种技术。

下面的程序演示了通用 Stack 的使用。程序以相反的顺序打印它的命令行参数并转换为大写。在从堆栈弹出的元素上调用 String 的 toUpperCase 方法不需要显式转换，自动生成的转换保证成功：

```
// Little program to exercise our generic Stack
public static void main(String[] args) {
    Stack<String> stack = new Stack<>();
    for (String arg : args)
        stack.push(arg);
    while (!stack.isEmpty())
        System.out.println(stack.pop().toUpperCase());
}
```

前面的例子可能与 [Item-28](/Chapter-5/Chapter-5-Item-28-Prefer-lists-to-arrays.md) 相矛盾，Item-28 鼓励优先使用列表而不是数组。在泛型中使用列表并不总是可能的或可取的。Java 本身不支持列表，因此一些泛型（如 ArrayList）必须在数组之上实现。其他泛型（如 HashMap）是在数组之上实现的，以提高性能。大多数泛型与我们的 Stack 示例相似，因为它们的类型参数没有限制：你可以创建 `Stack<Object>`、Stack<int[]>、Stack<List<String>> 或任何其他对象引用类型的堆栈。注意，不能创建基本类型的 Stack：试图创建 `Stack<int>` 或 `Stack<double>` 将导致编译时错误。

这是 Java 泛型系统的一个基本限制。你可以通过使用装箱的基本类型（[Item-61](/Chapter-9/Chapter-9-Item-61-Prefer-primitive-types-to-boxed-primitives.md)）来绕过这一限制。有一些泛型限制了其类型参数的允许值。例如，考虑 java.util.concurrent.DelayQueue，其声明如下：

```
class DelayQueue<E extends Delayed> implements BlockingQueue<E>
```

类型参数列表（<E extends Delayed>）要求实际的类型参数 E 是 java.util.concurrent.Delayed 的一个子类型。这允许 DelayQueue 实现及其客户端利用 DelayQueue 元素上的 Delayed 方法，而不需要显式转换或 ClassCastException 的风险。类型参数 E 称为有界类型参数。注意，子类型关系的定义使得每个类型都是它自己的子类型 [JLS, 4.10]，所以创建 `DelayQueue<Delayed>` 是合法的。

总之，泛型比需要在客户端代码中转换的类型更安全、更容易使用。在设计新类型时，请确保可以在不使用此类类型转换的情况下使用它们。这通常意味着使类型具有通用性。如果你有任何应该是泛型但不是泛型的现有类型，请对它们进行泛型。这将使这些类型的新用户在不破坏现有客户端（[Item-26](/Chapter-5/Chapter-5-Item-26-Do-not-use-raw-types.md)）的情况下更容易使用。
