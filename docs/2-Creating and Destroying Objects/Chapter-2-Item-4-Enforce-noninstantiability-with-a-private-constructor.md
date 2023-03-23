# 第四节: 用私有构造函数实施不可实例化

有时你会想要写一个类，它只是一个静态方法和静态字段的组合。这样的类已经获得了坏名声，因为有些人滥用它们来避免从对象角度思考，但是它们确有用途。它们可以用 java.lang.Math 或 java.util.Arrays 的方式，用于与原始值或数组相关的方法。它们还可以用于对以 java.util.Collections 的方式实现某些接口的对象分组静态方法，包括工厂（[Item-1](/Chapter-2/Chapter-2-Item-1-Consider-static-factory-methods-instead-of-constructors.md)）。（对于 Java 8，你也可以将这些方法放入接口中，假设你可以进行修改。）最后，这些类可用于对 final 类上的方法进行分组，因为你不能将它们放在子类中。

这样的实用程序类不是为实例化而设计的：实例是无意义的。然而，在没有显式构造函数的情况下，编译器提供了一个公共的、无参数的默认构造函数。对于用户来说，这个构造函数与其他构造函数没有区别。在已发布的 API 中看到无意中实例化的类是很常见的。

**试图通过使类抽象来实施不可实例化是行不通的。** 可以对类进行子类化，并实例化子类。此外，它误导用户认为类是为继承而设计的（[Item-19](/Chapter-4/Chapter-4-Item-19-Design-and-document-for-inheritance-or-else-prohibit-it.md)）。然而，有一个简单的习惯用法来确保不可实例化。只有当类不包含显式构造函数时，才会生成默认构造函数，因此**可以通过包含私有构造函数使类不可实例化：**

```
// Noninstantiable utility class
public class UtilityClass {
    // Suppress default constructor for noninstantiability
    private UtilityClass() {
        throw new AssertionError();
    } ... // Remainder omitted
}
```


因为显式构造函数是私有的，所以在类之外是不可访问的。AssertionError 不是严格要求的，但是它提供了保障，以防构造函数意外地被调用。它保证类在任何情况下都不会被实例化。这个习惯用法有点违反常规，因为构造函数是明确提供的，但不能调用它。因此，如上述代码所示，包含注释是明智的做法。

这个习惯用法也防止了类被子类化，这是一个副作用。所有子类构造函数都必须调用超类构造函数，无论是显式的还是隐式的，但这种情况下子类都没有可访问的超类构造函数可调用。
