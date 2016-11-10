---
title: 事务问题（一）
date: 2016-11-09 21:19:52
tags: 
  - 事务
  - 问题
categories: 编程
---

## 写在前面

目前做的系统，数据的一致性和准确性非常非常依赖数据库的事务，当事务跪了，结局可能要多惨有多惨（经历过的都懂的）。

当然MySQL本身BUG不多，大部分的出错都是业务代码写残了。下面来看现实中遇到过的一些问题！

## 遇到的问题

下面的场景都是Java操作__MySQL/InnoDB__、事务隔离级别为__RC__时遇到的：

### 不生效之一：代码问题

通常都是用@Transactional来管理事务的开始和提交，而当对AOP的实现不是很熟悉时，可能导致事务不生效的代码如下：

```java
public class Manager {
    public void func1() {
        func2();
    }

    @Transactional
    public void func2() {

    }
}
```

使用<u>manager.func2</u>时，事务时生效的，而使用<u>manager.func1</u>时并没有生效！在Spring中通过代理来实现拦截：

```java
public Object intercept(Object targe, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
    System.out.println("BEFORE");
    Object result = methodProxy.invokeSuper(targe, args);
    System.out.println("AFTER");
    return result;
}
```

调用fun2时顺序为：

![image](http://git.cn-hangzhou.oss.aliyun-inc.com/uploads/tianchi.gzt/note/eda3409819f4c596a56b0062ffba8ca0/image.png)

而调用func1的时候，调用顺序为：

![image](http://git.cn-hangzhou.oss.aliyun-inc.com/uploads/tianchi.gzt/note/7dbec4b5c57b765b31b887da6164d72e/image.png)

代理对象中发现并不需要代理<u>func1</u>方法，因此不会开启事务，到了target以后只会调用自己的<u>func2</u>，因此事务不会生效。

__解决办法__：在需要事务的方法入口方法上面都需要加注解。

### 不生效之二：配置问题

和事务相关的配置有：

- __datasource__
- __transactionManager__：需要datasource
- __annotation-driven__：需要transactionManager

当<u>transactionManager</u>或者<u>annotation-driven</u>设置的<u>datasource</u>与真正使用的不一致时，会出现事务不生效。

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

在执行sql之前会通过<u>TransactionSynchronizationManager.getResource</u>来获取链接：

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

这样，当transactionManager配置的datasource与实际的datasource不一致时，开启事务和执行sql会分别创建两个链接，这样就导致：

- 执行数据库操作使用的链接A
- 回滚事务用的链接B

因此会出现回滚不掉，事务也就不生效了（一般只有在迁移数据库的时候会配两个数据源吧）。

### 在Commit成功之后没有数据

常遇到是事务不生效，而遇到事务明明生效了，但是commit之后却没有数据，这时候就懵逼了！在提供服务时：

- 用线程池来处理任务，也就是说同一个线程可能处理多个任务
- 链接是绑定到线程上的

这样就可能导致出一个线程用一个链接处理了两个任务，

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

- 在方法开始前，认为已经在事务中了，所以不会开启新事务
- 在方法完成后，认为事务还没有处理完，所以并不会完结它

__简单来说__：谁开启了事务，谁负责把它结束掉。手动事务在用的时候一定要慎重，能不用就尽量别用了。

## 总结

大部分的事务问题是因为用法不对，与其在遇到问题时大海捞针般地找问题，还不如平时多熟悉逻辑到时候DEBUG一遍就知道原因了。

另外，查原因、解问题是最好的学习和积累的机会。