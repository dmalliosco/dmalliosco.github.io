---
title: KVC实现原理分析
tags: 源码分析
categories: 技术分享
---
## KVC实现原理分析
### 问题？
1. KVC在进行存取的时候，是怎么进行查找赋值的？
2. KVC的keypath中集合运算符是如何使用的？
3. 使用KVC的时候如果setvalue的属性没有实现会怎样？

### KVC是什么？
* KVC全称是Key value coding，定义在NSKeyValueCoding.h文件中，是一个非正式协议。
* KVC提供了一种间接访问属性方法/成员变量的机制。我们可以通过字符串来访问对应的属性方法或者成员变量。
* KVC的实现依赖其搜索规则。

### KVC的访问方法
* 在NSKeyValueCoding中我们可以看到提供了KVC的通用访问方法，getter方法：valueForKey:以及setter方法：setValue:forKey，以及其衍生出来的keyPath方法。
* 这两个方法由KVC提供了默认的实现。也可以重写对应的方法来更改实现。

### 操作对象
KVC主要针对三种类型进行操作，基础数据类型，对象和集合类型。

### KVC的集合操作符运算
* @sum 求和运算 比如@sum.classNumber
* @avg 求平均值运算 比如@avg.classNumber
* @count 求集合中元素的个数 比如@count

### 搜索规则
* KVC在通过key和keypath进行操作的时候，可以查找属性方法，成员变量。查找的时候可以兼容多种命名，查找的规则官方文档有介绍。
* 在KVC的实现中，依赖了`setter`和`getter`方法的实现。方法的命名要符合apple的规范。

#### 关键属性accessInstanceVariablesDirectly
这个属性表示是否允许读取实例变量的值，如果设置为YES,表明在KVC的查找中，从内存中读取实例变量的值。（在没有找到存取器的时候才会调用该方法）。

### 基础getter搜索模式
1. 这种搜索模式是valueForKey:的默认实现，给定一个key作为参数
2. 通过`getter`方法搜索实例，依次匹配  `get<key>`  `<key> ` `is<key>` ` _<key>` 如果找到，直接返回。需要注意的是：如果返回的是对象指针类型，直接返回结果，如果返回的是NSNumber转化所支持的变量之一，返回一个NSNumber否则返回的是NSValue。
3. 当没有找到getter方法的时候，调用`accessInstanceVariablesDirectly`询问，如果返回`yes`，从`_<key>`` _is<key>` ` <key> ` `is<key>`中查找对应的值。如果返回NO结束查找，并调用`valueForUndefindedKey`异常。

### 基础setter搜索模式
1. 这种搜索模式是`setValue:forKey`的默认实现，给定输入的value和key。在调用对象的内部，设置属性名为key的`value`。
2. 查找`set<key>:`或者`_set<key>`命名的setter。按照这个顺序，如果找到的话，调用这个方法执行。
3. 如果设置的`setter`但是`accessInstanceVariablesDirectly`返回YES,那么查找的命名规则为`_<key> _is<key> <key> is<key>`的实例变量。根据这个顺序将value赋值给实例变量。
4. 如果没有发现setter或实例变量。调用`setValueForUndefinedKey`并抛出异常。

### KVC的性能

从上面的描述中可以看出：KVC的性能访问属性并没有直接访问快，因为他是按照搜索规则进行搜索。所以我们在使用的时候，最好不 要手动设置`setter``和getter` 方法这样会导致搜索的步骤变长。本质上是操作方法列表以及在内存中去查找实例变量。这个操作对readonly和protected的成员变量，都可以正常访问。如果不想在外界访问的话，可以将`accessInstanceVariableDirectly`设置为NO。

### KVC缺点
* 因为我们在操作的时候传入`setvalueForKey`以及`setvalueForKeyPath valueForKey`以及`keyPath`是一个字符串。编译器在编译的时候不会报错，但是在运行的时候，如果设置或者获取的属性不存在就会报undefinedKey异常。
* 例如：在 iOS13之前我们可以通过改变私有变量的属性来更改一些设置。比如更改textField的颜色属性。iOS13之后更改的时候debug模式下会crash。

### JsonModel中的使用
主要是通过KVC进行赋值，例如在赋值的时候，循环遍历model中每一个解析出来的property结构，从dic中拿出对应的value，进行一系列的判断。如果value可用，就进行赋值。如果对应的property是一个jsonmodel的时候，就递归先将子model进行整理解析。如果包含protocol字段，表明是一个array或者dictionary。然后将这个protocol字段的对象解析。