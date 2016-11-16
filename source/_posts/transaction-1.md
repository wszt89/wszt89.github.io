---
title: 事务问题（一）
date: 2016-11-09 21:19:52
tags: 
  - 事务
  - 问题
categories: 编程
---

## 写在前面

现在开发的系统对准确性要求非常高，而数据的一致性和准确性几乎完全依赖数据库的事务，如果事务跪了，结局是要多惨有多惨（经历过的都懂的）。

当然MySQL本身也有一些BUG但并不多，大部分的出错都是业务代码写残了，下面来看遇到过的一些例子！

## 常遇问题

下面的场景都是Java操作__MySQL/InnoDB__、事务隔离级别为__RC__时遇到的：

### 不生效之一：代码问题

不生效模板代码如下：

```java
public class Manager {
    public void func1() {
        func2();
    }

    @Transactional
    public void func2() {
        // 数据库操作
    }
}
```

调用<u>manager.func2</u>事务是生效的，而调用<u>manager.func1</u>时不生效！开始分析原因：

1. Spring对@Transactional修饰的方法进行AOP拦截处理；
2. Spring中的AOP是通过代理实现的；

那么，先来看个代理的例子（CGLIB）：

```java
public Object intercept(Object targe, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
    System.out.println("BEFORE");
    Object result = methodProxy.invokeSuper(targe, args);
    System.out.println("AFTER");
    return result;
}
```

因此，调用fun2时顺序为：

![image](http://git.cn-hangzhou.oss.aliyun-inc.com/uploads/tianchi.gzt/note/eda3409819f4c596a56b0062ffba8ca0/image.png)

而调用func1时顺序为：

![image](http://git.cn-hangzhou.oss.aliyun-inc.com/uploads/tianchi.gzt/note/7dbec4b5c57b765b31b887da6164d72e/image.png)

代理对象中发现并不需要代理<u>func1</u>方法（因为它上面没有注解），因此不会开启事务，到了target以后自己调用自己的<u>func2</u>，虽然方法上有@Transactional，但是target并不管它（只有代理对象才管），因此事务并不生效。

__解决办法__：在需要事务的入口方法上面都加注解（略暴力）。

### 不生效之二：配置问题

和事务相关的配置常用的有以下三个：

- __datasource__
- __transactionManager__：需要datasource
- __annotation-driven__：需要transactionManager

当<u>transactionManager</u>或者<u>annotation-driven</u>设置的<u>datasource</u>与访问数据库所使用的不一致时，会出现事务不生效！下面来从Spring对事务的管理方式上找答案：

在开启事务，会调用<u>TransactionSynchronizationManager.bindResource</u>将链接绑定到ThreadLocal上，其中key为datasource，value为ConnectionHolder：

```java
public static void bindResource(Object key, Object value) throws IllegalStateException {
    // 根据datasource生成真正的key
    Object actualKey = TransactionSynchronizationUtils.unwrapResourceIfNecessary(key);
    Assert.notNull(value, "Value must not be null");
    Map<Object, Object> map = resources.get();
    // 初始化map
    if (map == null) {
	    map = new HashMap<Object, Object>();
        resources.set(map);
    }
    Object oldValue = map.put(actualKey, value);
    // ....
}
```

执行SQL前会通过<u>TransactionSynchronizationManager.getResource</u>来获取链接：

```java
public static Object getResource(Object key) {
    Object actualKey = TransactionSynchronizationUtils.unwrapResourceIfNecessary(key);
    Object value = doGetResource(actualKey);
    if (value != null && logger.isTraceEnabled()) {
        logger.trace("Retrieved value [" + value + "] for key [" + actualKey + "] bound to thread [" +
            Thread.currentThread().getName() + "]");
    }
    return value;
}
```

于是，当transactionManager配置的datasource与写数据的datasource不一样时，开启事务和执行SQL会分别用两个链接，这样就导致：

- 执行数据库操作使用的链接A
- 回滚事务用的链接B

当然也就回滚不掉了，这种情况都是在切换数据源的时候遇到的，干这种活的时候小心点就行了！

### 在Commit成功之后没有数据

常遇到是事务不生效，前端时间遇到事务明明生效，但COMMIT后却没有数据，这时候就直接懵逼了！

现象是：

> A调用B服务，B写数据库成功并将数据库ID返回给A，但B对应的数据库中并没有数据。

是不是感觉见鬼了。首先，我们来对B服务的执行过程描述一下：

- 数据库链接是绑定到线程上的；
- 用线程池来处理服务请求，也就是说同一个线程可能会依次处理多个请求；

也就是说：__两次不同的请求可能会互相影响__，而恰巧在B中使用的手动事务，那么该线程执行的流程如下：

```java
@Test
public void test() throws Exception {
    /********** 第一次HSF调用 **********/

    // 开启事务
    DefaultTransactionDefinition def = new DefaultTransactionDefinition();
    def.setIsolationLevel(DefaultTransactionDefinition.ISOLATION_READ_COMMITTED);
    def.setPropagationBehavior(DefaultTransactionDefinition.PROPAGATION_REQUIRED);
    TransactionStatus status = transactionManager.getTransaction(def);

    // 由于各种原因没有终结

    /********** 第二次HSF调用 **********/

    // 调用被@Transactional拦截的方法。

    // 返回成功，但是数据没有落库。
}
```

在手动事务没有完成（回滚或提交）时，对应的线程又调用了@Transactional拦截的方法：

- 在方法开始前：认为已经在事务中了，所以不会开启新事务
- 在方法完成后：认为事务还没有处理完，所以并不会完结它

简单来说：__谁开启了事务，谁负责把它结束掉__。手动事务在用的时候一定要慎重，能不用就别用了。

## 总结

事务是应用保证的最后一道防线，必须引起足够的重视，而且，感觉用的越简单越可靠越好！