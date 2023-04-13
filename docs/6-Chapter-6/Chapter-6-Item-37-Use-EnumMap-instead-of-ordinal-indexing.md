# 第三十七节: 使用 EnumMap 替换序数索引

偶尔你可能会看到使用 `ordinal()` 的返回值（[Item-35](../Chapter-6/Chapter-6-Item-35-Use-instance-fields-instead-of-ordinals)）作为数组或 list 索引的代码。例如，考虑这个简单的类，它表示一种植物：

```
class Plant {
    enum LifeCycle { ANNUAL, PERENNIAL, BIENNIAL }
    final String name;
    final LifeCycle lifeCycle;

    Plant(String name, LifeCycle lifeCycle) {
        this.name = name;
        this.lifeCycle = lifeCycle;
    }

    @Override public String toString() {
        return name;
    }
}
```

现在假设你有一个代表花园全部植物的 Plant 数组，你想要列出按生命周期（一年生、多年生或两年生）排列的植物。要做到这一点，你需要构造三个集合，每个生命周期一个，然后遍历整个数组，将每个植物放入适当的集合中：

```
// Using ordinal() to index into an array - DON'T DO THIS!
Set<Plant>[] plantsByLifeCycle =(Set<Plant>[]) new Set[Plant.LifeCycle.values().length];

for (int i = 0; i < plantsByLifeCycle.length; i++)
    plantsByLifeCycle[i] = new HashSet<>();

for (Plant p : garden)
    plantsByLifeCycle[p.lifeCycle.ordinal()].add(p);

// Print the results
for (int i = 0; i < plantsByLifeCycle.length; i++) {
    System.out.printf("%s: %s%n",
    Plant.LifeCycle.values()[i], plantsByLifeCycle[i]);
}
```

**译注：假设 Plant 数组如下：**
```
Plant[] garden = new Plant[]{
        new Plant("A", LifeCycle.ANNUAL),
        new Plant("B", LifeCycle.BIENNIAL),
        new Plant("C", LifeCycle.PERENNIAL),
        new Plant("D", LifeCycle.BIENNIAL),
        new Plant("E", LifeCycle.PERENNIAL),
};
```

输出结果为：
```
ANNUAL: [A]
PERENNIAL: [E, C]
BIENNIAL: [B, D]
```

这种技术是有效的，但它充满了问题。因为数组与泛型不兼容（[Item-28](../Chapter-5/Chapter-5-Item-28-Prefer-lists-to-arrays)），所以该程序需要 unchecked 的转换，否则不能顺利地编译。因为数组不知道它的索引表示什么，所以必须手动标记输出。但是这种技术最严重的问题是，当你访问一个由枚举序数索引的数组时，你有责任使用正确的 int 值；int 不提供枚举的类型安全性。如果你使用了错误的值，程序将静默执行错误的操作，如果幸运的话，才会抛出 ArrayIndexOutOfBoundsException。

有一种更好的方法可以达到同样的效果。该数组有效地充当从枚举到值的映射，因此你不妨使用 Map。更具体地说，有一种非常快速的 Map 实现，用于枚举键，称为 `java.util.EnumMap`。以下就是这个程序在使用 EnumMap 时的样子：

```
// Using an EnumMap to associate data with an enum
Map<Plant.LifeCycle, Set<Plant>> plantsByLifeCycle =new EnumMap<>(Plant.LifeCycle.class);

for (Plant.LifeCycle lc : Plant.LifeCycle.values())
    plantsByLifeCycle.put(lc, new HashSet<>());

for (Plant p : garden)
    plantsByLifeCycle.get(p.lifeCycle).add(p);

System.out.println(plantsByLifeCycle);
```

这个程序比原来的版本更短，更清晰，更安全，速度也差不多。没有不安全的转换；不需要手动标记输出，因为 Map 的键是能转换为可打印字符串的枚举；在计算数组索引时不可能出错。EnumMap 在速度上与有序索引数组相当的原因是，EnumMap 在内部使用这样的数组，但是它向程序员隐藏了实现细节，将 Map 的丰富的功能和类型安全性与数组的速度结合起来。注意，EnumMap 构造函数接受键类型的 Class 对象：这是一个有界类型标记，它提供运行时泛型类型信息（[Item-33](../Chapter-5/Chapter-5-Item-33-Consider-typesafe-heterogeneous-containers)）。

通过使用流（[Item-45](../Chapter-7/Chapter-7-Item-45-Use-streams-judiciously)）来管理映射，可以进一步缩短前面的程序。下面是基于流的最简单的代码，它在很大程度上复制了前一个示例的行为：

```
// Naive stream-based approach - unlikely to produce an EnumMap!
System.out.println(Arrays.stream(garden).collect(groupingBy(p -> p.lifeCycle)));
```

**译注：以上代码需要引入 `java.util.stream.Collectors.groupingBy`，输出结果如下：**
```
{BIENNIAL=[B, D], ANNUAL=[A], PERENNIAL=[C, E]}
```

这段代码的问题在于它选择了自己的 Map 实现，而实际上它不是 EnumMap，所以它的空间和时间性能与显式 EnumMap 不匹配。要纠正这个问题，可以使用 `Collectors.groupingBy` 的三参数形式，它允许调用者使用 mapFactory 参数指定 Map 实现：

```
// Using a stream and an EnumMap to associate data with an enum
System.out.println(
    Arrays.stream(garden).collect(groupingBy(p -> p.lifeCycle,() -> new EnumMap<>(LifeCycle.class), toSet()))
);
```

**译注：以上代码需要引入 `java.util.stream.Collectors.toSet`**

这种优化在示例程序中不值得去做，但在大量使用 Map 的程序中可能非常重要。

基于流的版本的行为与 EmumMap 版本略有不同。EnumMap 版本总是为每个植物生命周期生成一个嵌套 Map，而基于流的版本只在花园包含具有该生命周期的一个或多个植物时才生成嵌套 Map。例如，如果花园包含一年生和多年生植物，但没有两年生植物，plantsByLifeCycle 的大小在 EnumMap 版本中为 3，在基于流的版本中为 2。

你可能会看到被序数索引（两次！）的数组，序数用于表示两个枚举值的映射。例如，这个程序使用这样的一个数组来映射两个状态到一个状态的转换过程（液体到固体是冻结的，液体到气体是沸腾的，等等）：

```
// Using ordinal() to index array of arrays - DON'T DO THIS!
public enum Phase {
    SOLID, LIQUID, GAS;

    public enum Transition {
        MELT, FREEZE, BOIL, CONDENSE, SUBLIME, DEPOSIT;

        // Rows indexed by from-ordinal, cols by to-ordinal
        private static final Transition[][] TRANSITIONS = {
            { null, MELT, SUBLIME },
            { FREEZE, null, BOIL },
            { DEPOSIT, CONDENSE, null }
        };

        // Returns the phase transition from one phase to another
        public static Transition from(Phase from, Phase to) {
            return TRANSITIONS[from.ordinal()][to.ordinal()];
        }
    }
}
```

**译注：固体、液体、气体三态，对应的三组变化：融化 MELT，冻结 FREEZE（固态与液态）；沸腾 BOIL，凝固 CONDENSE（液态与气态）；升华 SUBLIME，凝华 DEPOSIT（固态与气态）。**

这个程序可以工作，甚至可能看起来很优雅，但外表可能具有欺骗性。就像前面展示的更简单的 garden 示例一样，编译器无法知道序数和数组索引之间的关系。如果你在转换表中出错，或者在修改 Phase 或 `Phase.Transition` 枚举类型时忘记更新，你的程序将在运行时失败。失败可能是抛出 ArrayIndexOutOfBoundsException、NullPointerException 或（更糟糕的）静默错误行为。并且即使非空项的数目更小，该表的大小也为状态数量的二次方。

同样，使用 EnumMap 可以做得更好。因为每个阶段转换都由一对阶段枚举索引，所以最好将这个关系用 Map 表示，从一个枚举（起始阶段）到第二个枚举（结束阶段）到结果（转换阶段）。与阶段转换相关联的两个阶段最容易捕捉到的是将它们与阶段过渡的 enum 联系起来，这样就可以用来初始化嵌套的 EnumMap：

```
// Using a nested EnumMap to associate data with enum pairs
public enum Phase {
    SOLID, LIQUID, GAS;

    public enum Transition {
        MELT(SOLID, LIQUID), FREEZE(LIQUID, SOLID),
        BOIL(LIQUID, GAS), CONDENSE(GAS, LIQUID),
        SUBLIME(SOLID, GAS), DEPOSIT(GAS, SOLID);
        private final Phase from;
        private final Phase to;

        Transition(Phase from, Phase to) {
            this.from = from;
            this.to = to;
        }

        // Initialize the phase transition map
        private static final Map<Phase_new, Map<Phase_new, Transition>> m = Stream.of(values())
                .collect(groupingBy(
                        t -> t.from,
                        () -> new EnumMap<>(Phase_new.class),
                        toMap(t -> t.to, t -> t, (x, y) -> y, () -> new EnumMap<>(Phase_new.class))
                        )
                );

        public static Transition from(Phase from, Phase to) {
            return m.get(from).get(to);
        }
    }
}
```

初始化阶段变化 Map 的代码有点复杂。Map 的类型是 `Map<Phase, Map<Phase, Transition>>`，这意味着「从（源）阶段 Map 到（目标）阶段 Map 的转换过程」。这个 Map 嵌套是使用两个收集器的级联序列初始化的。第一个收集器按源阶段对转换进行分组，第二个收集器使用从目标阶段到转换的映射创建一个 EnumMap。第二个收集器 ((x, y) -> y) 中的 merge 函数未使用；之所以需要它，只是因为我们需要指定一个 Map 工厂来获得 EnumMap，而 Collector 提供了伸缩工厂。本书的上一个版本使用显式迭代来初始化阶段转换映射。代码更冗长，但也更容易理解。

**译注：第二版中的实现代码如下：**
```
// Initialize the phase transition map
private static final Map<Phase, Map<Phase,Transition> m =
    new EnumMap<Phase, Map<Phase ,Transition>>(Phase.class);

    static{
        for (Phase p : Phase. values())
            m.put(p,new EnumMap<Phase,Transition (Phase.class));
        for (Transition trans : Transition.values() )
            m.get(trans. src).put(trans.dst, trans) ;
    }

public static Transition from(Phase src, Phase dst) {
    return m.get(src).get(dst);
}
```

现在假设你想向系统中加入一种新阶段：等离子体，或电离气体。这个阶段只有两个变化：电离，它把气体转为等离子体；去离子作用，把等离子体变成气体。假设要更新基于数组版本的程序，必须向 Phase 添加一个新常量，向 `Phase.Transition` 添加两个新常量，并用一个新的 16 个元素版本替换原来的数组中的 9 个元素数组。如果你向数组中添加了太多或太少的元素，或者打乱了元素的顺序，那么你就麻烦了：程序将编译，但在运行时将失败。相比之下，要更新基于 EnumMap 的版本，只需将 PLASMA 添加到 Phase 列表中，将 `IONIZE(GAS, PLASMA)` 和 `DEIONIZE(PLASMA, GAS)` 添加到 `Phase.Transition` 中：

```
// Adding a new phase using the nested EnumMap implementation
public enum Phase {
    SOLID, LIQUID, GAS, PLASMA;
    public enum Transition {
        MELT(SOLID, LIQUID), FREEZE(LIQUID, SOLID),
        BOIL(LIQUID, GAS), CONDENSE(GAS, LIQUID),
        SUBLIME(SOLID, GAS), DEPOSIT(GAS, SOLID),
        IONIZE(GAS, PLASMA), DEIONIZE(PLASMA, GAS);
        ... // Remainder unchanged
    }
}
```

这个程序会处理所有其他事情，实际上不会给你留下任何出错的机会。在内部，Map 的映射是用一个数组来实现的，因此你只需花费很少的空间或时间成本就可以获得更好的清晰度、安全性并易于维护。

为了简洁起见，最初的示例使用 null 表示没有状态更改（其中 to 和 from 是相同的）。这不是一个好的方式，可能会在运行时导致 NullPointerException。针对这个问题设计一个干净、优雅的解决方案是非常棘手的，并且生成的程序冗长，以至于它们会偏离条目中的主要内容。

总之，**用普通的序数索引数组是非常不合适的：应使用 EnumMap 代替。** 如果所表示的关系是多维的，则使用 `EnumMap<..., EnumMap<...>>`。这是一种特殊的基本原则，程序员很少（即使有的话）使用 `Enum.ordinal` （[Item-35](../Chapter-6/Chapter-6-Item-35-Use-instance-fields-instead-of-ordinals)）。
