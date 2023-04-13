# 第二节：当构造函数有多个参数时，考虑改用构建器

---

静态工厂和构造函数都有一个局限：它们不能对大量可选参数做很好的扩展。以一个类为例，它表示包装食品上的营养标签。这些标签上有一些字段是必需的，如：净含量、毛重和每单位份量的卡路里，另有超过 20 个可选的字段，如：总脂肪、饱和脂肪、反式脂肪、胆固醇、钠等等。大多数产品只有这些可选字段中的少数，且具有非零值。

应该为这样的类编写什么种类的构造函数或静态工厂呢？

## 1. 使用可伸缩构造函数

在这种模式中，只向构造函数提供必需的参数。即，向第一个构造函数提供单个可选参数，向第二个构造函数提供两个可选参数，以此类推，最后一个构造函数是具有所有可选参数的。这是它在实际应用中的样子。为了简洁起见，只展示具备四个可选字段的情况：

```Java
// Telescoping constructor pattern - does not scale well!
public class NutritionFacts {
    private final int servingSize; // (mL) required
    private final int servings; // (per container) required
    private final int calories; // (per serving) optional
    private final int fat; // (g/serving) optional
    private final int sodium; // (mg/serving) optional
    private final int carbohydrate; // (g/serving) optional

    public NutritionFacts(int servingSize, int servings) {
        this(servingSize, servings, 0);
    }

    public NutritionFacts(int servingSize, int servings, int calories) {
        this(servingSize, servings, calories, 0);
    }

    public NutritionFacts(int servingSize, int servings, int calories, int fat) {
        this(servingSize, servings, calories, fat, 0);
    }

    public NutritionFacts(int servingSize, int servings, int calories, int fat, int sodium) {
        this(servingSize, servings, calories, fat, sodium, 0);
    }

    public NutritionFacts(int servingSize, int servings, int calories, int fat, int sodium, int carbohydrate) {
        this.servingSize = servingSize;
        this.servings = servings;
        this.calories = calories;
        this.fat = fat;
        this.sodium = sodium;
        this.carbohydrate = carbohydrate;
    }
}
```

当你想要创建一个实例时，可以使用包含所需参数的最短参数列表的构造函数：

```
NutritionFacts cocaCola =new NutritionFacts(240, 8, 100, 0, 35, 27);
```

通常，这个构造函数包含许多额外的参数，但是你必须为它们传递一个值。在本例中，我们为 fat 传递了一个值 0。只有六个参数时，这可能看起来不那么糟，但随着参数的增加，它很快就会失控。

简单地说，**可伸缩构造函数模式可以工作，但是当有很多参数时，编写客户端代码是很困难的，而且读起来更困难。** 读者想知道所有这些值是什么意思，必须仔细清点参数。相同类型参数的长序列会导致细微的错误。如果客户端不小心倒转了两个这样的参数，编译器不会报错，但是程序会在运行时出错（[第五十一节](../Chapter-8/Chapter-8-Item-51-Design-method-signatures-carefully)）。

## 2. JavaBean 模式

在这种模式中，你调用一个无参数的构造函数来创建对象，然后调用 setter 方法来设置每个所需的参数和每个感兴趣的可选参数：

```
// JavaBeans Pattern - allows inconsistency, mandates mutability
public class NutritionFacts {
    // Parameters initialized to default values (if any)
    private int servingSize = -1; // Required; no default value
    private int servings = -1; // Required; no default value
    private int calories = 0;
    private int fat = 0;
    private int sodium = 0;
    private int carbohydrate = 0;
    public NutritionFacts() { }
    // Setters
    public void setServingSize(int val) { servingSize = val; }
    public void setServings(int val) { servings = val; }
    public void setCalories(int val) { calories = val; }
    public void setFat(int val) { fat = val; }
    public void setSodium(int val) { sodium = val; }
    public void setCarbohydrate(int val) { carbohydrate = val; }
}
```

这个模式没有可伸缩构造函数模式的缺点。创建实例很容易，虽然有点冗长，但很容易阅读生成的代码：

```
NutritionFacts cocaCola = new NutritionFacts();
cocaCola.setServingSize(240);
cocaCola.setServings(8);
cocaCola.setCalories(100);
cocaCola.setSodium(35);
cocaCola.setCarbohydrate(27);
```

不幸的是，JavaBean 模式本身有严重的缺点。因为构建是在多个调用之间进行的，所以 JavaBean 可能在构建的过程中处于不一致的状态。该类不能仅通过检查构造函数参数的有效性来强制一致性。在不一致的状态下尝试使用对象可能会导致错误的发生，而包含这些错误的代码很难调试。一个相关的缺点是，JavaBean 模式排除了使类不可变的可能性（[第十七节](../Chapter-4/Chapter-4-Item-17-Minimize-mutability)），并且需要程序员额外的努力来确保线程安全。

通过在对象构建完成时手动「冻结」对象，并在冻结之前不允许使用对象，可以减少这些缺陷，但是这种变通方式很笨拙，在实践中很少使用。此外，它可能在运行时导致错误，因为编译器不能确保程序员在使用对象之前调用它的 freeze 方法。

## 3. 构建器

它结合了可伸缩构造函数模式的安全性和 JavaBean 模式的可读性。它是建造者模式的一种形式。客户端不直接生成所需的对象，而是使用所有必需的参数调用构造函数（或静态工厂），并获得一个 builder 对象。然后，客户端在构建器对象上调用像 setter 这样的方法来设置每个感兴趣的可选参数。最后，客户端调用一个无参数的构建方法来生成对象，这通常是不可变的。构建器通常是它构建的类的静态成员类（[第二十四节](../Chapter-4/Chapter-4-Item-24-Favor-static-member-classes-over-nonstatic)）。下面是它在实际应用中的样子：

**若将该案例「构建机制」独立出来，或能广泛适应相似结构的构建需求，详见文末随笔**

```
// Builder Pattern
public class NutritionFacts {
    private final int servingSize;
    private final int servings;
    private final int calories;
    private final int fat;
    private final int sodium;
    private final int carbohydrate;

    public static class Builder {
        // Required parameters
        private final int servingSize;
        private final int servings;
        // Optional parameters - initialized to default values
        private int calories = 0;
        private int fat = 0;
        private int sodium = 0;
        private int carbohydrate = 0;

        public Builder(int servingSize, int servings) {
            this.servingSize = servingSize;
            this.servings = servings;
        }

        public Builder calories(int val) {
            calories = val;
            return this;
        }

        public Builder fat(int val) {
            fat = val;
            return this;
        }

        public Builder sodium(int val) {
            sodium = val;
            return this;
        }

        public Builder carbohydrate(int val) {
            carbohydrate = val;
            return this;
        }

        public NutritionFacts build() {
            return new NutritionFacts(this);
        }
    }

    private NutritionFacts(Builder builder) {
        servingSize = builder.servingSize;
        servings = builder.servings;
        calories = builder.calories;
        fat = builder.fat;
        sodium = builder.sodium;
        carbohydrate = builder.carbohydrate;
    }
}
```

NutritionFacts 类是不可变的，所有参数默认值都在一个位置。构建器的 setter 方法返回构建器本身，这样就可以链式调用，从而得到一个流畅的 API。下面是客户端代码的样子：

```
NutritionFacts cocaCola = new NutritionFacts.Builder(240, 8)
.calories(100).sodium(35).carbohydrate(27).build();
```

该客户端代码易于编写，更重要的是易于阅读。建造者模式模拟 Python 和 Scala 中的可选参数。

为了简洁，省略了有效性检查。为了尽快检测无效的参数，请检查构建器的构造函数和方法中的参数有效性。检查构建方法调用的构造函数中涉及多个参数的不变量。为了确保这些不变量不受攻击，在从构建器复制参数之后检查对象字段（[第五十节](../Chapter-8/Chapter-8-Item-50-Make-defensive-copies-when-needed)）。如果检查失败，抛出一个 IllegalArgumentException（[第七十二节](../Chapter-10/Chapter-10-Item-72-Favor-the-use-of-standard-exceptions)），它的详细消息指示哪些参数无效（[第七十五节](../Chapter-10/Chapter-10-Item-75-Include-failure-capture-information-in-detail-messages)）。

建造者模式非常适合于类层次结构。使用构建器的并行层次结构，每个构建器都嵌套在相应的类中。抽象类有抽象类构建器；具体类有具体类构建器。例如，考虑一个在层次结构处于最低端的抽象类，它代表各种比萨饼：

```
import java.util.EnumSet;
import java.util.Objects;
import java.util.Set;

// Builder pattern for class hierarchies
public abstract class Pizza {
    public enum Topping {HAM, MUSHROOM, ONION, PEPPER, SAUSAGE}

    final Set<Topping> toppings;

    abstract static class Builder<T extends Builder<T>> {
        EnumSet<Topping> toppings = EnumSet.noneOf(Topping.class);

        public T addTopping(Topping topping) {
            toppings.add(Objects.requireNonNull(topping));
            return self();
        }

        abstract Pizza build();

        // Subclasses must override this method to return "this"
        protected abstract T self();
    }

    Pizza(Builder<?> builder) {
        toppings = builder.toppings.clone(); // See Item 50
    }
}
```

:::tip

`Pizza.Builder` 是具有递归类型参数的泛型类型（[第三十一节](../Chapter-5/Chapter-5-Item-31-Use-bounded-wildcards-to-increase-API-flexibility)）。这与抽象 self 方法一起，允许方法链接在子类中正常工作，而不需要强制转换。对于 Java 缺少自类型这一事实，这种变通方法称为模拟自类型习惯用法。这里有两个具体的比萨子类，一个是标准的纽约风格的比萨，另一个是 calzone。前者有一个必需的尺寸大小参数，而后者让你指定酱料应该放在里面还是外面

:::

```
import java.util.Objects;

public class NyPizza extends Pizza {
    public enum Size {SMALL, MEDIUM, LARGE}

    private final Size size;

    public static class Builder extends Pizza.Builder<Builder> {
        private final Size size;

        public Builder(Size size) {
            this.size = Objects.requireNonNull(size);
        }

        @Override
        public NyPizza build() {
            return new NyPizza(this);
        }

        @Override
        protected Builder self() {
            return this;
        }
    }

    private NyPizza(Builder builder) {
        super(builder);
        size = builder.size;
    }
}

public class Calzone extends Pizza {
    private final boolean sauceInside;

    public static class Builder extends Pizza.Builder<Builder> {
        private boolean sauceInside = false; // Default

        public Builder sauceInside() {
            sauceInside = true;
            return this;
        }

        @Override
        public Calzone build() {
            return new Calzone(this);
        }

        @Override
        protected Builder self() {
            return this;
        }
    }

    private Calzone(Builder builder) {
        super(builder);
        sauceInside = builder.sauceInside;
    }
}
```

:::tip

注意，每个子类的构建器中的构建方法声明为返回正确的子类：构建的方法 `NyPizza.Builder` 返回 NyPizza，而在 `Calzone.Builder` 则返回 Calzone。这种技术称为协变返回类型，其中一个子类方法声明为返回超类中声明的返回类型的子类型。它允许客户使用这些构建器，而不需要强制转换。这些「层次构建器」的客户端代码与简单的 NutritionFacts 构建器的代码基本相同。为简洁起见，下面显示的示例客户端代码假定枚举常量上的静态导入

:::

```
NyPizza pizza = new NyPizza.Builder(SMALL)
.addTopping(SAUSAGE).addTopping(ONION).build();
Calzone calzone = new Calzone.Builder()
.addTopping(HAM).sauceInside().build();
```

与构造函数相比，构建器的一个小优点是构建器可以有多个变量参数，因为每个参数都是在自己的方法中指定的。或者，构建器可以将传递给一个方法的多个调用的参数聚合到单个字段中，如前面的 addTopping 方法中所示。

建造者模式非常灵活。一个构建器可以反复构建多个对象。构建器的参数可以在构建方法的调用之间进行调整，以改变创建的对象。构建器可以在创建对象时自动填充某些字段，例如在每次创建对象时增加的序列号。

建造者模式也有缺点。为了创建一个对象，你必须首先创建它的构建器。虽然在实际应用中创建这个构建器的成本可能并不显著，但在以性能为关键的场景下，这可能会是一个问题。而且，建造者模式比可伸缩构造函数模式更冗长，因此只有在有足够多的参数时才值得使用，比如有 4 个或更多参数时，才应该使用它。但是请记住，你可能希望在将来添加更多的参数。但是，如果你以构造函数或静态工厂开始，直至类扩展到参数数量无法控制的程度时，也会切换到构建器，但是过时的构造函数或静态工厂将很难处理。因此，最好一开始就从构建器开始。

总之，在设计构造函数或静态工厂的类时，建造者模式是一个很好的选择，特别是当许多参数是可选的或具有相同类型时。与可伸缩构造函数相比，使用构建器客户端代码更容易读写，而且构建器比 JavaBean 更安全。

## 随笔

每个内部 Builder 类要对每个字段建立相应方法，代码比较冗长。若将「构建机制」独立出来，或能广泛适应相似结构的构建需求。以下是针对原文案例的简要修改，仅供参考：

```
class EntityCreator<T> {

    private Class<T> classInstance;
    private T entityObj;

    public EntityCreator(Class<T> classInstance, Object... initParams) throws Exception {
        this.classInstance = classInstance;
        Class<?>[] paramTypes = new Class<?>[initParams.length];
        for (int index = 0, length = initParams.length; index < length; index++) {
            String checkStr = initParams[index].getClass().getSimpleName();
            if (checkStr.contains("Integer")) {
                paramTypes[index] = int.class;
            }
            if (checkStr.contains("Double")) {
                paramTypes[index] = double.class;
            }
            if (checkStr.contains("Boolean")) {
                paramTypes[index] = boolean.class;
            }
            if (checkStr.contains("String")) {
                paramTypes[index] = initParams[index].getClass();
            }
        }
        Constructor<T> constructor = classInstance.getDeclaredConstructor(paramTypes);
        constructor.setAccessible(true);
        this.entityObj = constructor.newInstance(initParams);
    }

    public EntityCreator<T> setValue(String paramName, Object paramValue) throws Exception {
        Field field = classInstance.getDeclaredField(paramName);
        field.setAccessible(true);
        field.set(entityObj, paramValue);
        return this;
    }

    public T build() {
        return entityObj;
    }
}
```

如此，可移除整个内部 Builder 类，NutritionFacts 类私有构造的参数仅包括两个必填的 servingSize、servings 字段：

```
public class NutritionFacts {
    // Required parameters
    private final int servingSize;
    private final int servings;
    // Optional parameters - initialized to default values
    private int calories = 0;
    private int fat = 0;
    private int sodium = 0;
    private int carbohydrate = 0;

    private NutritionFacts(int servingSize, int servings) {
        this.servingSize = servingSize;
        this.servings = servings;
    }
}
```

该案例的客户端代码改为：

```
NutritionFacts cocaCola = new EntityCreator<>(NutritionFacts.class, 240, 8)
        .setValue("calories", 100)
        .setValue("sodium", 35)
        .setValue("carbohydrate", 27).build();](/Chapter-2/Chapter-2-Item-3-Enforce-the-singleton-property-with-a-private-constructor-or-an-enum-type.md)**
```
