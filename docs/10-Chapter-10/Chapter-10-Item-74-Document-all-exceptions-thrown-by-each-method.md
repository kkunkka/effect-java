# 第七十四节: 为每个方法记录会抛出的所有异常

描述方法抛出的异常，是该方法文档的重要部分。因此，花时间仔细记录每个方法抛出的所有异常是非常重要的（[Item-56](/Chapter-8/Chapter-8-Item-56-Write-doc-comments-for-all-exposed-API-elements.md)）。

**始终单独声明 checked 异常，并使用 Javadoc 的 `@throw` 标记精确记录每次抛出异常的条件**。如果一个方法抛出多个异常，不要使用快捷方式声明这些异常的超类。作为一个极端的例子，即不要在公共方法声明 `throws Exception`，或者更糟，甚至 `throws Throwable`。除了不能向方法的用户提供会抛出哪些异常的任何消息之外，这样的声明还极大地阻碍了方法的使用，因为它掩盖了在相同上下文中可能抛出的任何其他异常。这个建议的一个特例是 main 方法，它可以安全地声明 `throw Exception`，因为它只被 VM 调用。

虽然 Java 不要求程序员为方法声明会抛出的 unchecked 异常，但明智的做法是，应该像 checked 异常一样仔细地记录它们。unchecked 异常通常表示编程错误（[Item-70](/Chapter-10/Chapter-10-Item-70-Use-checked-exceptions-for-recoverable-conditions-and-runtime-exceptions-for-programming-errors.md)），让程序员熟悉他们可能犯的所有错误可以帮助他们避免犯这些错误。将方法可以抛出的 unchecked 异常形成良好文档，可以有效描述方法成功执行的先决条件。每个公共方法的文档都必须描述它的先决条件（[Item-56](/Chapter-8/Chapter-8-Item-56-Write-doc-comments-for-all-exposed-API-elements.md)），记录它的 unchecked 异常是满足这个需求的最佳方法。

特别重要的是，接口中的方法要记录它们可能抛出的 unchecked 异常。此文档构成接口通用约定的一部分，并指明接口的多个实现之间应该遵守的公共行为。

**使用 Javadoc 的 `@throw` 标记记录方法会抛出的每个异常，但是不要对 unchecked 异常使用 throws 关键字。** 让使用你的 API 的程序员知道哪些异常是 checked 异常，哪些是 unchecked 异常是很重要的，因为程序员在这两种情况下的职责是不同的。Javadoc 的 `@throw` 标记生成的文档在方法声明中没有对应的 throws 子句，这向程序员提供了一个强烈的视觉提示，这是 unchecked 异常。

应该注意的是，记录每个方法会抛出的所有 unchecked 异常是理想的，但在实际中并不总是可以做到。当类进行修订时，如果将导出的方法修改，使其抛出额外的 unchecked 异常，这并不违反源代码或二进制兼容性。假设一个类 A 从另一个独立的类 B 中调用一个方法。A 类的作者可能会仔细记录每个方法抛出的 unchecked 异常，如果 B 类被修改了，使其抛出额外的 unchecked 异常，很可能 A 类（未经修改）将传播新的 unchecked 异常，尽管它没有在文档中声明。

**如果一个类中的许多方法都因为相同的原因抛出异常，你可以在类的文档注释中记录异常，** 而不是为每个方法单独记录异常。一个常见的例子是 NullPointerException。类的文档注释可以这样描述：「如果在任何参数中传递了 null 对象引用，该类中的所有方法都会抛出 NullPointerException」，或者类似意思的话。

总之，记录你所编写的每个方法可能引发的每个异常。对于 unchecked 异常、checked 异常、抽象方法、实例方法都是如此。应该在文档注释中采用 `@throw` 标记的形式。在方法的 throws 子句中分别声明每个 checked 异常，但不要声明 unchecked 异常。如果你不记录方法可能抛出的异常，其他人将很难或不可能有效地使用你的类和接口。
