---
title: JDK动态代理
date: 2018-09-11
categories: 
- Java

tags:
- AOP
---


首先来了解以下什么是代理模式

先来一个hello接口
```
public interface Hello {

    void sayHello(String name);
}
```

<!--more-->

以下是hello接口的实现类

```
public class HelloImpl implements Hello {

    @Override
    public void sayHello(String name) {
        System.out.println("hello," + name);
    }
}
```
如果要在实现hello接口的类中pritln前后都执行以下方法呢，如果把代码直接写里面感觉重复率太高了而且不够优雅，这个时候就要用代理模式

```
public class HelloProxy implements Hello {

    private Hello hello;

    public HelloProxy() {
        this.hello = new HelloImpl();
    }

    @Override
    public void sayHello(String name) {
        before();
        hello.sayHello(name);
        after();
    }

    public void before() {
        System.out.println("before");
    }

    public void after() {
        System.out.println("after");
    }
}
```
用HellpProxy类实现了Hello接口，测试以下

```
    public static void main(String[] args) {
        Hello helloProxy = new HelloProxy();
        helloProxy.sayHello("biubiubiu");
    }
```
打印结果：

```
before
hello,biubiubiu
after
```
可是除了Hello接口外又有Play接口，Eat接口怎么办，每一个都写一个代理还是很麻烦，到处都是XxxProxy，这个时候就需要动态代理了

下面是jdk提供的动态代理方案写的一个DynamicProxy:

```
public class DynamicProxy implements InvocationHandler {
    
    private Object target;

    public DynamicProxy(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        before();
        Object result = method.invoke(target, args);
        after();
        return result;
    }

    public void before() {
        System.out.println("before");
    }

    public void after() {
        System.out.println("after");
    }
}
```
通过反射，实现jdk提供的InvocationHandler接口，使用方法：

```
    public static void main(String[] args) {

        Hello hello = new HelloImpl();

        DynamicProxy dynamicProxy = new DynamicProxy(hello);

        Hello helloProxy = (Hello) Proxy.newProxyInstance(hello.getClass().getClassLoader(), hello.getClass().getInterfaces(), dynamicProxy);
        
        helloProxy.sayHello("wahaha");
    }
```
通过DynamicProxy包装HelloImpl实例，通过Proxy的工厂方法newProxyInstance动态地创建一个Hello接口的代理类，调用sayHello方法

还可以封装一层

```
    @SuppressWarnings("unchecked")//忽略警告
    public <T> T getProxy() {
        return (T) Proxy.newProxyInstance(target.getClass().getClassLoader(), target.getClass().getInterfaces(), this);
    }

    public static void main(String[] args) {

        DynamicProxy dynamicProxy = new DynamicProxy(new HelloImpl());
        Hello helloProxy = dynamicProxy.getProxy();
        helloProxy.sayHello("biubiubiu");
    }
```
