# 第十一节:  当覆盖 equals 方法时，总要覆盖 hashCode 方法

**在覆盖了 equals 方法的类中，必须覆盖 hashCode 方法。** 如果你没有这样做，该类将违反 hashCode 方法的一般约定，这将阻止该类在 HashMap 和 HashSet 等集合中正常运行。以下是根据 Object 规范修改的约定：

- 应用程序执行期间对对象重复调用 hashCode 方法时，它必须一致地返回相同的值，前提是不对 equals 方法中用于比较的信息进行修改。这个值不需要在应用程序的不同执行之间保持一致。

- 如果根据 `equals(Object)` 方法判断出两个对象是相等的，那么在两个对象上调用 hashCode 方法必须产生相同的整数结果。

- 如果根据 `equals(Object)` 方法判断出两个对象不相等，则不需要在每个对象上调用 hashCode 方法时必须产生不同的结果。但是，程序员应该知道，为不相等的对象生成不同的结果可能会提高散列表的性能。

**当你无法覆盖 hashCode 方法时，将违反第二个关键条款：相等的对象必须具有相等的散列码。** 根据类的 equals 方法，两个不同的实例在逻辑上可能是相等的，但是对于对象的 hashCode 方法来说，它们只是两个没有共同之处的对象。因此，Object 的 hashCode 方法返回两个看似随机的数字，而不是约定要求的两个相等的数字。例如，假设你尝试使用[Item-10](/Chapter-3/Chapter-3-Item-10-Obey-the-general-contract-when-overriding-equals.md)中的 PhoneNumber 类实例作为 HashMap 中的键：

```
Map<PhoneNumber, String> m = new HashMap<>();
m.put(new PhoneNumber(707, 867, 5309), "Jenny");
```

此时，你可能期望 `m.get(new PhoneNumber(707, 867,5309))` 返回「Jenny」，但是它返回 null。注意，这里涉及到两个 PhoneNumber 实例：一个用于插入到 HashMap 中，另一个相等的实例（被试图）用于检索。由于 PhoneNumber 类未能覆盖 hashCode 方法，导致两个相等的实例具有不相等的散列码，这违反了 hashCode 方法约定。因此，get 方法查找电话号码的散列桶可能会与 put 方法存储电话号码的散列桶不同。即使这两个实例碰巧分配在同一个散列桶上，get 方法几乎肯定会返回 null，因为 HashMap 有一个优化，它缓存每个条目相关联的散列码，如果散列码不匹配，就不会检查对象是否相等。

解决这个问题就像为 PhoneNumber 编写一个正确的 hashCode 方法一样简单。那么 hashCode 方法应该是什么样的呢？写一个反面例子很容易。例如，以下方法是合法的，但是不应该被使用：

```
// The worst possible legal hashCode implementation - never use!
@Override
public int hashCode() { return 42; }
```

它是合法的，因为它确保了相等的对象具有相同的散列码。同时它也很糟糕，因为它使每个对象都有相同的散列码。因此，每个对象都分配到同一个桶中，散列表退化为链表。这样，原本应该在线性阶 `O(n)` 运行的程序将在平方阶 `O(n^2)` 运行。对于大型散列表，这是工作和不工作的区别。

一个好的散列算法倾向于为不相等的实例生成不相等的散列码。这正是 hashCode 方法约定第三部分的含义。理想情况下，一个散列算法应该在所有 int 值上均匀合理分布所有不相等实例集合。实现这个理想是很困难的。幸运的是，实现一个类似的并不太难。这里有一个简单的方式：

声明一个名为 result 的 int 变量，并将其初始化为对象中第一个重要字段的散列码 c，如步骤 2.a 中计算的那样。（回想一下 [Item-10](/Chapter-3/Chapter-3-Item-10-Obey-the-general-contract-when-overriding-equals.md) 中的重要字段会对比较产生影响）


对象中剩余的重要字段 f，执行以下操作：

- 为字段计算一个整数散列码 c：

    1. 如果字段是基本数据类型，计算 `Type.hashCode(f)`，其中 type 是与 f 类型对应的包装类。

    2. 如果字段是对象引用，并且该类的 equals 方法通过递归调用 equals 方法来比较字段，则递归调用字段上的 hashCode 方法。如果需要更复杂的比较，则为该字段计算一个「canonical representation」，并在 canonical representation 上调用 hashCode 方法。如果字段的值为空，则使用 0（或其他常数，但 0 是惯用的）。

    3. 如果字段是一个数组，则将其每个重要元素都视为一个单独的字段。也就是说，通过递归地应用这些规则计算每个重要元素的散列码，并将每个步骤的值组合起来。如果数组中没有重要元素，则使用常量，最好不是 0。如果所有元素都很重要，那么使用 `Arrays.hashCode`。

- 将步骤 2.a 中计算的散列码 c 合并到 result 变量，如下所示：

```
result = 31 * result + c;
```

- Return result.

返回 result 变量。


当你完成了 hashCode 方法的编写之后，问问自己现在相同的实例是否具有相同的散列码。编写单元测试来验证你的直觉（除非你使用 AutoValue 生成你的 equals 方法和 hashCode 方法，在这种情况下你可以安全地省略这些测试）。如果相同的实例有不相等的散列码，找出原因并修复问题。

可以从散列码计算中排除派生字段。换句话说，你可以忽略任何可以从包含的字段计算其值的字段。你必须排除不用 `equals` 比较的任何字段，否则你可能会违反 hashCode 方法约定的第二个条款。

在步骤 2.b 中使用的乘法将使结果取决于字段的顺序，如果类有多个相似的字段，则会产生一个更好的散列算法。例如，如果字符串散列算法中省略了乘法，那么所有的字母顺序都有相同的散列码。选择 31 是因为它是奇素数。如果是偶数，乘法运算就会溢出，信息就会丢失，因为乘法运算等同于移位。使用素数的好处不太明显，但它是传统用法。31 有一个很好的特性，可以用移位和减法来代替乘法，从而在某些体系结构上获得更好的性能：`31 * i == (i <<5) – i`。现代虚拟机自动进行这种优化。

让我们将前面的方法应用到 PhoneNumber 类：

```
// Typical hashCode method
@Override
public int hashCode() {
    int result = Short.hashCode(areaCode);
    result = 31 * result + Short.hashCode(prefix);
    result = 31 * result + Short.hashCode(lineNum);
    return result;
}
```

因为这个方法返回一个简单的确定的计算结果，它的唯一输入是 PhoneNumber 实例中的三个重要字段，所以很明显，相等的 PhoneNumber 实例具有相等的散列码。实际上，这个方法是 PhoneNumber 的一个非常好的 hashCode 方法实现，与 Java 库中的 hashCode 方法实现相当。它很简单，速度也相当快，并且合理地将不相等的电话号码分散到不同的散列桶中。

虽然本条目中的方法产生了相当不错的散列算法，但它们并不是最先进的。它们的质量可与 Java 库的值类型中的散列算法相媲美，对于大多数用途来说都是足够的。如果你确实需要不太可能产生冲突的散列算法，请参阅 Guava 的 com.google.common.hash.Hashing [Guava]。

Objects 类有一个静态方法，它接受任意数量的对象并返回它们的散列码。这个名为 `hash` 的方法允许你编写只有一行代码的 hashCode 方法，这些方法的质量可以与本条目中提供的编写方法媲美。不幸的是，它们运行得更慢，因为它们需要创建数组来传递可变数量的参数，如果任何参数是原始类型的，则需要进行装箱和拆箱。推荐只在性能不重要的情况下使用这种散列算法。下面是使用这种技术编写的 PhoneNumber 的散列算法：

```
// One-line hashCode method - mediocre performance
@Override
public int hashCode() {
    return Objects.hash(lineNum, prefix, areaCode);
}
```

如果一个类是不可变的，并且计算散列码的成本非常高，那么你可以考虑在对象中缓存散列码，而不是在每次请求时重新计算它。如果你认为这种类型的大多数对象都将用作散列键，那么你应该在创建实例时计算散列码。否则，你可以选择在第一次调用 hashCode 方法时延迟初始化散列码。在一个延迟初始化的字段（[Item-83](/Chapter-11/Chapter-11-Item-83-Use-lazy-initialization-judiciously.md)）的情况下，需要注意以确保该类仍然是线程安全的。我们的 PhoneNumber 类不值得进行这种处理，但只是为了向你展示它是如何实现的，如下所示。注意，散列字段的初始值（在本例中为 0）不应该是通常创建的实例的散列码：

```
// hashCode method with lazily initialized cached hash code
private int hashCode; // Automatically initialized to 0
@Override
public int hashCode() {
    int result = hashCode;

    if (result == 0) {
        result = Short.hashCode(areaCode);
        result = 31 * result + Short.hashCode(prefix);
        result = 31 * result + Short.hashCode(lineNum);
        hashCode = result;
    }

    return result;
}
```

**不要试图从散列码计算中排除重要字段，以提高性能。** 虽然得到的散列算法可能运行得更快，但其糟糕的质量可能会将散列表的性能降低到无法使用的程度。特别是，散列算法可能会遇到大量实例，这些实例在你选择忽略的不同区域。如果发生这种情况，散列算法将把所有这些实例映射很少一部分散列码，使得原本应该在线性阶 `O(n)` 运行的程序将在平方阶 `O(n^2)` 运行。

这不仅仅是一个理论问题。在 Java 2 之前，字符串散列算法在字符串中，以第一个字符开始，最多使用 16 个字符。对于大量且分层次的集合（如 url），该函数完全展示了前面描述的病态行为。

**不要为 hashCode 返回的值提供详细的规范，这样客户端就不能理所应当的依赖它。这（也）给了你更改它的余地。** Java 库中的许多类，例如 String 和 Integer，都将 hashCode 方法返回的确切值指定为实例值的函数。这不是一个好主意，而是一个我们不得不面对的错误：它阻碍了在未来版本中提高散列算法的能力。如果你保留了未指定的细节，并且在散列算法中发现了缺陷，或者发现了更好的散列算法，那么你可以在后续版本中更改它。

总之，每次覆盖 equals 方法时都必须覆盖 hashCode 方法，否则程序将无法正确运行。你的 hashCode 方法必须遵守 Object 中指定的通用约定，并且必须合理地将不相等的散列码分配给不相等的实例。这很容易实现，如果有点枯燥，可使用第 51 页的方法。如 [Item-10](/Chapter-3/Chapter-3-Item-10-Obey-the-general-contract-when-overriding-equals.md) 所述，AutoValue 框架提供了一种能很好的替代手动编写 equals 方法和 hashCode 方法的功能，IDE 也提供了这种功能。
