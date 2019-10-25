---
title: CGlib动态代理

categories: 
- Java

tags:
- AOP
---

动态代理比起静态代理方便的多，但是jdk动态代理实现必须通过接口，如果要代理一个没有接口的类jdk动态就无法实现了，这个时候就要借助CGlib这个类库来动态生成代理类(spring hibernate框架都使用了该类库，使用前要先导入)

<!--more-->


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
