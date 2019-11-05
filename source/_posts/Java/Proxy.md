---
title: 代理
date: 2018-09-11
categories: 
- Java

tags:
- AOP
- 设计模式
---

### 代理模式

代理模式的定义：为其他对象提供一种代理以控制对这个对象的访问。在某些情况下，一个对象不适合或者不能直接引用另一个对象，而代理对象可以在客户端和目标对象之间起到中介的作用。


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
如果要在实现hello接口的类中println前后都执行以下方法呢，如果把代码直接写里面感觉重复率太高了而且不够优雅，这个时候就要用代理模式

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
用HelloProxy类实现了Hello接口，测试以下

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

### JDK动态代理

除了Hello接口外又有Play接口，Eat接口怎么办，每一个都写一个代理还是很麻烦，到处都是XxxProxy，这个时候就需要动态代理了

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



### CGlib代理

动态代理比起静态代理方便的多，但是jdk动态代理实现必须通过接口，如果要代理一个没有接口的类jdk动态就无法实现了，这个时候就要借助CGlib这个类库来动态生成代理类(spring hibernate框架都使用了该类库，使用前要先导入)


```
package proxy;

import net.sf.cglib.proxy.Enhancer;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

import java.lang.reflect.Method;


public class CGlibProxy implements MethodInterceptor {

    public <T> T getProxy(Class<T> tClass) {
        return (T) Enhancer.create(tClass, this);
    }

    @Override
    public Object intercept(Object object, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        before();
        Object result = proxy.invokeSuper(object, args);
        after();
        return result;
    }

    public void before() {
        System.out.println("before");
    }

    public void after() {
        System.out.println("after");
    }

    public static void main(String[] args) {
        CGlibProxy cGlibProxy = new CGlibProxy();
        Hello helloProxy = cGlibProxy.getProxy(Hello.class);
        helloProxy.sayHello("world");
    }
}

```

关于Hello类:

```
package proxy;


public class Hello {

    public void sayHello(String name) {
        System.out.println("hello " + name);
    }
}

```
