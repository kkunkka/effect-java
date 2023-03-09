# 第十六节: 在公共类中，使用访问器方法，而不是公共字段

有时候，可能会编写一些退化类，这些类除了对实例字段进行分组之外，没有其他用途：

```
// Degenerate classes like this should not be public!
class Point {
    public double x;
    public double y;
}
```

因为这些类的数据字段是直接访问的，所以这些类没有提供封装的好处（[Item-15](/Chapter-4/Chapter-4-Item-15-Minimize-the-accessibility-of-classes-and-members.md)）。不改变 API 就不能改变表现形式，不能实现不变量，也不能在访问字段时采取辅助操作。坚持面向对象思维的程序员会认为这样的类是令人厌恶的，应该被使用私有字段和公共访问方法 getter 的类所取代，对于可变类，则是赋值方法 setter：

```
// Encapsulation of data by accessor methods and mutators
class Point {
    private double x;
    private double y;
    public Point(double x, double y) {
        this.x = x;
        this.y = y;
    }
    public double getX() { return x; }
    public double getY() { return y; }
    public void setX(double x) { this.x = x; }
    public void setY(double y) { this.y = y; }
}
```

当然，当涉及到公共类时，强硬派是正确的：如果类可以在包之外访问，那么提供访问器方法来保持更改类内部表示的灵活性。如果一个公共类公开其数据字段，那么改变其表示形式的所有希望都将落空，因为客户端代码可以广泛分发。

但是，如果一个类是包级私有的或者是私有嵌套类，那么公开它的数据字段并没有什么本质上的错误（假设它们能够很好地描述类提供的抽象）。无论是在类定义还是在使用它的客户端代码中，这种方法产生的视觉混乱都比访问方法少。虽然客户端代码与类的内部表示绑定在一起，但这段代码仅限于包含该类的包。如果想要对表示形式进行更改，你可以在不接触包外部任何代码的情况下进行更改。对于私有嵌套类，更改的范围进一步限制在封闭类中。

Java 库中的几个类违反了公共类不应该直接公开字段的建议。突出的例子包括 `java.awt` 包中的 Point 和 Dimension。这些类不应被效仿，而应被视为警示。正如 [Item-67](/Chapter-9/Chapter-9-Item-67-Optimize-judiciously.md) 所述，公开 Dimension 类的内部结构导致了严重的性能问题，这种问题至今仍存在。

虽然公共类直接公开字段从来都不是一个好主意，但是如果字段是不可变的，那么危害就会小一些。你不能在不更改该类的 API 的情况下更改该类的表现形式，也不能在读取字段时采取辅助操作，但是你可以实施不变量。例如，这个类保证每个实例代表一个有效的时间：

```
// Public class with exposed immutable fields - questionable
public final class Time {
    private static final int HOURS_PER_DAY = 24;
    private static final int MINUTES_PER_HOUR = 60;
    public final int hour;
    public final int minute;

    public Time(int hour, int minute) {
        if (hour < 0 || hour >= HOURS_PER_DAY)
            throw new IllegalArgumentException("Hour: " + hour);
        if (minute < 0 || minute >= MINUTES_PER_HOUR)
            throw new IllegalArgumentException("Min: " + minute);
        this.hour = hour;
        this.minute = minute;
    } ... // Remainder omitted
}
```

总之，公共类不应该公开可变字段。对于公共类来说，公开不可变字段的危害要小一些，但仍然存在潜在的问题。然而，有时候包级私有或私有嵌套类需要公开字段，无论这个类是可变的还是不可变的。