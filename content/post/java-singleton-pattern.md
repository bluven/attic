---
title: "Singleton Design Pattern"
date: 2018-01-20T21:31:07+08:00
---

The Singleton design pattern comes under the creational pattern type.It is one of the most simple design pattern in terms of the modelling but on the other hand this is one of the most controversial pattern in terms of complexity of usage.

Sometimes it is necessary that one and only one instance of a particular class exists in the entire Java Virtual Machine.  This is achieved through the Singleton design pattern .

The Singleton pattern is widely used and has proved its usability in designing software. Although the pattern is not specific to Java, it has become a classic in Java programming.

Implementation of Singleton Deign Pattern

There are many ways this can be done in Java. All these ways differs in their implementation of the pattern, but it in the end, they all achieve the same end result of a single instance.

##1.  Lazy initialization

The Singleton pattern is implemented by creating a class with a method that creates a new instance of the object if one does not exist. If an instance already exists, it simply returns a reference to that object. To make sure that the object cannot be instantiated any other way, the constructor is made private. Although a Singleton can be implemented as a static instance, it can also be lazily constructed, requiring no memory or resources until needed.

    <!-- lang: java -->
    public class SingletonExample {

        // Static member holds only one instance of the
        // SingletonExample class
        private static SingletonExample singletonInstance;

        // SingletonExample prevents any other class from instantiating
        private SingletonExample() {
        }

        // Providing Global point of access
        public static SingletonExample getSingletonInstance() {
            if (null == singletonInstance) {
                singletonInstance = new SingletonExample();
            }
            return singletonInstance;
         }
      }
 
##2.   Lazy initialization with Double check locking

The above code works absolutely fine in a single threaded environment and processes the result faster because of lazy initialization. However the above code might create some abrupt behavior in the results in a multithreaded environment as in this situation multiple threads can possibly create multiple instance of the same SingletonExample class if they try to access the getSingletonInstance() method at the same time.

In the multithreading environment to prevent each thread to create another instance of singleton object and thus creating concurrency issue we will need to use locking mechanism. This can be achieved by synchronized keyword. By using this synchronized keyword we prevent Thread2 or Thread3 to access the singleton instance while Thread1 is inside the method getSingletonInstance().

From code perspective it means:

    <!-- lang: java -->
    public static synchronized SingletonExample getSingletonInstance() {
        if (null == singletonInstance) {
            singletonInstance = new SingletonExample();
        }
        return singletonInstance;
    }

So this means that every time the getSingletonInstance() is called it gives us an additional overhead . To prevent this expensive operation we will use double checked locking so that the synchronization happens only during the first call and we limit this expensive operation to happen only once.

    <!-- lang: java -->
    public static volatile SingletonExample getSingletonInstance() {
        if (null == singletonInstance) {
            synchronized (SingletonExample.class){
                    if (null == singletonInstance) {
            singletonInstance = new SingletonExample();
            }
        }
        }
        return singletonInstance;
    }

##3.   Eager initialization

If the program will always need an instance, or if the cost of creating the instance is not too large in terms of time/resources, the programmer can switch to eager initialization, which always creates an instance when the class is loaded into the JVM.

    <!-- lang: java -->
    public class SingletonExample {

        // Static member holds only one instance of the
        // SingletonExample class
        private static SingletonExample singletonInstance = 
                                                 new SingletonExample();
    
        // SingletonExample prevents any other class from instantiating
        private SingletonExample() {
        }
    
        // Providing Global point of access
        public static SingletonExample getSingletonInstance() {
            return singletonInstance;
        }
    }

##4.   Static block initialization

    <!-- lang: java -->
    public class SingletonExample {

    // Static member holds only one instance of the
    // SingletonExample class
        private static SingletonExample singletonInstance;
        static { 
           try {
               singletonInstance = new SingletonExample();
               } catch (IOException e) {
                  throw new RuntimeException("Darn, an error occurred!", e);
               }
        }
        // SingletonExample prevents any other class from instantiating
        private SingletonExample() {
        }
    
        // Providing Global point of access
        public static SingletonExample getSingletonInstance() {
            return singletonInstance;
        }
    }
    
##5.   Using Enum

From Java 5 onwards there is a new way to implement Singleton design pattern, by using Enum. Enum has some distinct benefits in terms of thread-safety during instance creation, serialization guarantee by JVM and amazingly reduce amount of code which makes it perfect choice of using as Singleton class.

    <!-- lang: java -->
    /**
    * Singleton pattern example using Java Enum
    */
    public enum EnumSingleton{
        INSTANCE;
    }
    
##6.   Initialization-on-demand holder idiom

This idiom derives its thread safety from the fact that operations that are part of class initialization, which is guaranteed by the JVM. It derives its lazy initialization from the fact that the inner class is not loaded until some thread references one of its fields or methods. This provides the best of both options and guaranteed to be thread-safe by the JVM.

The trick is to use private nested class to hold Singleton instance.

    <!-- lang: java -->
    public class SingletonExample {

        // Static member class member that holds only one instance of the
        // SingletonExample class
        private static class SingletonHolder{
        public static SingletonExample singletonInstance =
                                              new SingletonExample();
        }
        // SingletonExample prevents any other class from instantiating
        private SingletonExample() {
        }
    
        // Providing Global point of access
        public static SingletonExample getSingletonInstance() {
            return SingletonHolder.singletonInstance;
        }
    }

This solution leverages lazy initialization. Static instance field is initialized no earlier than SingletonHolder class is loaded and initialized. SingletonHolder class in its turn is loaded and initialized no earlier than it is first referenced. Finally SingletonHolder class is first referenced no earlier than getSingletonInstance() method is called. And this is exactly what we need. This implementation ,similar to eager and static block initialization is also based on the class initialization procedure which is guaranteed by Java Language Specification to be thread-safe.

##Singleton and Serialization

Serialization allows storing the object in some data store and re create it later on. However when we serialize a singleton class and invoke deserialization multiple times, we can end up with multiple objects of the singleton class.

In order to make serialization and singleton work properly,we have to introduce readResolve() method in the singleton class. readResolve() method lets developer control what object should be returned  on deserialization.

    <!-- lang: java -->
    public class SingletonExample implements Serializable {

        // Static member holds only one instance of the
        // SingletonExample class
        private static SingletonExample singletonInstance = 
                                                 new SingletonExample();
    
        // SingletonExample prevents any other class from instantiating
        private SingletonExample() {
        }    // Providing Global point of access
        public static SingletonExample getSingletonInstance() {
            return singletonInstance;
        }
        // This method is called immediately after an object of this class is
        // deserialized.
         protected Object readResolve() {
            // Instead of the object we’re on, return the class variable singleton
           return singletonInstance;
        }
    }

##Cloning can spoil the game

    <!-- lang: java -->
    public Object clone() throws CloneNotSupportedException {
       throw new CloneNotSupportedException();
    }
    
##When to use Static Class in place of Singleton in Java

The Singleton pattern has several advantages over static classes. First, a singleton can extend classes and implement interfaces, while a static class cannot (it can extend classes, but it does not inherit their instance members). A singleton can be initialized lazily or asynchronously while a static class is generally initialized when it is first loaded, leading to potential class loader issues. However the most important advantage, though, is that singletons can be handled polymorphically without forcing their users to assume that there is only one instance.

But there are some situations, where static classes makes sense than Singleton. Prime example of this is java.lang.Math which is not Singleton, instead a class with all static methods. Here are few situation where I think using static class over Singleton pattern make sense:

 1. If your Singleton is not maintaining any state, and just providing global access to methods, than consider using static class, as static methods are much faster than Singleton, because of static binding during compile time. But remember its not advised to maintain state inside static class, especially in concurrent environment, where it could lead subtle race conditions when modified parallel by multiple threads without adequate synchronization.
2. You can also choose to use static method, if you need to combine bunch of utility method together. Anything else, which requires singles access to some resource, should use Singleton design pattern.

##Advantage of Singleton Pattern over Static Class in Java

Main advantage of Singleton over static is that former is more object oriented than later. With Singleton, you can use Inheritance and Polymorphism to extend a base class, implement an interface and capable of providing different implementations. If we talk about java.lang.Runtime, which is a Singleton in Java, call to getRuntime() method return different implementations based on different JVM, but guarantees only one instance per JVM, had java.lang.Runtime an static class, it’s not possible to return different implementation for different JVM.

 There is not one universe. There are many: A multiverse. We have the technology to travel between universes, but travel is highly restricted and policed. There is not one you. There are many. Each of us exists in present time, in parallel universes. There was balance in the system, but now a force exists who seeks to destroy the balance so he can become The One.
