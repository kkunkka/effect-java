# 第五十三节: 明智地使用可变参数

可变参数方法的正式名称是 variable arity methods [JLS, 8.4.1]，它接受指定类型的零个或多个参数。可变参数首先创建一个数组，其大小是在调用点上传递的参数数量，然后将参数值放入数组，最后将数组传递给方法。

例如，这里有一个可变参数方法，它接受一系列 int 参数并返回它们的和。如你所料，`sum(1, 2, 3)` 的值为 6，`sum()` 的值为 0：

```
// Simple use of varargs
static int sum(int... args) {
    int sum = 0;
    for (int arg : args)
        sum += arg;
    return sum;
}
```

有时，编写一个方法需要一个或多个某种类型的参数，而不是零个或多个参数，这是合适的。例如，假设你想编写一个函数来计算其参数的最小值。如果客户端不传递参数，则此函数定义得不好。你可以在运行时检查数组长度：

```
// The WRONG way to use varargs to pass one or more arguments!
static int min(int... args) {
    if (args.length == 0)
        throw new IllegalArgumentException("Too few arguments");
    int min = args[0];
    for (int i = 1; i < args.length; i++)
        if (args[i] < min)
    min = args[i];
    return min;
}
```

这个解决方案有几个问题。最严重的情况是，如果客户端不带参数调用此方法，那么它将在运行时而不是编译时失败。另一个问题是它不美观。必须包含对 args 的显式有效性检查，并且不能使用 for-each 循环，除非将 min 初始化为 Integer.MAX_VALUE，也很不美观。

幸运的是，有一种更好的方法可以达到预期的效果。声明方法获取两个参数，一个指定类型的常规参数和一个该类型的可变参数。这个解决方案弥补了前一个解决方案的所有不足：

```
// The right way to use varargs to pass one or more arguments
static int min(int firstArg, int... remainingArgs) {
    int min = firstArg;
    for (int arg : remainingArgs)
        if (arg < min)
    min = arg;
    return min;
}
```

从这个例子中可以看出，在方法需要参数数量可变的情况下，可变参数是有效的。可变参数是为 printf 和经过改造的核心反射机制（[Item-65](../Chapter-9/Chapter-9-Item-65-Prefer-interfaces-to-reflection)）而设计的，它们与可变参数同时被添加到 JDK，printf 和 reflection 都从可变参数中受益匪浅。

在性能关键的情况下使用可变参数时要小心。每次调用可变参数方法都会导致数组分配和初始化。如果你已经从经验上确定你负担不起这个成本，但是你仍需要可变参数的灵活性，那么有一种模式可以让你鱼与熊掌兼得。假设你已经确定对方法 95% 的调用只需要三个或更少的参数。可以声明该方法的 5 个重载，每个重载 0 到 3 个普通参数，当参数数量超过 3 个时引入可变参数：

```
public void foo() { }
public void foo(int a1) { }
public void foo(int a1, int a2) { }
public void foo(int a1, int a2, int a3) { }
public void foo(int a1, int a2, int a3, int... rest) { }
```

现在你知道，在所有参数数量超过 3 的调用中，只有 5% 的调用需要付出创建数组的成本。与大多数性能优化一样，这种技术使用并不广泛，但当它合适出现时，就是一个救星。

EnumSet 的静态工厂使用这种技术将创建枚举集的成本降到最低。这是适当的，因为 enum 集合为位字段提供具有性能竞争力的替代方法是至关重要的（[Item-36](../Chapter-6/Chapter-6-Item-36-Use-EnumSet-instead-of-bit-fields)）。

总之，当你需要定义具有不确定数量参数的方法时，可变参数是非常有用的。在可变参数之前加上任何必需的参数，并注意使用可变参数可能会引发的性能后果。
