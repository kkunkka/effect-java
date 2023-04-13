# 第二十三节: 类层次结构优于带标签的类

有时候，你可能会遇到这样一个类，它的实例有两种或两种以上的样式，并且包含一个标签字段来表示实例的样式。例如，考虑这个类，它能够表示一个圆或一个矩形：

```
// Tagged class - vastly inferior to a class hierarchy!
class Figure {
    enum Shape {RECTANGLE, CIRCLE};

    // Tag field - the shape of this figure
    final Shape shape;

    // These fields are used only if shape is RECTANGLE
    double length;

    double width;

    // This field is used only if shape is CIRCLE
    double radius;

    // Constructor for circle
    Figure(double radius) {
        shape = Shape.CIRCLE;
        this.radius = radius;
    }

    // Constructor for rectangle
    Figure(double length, double width) {
        shape = Shape.RECTANGLE;
        this.length = length;
        this.width = width;
    }

    double area() {
        switch (shape) {
            case RECTANGLE:
                return length * width;
            case CIRCLE:
                return Math.PI * (radius * radius);
            default:
                throw new AssertionError(shape);
        }
    }
}
```

这样的标签类有许多缺点。它们充斥着样板代码，包括 enum 声明、标签字段和 switch 语句。因为多个实现在一个类中混杂，会造成可读性受损。内存占用也增加了，因为实例被其他类型的不相关字段所拖累。除非构造函数初始化不相关的字段，否则不能将字段设置为 final，但这会导致更多的样板文件。构造函数必须设置标签字段并在没有编译器帮助的情况下初始化正确的数据字段：如果初始化了错误的字段，程序将在运行时失败。除非你能够修改它的源文件，否则你不能向标签类添加样式。如果你确实添加了一个样式，那么你必须记住要为每个 switch 语句添加一个 case，否则类将在运行时出错。最后，实例的数据类型没有给出它任何关于样式的提示。简而言之，**标签类冗长、容易出错和低效。**

幸运的是，面向对象的语言（如 Java）提供了一个更好的选择来定义能够表示多种类型对象的单一数据类型：子类型。**标签类只是类层次结构的简易模仿。**

要将已标签的类转换为类层次结构，首先为标签类中的每个方法定义一个包含抽象方法的抽象类，其行为依赖于标签值。在 Figure 类中，只有一个这样的方法，即 area 方法。这个抽象类是类层次结构的根。如果有任何方法的行为不依赖于标签的值，请将它们放在这个类中。类似地，如果有任何数据字段被所有样式使用，将它们放在这个类中。在 Figure 类中没有这样的独立于样式的方法或字段。

接下来，为原始标签类的每个类型定义根类的具体子类。在我们的例子中，有两个：圆形和矩形。在每个子类中包含特定于其样式的数据字段。在我们的例子中，半径是特定于圆的，长度和宽度是特定于矩形的。还应在每个子类中包含根类中每个抽象方法的适当实现。下面是原 Figure 类对应的类层次结构：

```
// Class hierarchy replacement for a tagged class
abstract class Figure {
    abstract double area();
}

class Circle extends Figure {
    final double radius;

    Circle(double radius) {
        this.radius = radius;
    }

    @Override
    double area() {
        return Math.PI * (radius * radius);
    }
}

class Rectangle extends Figure {
    final double length;
    final double width;

    Rectangle(double length, double width) {
        this.length = length;
        this.width = width;
    }

    @Override
    double area() {
        return length * width;
    }
}
```

这个类层次结构纠正了前面提到的标签类的所有缺点。代码简单明了，不包含原始代码中的样板代码。每种样式的实现都分配有自己的类，这些类没有被不相关的数据字段拖累。所有字段为 final 字段。编译器确保每个类的构造函数初始化它的数据字段，并且每个类对于根类中声明的抽象方法都有一个实现。这消除了由于缺少 switch case 而导致运行时出错的可能性。多个程序员可以独立地、可互操作地扩展层次结构，而无需查看根类的源代码。每种样式都有一个单独的数据类型，允许程序员指出变量的样式，并将变量和输入参数限制为特定的样式。

类层次结构的另一个优点是，可以反映类型之间的自然层次关系，从而提高灵活性和更好的编译时类型检查。假设原始示例中的标签类也允许使用正方形。类层次结构可以反映这样一个事实：正方形是一种特殊的矩形（假设两者都是不可变的）：

```
class Square extends Rectangle {
  Square(double side) {
    super(side, side);
  }
}
```

注意，上面层次结构中的字段是直接访问的，而不是通过访问器方法访问的。这样做是为了简洁，如果层次结构是公共的，那么这将是一个糟糕的设计（[Item-16](/Chapter-4/Chapter-4-Item-16-In-public-classes-use-accessor-methods-not-public-fields.md)）。

总之，标签类很少有合适的使用场景。如果想编写一个带有显式标签字段的类，请考虑是否可以删除标签并用层次结构替换。当遇到具有标签字段的现有类时，请考虑将其重构为层次结构。