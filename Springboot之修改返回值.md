# SpringBoot之修改返回值

最近在做字典业务,涉及到字典的话必然就有字典的转换

当然普通的字典显然可以满足大多数的需求,而我们这此所考虑的点是可以当用户支持改名操作

这样一来业务端在转换字典的时候,必然会很繁琐(先去查用户自定义字典,不存在去查询系统字典)

## 最开始的想法

最开始想的将字典转换的操作封装,然后以`jar`的方式提供给多个服务

提供`DictionaryService`服务,根据字典类型和字典`key`来转换

但是这样一来所有的业务只要涉及到字典转换的都要其他开发人员手动调用这个类来进行转换

这样很显然可以解决问题,但是我并不想这样去做. 感觉上对原来的代码有侵入性

## 在返回值上做手脚

有了上面一版的方案,后面又在想.如果不想让业务端处理,我是不是可以在返回值上给他做转换呢?

有了思路就开始实现

### HandlerMethodReturnValueHandler

首先我们可以先看看`@ResponseBody`是怎么把结果转成`JSON`的

通过查阅资料知道它是通过`RequestResponseBodyMethodProcessor`类的`handleReturnValue`方法处理

而`handleReturnValue`方法时继承自`HandlerMethodReturnValueHandler`接口

是不是可以通过自己实现`HandlerMethodReturnValueHandler`接口来修改我们的返回值呢?

但是在`RequestMappingHandlerAdapter`类中`getDefaultReturnValueHandlers`,使用了该类

通过方法名可以得知,该方法是获取默认的返回值处理程序,看了一下替换还是很麻烦,谁让我是一个嫌麻烦的人呢

### ResponseBodyAdvice

而在`handleReturnValue`方法中有一行注释`Try even with null return value. ResponseBodyAdvice could get involved.`

```java
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
        ModelAndViewContainer mavContainer, NativeWebRequest webRequest)
        throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {

    mavContainer.setRequestHandled(true);
    ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
    ServletServerHttpResponse outputMessage = createOutputMessage(webRequest);

    // Try even with null return value. ResponseBodyAdvice could get involved.
    writeWithMessageConverters(returnValue, returnType, inputMessage, outputMessage);
}
```

其中提到了`ResponseBodyAdvice`类.通过查阅资料,和自己动手试验发现使用该类非常简单就实现了我们的目的

```java
@ControllerAdvice
public class DictionaryAdvice implements ResponseBodyAdvice {

    // 通过该方法返回的参数判断是否调用`beforeBodyWrite`方法
    @Override
    public boolean supports(MethodParameter returnType, Class converterType) {
        // 这里可以加判断
        return true;
    }

    @Override
    public Object beforeBodyWrite(Object body, MethodParameter returnType, MediaType selectedContentType, Class selectedConverterType, ServerHttpRequest request, ServerHttpResponse response) {
        // 对返回值进行修改
        return body;
    }
}
```

### HttpMessageConverter

在查询`@ResponseBody`如果处理返回值时,我看到很多资料中提到了`HttpMessageConverter`(消息转换器)这么个东西

然后我寻找`HttpMessageConverter`的资料,发现它是用来将我们请求(`Request`),或者返回(`Response`)的参数去进行转换

讲到这里是不是知道了,我们可以在我们的对象还没有在转换前拿到并修改它

这里拿`JSON`转换器举例,因为我项目中都是`JSON`传输.

而在`SpringBoot`中默认的`JSON`转换器是`MappingJackson2HttpMessageConverter`

```java
public class DictionaryHttpMessageConverter extends MappingJackson2HttpMessageConverter {

    @Override
    protected void writeInternal(Object object, Type type, HttpOutputMessage outputMessage) throws IOException, HttpMessageNotWritableException {
        // object 就是要转换的数据
        super.writeInternal(object, type, outputMessage);
    }
}
```

首先通过继承`MappingJackson2HttpMessageConverter`并重写`writeInternal`方法

这个时候是不会生效的,因为现在还是用的默认的`MappingJackson2HttpMessageConverter`

我们需要将默认的`Bean`替换成我们自己的,如下:

```java
@Primary    //主要
@Bean
public MappingJackson2HttpMessageConverter DictionaryHttpMessageConverter() {
    log.info("create dictionary converter ...");
    return new DictionaryHttpMessageConverter();
}
```

## 总结

- `HandlerMethodReturnValueHandler` 

  理论上可以实现,但是现在好像没有找到资料,我也不愿意研究(嫌麻烦~)

- `ResponseBodyAdvice`              

  可以实现,相对简单.但是需要理解一下`@ControllerAdvice`注解的意思

- `HttpMessageConverter`            

  可以实现,只需要覆盖一个方法即可,**但是需要替换掉原有的转换器,不然不生效**

> `ResponseBodyAdvice`能在`HttpMessageConverter`前拿到结果

最后我使用`HttpMessageConverter`转换器的方法实现转换,在配合使用自定义注解`DictionaryParam`,添加在需要转换的对象属性上,当读取到该注解时才会进行转换

本次说的是修改返回值的几个方法,所以具体的字典转换细节并不会详说,具体的实现(修改返回值)还要看实际情况

[`ResponseBodyAdvice` 参考资料](https://www.cnblogs.com/chenss15060100790/p/9095584.html)

[`@ControllerAdvice` 参考资料](https://blog.csdn.net/zxfryp909012366/article/details/82955259)

[`@RequestBody和@ResponseBody` 参考资料](https://my.oschina.net/u/2377110/blog/1552979)

[`HttpMessageConverter` 参考资料](https://xuanjian1992.top/2019/08/14/Spring-MVC-HttpMessageConverter%E8%BD%AC%E6%8D%A2%E8%AF%B7%E6%B1%82%E5%92%8C%E5%93%8D%E5%BA%94%E6%95%B0%E6%8D%AE%E7%9A%84%E8%BF%87%E7%A8%8B%E5%88%86%E6%9E%90/)