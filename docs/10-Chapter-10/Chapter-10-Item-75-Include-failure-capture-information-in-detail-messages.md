# 第七十五节: 异常详细消息中应包含捕获失败的信息

当程序由于未捕获异常而失败时，系统可以自动打印出异常的堆栈跟踪。堆栈跟踪包含异常的字符串表示，这是调用其 toString 方法的结果。这通常包括异常的类名及其详细信息。通常，这是程序员或管理员在调查软件故障时所掌握的唯一信息。如果失败不容易重现，想获得更多的信息会非常困难。因此，异常的 toString 方法返回尽可能多的关于失败原因的信息是非常重要的。换句话说，由失败导致的异常的详细信息应该被捕获，以便后续分析。

**要捕获失败，异常的详细消息应该包含导致异常的所有参数和字段的值。** 例如，IndexOutOfBoundsException 的详细消息应该包含下界、上界和未能位于下界之间的索引值。这些信息说明了很多关于失败的信息。这三个值中的任何一个或所有值都可能是错误的。索引可以小于或等于上界（「越界错误」），也可以是一个无效值，太小或太大。下界可能大于上界（严重的内部故障）。每一种情况都指向一个不同的问题，如果你知道你在寻找什么样的错误，这对诊断有很大的帮助。

提及一个与安全敏感信息有关的警告。因为许多人在诊断和修复软件问题的过程中可能会看到堆栈跟踪，所以 **不应包含密码、加密密钥等详细信息。**

虽然在异常的详细信息中包含所有相关数据非常重要，但通常不需要包含大量的描述。堆栈跟踪将与文档一起分析，如果需要，还将与源代码一起分析。它通常包含抛出异常的确切文件和行号，以及堆栈上所有方法调用的文件和行号。冗长的描述对一个失败问题而言是多余的；可以通过阅读文档和源代码来收集信息。

异常的详细信息不应该与用户层的错误消息混淆，因为用户层错误消息最终必须被用户理解。与用户层错误消息不同，详细消息主要是为程序员或管理员在分析故障时提供的。因此，信息内容远比可读性重要。用户层错误消息通常是本地化的，而异常详细信息消息很少本地化。确保异常在其详细信息中包含足够的故障捕获信息的一种方法是，在其构造函数中配置，而不是以传入字符串方式引入这些信息。之后可以自动生成详细信息来包含细节。例如，IndexOutOfBoundsException 构造函数不包含 String 参数，而是像这样：

```
/**
* Constructs an IndexOutOfBoundsException.
**
@param lowerBound the lowest legal index value
* @param upperBound the highest legal index value plus one
* @param index the actual index value
*/
public IndexOutOfBoundsException(int lowerBound, int upperBound, int index) {
    // Generate a detail message that captures the failure
    super(String.format("Lower bound: %d, Upper bound: %d, Index: %d",lowerBound, upperBound, index));
    // Save failure information for programmatic access
    this.lowerBound = lowerBound;
    this.upperBound = upperBound;
    this.index = index;
}
```

从 Java 9 开始，IndexOutOfBoundsException 最终获得了一个接受 int 值索引参数的构造函数，但遗憾的是它忽略了下界和上界参数。更一般地说，Java 库不会大量使用这个习惯用法，但是强烈推荐使用它。它使程序员很容易通过抛出异常来捕获失败。事实上，它使程序员不想捕获失败都难！实际上，这个习惯用法将集中在异常类中生成高质量的详细信息，而不是要求该类的每个用户都生成冗余的详细信息。

**译注：IndexOutOfBoundsException 有关 int 参数的构造函数源码**

```
/**
     * Constructs a new {@code IndexOutOfBoundsException} class with an
     * argument indicating the illegal index.
     *
     * <p>The index is included in this exception's detail message.  The
     * exact presentation format of the detail message is unspecified.
     *
     * @param index the illegal index.
     * @since 9
     */
    public IndexOutOfBoundsException(int index) {
        super("Index out of range: " + index);
    }
```

正如 [Item-70](../Chapter-10/Chapter-10-Item-70-Use-checked-exceptions-for-recoverable-conditions-and-runtime-exceptions-for-programming-errors) 中建议的，异常为其故障捕获信息提供访问器方法是适合的（上面示例中的下界、上界和索引）。在 checked 异常上提供此类访问器方法比 unchecked 异常上提供此类访问器方法更为重要，因为故障捕获信息可能有助于程序从故障中恢复。程序员可能希望通过编程访问 unchecked 异常的详细信息，但这是很少见的（尽管是可以想象的）。然而，即使对于 unchecked 异常，根据一般原则，提供这些访问器也是可以的（[Item-12](../Chapter-3/Chapter-3-Item-12-Always-override-toString)，第 57 页）。

