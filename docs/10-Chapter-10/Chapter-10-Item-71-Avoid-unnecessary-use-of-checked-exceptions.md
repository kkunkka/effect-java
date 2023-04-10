# 第七十一节: 避免不必要地使用 checked 异常

许多 Java 程序员不喜欢 checked 异常，但是如果使用得当，它们可以有利于 API 和程序。与返回代码和 unchecked 异常不同，它们强制程序员处理问题，提高了可靠性。相反，在 API 中过度使用 checked 异常会变得不那么令人愉快。如果一个方法抛出 checked 异常，调用它的代码必须在一个或多个 catch 块中处理它们；或者通过声明抛出，让它们向外传播。无论哪种方式，它都给 API 的用户带来了负担。Java 8 中，这一负担增加得更多，因为会抛出 checked 异常的方法不能直接在流（Item 45-48）中使用。

如果（1）正确使用 API 也不能防止异常情况，（2）并且使用 API 的程序员在遇到异常时可以采取一些有用的操作，那么这种负担是合理的。除非满足这两个条件，否则可以使用 unchecked 异常。作为程序能否成功的试金石，程序员应该问问自己将如何处理异常。这是最好的办法吗？

```
} catch (TheCheckedException e) {
    throw new AssertionError(); // Can't happen!
}
```

Or this?

或者这样？

```
} catch (TheCheckedException e) {
    e.printStackTrace(); // Oh well, we lose.
    System.exit(1);
}
```

如果程序员不能做得更好，则需要一个 unchecked 异常。

如果 checked 异常是方法抛出的唯一 checked 异常，那么 checked 异常给程序员带来的额外负担就会大得多。如果还有其他 checked 异常，则该方法一定已经在 try 块中了，因此该异常最多需要另一个 catch 块而已。如果一个方法抛出单个 checked 异常，那么这个异常就是该方法必须出现在 try 块中而不能直接在流中使用的唯一原因。在这种情况下，有必要问问自己是否有办法避免 checked 异常。

消除 checked 异常的最简单方法是返回所需结果类型的 Optional 对象（[Item-55](/Chapter-8/Chapter-8-Item-55-Return-optionals-judiciously.md)）。该方法只返回一个空的 Optional 对象，而不是抛出一个 checked 异常。这种技术的缺点是，该方法不能返回任何详细说明其无法执行所需计算的附加信息。相反，异常具有描述性类型，并且可以导出方法来提供附加信息（[Item-70](/Chapter-10/Chapter-10-Item-70-Use-checked-exceptions-for-recoverable-conditions-and-runtime-exceptions-for-programming-errors.md)）。

你还可以通过将抛出异常的方法拆分为两个方法，从而将 checked 异常转换为 unchecked 异常，第一个方法返回一个布尔值，指示是否将抛出异常。这个 API 重构将调用序列转换为：

```
// Invocation with checked exception
try {
    obj.action(args);
}
catch (TheCheckedException e) {
    ... // Handle exceptional condition
}
```

转换为这种形式：

```
// Invocation with state-testing method and unchecked exception
if (obj.actionPermitted(args)) {
    obj.action(args);
}
else {
    ... // Handle exceptional condition
}
```

这种重构并不总是适当的，但是只要在适当的地方，它就可以使 API 更易于使用。虽然后一种调用序列并不比前一种调用序列漂亮，但是经过重构的 API 更加灵活。如果程序员知道调用会成功，或者不介意由于调用失败而导致的线程终止，那么该重构还可以接受更简单的调用序列：

```
obj.action(args);
```

如果你不认为上文更简单的调用序列是一种常态，那么 API 重构可能是合适的。重构之后的 API 在本质上等同于 [Item-69](/Chapter-10/Chapter-10-Item-69-Use-exceptions-only-for-exceptional-conditions.md) 中的「状态测试」方法，并且，也有同样的告诫：如果对象将在缺少外部同步的情况下被并发访问，或者可被外界改变状态，这种重构就是不恰当的，因为在 actionPermitted 和 action 这两个调用的间隔，对象的状态有可能会发生变化。如果单独的 actionPermitted 方法必须重复 action 方法的工作，出于性能的考虑，这种 API 重构就不值得去做。

总之，如果谨慎使用，checked 异常可以提高程序的可靠性；当过度使用时，它们会使 API 难以使用。如果调用者不应从失败中恢复，则抛出 unchecked 异常。如果恢复是可能的，并且你希望强制调用者处理异常条件，那么首先考虑返回一个 Optional 对象。只有当在失败的情况下，提供的信息不充分时，你才应该抛出一个 checked 异常。
