---
sidebar_label: 第三章 对象的通用方法
---
# 章节介绍

虽然 Object 是一个具体的类，但它主要是为扩展而设计的。它的所有非 final 方法（equals、hashCode、toString、clone 和 finalize）都有显式的通用约定，因为它们的设计目的是被覆盖。任何类都有责任覆盖这些方法并将之作为一般约定；如果不这样做，将阻止依赖于约定的其他类（如 HashMap 和 HashSet）与之一起正常工作。

本章将告诉你何时以及如何覆盖 Object 类的非 final 方法。finalize 方法在本章中被省略，因为它在 [Item-8](../Chapter-2/Chapter-2-Item-8-Avoid-finalizers-and-cleaners) 中讨论过。虽然 Comparable.compareTo 不是 Object 类的方法，但是由于具有相似的特性，所以本章也对它进行讨论。
