# 第十八节: 优先选择复合而不是继承

继承是实现代码复用的一种强大方法，但它并不总是最佳的工具。使用不当会导致软件变得脆弱。在同一个包中使用继承是安全的，其中子类和超类实现由相同的程序员控制。在对专为扩展而设计和文档化的类时使用继承也是安全的（[Item-19](/Chapter-4/Chapter-4-Item-19-Design-and-document-for-inheritance-or-else-prohibit-it.md)）。然而，对普通的具体类进行跨越包边界的继承是危险的。作为提醒，本书使用「继承」一词来表示实现继承（当一个类扩展另一个类时）。本条目中讨论的问题不适用于接口继承（当类实现接口或一个接口扩展另一个接口时）。

与方法调用不同，继承破坏了封装 [Snyder86]。换句话说，子类的功能正确与否依赖于它的超类的实现细节。超类的实现可能在版本之间发生变化，如果发生了变化，子类可能会崩溃，即使子类的代码没有被修改过。因此，子类必须与其超类同步发展，除非超类是专门为扩展的目的而设计的，并具有很明确的文档说明。

为了使问题更具体一些，让我们假设有一个使用 HashSet 的程序。为了优化程序的性能，我们需要查询 HashSet，以确定自创建以来添加了多少元素（不要与当前的大小混淆，当元素被删除时，当前的大小会递减）。为了提供这个功能，我们编写了一个变量，它记录试图插入 HashSet 的元素数量，并为这个计数变量导出一个访问器。HashSet 类包含两个能够添加元素的方法，add 和 addAll，因此我们覆盖这两个方法：

```
// Broken - Inappropriate use of inheritance!
public class InstrumentedHashSet<E> extends HashSet<E> {

    // The number of attempted element insertions
    private int addCount = 0;

    public InstrumentedHashSet() {
    }

    public InstrumentedHashSet(int initCap, float loadFactor) {
        super(initCap, loadFactor);
    }

    @Override
    public boolean add(E e) {
        addCount++;
        return super.add(e);
    }

    @Override
    public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return super.addAll(c);
    }

    public int getAddCount() {
        return addCount;
    }
}
```

这个类看起来是合理的，但是它不起作用。假设我们创建了一个实例，并使用 addAll 方法添加了三个元素。顺便说一下，我们使用 Java 9 中添加的静态工厂方法 `List.of` 创建了一个列表；如果你使用的是早期版本，那么使用 `Arrays.asList`:

```
InstrumentedHashSet<String> s = new InstrumentedHashSet<>();
s.addAll(List.of("Snap", "Crackle", "Pop"));
```

我们希望 getAddCount 方法此时返回 3，但它返回 6。到底是哪里出了错？在 HashSet 内部，addAll 方法是基于 add 方法实现的，尽管 HashSet 理所当然的没有记录这个实现细节。InstrumentedHashSet 中的 addAll 方法使 addCount 变量增加了 3，然后通过 `super.addAll` 调用 HashSet 的 addAll 实现。这相当于反过来调用在 InstrumentedHashSet 中被覆盖的 add 方法，每个元素一次。这三个调用中的每一个都使 addCount 变量增加了 1，总共增加了 6，即使用 addAll 方法添加的每个元素都被重复计数。

我们可以通过移除子类覆盖的 addAll 方法来「修复」。虽然生成的类可以工作，但它的正确功能取决于 HashSet 的 addAll 方法是基于 add 方法实现的事实。这种「自用」是实现细节，不能保证在 Java 平台的所有实现中都保留，并且在不同的版本中可能会有变化。因此，生成的 InstrumentedHashSet 类是脆弱的。

如果通过覆盖 addAll 方法来遍历指定的集合，对每个元素调用一次 add 方法会稍好一些。无论 HashSet 的 addAll 方法是否基于 add 方法实现都能保证得到正确结果，因为 HashSet 的 addAll 实现将不再被调用。然而，这种技术并不能解决我们所有的问题。它相当于重新实现超类方法，这可能会导致「自用」，也可能不会，这是困难的、耗时的、容易出错的，并且可能会降低性能。此外，这并不总是可行的，如果子类无法访问某些私有字段，这些方法就无法实现。

子类脆弱的一个原因是他们的超类可以在后续版本中获得新的方法。假设一个程序的安全性取决于插入到某个集合中的所有元素满足某个断言。这可以通过子类化集合和覆盖每个能够添加元素的方法来确保在添加元素之前满足断言。这可以很好地工作，直到在后续版本中向超类中添加能够插入元素的新方法。一旦发生这种情况，只需调用新方法就可以添加「非法」元素，而新方法在子类中不会被覆盖。这不是一个纯粹的理论问题。当 Hashtable 和 Vector 被重新改装以加入 Collections 框架时，必须修复几个这种性质的安全漏洞。

这两个问题都源于覆盖方法。你可能认为，如果只添加新方法，并且不覆盖现有方法，那么扩展类是安全的。虽然这种扩展会更安全，但也不是没有风险。如果超类在随后的版本中获得了一个新方法，而子类具有一个相同签名和不同返回类型的方法，那么你的子类将不能编译 [JLS, 8.4.8.3]。如果给子类一个方法，该方法具有与新超类方法相同的签名和返回类型，那么现在要覆盖它，因此你要面对前面描述的问题。此外，你的方法是否能够完成新的超类方法的约定是值得怀疑的，因为在你编写子类方法时，该约定还没有被写入。

幸运的是，有一种方法可以避免上述所有问题。与其扩展现有类，不如为新类提供一个引用现有类实例的私有字段。这种设计称为复合，因为现有的类是新类的一个组件。新类中的每个实例方法调用现有类实例的对应方法，并返回结果。这称为转发，新类中的方法称为转发方法。生成的类将非常坚固，不依赖于现有类的实现细节。即使向现有类添加新方法，也不会对新类产生影响。为了说明问题，这里有一个使用复合和转发方法的案例，用以替代 InstrumentedHashSet。注意，实现被分成两部分，类本身和一个可复用的转发类，其中包含所有的转发方法，没有其他内容：

```
// Wrapper class - uses composition in place of inheritance
public class InstrumentedSet<E> extends ForwardingSet<E> {

    private int addCount = 0;

    public InstrumentedSet(Set<E> s) {
        super(s);
    }

    @Override
    public boolean add(E e) {
        addCount++;
        return super.add(e);
    }

    @Override
    public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return super.addAll(c);
    }

    public int getAddCount() {
        return addCount;
    }
}

// Reusable forwarding class
public class ForwardingSet<E> implements Set<E> {
    private final Set<E> s;
    public ForwardingSet(Set<E> s) { this.s = s; }
    public void clear() { s.clear(); }
    public boolean contains(Object o) { return s.contains(o); }
    public boolean isEmpty() { return s.isEmpty(); }
    public int size() { return s.size(); }
    public Iterator<E> iterator() { return s.iterator(); }
    public boolean add(E e) { return s.add(e); }
    public boolean remove(Object o) { return s.remove(o); }
    public boolean containsAll(Collection<?> c)
    { return s.containsAll(c); }
    public boolean addAll(Collection<? extends E> c)
    { return s.addAll(c); }
    public boolean removeAll(Collection<?> c)
    { return s.removeAll(c); }
    public boolean retainAll(Collection<?> c)
    { return s.retainAll(c); }
    public Object[] toArray() { return s.toArray(); }
    public <T> T[] toArray(T[] a) { return s.toArray(a); }

    @Override
    public boolean equals(Object o){ return s.equals(o); }

    @Override
    public int hashCode() { return s.hashCode(); }

    @Override
    public String toString() { return s.toString(); }
}
```

InstrumentedSet 类的设计是通过 Set 接口来实现的，这个接口可以捕获 HashSet 类的功能。除了健壮外，这个设计非常灵活。InstrumentedSet 类实现了 Set 接口，并且有一个参数也是 Set 类型的构造函数。实际上，该类具有「插装」功能，可间接将一种 Set 的转化为另一种 Set。基于继承的方法只适用于单个具体类，并且需要为超类中每个受支持的构造函数提供单独的构造函数，与此不同的是，包装器类可用于插装任何 Set 实现，并将与任何现有构造函数一起工作：

```
Set<Instant> times = new InstrumentedSet<>(new TreeSet<>(cmp));
Set<E> s = new InstrumentedSet<>(new HashSet<>(INIT_CAPACITY));
```

InstrumentedSet 类甚至还可以用来临时配置一个没有「插装」功能的 Set 实例：

```
static void walk(Set<Dog> dogs) {
InstrumentedSet<Dog> iDogs = new InstrumentedSet<>(dogs);
... // Within this method use iDogs instead of dogs
}
```

InstrumentedSet 类被称为包装器类，因为每个 InstrumentedSet 实例都包含（「包装」）了另一个 Set 实例。这也称为装饰者模式 [Gamma95]，因为 InstrumentedSet 类通过添加「插装」来「装饰」一个集合。有时复合和转发的组合被不当地称为委托。严格来说，除非包装器对象将自身传递给包装对象，否则它不是委托 [Lieberman86; Gamma95]。

包装器类的缺点很少。一个需要注意的点是：包装器类不适合在回调框架中使用，在回调框架中，对象为后续调用(「回调」)将自定义传递给其他对象。因为包装对象不知道它对应的包装器，所以它传递一个对它自己的引用（this），回调避开包装器。这就是所谓的「自用」问题。有些人担心转发方法调用的性能影响或包装器对象的内存占用影响。这两种方法在实践中都没有多大影响。编写转发方法很麻烦，但是你必须只为每个接口编写一次可复用的转发类，而且可能会为你提供转发类。例如，Guava 为所有的集合接口提供了转发类 [Guava]。

只有子类确实是超类的子类型的情况下，继承才合适。换句话说，两个类 A、B 之间只有 B 满足「is-a」关系时才应该扩展 A。如果你想让 B 扩展 A，那就问问自己：每个 B 都是 A 吗？如果不能对这个问题给出肯定回答，B 不应该扩展 A；如果答案是否定的，通常情况下，B 应该包含 A 的私有实例并暴露不同的 API：A 不是 B 的基本组成部分，而仅仅是其实现的一个细节。

在 Java 库中有许多明显违反这一原则的地方。例如，stack 不是 vector，因此 Stack 不应该继承 Vector。类似地，property 列表不是 hash 表，因此 Properties 不应该继承 Hashtable。在这两种情况下，复合都是可取的。

如果在复合适用的地方使用了继承，就会不必要地暴露实现细节。生成的 API 将你与原始实现绑定在一起，永远限制了类的性能。更严重的是，通过暴露内部组件，你可以让客户端直接访问它们。至少，它会导致语义混乱。例如，如果 p 引用了一个 Properties 类的实例，那么 `p.getProperty(key)` 可能会产生与 `p.get(key)` 不同的结果：前者考虑了默认值，而后者（从 Hashtable 继承而来）则不会。最严重的是，客户端可以通过直接修改超类来破坏子类的不变量。对于 Properties 类，设计者希望只允许字符串作为键和值，但是直接访问底层 Hashtable 允许违反这个不变性。一旦违反，就不再可能使用 Properties API 的其他部分（加载和存储）。当发现这个问题时，已经来不及纠正了，因为客户端依赖于非字符串键和值的使用。

在决定使用继承而不是复合之前，你应该问自己最后一组问题。你打算扩展的类在其 API 中有任何缺陷吗？如果是这样，你是否愿意将这些缺陷传播到类的 API 中？继承传播超类 API 中的任何缺陷，而复合允许你设计一个新的 API 来隐藏这些缺陷。

总而言之，继承是强大的，但是它是有问题的，因为它打破了封装。只有当子类和超类之间存在真正的子类型关系时才合适。即使这样，如果子类与超类在不同的包中，并且超类不是为继承而设计的，继承也可能导致程序脆弱。为了避免这种缺陷，应使用复合和转发，特别是存在适当接口能实现包装器类时更应如此。包装类不仅比子类更健壮，而且功能更强大。