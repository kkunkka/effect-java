# 第三十六: 用 EnumSet 替代位字段

如果枚举类型的元素主要在 Set 中使用，传统上使用 int 枚举模式（[Item-34](../Chapter-6/Chapter-6-Item-34-Use-enums-instead-of-int-constants)），通过不同的 2 平方数为每个常量赋值：

```
// Bit field enumeration constants - OBSOLETE!
public class Text {
    public static final int STYLE_BOLD = 1 << 0; // 1
    public static final int STYLE_ITALIC = 1 << 1; // 2
    public static final int STYLE_UNDERLINE = 1 << 2; // 4
    public static final int STYLE_STRIKETHROUGH = 1 << 3; // 8
    // Parameter is bitwise OR of zero or more STYLE_ constants
    public void applyStyles(int styles) { ... }
}
```

这种表示方式称为位字段，允许你使用位运算的 OR 操作将几个常量组合成一个 Set：

```
text.applyStyles(STYLE_BOLD | STYLE_ITALIC);
```

位字段表示方式允许使用位运算高效地执行 Set 操作，如并集和交集。但是位字段具有 int 枚举常量所有缺点，甚至更多。当位字段被打印为数字时，它比简单的 int 枚举常量更难理解。没有一种简单的方法可以遍历由位字段表示的所有元素。最后，你必须预测在编写 API 时需要的最大位数，并相应地为位字段（通常是 int 或 long）选择一种类型。一旦选择了一种类型，在不更改 API 的情况下，不能超过它的宽度（32 或 64 位）。

一些使用枚举而不是 int 常量的程序员在需要传递常量集时仍然坚持使用位字段。没有理由这样做，因为存在更好的选择。`java.util` 包提供 EnumSet 类来有效地表示从单个枚举类型中提取的值集。这个类实现了 Set 接口，提供了所有其他 Set 实现所具有的丰富性、类型安全性和互操作性。但在内部，每个 EnumSet 都表示为一个位向量。如果底层枚举类型有 64 个或更少的元素（大多数都是），则整个 EnumSet 用一个 long 表示，因此其性能与位字段的性能相当。批量操作（如 removeAll 和 retainAll）是使用逐位算法实现的，就像手动处理位字段一样。但是，你可以避免因手工修改导致产生不良代码和潜在错误：EnumSet 为你完成了这些繁重的工作。

当之前的示例修改为使用枚举和 EnumSet 而不是位字段时。它更短，更清晰，更安全：

```
// EnumSet - a modern replacement for bit fields
public class Text {
    public enum Style { BOLD, ITALIC, UNDERLINE, STRIKETHROUGH }
    // Any Set could be passed in, but EnumSet is clearly best
    public void applyStyles(Set<Style> styles) { ... }
}
```

下面是将 EnumSet 实例传递给 applyStyles 方法的客户端代码。EnumSet 类提供了一组丰富的静态工厂，可以方便地创建 Set，下面的代码演示了其中的一个：

```
text.applyStyles(EnumSet.of(Style.BOLD, Style.ITALIC));
```

请注意，applyStyles 方法采用 `Set<Style>` 而不是 `EnumSet<Style>`。虽然似乎所有客户端都可能将 EnumSet 传递给该方法，但通常较好的做法是接受接口类型而不是实现类型（[Item-64](../Chapter-9/Chapter-9-Item-64-Refer-to-objects-by-their-interfaces)）。这允许特殊的客户端传入其他 Set 实现的可能性。

总之，**因为枚举类型将在 Set 中使用，没有理由用位字段表示它。** EnumSet 类结合了位字段的简洁性和性能，以及 [Item-34](../Chapter-6/Chapter-6-Item-34-Use-enums-instead-of-int-constants) 中描述的枚举类型的许多优点。EnumSet 的一个真正的缺点是，从 Java 9 开始，它不能创建不可变的 EnumSet，在未来发布的版本中可能会纠正这一点。同时，可以用 `Collections.unmodifiableSet` 包装 EnumSet，但简洁性和性能将受到影响。
