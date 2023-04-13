# 第八十八节: 防御性地编写 readObject 方法

[Item-50](../Chapter-8/Chapter-8-Item-50-Make-defensive-copies-when-needed) 包含一个具有可变私有 Date 字段的不可变日期范围类。该类通过在构造函数和访问器中防御性地复制 Date 对象，不遗余力地保持其不变性和不可变性。它是这样的：

```
// Immutable class that uses defensive copying
public final class Period {
    private final Date start;
    private final Date end;

    /**
    * @param start the beginning of the period
    * @param end the end of the period; must not precede start
    * @throws IllegalArgumentException if start is after end
    * @throws NullPointerException if start or end is null
    */
    public Period(Date start, Date end) {
        this.start = new Date(start.getTime());
        this.end = new Date(end.getTime());
        if (this.start.compareTo(this.end) > 0)
            throw new IllegalArgumentException(start + " after " + end);
    }

    public Date start () { return new Date(start.getTime()); }

    public Date end () { return new Date(end.getTime()); }

    public String toString() { return start + " - " + end; }

    ... // Remainder omitted
}
```

假设你决定让这个类可序列化。由于 Period 对象的物理表示精确地反映了它的逻辑数据内容，所以使用默认的序列化形式是合理的（[Item-87](../Chapter-12/Chapter-12-Item-87-Consider-using-a-custom-serialized-form)）。因此，要使类可序列化，似乎只需将实现 Serializable 接口。但是，如果这样做，该类将不再保证它的临界不变量。

问题是 readObject 方法实际上是另一个公共构造函数，它与任何其他构造函数有相同的注意事项。如，构造函数必须检查其参数的有效性（[Item-49](../Chapter-8/Chapter-8-Item-49-Check-parameters-for-validity)）并在适当的地方制作防御性副本（[Item-50](../Chapter-8/Chapter-8-Item-50-Make-defensive-copies-when-needed)）一样，readObject 方法也必须这样做。如果 readObject 方法没有做到这两件事中的任何一件，那么攻击者就很容易违反类的不变性。

不严格地说，readObject 是一个构造函数，它唯一的参数是字节流。在正常使用中，字节流是通过序列化一个正常构造的实例生成的。当 readObject 呈现一个字节流时，问题就出现了，这个字节流是人为构造的，用来生成一个违反类不变性的对象。这样的字节流可用于创建一个不可思议的对象，而该对象不能使用普通构造函数创建。

假设我们只是简单地让 Period 实现 Serializable 接口。然后，这个有问题的程序将生成一个 Period 实例，其结束比起始时间还要早。对其高位位设置的字节值进行强制转换，这是由于 Java 缺少字节字面值，再加上让字节类型签名的错误决定导致的：

```
public class BogusPeriod {
// Byte stream couldn't have come from a real Period instance!
    private static final byte[] serializedForm = {
        (byte)0xac, (byte)0xed, 0x00, 0x05, 0x73, 0x72, 0x00, 0x06,
        0x50, 0x65, 0x72, 0x69, 0x6f, 0x64, 0x40, 0x7e, (byte)0xf8,
        0x2b, 0x4f, 0x46, (byte)0xc0, (byte)0xf4, 0x02, 0x00, 0x02,
        0x4c, 0x00, 0x03, 0x65, 0x6e, 0x64, 0x74, 0x00, 0x10, 0x4c,
        0x6a, 0x61, 0x76, 0x61, 0x2f, 0x75, 0x74, 0x69, 0x6c, 0x2f,
        0x44, 0x61, 0x74, 0x65, 0x3b, 0x4c, 0x00, 0x05, 0x73, 0x74,
        0x61, 0x72, 0x74, 0x71, 0x00, 0x7e, 0x00, 0x01, 0x78, 0x70,
        0x73, 0x72, 0x00, 0x0e, 0x6a, 0x61, 0x76, 0x61, 0x2e, 0x75,
        0x74, 0x69, 0x6c, 0x2e, 0x44, 0x61, 0x74, 0x65, 0x68, 0x6a,
        (byte)0x81, 0x01, 0x4b, 0x59, 0x74, 0x19, 0x03, 0x00, 0x00,
        0x78, 0x70, 0x77, 0x08, 0x00, 0x00, 0x00, 0x66, (byte)0xdf,
        0x6e, 0x1e, 0x00, 0x78, 0x73, 0x71, 0x00, 0x7e, 0x00, 0x03,
        0x77, 0x08, 0x00, 0x00, 0x00, (byte)0xd5, 0x17, 0x69, 0x22,
        0x00, 0x78
    };

    public static void main(String[] args) {
        Period p = (Period) deserialize(serializedForm);
        System.out.println(p);
    }

    // Returns the object with the specified serialized form
    static Object deserialize(byte[] sf) {
        try {
            return new ObjectInputStream(new ByteArrayInputStream(sf)).readObject();
        } catch (IOException | ClassNotFoundException e) {
            throw new IllegalArgumentException(e);
        }
    }
}
```

用于初始化 serializedForm 的字节数组文本是通过序列化一个普通 Period 实例并手工编辑得到的字节流生成的。流的细节对示例并不重要，但是如果你感兴趣，可以在《JavaTM Object Serialization Specification》[serialization, 6]中查到序列化字节流的格式描述。如果你运行这个程序，它将打印 `Fri Jan 01 12:00:00 PST 1999 - Sun Jan 01 12:00:00 PST 1984`。只需声明 Period 可序列化，就可以创建一个违反其类不变性的对象。

要解决此问题，请为 Period 提供一个 readObject 方法，该方法调用 defaultReadObject，然后检查反序列化对象的有效性。如果有效性检查失败，readObject 方法抛出 InvalidObjectException，阻止反序列化完成：

```
// readObject method with validity checking - insufficient!
private void readObject(ObjectInputStream s) throws IOException, ClassNotFoundException {
    s.defaultReadObject();
    // Check that our invariants are satisfied
    if (start.compareTo(end) > 0)
        throw new InvalidObjectException(start +" after "+ end);
}
```

虽然这可以防止攻击者创建无效的 Period 实例，但还有一个更微妙的问题仍然潜伏着。可以通过字节流来创建一个可变的 Period 实例，该字节流以一个有效的 Period 实例开始，然后向 Period 实例内部的私有日期字段追加额外的引用。攻击者从 ObjectInputStream 中读取 Period 实例，然后读取附加到流中的「恶意对象引用」。这些引用使攻击者能够访问 Period 对象中的私有日期字段引用的对象。通过修改这些日期实例，攻击者可以修改 Period 实例。下面的类演示了这种攻击：

```
public class MutablePeriod {
    // A period instance
    public final Period period;

    // period's start field, to which we shouldn't have access
    public final Date start;

    // period's end field, to which we shouldn't have access
    public final Date end;

    public MutablePeriod() {
        try {
            ByteArrayOutputStream bos = new ByteArrayOutputStream();
            ObjectOutputStream out = new ObjectOutputStream(bos);

            // Serialize a valid Period instance
            out.writeObject(new Period(new Date(), new Date()));

            /*
            * Append rogue "previous object refs" for internal
            * Date fields in Period. For details, see "Java
            * Object Serialization Specification," Section 6.4.
            */
            byte[] ref = { 0x71, 0, 0x7e, 0, 5 }; // Ref #5
            bos.write(ref); // The start field
            ref[4] = 4; // Ref # 4
            bos.write(ref); // The end field

            // Deserialize Period and "stolen" Date references
            ObjectInputStream in = new ObjectInputStream(new ByteArrayInputStream(bos.toByteArray()));
            period = (Period) in.readObject();
            start = (Date) in.readObject();
            end = (Date) in.readObject();
        } catch (IOException | ClassNotFoundException e) {
            throw new AssertionError(e);
        }
    }
}
```

要查看攻击的实际效果，请运行以下程序：

```
public static void main(String[] args) {
    MutablePeriod mp = new MutablePeriod();
    Period p = mp.period;
    Date pEnd = mp.end;

    // Let's turn back the clock
    pEnd.setYear(78);
    System.out.println(p);

    // Bring back the 60s!
    pEnd.setYear(69);
    System.out.println(p);
}
```

在我的语言环境中，运行这个程序会产生以下输出：

```
Wed Nov 22 00:21:29 PST 2017 - Wed Nov 22 00:21:29 PST 1978
Wed Nov 22 00:21:29 PST 2017 - Sat Nov 22 00:21:29 PST 1969
```

虽然创建 Period 实例时保留了它的不变性，但是可以随意修改它的内部组件。一旦拥有一个可变的 Period 实例，攻击者可能会将实例传递给一个依赖于 Period 的不变性来保证其安全性的类，从而造成极大的危害。这并不是牵强附会的：有些类依赖于 String 的不变性来保证其安全。

问题的根源在于 Period 的 readObject 方法没有进行足够的防御性复制。**当对象被反序列化时，对任何客户端不能拥有的对象引用的字段进行防御性地复制至关重要。** 因此，对于每个可序列化的不可变类，如果它包含了私有的可变组件，那么在它的 readObjec 方法中，必须要对这些组件进行防御性地复制。下面的 readObject 方法足以保证周期的不变性，并保持其不变性：

```
// readObject method with defensive copying and validity checking
private void readObject(ObjectInputStream s) throws IOException, ClassNotFoundException {
    s.defaultReadObject();
    // Defensively copy our mutable components
    start = new Date(start.getTime());
    end = new Date(end.getTime());
    // Check that our invariants are satisfied
    if (start.compareTo(end) > 0)
        throw new InvalidObjectException(start +" after "+ end);
}
```

注意，防御副本是在有效性检查之前执行的，我们没有使用 Date 的 clone 方法来执行防御副本。这两个细节对于保护 Period 免受攻击是必要的(第50项)。还要注意，防御性复制不可能用于 final 字段。要使用 readObject 方法，必须使 start 和 end 字段非 final。这是不幸的，但却是权衡利弊后的方案。使用新的 readObject 方法，并从 start 和 end 字段中删除 final 修饰符，MutablePeriod 类将无效。上面的攻击程序现在生成这个输出：

```
Wed Nov 22 00:23:41 PST 2017 - Wed Nov 22 00:23:41 PST 2017
Wed Nov 22 00:23:41 PST 2017 - Wed Nov 22 00:23:41 PST 2017
```

下面是一个简单的测试，用于判断默认 readObject 方法是否可用于类：你是否愿意添加一个公共构造函数，该构造函数将对象中每个非 transient 字段的值作为参数，并在没有任何验证的情况下将值存储在字段中？如果没有，则必须提供 readObject 方法，并且它必须执行构造函数所需的所有有效性检查和防御性复制。或者，你可以使用序列化代理模式（[Item-90](../Chapter-12/Chapter-12-Item-90-Consider-serialization-proxies-instead-of-serialized-instances)）。强烈推荐使用这种模式，否则会在安全反序列化方面花费大量精力。

readObject 方法和构造函数之间还有一个相似之处，适用于非 final 序列化类。与构造函数一样，readObject 方法不能直接或间接调用可覆盖的方法（[Item-19](../Chapter-4/Chapter-4-Item-19-Design-and-document-for-inheritance-or-else-prohibit-it)）。如果违反了这条规则，并且涉及的方法被覆盖，则覆盖方法将在子类的状态反序列化之前运行。很可能导致程序失败 [Bloch05, Puzzle 91]。

总而言之，无论何时编写 readObject 方法，都要采用这样的思维方式，即编写一个公共构造函数，该构造函数必须生成一个有效的实例，而不管给定的是什么字节流。不要假设字节流表示实际的序列化实例。虽然本条目中的示例涉及使用默认序列化形式的类，但是所引发的所有问题都同样适用于具有自定义序列化形式的类。下面是编写 readObject 方法的指导原则：

- 对象引用字段必须保持私有的的类，应防御性地复制该字段中的每个对象。不可变类的可变组件属于这一类。

- 检查任何不变量，如果检查失败，则抛出 InvalidObjectException。检查动作应该跟在任何防御性复制之后。

- 如果必须在反序列化后验证整个对象图，那么使用 ObjectInputValidation 接口（在本书中没有讨论）。

- 不要直接或间接地调用类中任何可被覆盖的方法。

