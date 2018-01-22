---
title: "SICP 语言的三要素"
date: 2018-01-15T22:40:36+08:00
draft: false
---

## 语言的基本要素

中文：
> 一个强有力的程序设计语言，不仅仅是一种指挥计算机执行任务的方式，它还应该成为一种框架，使我们能够在其中组织自己有关计算过程的思想。 这样，当我们描述一个语言时，就需要将注意力特别放在这一语言所提供的，能够将简单的认识组合起来形成更复杂认识的方法方面。 每一种强有力的语言都为此提供了三种机制：

> * 基本表达形式，用于表示语言所关心的最简单的个体。
> * 组合的方法，通过它们可以从较简单的东西出发构造出复合的元素。
> * 抽象的方法，通过它们可以为复合对象命名，并将他们当做单元去操作。

英文：
> A powerful programming language is more than just a means for instructing a computer to perform tasks. The language also serves as a framework within which we organize our ideas about processes. Thus, when we describe a language, we should pay particular attention to the means that the language provides for combining simple ideas to form more complex ideas. Every powerful language has three mechanisms for accomplishing this:

> * primitive expressions, which represent the simplest entities the language is concerned with,

> * means of combination, by which compound elements are built from simpler ones, and

> * means of abstraction, by which compound elements can be named and manipulated as units.

## 例子： Django Rest Framework CRUD Views

[DRF框架][1]提供了一套泛型视图, 基本要素有如下几个类：

 - GenericAPIView
 - CreateModelMixin
 - ListModelMixin
 - RetrieveModelMixin
 - UpdateModelMixin
 - DestroyModelMixin
    
Python支持继承多个类，利用这种组合方式，DRF在上面这些类的基础上又提供了如下视图：

```python
class CreateAPIView(mixins.CreateModelMixin, GenericAPIView): pass
class ListAPIView(mixins.ListModelMixin, GenericAPIView): pass
class RetrieveAPIView(mixins.RetrieveModelMixin, GenericAPIView): pass
class DestroyAPIView(mixins.DestroyModelMixin,  GenericAPIView): pass
class UpdateAPIView(mixins.UpdateModelMixin, GenericAPIView): pass

class ListCreateAPIView(mixins.ListModelMixin, mixins.CreateModelMixin, GenericAPIView): 
    pass

class RetrieveUpdateAPIView(mixins.RetrieveModelMixin, mixins.UpdateModelMixin, GenericAPIView): 
    pass
    
class RetrieveDestroyAPIView(mixins.RetrieveModelMixin, mixins.DestroyModelMixin, GenericAPIView):            pass

class RetrieveUpdateDestroyAPIView(mixins.RetrieveModelMixin,
                               mixins.UpdateModelMixin,
                               mixins.DestroyModelMixin,
                               GenericAPIView):
    pass
```

 在这一层面的设计中，GenericAPIView，XXXXModelMixin扮演了基本元素，Python的多类继承提供了组合方式， 各种APIView是复合元素， python的类定义实现了抽象。
 
  
### 例子： Java InputStream Decorator Pattern
  
  Java的[InputStream][2]是XXXInputStream的类的基类，它包含有如下几个子类：
  
 - AudioInputStream    
 - ByteArrayInputStream   
 - FileInputStream  
 - FilterInputStream   
 - InputStream   
 - ObjectInputStream 
 - PipedInputStream   
 - SequenceInputStream   
 - StringBufferInputStream

这些类采用了装饰器模式的设计，可以像下面这样使用：

```java
FileInputStream fis = new FileInputStream("/objects.gz");
BufferedInputStream bis = new BufferedInputStream(fis);
GzipInputStream gis = new GzipInputStream(bis);
ObjectInputStream ois = new ObjectInputStream(gis);
SomeObject someObject = (SomeObject) ois.readObject();
```
*[source][3]*

一个stream可以作为另一种stream的来源，从而实现丰富的组合。
在这个例子中，各种Stream是基本元素，组合方式式层层嵌套，对复合stream的命名由Java的变量实现。

一直觉得SICP第二章的Picture Language跟这个例子很相似，都是由一个基本的元素通过层层装饰，形成一个复杂的东西。

  [1]: http://www.django-rest-framework.org/
  [2]: https://docs.oracle.com/javase/7/docs/api/java/io/InputStream.html
  [3]: http://stackoverflow.com/questions/6366385/decorator-pattern-for-io
  
  原本想继续说一下UNIX的命令与管道的，但是不知道如何说起。这个例子中，每个命令貌似算不上基本元素，而且怎么也无法把他们想象成语言，想到这里，书上只是说一个强力的语言应该提供这三个要素，但是并没有说者满足这三个要素的就是语言，或许我该放弃“语言”这个概念，单纯从基本元素，组合，抽象这些角度去思考。
  
突然想起第二章的Language levels, 这里又涉及到抽象屏障(abstract barrier),每一层都有自己的primitive，但这种primitive在下一层级却是复合对象，所以把linux的各个命令视为这个层级的基本元素也是可以的。

## 例子： UNIX的Rule of Composition

Rule of Composition: Design programs to be connected to other programs.

这里的programs应该就是我们常用的shell下的各种命令了，这里我将各种命令视为基本元素，组合方式就是管道了，那么脚本或者函数就是抽象的方式了。


## 最后

前面找了几个算是接触过的例子，其实应该还有很多，只是一时没想到。通过基本元素组合成更复杂的元素，应该是跨越编程领域甚至计算机领域的一种构建方式，这种方式配合抽象屏障，从而很好地控制了一个大型系统的复杂性，将复杂性划分成一个个级别，每次只关注一个级别层面的实现，从而达到关注点分离（separation of concerns）的目的。


