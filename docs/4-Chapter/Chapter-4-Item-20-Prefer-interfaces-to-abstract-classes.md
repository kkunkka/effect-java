# 第二十节: 接口优于抽象类

Java 有两种机制来定义允许多种实现的类型：接口和抽象类。由于 Java 8 [JLS 9.4.3]中引入了接口的默认方法，这两种机制都允许你为一些实例方法提供实现。一个主要区别是，一个类要实现抽象类定义的类型，该类必须是抽象类的子类。因为 Java 只允许单一继承，这种限制对抽象类而言严重制约了它们作为类型定义的使用。任何定义了所有必需的方法并遵守通用约定的类都允许实现接口，而不管该类驻留在类层次结构中何处。

**译注：**

**1、抽象类的局限：一个类要实现抽象类定义的类型，该类必须是抽象类的子类。因为 Java 只允许单一继承，这种限制对抽象类而言严重制约了它们作为类型定义的使用。**

**2、接口的优点：任何定义了所有必需的方法并遵守通用约定的类都允许实现接口，而不管该类驻留在类层次结构中何处。**

**可以很容易地对现有类进行改造，以实现新的接口。** 你所要做的就是添加所需的方法（如果它们还不存在的话），并向类声明中添加一个 implements 子句。例如，许多现有的类在添加到 JDK 时进行了修改，以实现 Comparable、Iterable 和 Autocloseable 接口。一般来说，现有的类不能被修改以扩展新的抽象类。如果你想让两个类扩展同一个抽象类，你必须把它放在类型层次结构的高层，作为两个类的祖先。不幸的是，这可能会对类型层次结构造成巨大的附带损害，迫使新抽象类的所有后代对其进行子类化，无论它是否合适。

**接口是定义 mixin（混合类型）的理想工具。** 粗略地说，mixin 是类除了「基本类型」之外还可以实现的类型，用于声明它提供了一些可选的行为。例如，Comparable 是一个 mixin 接口，它允许类的实例可以与其他的可相互比较的对象进行排序。这样的接口称为 mixin，因为它允许可选功能「混合」到类型的主要功能中。抽象类不能用于定义 mixin，原因与它们不能被修改到现有类相同：一个类不能有多个父类，而且在类层次结构中没有插入 mixin 的合理位置。

**接口允许构造非层次化类型框架。** 类型层次结构对于组织一些事情很好，但是其他事情不能整齐地归入严格的层次结构。例如，假设我们有一个代表歌手的接口和另一个代表词曲作者的接口：

```
public interface Singer {
    AudioClip sing(Song s);
}

public interface Songwriter {
    Song compose(int chartPosition);
}
```

在现实生活中，一些歌手也是词曲作者。因为我们使用接口而不是抽象类来定义这些类型，所以完全允许单个类同时实现歌手和词曲作者。事实上，我们可以定义第三个接口，扩展歌手和词曲作者，并添加适合这种组合的新方法：

```
public interface SingerSongwriter extends Singer, Songwriter {
    AudioClip strum();
    void actSensitive();
}
```

你并不总是需要这种级别的灵活性，但是当你需要时，接口就是救星。另一种选择是一个臃肿的类层次结构，它为每个受支持的属性组合包含一个单独的类。如果类型系统中有 n 个属性，那么可能需要支持 2<sup>n</sup>种组合。这就是所谓的组合爆炸。臃肿的类层次结构可能导致类也臃肿，其中许多方法只在其参数的类型上有所不同，因为类层次结构中没有类型来捕获公共行为。

通过 [Item-18](/Chapter-4/Chapter-4-Item-18-Favor-composition-over-inheritance.md) 介绍的包装类，接口能够支持安全、强大的功能增强。如果你使用抽象类来定义类型，那么希望添加功能的程序员除了继承之外别无选择。最终生成的类不如包装类强大，也更脆弱。

如果接口方法的实现与其他接口方法类似，那么可以考虑以默认方法的形式为程序员提供实现帮助。有关此技术的示例，请参阅第 104 页的 removeIf 方法。如果提供了默认方法，请使用 `@implSpec` 标签，并确保在文档中记录他们的继承关系（[Item-19](/Chapter-4/Chapter-4-Item-19-Design-and-document-for-inheritance-or-else-prohibit-it.md)）。

默认方法为实现提供的帮助有限。尽管许多接口指定了诸如 equals 和 hashCode 等对象方法的行为，但是不允许为它们提供默认方法。此外，接口不允许包含实例字段或非公共静态成员（私有静态方法除外）。最后，你不能向你不控制的接口添加默认方法。

但是，你可以通过提供一个抽象骨架实现类来结合接口和抽象类的优点。接口定义了类型，可能提供了一些默认方法，而骨架实现类在基本接口方法之上实现了其余的非基本接口方法。扩展骨架实现需要完成实现接口的大部分工作。这是模板方法模式 [Gamma95]。

按照惯例，骨架实现类称为 AbstractInterface，其中 Interface 是它们实现的接口的名称。例如，Collections Framework 提供了一个骨架实现来配合每个主要的集合接口：AbstractCollection、AbstractSet、AbstractList 和 AbstractMap。可以说，将它们称为 SkeletalCollection、SkeletalSet、SkeletalList 和 SkeletalMap 是有意义的，但 Abstract 的用法现在已经根深蒂固。如果设计得当，骨架实现（无论是单独的抽象类，还是仅仅由接口上的默认方法组成）可以使程序员非常容易地提供他们自己的接口实现。例如，这里有一个静态工厂方法，它在 AbstractList 上包含一个完整的、功能完整的 List 实现：

```
// Concrete implementation built atop skeletal implementation
static List<Integer> intArrayAsList(int[] a) {
        Objects.requireNonNull(a);
        // The diamond operator is only legal here in Java 9 and later
        // If you're using an earlier release, specify <Integer>
        return new AbstractList<>() {
            @Override
            public Integer get(int i) {
                return a[i]; // Autoboxing (Item 6)
            }

            @Override
            public Integer set(int i, Integer val) {
                int oldVal = a[i];
                a[i] = val; // Auto-unboxing
                return oldVal; // Autoboxing
            }

            @Override
            public int size() {
                return a.length;
            }
        };
}
```

当你考虑到 List 实现为你做的所有事情时，这个例子是一个令人印象深刻的演示，体现了骨架实现的强大功能。顺便说一句，这个示例是一个 Adapter（适配器）[Gamma95]，它允许将 int 数组视为 Integer 实例的 list。因为在 int 值和 Integer 实例（装箱和拆箱）之间来回转换，所以它的性能不是很好。注意，实现的形式是匿名类（[Item-24](/Chapter-4/Chapter-4-Item-24-Favor-static-member-classes-over-nonstatic.md)）。

骨架实现类的美妙之处在于，它们提供了抽象类的所有实现帮助，而不像抽象类作为类型定义时那样受到严格的约束。对于具有骨架实现类的接口的大多数实现来说，扩展这个类是显而易见的选择，但它并不是必需的。如果不能使类扩展骨架实现，则类总是可以直接实现接口。类仍然受益于接口本身的任何默认方法。此外，骨架实现仍然可以帮助实现人员完成任务。实现接口的类可以将接口方法的调用转发给扩展骨架实现的私有内部类的包含实例。这种技术称为模拟多重继承，与[Item-18](/Chapter-4/Chapter-4-Item-18-Favor-composition-over-inheritance.md)中讨论的包装类密切相关。它提供了多重继承的许多好处，同时避免了缺陷。

编写一个骨架实现是一个相对简单的过程，尽管有点乏味。首先，研究接口并决定哪些方法是基本方法，以便其他方法可以根据它们实现。这些基本方法将是你的骨架实现中的抽象方法。接下来，在接口中为所有可以直接在基本方法之上实现的方法提供默认方法，但请记住，你可能不会为诸如 equals 和 hashCode 之类的对象方法提供默认方法。如果基本方法和默认方法覆盖了接口，那么就完成了，不需要一个骨架实现类。否则，编写一个声明为实现接口的类，并实现所有剩余的接口方法。该类可能包含任何适合于任务的非公共字段和方法。

作为一个简单的例子，考虑一下 `Map.Entry` 接口。最明显的基本方法是 getKey、getValue 和（可选的）setValue。该接口指定了 equals 和 hashCode 的行为，并且在基本方法方面有 toString 的明显实现。由于不允许为对象方法提供默认实现，所有实现都放在骨架实现类中：

```
// Skeletal implementation class
public abstract class AbstractMapEntry<K,V> implements Map.Entry<K,V> {

    // Entries in a modifiable map must override this method
    @Override public V setValue(V value) {
        throw new UnsupportedOperationException();
    }

    // Implements the general contract of Map.Entry.equals
    @Override public boolean equals(Object o) {
        if (o == this)
            return true;
        if (!(o instanceof Map.Entry))
            return false;
        Map.Entry<?,?> e = (Map.Entry) o;
        return Objects.equals(e.getKey(), getKey()) && Objects.equals(e.getValue(), getValue());
    }

    // Implements the general contract of Map.Entry.hashCode
    @Override public int hashCode() {
        return Objects.hashCode(getKey())^ Objects.hashCode(getValue());
    }

    @Override public String toString() {
        return getKey() + "=" + getValue();
    }
}
```

注意，这个骨架实现不能在 `Map.Entry` 接口或子接口中实现，因为不允许默认方法覆盖诸如 equals、hashCode 和 toString 等对象方法。

因为骨架实现是为继承而设计的，所以你应该遵循 [Item-19](/Chapter-4/Chapter-4-Item-19-Design-and-document-for-inheritance-or-else-prohibit-it.md) 中的所有设计和文档指南。为了简洁起见，在前面的示例中省略了文档注释，但是优秀的文档对于骨架实现来说是绝对必要的，不管它是由接口上的默认方法还是单独的抽象类组成。

骨架实现的一个小变种是简单实现，例如 `AbstractMap.SimpleEntry`。一个简单的实现就像一个骨架实现，因为它实现了一个接口，并且是为继承而设计的，但是它的不同之处在于它不是抽象的：它是最简单的工作实现。你可以根据它的状态使用它，也可以根据情况对它进行子类化。

总之，接口通常是定义允许多种实现的类型的最佳方法。如果导出了一个重要的接口，则应该强烈考虑提供一个骨架实现。尽可能地，你应该通过接口上的默认方法提供骨架实现，以便接口的所有实现者都可以使用它。也就是说，对接口的限制通常要求框架实现采用抽象类的形式。
