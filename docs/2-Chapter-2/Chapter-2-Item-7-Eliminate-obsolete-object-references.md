# 第七节: 排除过时的对象引用

如果你从需要手动管理内存的语言（如 C 或 c++）切换到具有垃圾回收机制的语言（如 Java），当你使用完对象后，会感觉程序员工作轻松很多。当你第一次体验它的时候，它几乎就像魔术一样。这很容易让人觉得你不需要考虑内存管理，但这并不完全正确。

考虑以下简单的堆栈实现：

```
import java.util.Arrays;
import java.util.EmptyStackException;

// Can you spot the "memory leak"?
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
        return elements[--size];
    }

    /**
     * Ensure space for at least one more element, roughly
     * doubling the capacity each time the array needs to grow.
     */
    private void ensureCapacity() {
        if (elements.length == size)
            elements = Arrays.copyOf(elements, 2 * size + 1);
    }
}
```

这个程序没有明显的错误（但是通用版本请参阅 [Item-29](../Chapter-5/Chapter-5-Item-29-Favor-generic-types)）。你可以对它进行详尽的测试，它会以优异的成绩通过所有的测试，但是有一个潜在的问题。简单地说，该程序有一个「内存泄漏」问题，由于垃圾收集器活动的增加或内存占用的增加，它可以悄无声息地表现为性能的降低。在极端情况下，这种内存泄漏可能导致磁盘分页，甚至出现 OutOfMemoryError 程序故障，但这种故障相对少见。

那么内存泄漏在哪里呢？如果堆栈增长，然后收缩，那么从堆栈中弹出的对象将不会被垃圾收集，即使使用堆栈的程序不再引用它们。这是因为栈保留了这些对象的旧引用。一个过时的引用，是指永远不会被取消的引用。在本例中，元素数组的「活动部分」之外的任何引用都已过时。活动部分由索引小于大小的元素组成。

垃圾收集语言中的内存泄漏（更确切地说是无意的对象保留）是暗藏的风险。如果无意中保留了对象引用，那么对象不仅被排除在垃圾收集之外，该对象引用的任何对象也被排除在外，依此类推。即使只是无意中保留了一些对象引用，许多许多的对象也可能被阻止被垃圾收集，从而对性能产生潜在的巨大影响。

解决这类问题的方法很简单：一旦引用过时，就将置空。在我们的 Stack 类中，对某个项的引用一旦从堆栈中弹出就会过时。pop 方法的正确版本如下：

```
public Object pop() {
    if (size == 0)
        throw new EmptyStackException();
    Object result = elements[--size];
    elements[size] = null; // Eliminate obsolete reference
    return result;
}
```

用 null 处理过时引用的另一个好处是，如果它们随后被错误地关联引用，程序将立即失败，出现 NullPointerException，而不是悄悄地做错误的事情。尽可能快地检测编程错误总是有益的。

当程序员第一次被这个问题困扰时，他们可能会过度担心，一旦程序使用完它，他们就会取消所有对象引用。这既无必要也不可取；它不必要地搞乱了程序。清除对象引用应该是例外，而不是规范。消除过时引用的最佳方法是让包含引用的变量脱离作用域。如果你在最狭窄的范围内定义每个变量（[Item-57](../Chapter-9/Chapter-9-Item-57-Minimize-the-scope-of-local-variables)），那么这种情况自然会发生。


那么，什么时候应该取消引用呢？Stack 类的哪些方面容易导致内存泄漏？简单地说，它管理自己的内存。存储池包含元素数组的元素（对象引用单元，而不是对象本身）数组的活动部分（如前面所定义的）中的元素被分配，而数组其余部分中的元素是空闲的。垃圾收集器没有办法知道这一点；对于垃圾收集器，元素数组中的所有对象引用都同样有效。只有程序员知道数组的非活动部分不重要。只要数组元素成为非活动部分的一部分，程序员就可以通过手动清空数组元素，有效地将这个事实传递给垃圾收集器。

一般来说，一个类管理它自己的内存时，程序员应该警惕内存泄漏。当释放一个元素时，该元素中包含的任何对象引用都应该被置为 null。

另一个常见的内存泄漏源是缓存。一旦将对象引用放入缓存中，就很容易忘记它就在那里，并且在它变得无关紧要之后很久仍将它留在缓存中。有几个解决这个问题的办法。如果你非常幸运地实现了一个缓存，只要缓存外有对其键的引用，那么就将缓存表示为 WeakHashMap；当条目过时后，条目将被自动删除。记住，WeakHashMap 只有在缓存条目的预期生存期由键的外部引用（而不是值）决定时才有用。

更常见的情况是，缓存条目的有效生存期定义不太好，随着时间的推移，条目的价值会越来越低。在这种情况下，缓存偶尔应该清理那些已经停用的条目。这可以通过后台线程（可能是 ScheduledThreadPoolExecutor）或向缓存添加新条目时顺便完成。LinkedHashMap 类通过其 removeEldestEntry 方法简化了后一种方法。对于更复杂的缓存，你可能需要直接使用 java.lang.ref。

**内存泄漏的第三个常见来源是侦听器和其他回调。** 如果你实现了一个 API，其中客户端注册回调，但不显式取消它们，除非你采取一些行动，否则它们将累积。确保回调被及时地垃圾收集的一种方法是仅存储对它们的弱引用，例如，将它们作为键存储在 WeakHashMap 中。

由于内存泄漏通常不会表现为明显的故障，它们可能会在系统中存在多年。它们通常只能通过仔细的代码检查或借助一种称为堆分析器的调试工具来发现。因此，学会在这样的问题发生之前预测并防止它们发生是非常可取的。
