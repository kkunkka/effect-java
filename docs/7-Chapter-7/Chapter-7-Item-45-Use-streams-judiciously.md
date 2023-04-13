# 第四十五节: 明智地使用流

在 Java 8 中添加了流 API，以简化序列或并行执行批量操作的任务。这个 API 提供了两个关键的抽象：流（表示有限或无限的数据元素序列）和流管道（表示对这些元素的多阶段计算）。流中的元素可以来自任何地方。常见的源包括集合、数组、文件、正则表达式的 Pattern 匹配器、伪随机数生成器和其他流。流中的数据元素可以是对象的引用或基本数据类型。支持三种基本数据类型：int、long 和 double。

流管道由源流、零个或多个 Intermediate 操作和一个 Terminal 操作组成。每个 Intermediate 操作以某种方式转换流，例如将每个元素映射到该元素的一个函数，或者过滤掉不满足某些条件的所有元素。中间操作都将一个流转换为另一个流，其元素类型可能与输入流相同，也可能与输入流不同。Terminal 操作对最后一次 Intermediate 操作所产生的流进行最终计算，例如将其元素存储到集合中、返回特定元素、或打印其所有元素。

流管道的计算是惰性的：直到调用 Terminal 操作时才开始计算，并且对完成 Terminal 操作不需要的数据元素永远不会计算。这种惰性的求值机制使得处理无限流成为可能。请注意，没有 Terminal 操作的流管道是无动作的，因此不要忘记包含一个 Terminal 操作。

流 API 是流畅的：它被设计成允许使用链式调用将组成管道的所有调用写到单个表达式中。实际上，可以将多个管道链接到一个表达式中。

默认情况下，流管道按顺序运行。让管道并行执行与在管道中的任何流上调用并行方法一样简单，但是这样做不一定合适（[Item-48](../Chapter-7/Chapter-7-Item-48-Use-caution-when-making-streams-parallel)）。

流 API 非常通用，实际上任何计算都可以使用流来执行，但这并不意味着你就应该这样做。如果使用得当，流可以使程序更短、更清晰；如果使用不当，它们会使程序难以读取和维护。对于何时使用流没有硬性的规则，但是有一些启发式的规则。

考虑下面的程序，它从字典文件中读取单词并打印所有大小满足用户指定最小值的变位组。回想一下，如果两个单词以不同的顺序由相同的字母组成，那么它们就是字谜。该程序从用户指定的字典文件中读取每个单词，并将这些单词放入一个 Map 中。Map 的键是按字母顺序排列的单词，因此「staple」的键是「aelpst」，而「petals」的键也是「aelpst」：这两个单词是字谜，所有的字谜都有相同的字母排列形式（有时称为字母图）。Map 的值是一个列表，其中包含共享按字母顺序排列的表单的所有单词。在字典被处理之后，每个列表都是一个完整的字谜组。然后，该程序遍历 Map 的 values() 视图，并打印大小满足阈值的每个列表：

```
// Prints all large anagram groups in a dictionary iteratively
public class Anagrams {
    public static void main(String[] args) throws IOException {
        File dictionary = new File(args[0]);
        int minGroupSize = Integer.parseInt(args[1]);
        Map<String, Set<String>> groups = new HashMap<>();
        try (Scanner s = new Scanner(dictionary)) {
            while (s.hasNext()) {
                String word = s.next();
                groups.computeIfAbsent(alphabetize(word),(unused) -> new TreeSet<>()).add(word);
            }
        }
        for (Set<String> group : groups.values())
        if (group.size() >= minGroupSize)
            System.out.println(group.size() + ": " + group);
    }

    private static String alphabetize(String s) {
        char[] a = s.toCharArray();
        Arrays.sort(a);
        return new String(a);
    }
}
```

这个程序中的一个步骤值得注意。将每个单词插入到 Map 中（以粗体显示）使用 computeIfAbsent 方法，该方法是在 Java 8 中添加的。此方法在 Map 中查找键：如果键存在，则该方法仅返回与其关联的值。若不存在，则该方法通过将给定的函数对象应用于键来计算一个值，将该值与键关联，并返回计算的值。computeIfAbsent 方法简化了将多个值与每个键关联的 Map 的实现。

现在考虑下面的程序，它解决了相同的问题，但是大量使用了流。注意，除了打开字典文件的代码之外，整个程序都包含在一个表达式中。在单独的表达式中打开字典的唯一原因是允许使用 `try with-resources` 语句，该语句确保字典文件是关闭的：

```
// Overuse of streams - don't do this!
public class Anagrams {
    public static void main(String[] args) throws IOException {
        Path dictionary = Paths.get(args[0]);
        int minGroupSize = Integer.parseInt(args[1]);
        try (Stream<String> words = Files.lines(dictionary)) {
            words.collect(
            groupingBy(word -> word.chars().sorted()
            .collect(StringBuilder::new,(sb, c) -> sb.append((char) c),
            StringBuilder::append).toString()))
            .values().stream()
            .filter(group -> group.size() >= minGroupSize)
            .map(group -> group.size() + ": " + group)
            .forEach(System.out::println);
        }
    }
}
```

如果你发现这段代码难以阅读，不要担心；不单是你有这样的感觉。它虽然更短，但可读性也更差，特别是对于不擅长流使用的程序员来说。过度使用流会使得程序难以读取和维护。幸运的是，有一个折衷的办法。下面的程序解决了相同的问题，在不过度使用流的情况下使用流。结果，这个程序比原来的程序更短，也更清晰：

```
// Tasteful use of streams enhances clarity and conciseness
public class Anagrams {
    public static void main(String[] args) throws IOException {
        Path dictionary = Paths.get(args[0]);
        int minGroupSize = Integer.parseInt(args[1]);
        try (Stream<String> words = Files.lines(dictionary)) {
            words.collect(groupingBy(word -> alphabetize(word)))
            .values().stream()
            .filter(group -> group.size() >= minGroupSize)
            .forEach(g -> System.out.println(g.size() + ": " + g));
        }
    }
    // alphabetize method is the same as in original version
}
```

即使你以前很少接触流，这个程序也不难理解。它在带有资源的 try 块中打开字典文件，获得由文件中所有行组成的流。流变量名为 words，表示流中的每个元素都是一个单词。此流上的管道没有 Intermediate 操作；它的 Terminal 操作将所有单词收集到一个 Map 中，该 Map 按字母顺序将单词分组（[Item-46](../Chapter-7/Chapter-7-Item-46-Prefer-side-effect-free-functions-in-streams)）。这与在程序的前两个版本中构造的 Map 完全相同。然后在 Map 的 values() 视图上打开一个新的 `Stream<List<String>>`。这个流中的元素当然是字谜组。对流进行过滤，以便忽略所有大小小于 minGroupSize 的组，最后，Terminal 操作 forEach 打印其余组。

注意，lambda 表达式参数名称是经过仔细选择的。参数 g 实际上应该命名为 group，但是生成的代码行对于本书排版来说太宽了。**在没有显式类型的情况下，lambda 表达式参数的谨慎命名对于流管道的可读性至关重要。**

还要注意，单词的字母化是在一个单独的字母化方法中完成的。这通过为操作提供一个名称并将实现细节排除在主程序之外，从而增强了可读性。**在流管道中使用 helper 方法比在迭代代码中更重要**，因为管道缺少显式类型信息和命名的临时变量。

本来可以重新实现字母顺支持基本字符流（这并不意味着 Java 应该支持字符流；这样做是不可行的）。要演示使用流处理 char 值的危害，请考虑以下代码：

```
"Hello world!".chars().forEach(System.out::print);
```

你可能希望它打印 Hello world!，但如果运行它，你会发现它打印 721011081081113211911111410810033。这是因为 `"Hello world!".chars()` 返回的流元素不是 char 值，而是 int 值，因此调用了 print 的 int 重载。无可否认，一个名为 chars 的方法返回一个 int 值流是令人困惑的。你可以通过强制调用正确的重载来修复程序：

```
"Hello world!".chars().forEach(x -> System.out.print((char) x));
```

但理想情况下，你应该避免序方法来使用流，但是基于流的字母顺序方法就不那么清晰了，更难于正确地编写，而且可能会更慢。这些缺陷是由于 Java 不使用流来处理 char 值。当你开始使用流时，你可能会有将所有循环转换为流的冲动，但是要抵制这种冲动。虽然这是可实施的，但它可能会损害代码库的可读性和可维护性。通常，即使是中等复杂的任务，也最好使用流和迭代的某种组合来完成，如上面的 Anagrams 程序所示。因此，**重构现有代码或是在新代码中，都应该在合适的场景使用流。**

如本项中的程序所示，流管道使用函数对象（通常是 lambda 表达式或方法引用）表示重复计算，而迭代代码使用代码块表示重复计算。有些事情你可以对代码块做，而你不能对函数对象做：

- 从代码块中，可以读取或修改作用域中的任何局部变量；在 lambda 表达式中，只能读取 final 或有效的 final 变量 [JLS 4.12.4]，不能修改任何局部变量。

- 从代码块中，可以从封闭方法返回、中断或继续封闭循环，或抛出声明要抛出的任何已检查异常；在 lambda 表达式中，你不能做这些事情。

如果使用这些技术最好地表达计算，那么它可能不适合流。相反，流使做一些事情变得非常容易：

- 元素序列的一致变换

- 过滤元素序列

- 使用单个操作组合元素序列（例如添加它们、连接它们或计算它们的最小值）

- 将元素序列累积到一个集合中，可能是按某个公共属性对它们进行分组

- 在元素序列中搜索满足某些条件的元素

如果使用这些技术能够最好地表达计算，那么它就是流的一个很好的使用场景。

使用流很难做到的一件事是从管道的多个阶段同时访问相应的元素：一旦你将一个值映射到另一个值，原始值就会丢失。一个解决方案是将每个值映射到包含原始值和新值的 pair 对象，但这不是一个令人满意的解决方案，特别是如果管道的多个阶段都需要 pair 对象的话。生成的代码混乱而冗长，这违背了流的主要目的。当它适用时，更好的解决方案是在需要访问早期阶段值时反转映射。

例如，让我们编写一个程序来打印前 20 个 Mersenne 素数。刷新你的记忆,一个 Mersenne 素数的数量是一个数字形式 2p − 1。如果 p 是素数，对应的 Mersenne 数可以是素数；如果是的话，这就是 Mersenne 素数。作为管道中的初始流，我们需要所有质数。这里有一个返回（无限）流的方法。我们假设已经使用静态导入来方便地访问 BigInteger 的静态成员：

```
static Stream<BigInteger> primes() {
    return Stream.iterate(TWO, BigInteger::nextProbablePrime);
}
```

方法的名称（素数）是描述流元素的复数名词。强烈推荐所有返回流的方法使用这种命名约定，因为它增强了流管道的可读性。该方法使用静态工厂 `Stream.iterate`，它接受两个参数：流中的第一个元素和一个函数，用于从前一个元素生成流中的下一个元素。下面是打印前 20 个 Mersenne 素数的程序：

```
public static void main(String[] args) {
    primes().map(p -> TWO.pow(p.intValueExact()).subtract(ONE))
    .filter(mersenne -> mersenne.isProbablePrime(50))
    .limit(20)
    .forEach(System.out::println);
}
```

这个程序是上述散文描述的一个简单编码：它从素数开始，计算相应的 Mersenne 数，过滤掉除素数以外的所有素数（魔法数字 50 控制概率素数测试），将结果流限制为 20 个元素，并打印出来。

现在假设我们想要在每个 Mersenne 素数之前加上它的指数 (p)，这个值只在初始流中存在，因此在输出结果的终端操作中是不可访问的。幸运的是，通过对第一个中间操作中发生的映射求逆，可以很容易地计算出 Mersenne 数的指数。指数仅仅是二进制表示的比特数，因此这个终端操作产生了想要的结果：

```
.forEach(mp -> System.out.println(mp.bitLength() + ": " + mp));
```

在许多任务中，使用流还是迭代并不明显。例如，考虑初始化一副新纸牌的任务。假设 Card 是一个不可变的值类，它封装了 Rank 和 Suit，它们都是 enum 类型。此任务代表需要计算可从两个集合中选择的所有元素对的任何任务。数学家称之为这两个集合的笛卡尔积。下面是一个嵌套 for-each 循环的迭代实现，你应该非常熟悉它：

```
// Iterative Cartesian product computation
private static List<Card> newDeck() {
    List<Card> result = new ArrayList<>();
    for (Suit suit : Suit.values())
    for (Rank rank : Rank.values())
    result.add(new Card(suit, rank));
    return result;
}
```

这是一个基于流的实现，它使用了中间操作 flatMap。此操作将流中的每个元素映射到流，然后将所有这些新流连接到单个流中（或将其扁平化）。注意，这个实现包含一个嵌套 lambda 表达式，用粗体显示:

```
// Stream-based Cartesian product computation
private static List<Card> newDeck() {
    return Stream.of(Suit.values())
    .flatMap(suit ->Stream.of(Rank.values())
    .map(rank -> new Card(suit, rank)))
    .collect(toList());
}
```

两个版本的 newDeck 哪个更好？这可以归结为个人偏好和编程环境。第一个版本更简单，可能感觉更自然。大部分 Java 程序员将能够理解和维护它，但是一些程序员将对第二个（基于流的）版本感到更舒服。如果你相当精通流和函数式编程，那么它会更简洁，也不会太难理解。如果你不确定你更喜欢哪个版本，迭代版本可能是更安全的选择。如果你更喜欢流版本，并且相信与代码一起工作的其他程序员也会分享你的偏好，那么你应该使用它。

总之，有些任务最好使用流来完成，有些任务最好使用迭代来完成。许多任务最好通过结合这两种方法来完成。对于选择任务使用哪种方法，没有硬性的规则，但是有一些有用的启发。在许多情况下，使用哪种方法是清楚的；在某些情况下很难决定。如果你不确定一个任务是通过流还是通过迭代更好地完成，那么同时尝试这两种方法，看看哪一种更有效。
