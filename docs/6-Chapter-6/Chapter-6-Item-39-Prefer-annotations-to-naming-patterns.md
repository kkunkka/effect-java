# 第三十九节: 注解优于命名模式

从历史上看，使用命名模式来标明某些程序元素需要工具或框架特殊处理的方式是很常见的。例如，在版本 4 之前，JUnit 测试框架要求其用户通过以字符 test [Beck04] 开头的名称来指定测试方法。这种技术是有效的，但是它有几个很大的缺点。首先，排版错误会导致没有提示的失败。例如，假设你意外地将一个测试方法命名为 tsetSafetyOverride，而不是 testSafetyOverride。JUnit 3 不会报错，但它也不会执行测试，这导致一种正确执行了测试的假象。

命名模式的第二个缺点是，无法确保只在相应的程序元素上使用它们。例如，假设你调用了一个类 TestSafetyMechanisms，希望 JUnit 3 能够自动测试它的所有方法，而不管它们的名称是什么。同样，JUnit 3 不会报错，但它也不会执行测试。

命名模式的第三个缺点是，它们没有提供将参数值与程序元素关联的好方法。例如，假设你希望支持只有在抛出特定异常时才成功的测试类别。异常类型本质上是测试的一个参数。你可以使用一些精心设计的命名模式，将异常类型名称编码到测试方法名称中，但这样的代码将不好看且脆弱（[Item-62](../Chapter-9/Chapter-9-Item-62-Avoid-strings-where-other-types-are-more-appropriate)）。编译器将无法检查这些用于命名异常的字符串是否确实执行了。如果指定的类不存在或不是异常，则在运行测试之前不会被发现。

注解 [JLS, 9.7] 很好地解决了所有这些问题，JUnit 从版本 4 开始就采用了它们。在本条目中，我们将编写自己的示例测试框架来展示注解是如何工作的。假设你希望定义注解类型，以指定自动运行的简单测试，并在抛出异常时失败。下面是这种名为 Test 的注解类型的概貌：

```
// Marker annotation type declaration
import java.lang.annotation.*;

/**
* Indicates that the annotated method is a test method.
* Use only on parameterless static methods.
*/
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Test {

}
```

Test 注解类型的声明本身带有 Retention 注解和 Target 注解。这种注解类型声明上的注解称为元注解。`@Retention(RetentionPolicy.RUNTIME)` 元注解表明测试注解应该在运行时保留。没有它，测试工具将无法识别测试注解。`@Target.get(ElementType.METHOD)` 元注解表明测试注解仅对方法声明合法：它不能应用于类声明、字段声明或其他程序元素。

**译注 1：注解的保留策略**

保留策略决定了在什么位置丢弃注解。Java 定义了 3 种策略，它们被封装到 `java.lang.annotation.RetentionPolicy` 枚举中。这 3 种策略分别是 SOURCE、CLASS 和 RUNTIME。
- 使用 SOURCE 保留策略的注解，只在源文件中保留，在编译期间会被抛弃。
- 使用 CLASS 保留策略的注解，在编译时被存储到 `.class` 文件中。但是，在运行时不能通过 JVM 得到这些注解。
- 使用 RUNTIME 保留策略的注解，在编译时被存储到 `.class` 文件中，并且在运行时可以通过 JVM 获取这些注解。因此，RUNTIME 保留策略提供了最永久的注解。

**译注 2：ElementType 各常量定义的范围**

- ElementType.TYPE
  - Class, interface (including annotation type), or enum declaration（类、接口、注解、枚举）
- ElementType.FIELD
  - Field declaration (includes enum constants)（字段、枚举常量）
- ElementType.METHOD
  - Method declaration（方法）
- ElementType.PARAMETER
  - Formal parameter declaration（方法参数）
- ElementType.CONSTRUCTOR
  - Constructor declaration（构造）
- ElementType.LOCAL_VARIABLE
  - Local variable declaration（局部变量）
- ElementType.ANNOTATION_TYPE
  - Annotation type declaration（注解）
- ElementType.PACKAGE
  - Package declaration（包）
- ElementType.TYPE_PARAMETER
  - Type parameter declaration（泛型参数）
  - Since: 1.8
- ElementType.TYPE_USE
  - Use of a type（任意类型，获取 class 对象和 import 两种情况除外）
  - Since: 1.8
- ElementType.MODULE
  - Module declaration（[模块](https://docs.oracle.com/javase/9/whatsnew/toc.htm#JSNEW-GUID-C23AFD78-C777-460B-8ACE-58BE5EA681F6)）
  - Since: 9

Test 注解声明之前的代码注释是这么描述的:「Use only on parameterless static methods.（只对无参数的静态方法使用）」如果编译器能够强制执行这一点，那就太好了，但是它不能，除非你编写代码注释处理器来执行。有关此主题的更多信息，请参阅 `javax.annotation.processing` 的文档。在没有这样的代码注释处理程序的情况下，如果你将 Test 注解放在实例方法的声明上，或者放在带有一个或多个参数的方法上，测试程序仍然会编译，让测试工具在运行时处理。

下面是 Test 注解实际使用时的样子。它被称为标记注解，因为它没有参数，只是对带注解的元素进行「标记」。如果程序员拼错 Test 或将 Test 注解应用于除方法声明之外的程序元素，程序将无法编译：

```
// Program containing marker annotations
public class Sample {
    @Test
    public static void m1() { } // Test should pass

    public static void m2() { }

    @Test
    public static void m3() { // Test should fail
        throw new RuntimeException("Boom");
    }

    public static void m4() { }

    @Test
    public void m5() { } // INVALID USE: nonstatic method

    public static void m6() { }

    @Test
    public static void m7() { // Test should fail
        throw new RuntimeException("Crash");
    }

    public static void m8() { }
}
```

Sample 类有 7 个静态方法，其中 4 个被注解为 Test。其中两个方法 m3 和 m7 抛出异常，另外两个 m1 和 m5 没有抛出异常。但是，不抛出异常的带注解的方法 m5 是一个实例方法，因此它不是注解的有效使用。总之，Sample 包含四个测试：一个通过，两个失败，一个无效。没有使用 Test 注释的四个方法将被测试工具忽略。

Test 注解对 Sample 类的语义没有直接影响。它们仅用于向相关程序提供信息。更普遍的是，注解不会改变被注解代码的语义，而是通过工具（就像如下这个简单的 RunTests 类）对其进行特殊处理：

```
// Program to process marker annotations
import java.lang.reflect.*;

public class RunTests {
    public static void main(String[] args) throws Exception {
        int tests = 0;
        int passed = 0;
        Class<?> testClass = Class.forName(args[0]);
        for (Method m : testClass.getDeclaredMethods()) {
            if (m.isAnnotationPresent(Test.class)) {
                tests++;
                try {
                    m.invoke(null);
                    passed++;
                } catch (InvocationTargetException wrappedExc) {
                    Throwable exc = wrappedExc.getCause();
                    System.out.println(m + " failed: " + exc);
                } catch (Exception exc) {
                    System.out.println("Invalid @Test: " + m);
                }
        }
    }
    System.out.printf("Passed: %d, Failed: %d%n",passed, tests - passed);
    }
}
```

test runner 工具以命令行方式接受一个完全限定的类名，并通过调用 `Method.invoke` 以反射方式运行类的所有带测试注解的方法。isAnnotationPresent 方法告诉工具要运行哪些方法。如果测试方法抛出异常，反射工具将其封装在 InvocationTargetException 中。该工具捕获这个异常并打印一个失败报告，其中包含测试方法抛出的原始异常，该异常是用 getCause 方法从 InvocationTargetException 提取的。

如果通过反射调用测试方法时抛出除 InvocationTargetException 之外的任何异常，则表明在编译时存在未捕获的 Test 注解的无效用法。这些用途包括实例方法的注解、带有一个或多个参数的方法的注解或不可访问方法的注解。测试运行程序中的第二个 catch 块捕获这些 Test 使用错误并打印对应的错误消息。如果在 Sample 上运行 RunTests，输出如下：

```
public static void Sample.m3() failed: RuntimeException: Boom
Invalid @Test: public void Sample.m5()
public static void Sample.m7() failed: RuntimeException: Crash
Passed: 1, Failed: 3
```

现在让我们添加一个只在抛出特定异常时才成功的测试支持。我们需要一个新的注解类型：

```
// Annotation type with a parameter
import java.lang.annotation.*;

/**
* Indicates that the annotated method is a test method that
* must throw the designated exception to succeed.
*/
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface ExceptionTest {
    Class<? extends Throwable> value();
}
```

这个注解的参数类型是 `Class<? extends Throwable>`，这个通配符类型确实很复杂。在英语中，它的意思是「某个扩展自 Throwable 的类的 Class 对象」，它允许注解的用户指定任何异常（或错误）类型。这种用法是有界类型令牌（[Item-33](../Chapter-5/Chapter-5-Item-33-Consider-typesafe-heterogeneous-containers)）的一个示例。下面是这个注解在实际应用时的样子。注意，类的字面量被用作注解参数的值：

```
// Program containing annotations with a parameter
public class Sample2 {
    @ExceptionTest(ArithmeticException.class)
    public static void m1() { // Test should pass
        int i = 0;
        i = i / i;
    }

    @ExceptionTest(ArithmeticException.class)
    public static void m2() { // Should fail (wrong exception)
        int[] a = new int[0];
        int i = a[1];
    }

    @ExceptionTest(ArithmeticException.class)
    public static void m3() { } // Should fail (no exception)
}
```

现在让我们修改 test runner 工具来处理新的注解。向 main 方法添加以下代码：

```
if (m.isAnnotationPresent(ExceptionTest.class)) {
    tests++;
    try {
        m.invoke(null);
        System.out.printf("Test %s failed: no exception%n", m);
    } catch (InvocationTargetException wrappedEx) {
        Throwable exc = wrappedEx.getCause();
        Class<? extends Throwable> excType =m.getAnnotation(ExceptionTest.class).value();
        if (excType.isInstance(exc)) {
            passed++;
        } else {
            System.out.printf("Test %s failed: expected %s, got %s%n",m, excType.getName(), exc);
        }
    }
    catch (Exception exc) {
        System.out.println("Invalid @Test: " + m);
    }
}
```

这段代码与我们用来处理 Test 注解的代码类似，只有一个不同：这段代码提取注解参数的值，并使用它来检查测试抛出的异常是否是正确的类型。这里没有显式的强制类型转换，因此没有 ClassCastException 的危险。编译的测试程序保证其注解参数表示有效的异常类型，但有一点需要注意：如果注解参数在编译时有效，但表示指定异常类型的类文件在运行时不再存在，那么测试运行程序将抛出 TypeNotPresentException。


进一步修改我们的异常测试示例，如果它抛出几个指定异常中的任意一个，那么可以认为测试通过了。注解机制具有一种工具，可以轻松地支持这种用法。假设我们将 ExceptionTest 注解的参数类型更改为一个 Class 对象数组：

```
// Annotation type with an array parameter
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface ExceptionTest {
    Class<? extends Exception>[] value();
}
```

注解中数组参数的语法是灵活的。它针对单元素数组进行了优化。前面的 ExceptionTest 注解对于 ExceptionTest 的新数组参数版本仍然有效，并且可以生成单元素数组。要指定一个多元素数组，用花括号包围元素，并用逗号分隔它们：

```
// Code containing an annotation with an array parameter
@ExceptionTest({ IndexOutOfBoundsException.class,NullPointerException.class })
public static void doublyBad() {
    List<String> list = new ArrayList<>();
    // The spec permits this method to throw either
    // IndexOutOfBoundsException or NullPointerException
    list.addAll(5, null);
}
```

修改测试运行器工具来处理 ExceptionTest 的新版本是相当简单的。这段代码替换了原来的版本：

```
if (m.isAnnotationPresent(ExceptionTest.class)) {
    tests++;
    try {
        m.invoke(null);
        System.out.printf("Test %s failed: no exception%n", m);
    } catch (Throwable wrappedExc) {
        Throwable exc = wrappedExc.getCause();
        int oldPassed = passed;
        Class<? extends Exception>[] excTypes =m.getAnnotation(ExceptionTest.class).value();
        for (Class<? extends Exception> excType : excTypes) {
            if (excType.isInstance(exc)) {
                passed++;
                break;
            }
        }
        if (passed == oldPassed)
            System.out.printf("Test %s failed: %s %n", m, exc);
    }
}
```

在 Java 8 中，还有另一种方法可以执行多值注解。你可以在注解声明上使用 `@Repeatable` 元注解，以表明注解可以重复地应用于单个元素，而不是使用数组参数来声明注解类型。这个元注解只接受一个参数，这个参数是包含注解类型的类对象，它的唯一参数是注解类型的数组 [JLS, 9.6.3]。如果我们对 ExceptionTest 注解采用这种方法，那么注解声明是这样的。注意，包含的注解类型必须使用适当的 Retention 注解和 Target 注解，否则声明将无法编译：

```
// Repeatable annotation type
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@Repeatable(ExceptionTestContainer.class)
public @interface ExceptionTest {
    Class<? extends Exception> value();
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface ExceptionTestContainer {
    ExceptionTest[] value();
}
```

下面是使用重复注解代替数组值注解的 doublyBad 测试：

```
// Code containing a repeated annotation
@ExceptionTest(IndexOutOfBoundsException.class)
@ExceptionTest(NullPointerException.class)
public static void doublyBad() { ... }
```

处理可重复注解需要小心。「重复状态」会生成名为「容器注解类型」的合成注解。getAnnotationsByType 方法可忽略这一区别，它可以用于访问可重复注解类型的「重复状态」和「非重复状态」。但是 isAnnotationPresent 明确指出，「重复状态」的情况不属于注解类型，而是「容器注解类型」。如果一个元素是某种类型的「重复状态」注解，并且你使用 isAnnotationPresent 方法检查该元素是否具有该类型的注解，你将发现它提示不存在。因此，使用此方法检查注解类型的存在与否，将导致你的程序忽略「重复状态」。类似地，使用此方法检查「容器注解类型」将导致程序忽略「非重复状态」。要使用 isAnnotationPresent 检测「重复状态」和「非重复状态」，需要同时检查注解类型及其「容器注解类型」。下面是我们的 RunTests 程序的相关部分修改为使用 ExceptionTest 注解的可重复版本时的样子：

```
// Processing repeatable annotations
if (m.isAnnotationPresent(ExceptionTest.class)|| m.isAnnotationPresent(ExceptionTestContainer.class)) {
    tests++;
    try {
        m.invoke(null);
        System.out.printf("Test %s failed: no exception%n", m);
    } catch (Throwable wrappedExc) {
        Throwable exc = wrappedExc.getCause();
        int oldPassed = passed;
        ExceptionTest[] excTests =m.getAnnotationsByType(ExceptionTest.class);
        for (ExceptionTest excTest : excTests) {
            if (excTest.value().isInstance(exc)) {
                passed++;
                break;
            }
        }
        if (passed == oldPassed)
            System.out.printf("Test %s failed: %s %n", m, exc);
    }
}
```

**译注：比较原文中提及的 getAnnotationsByType 与 isAnnotationPresent 在可重复注解的「重复状态」和「非重复状态」下的使用差别：**

**原 doublyBad 方法不变，属于「重复状态」（重复注解大于等于两个的，都属于「重复状态」）；新增一个 doublyBad2 方法，仅使用一个重复注解，属于「非重复状态」**
```
class Simple4 {
    // Code containing a repeated annotation
    @ExceptionTest(IndexOutOfBoundsException.class)
    @ExceptionTest(NullPointerException.class)
    public static void doublyBad() {
    }

    @ExceptionTest(ArithmeticException.class)
    public static void doublyBad2() {
    }
}
```
**测试代码**
```
public static void main(String[] args) throws NoSuchMethodException {
    Class<?> testClass = Simple4.class;
    for (int count = 1; count <= 2; count++) {
        Method m = testClass.getMethod(count == 1 ? "doublyBad" : "doublyBad" + count);
        System.out.println(m.getName() + "「重复状态」：" + m.isAnnotationPresent(ExceptionTest.class));
        System.out.println(m.getName() + "「容器注解类型」：" + m.isAnnotationPresent(ExceptionTestContainer.class));
        System.out.println(m.getName() + "「非重复状态」：" + m.isAnnotationPresent(ExceptionTest.class));
        System.out.println(m.getName() + "「重复状态」：" + m.getAnnotationsByType(ExceptionTest.class));
        System.out.println(m.getName() + "「容器注解类型」：" + m.getAnnotationsByType(ExceptionTestContainer.class));
        System.out.println(m.getName() + "「非重复状态」：" + m.getAnnotationsByType(ExceptionTest.class));
    }
}
```
**结果**
```
doublyBad「重复状态」：false
doublyBad「容器注解类型」：true
doublyBad「非重复状态」：false
doublyBad「重复状态」：[LItem_39.ExceptionTest;@1593948d
doublyBad「容器注解类型」：[LItem_39.ExceptionTestContainer;@1b604f19
doublyBad「非重复状态」：[LItem_39.ExceptionTest;@7823a2f9

doublyBad2「重复状态」：true
doublyBad2「容器注解类型」：false
doublyBad2「非重复状态」：true
doublyBad2「重复状态」：[LItem_39.ExceptionTest;@cb5822
doublyBad2「容器注解类型」：[LItem_39.ExceptionTestContainer;@4b9e13df
doublyBad2「非重复状态」：[LItem_39.ExceptionTest;@2b98378d
```

添加可重复注解是为了提高源代码的可读性，源代码在逻辑上将同一注解类型的多个实例应用于给定的程序元素。如果你觉得它们增强了源代码的可读性，那么就使用它们，但是请记住，在声明和处理可重复注解方面有更多的样板，并且处理可重复注解很容易出错。

本条目中的测试框架只是一个示例，但是它清楚地展示了注解相对于命名模式的优势，并且它只涉及到你可以使用它们做什么。如果你编写的工具要求程序员向源代码中添加信息，请定义适当的注解类型。**如果可以使用注解，那么就没有理由使用命名模式。**

也就是说，除了 toolsmiths 之外，大多数程序员不需要定义注解类型。但是所有程序员都应该使用 Java 提供的预定义注解类型（[Item-40](../Chapter-6/Chapter-6-Item-40-Consistently-use-the-Override-annotation) 和 [Item-27](../Chapter-5/Chapter-5-Item-27-Eliminate-unchecked-warnings)）。另外，考虑使用 IDE 或静态分析工具提供的注解。这些注解可以提高这些工具提供的诊断信息的质量。但是，请注意，这些注解还没有标准化，因此，如果你切换了工具或出现了标准，那么你可能需要做一些工作。
