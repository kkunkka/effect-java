# 第九节: 使用 try-with-resources 优于 try-finally

Java 库包含许多必须通过调用 close 方法手动关闭的资源。常见的有 InputStream、OutputStream 和 java.sql.Connection。关闭资源常常会被客户端忽略，这会导致可怕的性能后果。虽然这些资源中的许多都使用终结器作为安全网，但终结器并不能很好地工作（[Item-8](../Chapter-2/Chapter-2-Item-8-Avoid-finalizers-and-cleaners)）。

从历史上看，try-finally 语句是确保正确关闭资源的最佳方法，即使在出现异常或返回时也是如此：

```
// try-finally - No longer the best way to close resources!
static String firstLineOfFile(String path) throws IOException {
    BufferedReader br = new BufferedReader(new FileReader(path));
    try {
        return br.readLine();
    } finally {
        br.close();
    }
}
```

这可能看起来不坏，但添加第二个资源时，情况会变得更糟：

```
// try-finally is ugly when used with more than one resource!
static void copy(String src, String dst) throws IOException {
    InputStream in = new FileInputStream(src);
    try {
        OutputStream out = new FileOutputStream(dst);
    try {
        byte[] buf = new byte[BUFFER_SIZE];
        int n;
        while ((n = in.read(buf)) >= 0)
            out.write(buf, 0, n);
    } finally {
        out.close();
        }
    }
    finally {
        in.close();
    }
}
```

这可能难以置信。在大多数情况下，即使是优秀的程序员也会犯这种错误。首先，我在 Java Puzzlers [Bloch05]的 88 页上做错了，多年来没有人注意到。事实上，2007 年发布的 Java 库中三分之二的 close 方法使用都是错误的。

**译注：《Java Puzzlers》的中文译本为《Java 解惑》**

使用 try-finally 语句关闭资源的正确代码（如前两个代码示例所示）也有一个细微的缺陷。try 块和 finally 块中的代码都能够抛出异常。例如，在 firstLineOfFile 方法中，由于底层物理设备发生故障，对 readLine 的调用可能会抛出异常，而关闭的调用也可能出于同样的原因而失败。在这种情况下，第二个异常将完全覆盖第一个异常。异常堆栈跟踪中没有第一个异常的记录，这可能会使实际系统中的调试变得非常复杂（而这可能是希望出现的第一个异常，以便诊断问题）。虽然可以通过编写代码来抑制第二个异常而支持第一个异常，但实际上没有人这样做，因为它太过冗长。

当 Java 7 引入 try-with-resources 语句 [JLS, 14.20.3]时，所有这些问题都一次性解决了。要使用这个结构，资源必须实现 AutoCloseable 接口，它由一个单独的 void-return close 方法组成。Java 库和第三方库中的许多类和接口现在都实现或扩展了 AutoCloseable。如果你编写的类存在必须关闭的资源，那么也应该实现 AutoCloseable。

下面是使用 try-with-resources 的第一个示例：

```
// try-with-resources - the the best way to close resources!
static String firstLineOfFile(String path) throws IOException {
    try (BufferedReader br = new BufferedReader(new FileReader(path))) {
        return br.readLine();
    }
}
```

下面是使用 try-with-resources 的第二个示例：

```
// try-with-resources on multiple resources - short and sweet
static void copy(String src, String dst) throws IOException {
    try (InputStream in = new FileInputStream(src);OutputStream out = new FileOutputStream(dst)) {
        byte[] buf = new byte[BUFFER_SIZE];
        int n;
        while ((n = in.read(buf)) >= 0)
            out.write(buf, 0, n);
    }
}
```

和使用 try-finally 的原版代码相比，try-with-resources 为开发者提供了更好的诊断方式。考虑 firstLineOfFile 方法。如果异常是由 readLine 调用和不可见的 close 抛出的，则后一个异常将被抑制，以支持前一个异常。实际上，还可能会抑制多个异常，以保留实际希望看到的异常。这些被抑制的异常不会仅仅被抛弃；它们会被打印在堆栈跟踪中，并标记它们被抑制。可以通过编程方式使用 getSuppressed 方法访问到它们，该方法是在 Java 7 中添加到 Throwable 中的。

可以在带有资源的 try-with-resources 语句中放置 catch 子句，就像在常规的 try-finally 语句上一样。这允许处理异常时不必用另一层嵌套来影响代码。作为一个特指的示例，下面是我们的 firstLineOfFile 方法的一个版本，它不抛出异常，但如果无法打开文件或从中读取文件，则返回一个默认值：

```
// try-with-resources with a catch clause
static String firstLineOfFile(String path, String defaultVal) {
    try (BufferedReader br = new BufferedReader(new FileReader(path))) {
        return br.readLine();
    } catch (IOException e) {
        return defaultVal;
    }
}
```

教训很清楚：在使用必须关闭的资源时，总是优先使用 try-with-resources，而不是 try-finally。前者的代码更短、更清晰，生成的异常更有用。使用 try-with-resources 语句可以很容易地为必须关闭的资源编写正确的代码，而使用 try-finally 几乎是不可能的。
