# 第八节: 避免使用终结器和清除器

**终结器是不可预测的，通常是危险的，也是不必要的。** 它们的使用可能导致不稳定的行为、糟糕的性能和可移植性问题。终结器有一些有效的用途，我们将在后面的文章中介绍，但是作为规则，你应该避免使用它们。在 Java 9 中，终结器已经被弃用，但是 Java 库仍然在使用它们。Java 9 替代终结器的是清除器。**清除器的危险比终结器小，但仍然不可预测、缓慢，而且通常是不必要的。**

c++ 程序员被告诫不要把终结器或清除器当成 Java 的 c++ 析构函数。在 c++ 中，析构函数是回收与对象相关联的资源的常用方法，对象是构造函数的必要对等物。在 Java 中，当对象变得不可访问时，垃圾收集器将回收与之关联的存储，无需程序员进行任何特殊工作。c++ 析构函数还用于回收其他非内存资源。在 Java 中，使用带有资源的 try-with-resources 或 try-finally 块用于此目的（[Item-9](/Chapter-2/Chapter-2-Item-9-Prefer-try-with-resources-to-try-finally.md)）。

终结器和清除器的一个缺点是不能保证它们会被立即执行 [JLS, 12.6]。当对象变得不可访问，终结器或清除器对它进行操作的时间是不确定的。这意味着永远不应该在终结器或清除器中执行任何对时间要求很严格的操作。例如，依赖终结器或清除器关闭文件就是一个严重错误，因为打开的文件描述符是有限的资源。如果由于系统在运行终结器或清除器的延迟导致许多文件处于打开状态，程序可能会运行失败，因为它不能再打开其他文件。

终结器和清除器执行的快速性主要是垃圾收集算法的功能，在不同的实现中存在很大差异。依赖于终结器的及时性或更清晰的执行的程序的行为可能也会发生变化。这样的程序完全有可能在测试它的 JVM 上完美地运行，然后在最重要的客户喜欢的 JVM 上悲惨地失败。

姗姗来迟的定稿不仅仅是一个理论上的问题。为类提供终结器可以任意延迟其实例的回收。一位同事调试了一个长期运行的 GUI 应用程序，该应用程序神秘地终结于 OutOfMemoryError 错误。分析显示，在应用程序终结的时候，终结器队列上有数千个图形对象等待最终完成和回收。不幸的是，终结器线程运行的优先级低于另一个应用程序线程，因此对象不能以适合终结器的速度完成。语言规范没有保证哪个线程将执行终结器，因此除了避免使用终结器之外，没有其他可移植的方法来防止这类问题。在这方面，清洁器比终结器要好一些，因为类作者可以自己控制是否清理线程，但是清洁器仍然在后台运行，在垃圾收集器的控制下运行，所以不能保证及时清理。

该规范不仅不能保证终结器或清洁剂能及时运行；它并不能保证它们能运行。完全有可能，甚至很有可能，程序在某些不再可访问的对象上运行而终止。因此，永远不应该依赖终结器或清除器来更新持久状态。例如，依赖终结器或清除器来释放共享资源（如数据库）上的持久锁，是让整个分布式系统停止工作的好方法。

不要被 System.gc 和 System.runFinalization 的方法所诱惑。它们可能会增加终结器或清除器被运行的几率，但它们不能保证一定运行。曾经有两种方法声称可以保证这一点：System.runFinalizersOnExit 和它的孪生兄弟 Runtime.runFinalizersOnExit。这些方法存在致命的缺陷，并且已经被废弃了几十年[ThreadStop]。

终结器的另一个问题是，在终结期间抛出的未捕获异常被忽略，该对象的终结终止 [JLS, 12.6]。未捕获的异常可能会使其他对象处于损坏状态。如果另一个线程试图使用这样一个损坏的对象，可能会导致任意的不确定性行为。正常情况下，未捕获的异常将终止线程并打印堆栈跟踪，但如果在终结器中出现，则不会打印警告。清除器没有这个问题，因为使用清除器的库可以控制它的线程。


使用终结器和清除器会严重影响性能。在我的机器上，创建一个简单的 AutoCloseable 对象，使用 try-with-resources 关闭它以及让垃圾收集器回收它的时间大约是 12ns。相反，使用终结器将时间增加到 550ns。换句话说，使用终结器创建和销毁对象大约要慢 50 倍。这主要是因为终结器抑制了有效的垃圾收集。如果使用清除器清除的所有实例（在我的机器上每个实例大约 500ns），那么清除器的速度与终结器相当，但是如果只将它们作为安全网来使用，清除器的速度要快得多，如下所述。在这种情况下，在我的机器上创建、清理和销毁一个对象需要花费 66ns 的时间，这意味着如果你不使用它，你需要多出五倍（而不是五十倍）的保障成本。


终结器有一个严重的安全问题：它们会让你的类受到终结器攻击。终结器攻击背后的思想很简单：如果从构造函数或它的序列化等价物（readObject 和 readResolve 方法（[Item-12](/Chapter-3/Chapter-3-Item-12-Always-override-toString.md)））抛出一个异常，恶意子类的终结器就可以运行在部分构造的对象上，而这个对象本来应该「胎死腹中」。这个终结器可以在静态字段中记录对对象的引用，防止它被垃圾收集。一旦记录了畸形对象，就很容易在这个对象上调用本来就不应该存在的任意方法。从构造函数抛出异常应该足以防止对象的出现；在有终结器的情况下，就不是这样了。这样的攻击可能会造成可怕的后果。最终类对终结器攻击免疫，因为没有人能够编写最终类的恶意子类。为了保护非最终类不受终结器攻击，编写一个不执行任何操作的最终终结方法。

那么，如果一个类的对象封装了需要终止的资源，例如文件或线程，那么应该做什么，而不是为它编写终结器或清除器呢？只有你的类实现 AutoCloseable，要求其客户端每个实例在不再需要时调用关闭方法，通常使用 try-with-resources 确保终止，即使面对异常（[Item-9](/Chapter-2/Chapter-2-Item-9-Prefer-try-with-resources-to-try-finally.md)）。一个值得一提的细节是实例必须跟踪是否已经关闭：close 方法必须在字段中记录对象不再有效，其他方法必须检查这个字段，如果在对象关闭后调用它们，则必须抛出一个 IllegalStateException。


那么，清除器和终结器有什么用呢？它们可能有两种合法用途。一种是充当一个安全网，以防资源的所有者忽略调用它的 close 方法。虽然不能保证清除器或终结器将立即运行（或根本不运行），但如果客户端没有这样做，最好是延迟释放资源。如果你正在考虑编写这样一个安全网络终结器，那就好好考虑一下这种保护是否值得。一些 Java 库类，如 FileInputStream、FileOutputStream、ThreadPoolExecutor 和 java.sql.Connection，都有终结器作为安全网。

清洁器的第二个合法使用涉及到与本机对等体的对象。本机对等点是普通对象通过本机方法委托给的本机（非 java）对象。因为本机对等点不是一个正常的对象，垃圾收集器不知道它，并且不能在回收 Java 对等点时回收它。如果性能是可接受的，并且本机对等体不持有任何关键资源，那么更清洁或终结器可能是完成这项任务的合适工具。如果性能不可接受，或者本机对等体持有必须立即回收的资源，则类应该具有前面描述的关闭方法。

清除器的使用有些棘手。下面是一个简单的 Room 类，展示了这个设施。让我们假设房间在回收之前必须被清理。Room 类实现了 AutoCloseable；它的自动清洗安全网使用了清除器，这只是一个实现细节。与终结器不同，清除器不会污染类的公共 API：

```
import sun.misc.Cleaner;

// An autocloseable class using a cleaner as a safety net
public class Room implements AutoCloseable {
    private static final Cleaner cleaner = Cleaner.create();

    // Resource that requires cleaning. Must not refer to Room!
    private static class State implements Runnable {
        int numJunkPiles; // Number of junk piles in this room

        State(int numJunkPiles) {
            this.numJunkPiles = numJunkPiles;
        }

        // Invoked by close method or cleaner
        @Override
        public void run() {
            System.out.println("Cleaning room");
            numJunkPiles = 0;
        }
    }

    // The state of this room, shared with our cleanable
    private final State state;
    // Our cleanable. Cleans the room when it’s eligible for gc
    private final Cleaner.Cleanable cleanable;

    public Room(int numJunkPiles) {
        state = new State(numJunkPiles);
        cleanable = cleaner.register(this, state);
    }

    @Override
    public void close() {
        cleanable.clean();
    }
}
```

静态嵌套 State 类持有清洁器清洁房间所需的资源。在这种情况下，它仅仅是 numJunkPiles 字段，表示房间的混乱程度。更实际地说，它可能是最后一个包含指向本机对等点的 long 指针。State 实现了 Runnable，它的运行方法最多被调用一次，由我们在 Room 构造器中向 cleaner 实例注册状态实例时得到的 Cleanable 调用。对 run 方法的调用将由以下两种方法之一触发：通常是通过调用 Room 的 close 方法来触发，调用 Cleanable 的 clean 方法。如果当一个 Room 实例有资格进行垃圾收集时，客户端没有调用 close 方法，那么清除器将调用 State 的 run 方法（希望如此）。

状态实例不引用其 Room 实例是非常重要的。如果它这样做了，它将创建一个循环，以防止 Room 实例有资格进行垃圾收集（以及自动清理）。因此，状态必须是一个静态嵌套类，因为非静态嵌套类包含对其封闭实例的引用（[Item-24](/Chapter-4/Chapter-4-Item-24-Favor-static-member-classes-over-nonstatic.md)）。同样不建议使用 lambda，因为它们可以很容易地捕获对包围对象的引用。

就像我们之前说的，Room 类的清除器只是用作安全网。如果客户端将所有 Room 实例包围在带有资源的 try 块中，则永远不需要自动清理。这位表现良好的客户端展示了这种做法：

```
public class Adult {
    public static void main(String[] args) {
        try (Room myRoom = new Room(7)) {
            System.out.println("Goodbye");
        }
    }
}
```

如你所料，运行 Adult 程序打印「Goodbye」，然后是打扫房间。但这个从不打扫房间的不守规矩的程序怎么办？

```
public class Teenager {
    public static void main(String[] args) {
        new Room(99);
        System.out.println("Peace out");
    }
}
```

你可能期望它打印出「Peace out」，然后打扫房间，但在我的机器上，它从不打扫房间；它只是退出。这就是我们之前提到的不可预测性。Cleaner 规范说：「在 System.exit 中，清洁器的行为是特定于实现的。不保证清理操作是否被调用。」虽然规范没有说明，但对于普通程序退出来说也是一样。在我的机器上，将 System.gc() 添加到 Teenager 的主要方法中就足以让它在退出之前打扫房间，但不能保证在其他机器上看到相同的行为。总之，不要使用清洁器，或者在 Java 9 之前的版本中使用终结器，除非是作为安全网或终止非关键的本机资源。即便如此，也要小心不确定性和性能后果。