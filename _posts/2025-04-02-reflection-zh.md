---
layout: post
title:  "Java 反射简介"
categories: java
tags: reflect zh
---

Java 语言的标准库中包括了反射，名为 core reflection（核心反射），可以在运行时检索类的结构，包括在编译时不存在的类。同时也支持检索 Java 语言中的类型和注解。`java.lang.Class` 类上一些方法提供这些信息。这些模型类存在于 `java.lang.reflect` 包中。

## 类的结构

Java 语言中，类中的结构有字段（`Field`）、方法（`Method`）、构造器（`Constructor`）、参数（`Parameter`）、record component（`RecordComponent`）。

对字段、方法、构造器，`Class` 中有命名如 `get(Declared)Xxxs` 方法批量获得此类结构，例如 `getDeclaredFields`。
 - 如果名称中包含 `Declared`，获得的结构不包含从上级类继承的，但是包括所有本类中定义的此类结构，包括非 `public` 结构。
 - 否则返回的结构包括上级继承的结构（最优先本类定义结构，然后字段继承先接口（只有静态字段）再上级类，方法继承优先上级类再接口（接口静态方法无继承）详见 JVMS 5.4，构造器无继承），只包括 `public` 结构。（不包含从上级类继承的 `protected` 结构）

同时还有 `get(Declared)Xxx` 接收参数，精准获得符合条件的结构。字段额外接收字段名称，方法接收方法名称和参数类型数组，构造器接收参数类型数组。能返回的结构必定存在于 `get(Declared)Xxxs` 返回中，如果无符合则报错。

`Parameter` 可以通过 `Executable::getParameters` （方法和构造器的公共上级类中）获得，`RecordComponent` 可以通过 `Class::getRecordComponents` 获得。

这些方法返回数组，所以标准库每次返回时会复制一份数组。字段、方法、构造器还是 `AccessibleObject` 子类，这个类有 `setAccessible`，是可变对象，所以这些结构在标准库返回时也会被复制一次。为了避免额外开销，使用这些方法时最好获取一次数组或者结构，然后缓存返回的数组或结构重复使用，避免获取和复制开销。

这些结构适合用来获得结构上的 Java 语言类型（例如泛型）和注解。字段、方法、构造器还提供方法获得及改变字段值和呼叫方法和构造器；这些用途方便一次性使用，但如果需要多次使用这些功能，使用 `MethodHandles.Lookup` 获得对应的 `MethodHandle` 或者 `VarHandle` 更合适，因为这些呼叫和改变方法每次使用时都会进行权限检查，影响性能。详见 `java.lang.invoke` 包相关介绍。

## Java 语言中的类型

Java 语言定义（Java Language Specification）第四章定义了 Java 语言中的类型，核心反射也有模型类，为了和已有的 `Class` 兼容，和语言中的类型有比较复杂的对应关系。

<!--
<table class="striped">
<thead>
<tr><th colspan="3">类型或泛型参数
    <th>举例
    <th>模型类
</th>
<tbody>
<tr><td colspan="3">基础类 (JLS 4.2)
    <td>int
    <td rowspan="3">Class
<tr><td rowspan="7">引用类(JLS 4.3)
    <td rowspan="3">类与接口
    <td>非泛型类与接口 (JLS 8.1.3, 9.1.3)
    <td>String
<tr><td>去参数类型 (JLS 4.8)
    <td>List
<tr><td>带参数类型 (JLS 4.5)
    <td>List&lt;String&gt;
    <td>ParameterizedType
<tr><td colspan="2">类型参数 (JLS 4.4)
    <td>T
    <td>TypeVariable
<tr><td rowspan="3">数组 (JLS 10.1)
    <td>带参数成员类型
    <td>List&lt;String&gt;[]
    <td rowspan="2">GenericArrayType
<tr><td>类型参数成员类型
    <td>T[]
<tr><td>其他成员类型
    <td>int[]、String[]
    <td>Class
<tr><td colspan="3">Wildcard Type Arguments (JLS 4.5.1)
    <td>? extends String
    <td>WildcardType</td>
</tr>
</tbody>
</table>
-->

各种结构中获得类型的接口 `getXxxType(s)` 返回 `Class` 类型，同时有 `getGenericXxxType(s)` 返回 `Type` 类型，可能是列表中的某一个模型类。

标准库返回的类型模型对象不可变，但是数组也是可变对象，有额外复制开销，所以如果返回的对象需要重复使用，推荐缓存数组或模型对象重复使用。

## 注解

注解可以携带自定义数据，可以存在于定义（结构）上（declaration annotation）或类型的用途中（type(-use) annotation）。携带类型的模型类都实现 `AnnotatedElement` 接口。以上提到的结构和类型都可以携带注解。核心反射能发现的注解都要有 `@Retention(RetentionPolicy.RUNTIME)` 元注解。

注解也有 `get(Declared)Annotation(s)` 的区分，类似结构；注解继承只存在于类与接口上，影响很小。Java 8 允许注解重复，有特殊的 `get(Declared)AnnotationsByType` 可以处理重复注解。

类型用途使用注解有一套模型类，和语言中的类型模型类相似但有些地方有细微差别，例如 `AnnotatedArrayType` 模型包含 `int[]`，但在语言类型模型中不由 `GenericArrayType`，而由 `Class` 模型此类型。

核心反射提供的注解皆通过代理（`Proxy`）实现，可序列化。所以性能会有些劣势。因为返回的数组可变，有复制开销，推荐有需要缓存注解及其数组重复使用。

## 代理

为了方便 reflect 提供了一个代理工具，允许提供一个类加载器和一个接口列表，返回一个实现全部接口，类由指定类加载器加载的代理实例。代理的工厂方法还接收一个处理器，所有代理对象的呼叫（包括所有继承的抽象与非抽象接口方法和`Object`的`equals`、`hashCode`、`toString`方法）都会转到处理器。

这个工具过去常用于实现 AOP，现在推荐能编译时预处理可以考虑编译时预处理。它所有方法都要转到处理器，对 JIT 编译采样及其不友好，所以性能不堪入目；现在有新的类文件接口，条件允许可以生成自己的高性能动态类替代。


<script src="https://giscus.app/client.js"
        data-repo="liachmodded/liachmodded.github.io"
        data-repo-id="MDEwOlJlcG9zaXRvcnkxMTU2NzU0Mjc="
        data-category="Announcements"
        data-category-id="DIC_kwDOBuURI84CfnKT"
        data-mapping="pathname"
        data-strict="0"
        data-reactions-enabled="0"
        data-emit-metadata="1"
        data-input-position="top"
        data-theme="preferred_color_scheme"
        data-lang="en"
        data-loading="lazy"
        crossorigin="anonymous"
        async>
</script>
