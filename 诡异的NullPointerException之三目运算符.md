# 诡异的NullPointerException之三目运算符

## 起因

上周在工作时,有一段代码一直出现`NullPointerException`异常.通过日志定位后发现是一行三元运算符

```java
Integer b = false ? 0 : a;
```

这里的`a`也是`Integer`类型,完整就是下面这样

```java
Integer a = null;
Integer b = false ? 0 : a;
```

当时想呀就算是`a`是`null`的话也没问题吧,怎么会出现空指针呢?这是什么鬼操作?

然后问了下旁边的同事,他表示也不知道,所以我只能请教度娘

## 解开谜团

在搜索一番后,我发现是因为JDK自动拆箱的原因

通过`javap -c` 命令可以看到

```text
3: invokevirtual #2                  // Method java/lang/Integer.intValue:()I
6: invokestatic  #3                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
```

先对`a`进行拆箱(`Integer.intValue`),然后计算的结果在进行装箱(`Integer.valueOf`)赋值给`b`

**问题就出在JDK自动帮我们把`null`给拆箱了,也就是调用了`Integer.intValue:()`方法**

## 解决办法

可以通过统一类型来解决

```java
Integer a = null;
Integer b = false ? Integer.valueOf(0) : a;
```

