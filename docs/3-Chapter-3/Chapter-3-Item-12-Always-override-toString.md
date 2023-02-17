# 第十二条: 始终覆盖 toString 方法

虽然 Object 提供 toString 方法的实现，但它返回的字符串通常不是类的用户希望看到的。它由后跟「at」符号（@）的类名和 hash 代码的无符号十六进制表示（例如 PhoneNumber@163b91）组成。toString 的通用约定是这么描述的，返回的字符串应该是「简洁但信息丰富的表示，易于阅读」。虽然有人认为 PhoneNumber@163b91 简洁易懂，但与 707-867-5309 相比，它的信息量并不大。toString 约定接着描述，「建议所有子类覆盖此方法。」好建议，确实！

虽然它不如遵守 equals 和 hashCode 约定（[Item-10](/Chapter-3/Chapter-3-Item-10-Obey-the-general-contract-when-overriding-equals.md) 和 [Item-11](/Chapter-3/Chapter-3-Item-11-Always-override-hashCode-when-you-override-equals.md)）那么重要，但是 **提供一个好的 toString 实现（能）使类更易于使用，使用该类的系统（也）更易于调试。** 当对象被传递给 println、printf、字符串连接操作符或断言或由调试器打印时，将自动调用 toString 方法。即使你从来没有调用 toString 对象，其他人也可能（使用）。例如，有对象引用的组件可以在日志错误消息中包含对象的字符串表示。如果你未能覆盖 toString，则该消息可能完全无用。

如果你已经为 PhoneNumber 提供了一个好的 toString 方法，那么生成一个有用的诊断消息就像这样简单：

```
System.out.println("Failed to connect to " + phoneNumber);
```

无论你是否覆盖 toString，程序员都会以这种方式生成诊断消息，但是除非你（覆盖 toString），否则这些消息不会有用。提供好的 toString 方法的好处不仅仅是将类的实例扩展到包含对这些实例的引用的对象，特别是集合。在打印 map 时，你更愿意看到哪个，{Jenny=PhoneNumber@163b91} 还是 {Jenny=707-867-5309}？

**当实际使用时，toString 方法应该返回对象中包含的所有有趣信息，** 如电话号码示例所示。如果对象很大，或者包含不利于字符串表示的状态，那么这种方法是不切实际的。在这种情况下，toString 应该返回一个摘要，例如曼哈顿住宅电话目录（1487536 号清单）或 Thread[main,5,main]。理想情况下，字符串应该是不言自明的。（线程示例未能通过此测试。）如果没有在字符串表示中包含所有对象的有趣信息，那么一个特别恼人的惩罚就是测试失败报告，如下所示：

```
Assertion failure: expected {abc, 123}, but was {abc, 123}.
```

在实现 toString 方法时，你必须做的一个重要决定是是否在文档中指定返回值的格式。建议你针对值类（如电话号码或矩阵）这样做。指定格式的优点是，它可以作为对象的标准的、明确的、人类可读的表示。这种表示可以用于输入和输出，也可以用于持久的人类可读数据对象，比如 CSV 文件。如果指定了格式，提供一个匹配的静态工厂或构造函数通常是一个好主意，这样程序员就可以轻松地在对象及其字符串表示之间来回转换。Java 库中的许多值类都采用这种方法，包括 BigInteger、BigDecimal 和大多数包装类。

指定 toString 返回值的格式的缺点是，一旦指定了它，就会终生使用它，假设你的类被广泛使用。程序员将编写代码来解析表示、生成表示并将其嵌入持久数据中。如果你在将来的版本中更改了表示形式，你将破坏它们的代码和数据，它们将发出大量的消息。通过选择不指定格式，你可以保留在后续版本中添加信息或改进格式的灵活性。

**无论你是否决定指定格式，你都应该清楚地记录你的意图。** 如果指定了格式，则应该精确地指定格式。例如，这里有一个 toString 方法用于[Item-11](/Chapter-3/Chapter-3-Item-11-Always-override-hashCode-when-you-override-equals.md)中的 PhoneNumber 类：

```
/**
* Returns the string representation of this phone number.
* The string consists of twelve characters whose format is
* "XXX-YYY-ZZZZ", where XXX is the area code, YYY is the
* prefix, and ZZZZ is the line number. Each of the capital
* letters represents a single decimal digit.
**
If any of the three parts of this phone number is too small
* to fill up its field, the field is padded with leading zeros.
* For example, if the value of the line number is 123, the last
* four characters of the string representation will be "0123".
*/
@Override
public String toString() {
    return String.format("%03d-%03d-%04d", areaCode, prefix, lineNum);
}
```

如果你决定不指定一种格式，文档注释应该如下所示：

```
/**
* Returns a brief description of this potion. The exact details
* of the representation are unspecified and subject to change,
* but the following may be regarded as typical:
**
"[Potion #9: type=love, smell=turpentine, look=india ink]"
*/
@Override
public String toString() { ... }
```

在阅读了这篇文档注释之后，当格式被更改时，生成依赖于格式细节的代码或持久数据的程序员将只能怪他们自己。

无论你是否指定了格式，都要 **提供对 toString 返回值中包含的信息的程序性访问。** 例如，PhoneNumber 类应该包含区域代码、前缀和行号的访问器。如果做不到这一点，就会迫使需要这些信息的程序员解析字符串。除了降低性能和使程序员不必要的工作之外，这个过程很容易出错，并且会导致脆弱的系统在你更改格式时崩溃。由于没有提供访问器，你可以将字符串格式转换为事实上的 API，即使你已经指定了它可能会发生更改。

在静态实用程序类中编写 toString 方法是没有意义的（[Item-4](/Chapter-2/Chapter-2-Item-4-Enforce-noninstantiability-with-a-private-constructor.md)），在大多数 enum 类型中也不应该编写 toString 方法（[Item-34](/Chapter-6/Chapter-6-Item-34-Use-enums-instead-of-int-constants.md)），因为 Java 为你提供了一个非常好的方法。但是，你应该在任何抽象类中编写 toString 方法，该类的子类共享公共的字符串表示形式。例如，大多数集合实现上的 toString 方法都继承自抽象集合类。

谷歌的开放源码自动值工具（在 [Item-10](/Chapter-3/Chapter-3-Item-10-Obey-the-general-contract-when-overriding-equals.md) 中讨论）将为你生成 toString 方法，大多数 IDE 也是如此。这些方法可以很好地告诉你每个字段的内容，但并不专门针对类的含义。因此，例如，对于 PhoneNumber 类使用自动生成的 toString 方法是不合适的（因为电话号码具有标准的字符串表示形式），但是对于 Potion 类来说它是完全可以接受的。也就是说，一个自动生成的 toString 方法要比从对象继承的方法好得多，对象继承的方法不会告诉你对象的值。

回顾一下，在你编写的每个实例化类中覆盖对象的 toString 实现，除非超类已经这样做了。它使类更易于使用，并有助于调试。toString 方法应该以美观的格式返回对象的简明、有用的描述。