# Maven插件 - maven-source-plugin

> maven-source-plugin 插件主要可以用来生成源码包

## 资料

[maven-source-plugin 官方页面](http://maven.apache.org/plugins/maven-source-plugin/index.html)

[maven 生命周期 官方页面](http://maven.apache.org/ref/3.6.3/maven-core/lifecycles.html)

本次只需要注意到`default`生命周期即可.**`deploy`是`default`生命周期的最后一个周期**

## 起因

最近因为偷懒在上传jar包到私服时并没有上传源码,直接就在idea的maven插件中执行`deploy`

因为没有源码,同事在使用的过程中需要看源码就看不到不知道我在写啥,他们看的都是idea反编译后的内容

这就很烦了,有的时候就是懒或者忘记了输命令

   - 命令行 `maven source:jar deploy`

于是我开始搜索,怎么用插件来替代这行命令

一番过后我找到`maven-source-plugin`这么一个插件,通过网上其他人的例子和官网的介绍,知道了大概的用法.如下:

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-source-plugin</artifactId>
            <version>3.2.0</version>
            <executions>
                <execution>
                    <id>attach-sources</id>
                    <phase>verify</phase>
                    <goals>
                        <goal>jar-no-fork</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

**`phase` 为生命周期,官网给的例子默认是`verify`(验证),不填默认是`package`(打包)**

而我在想我既然是在`deploy`生命周期的话,我肯定要改成`deploy`

```xml
...
<execution>
    <id>attach-sources</id>
    <phase>deploy</phase>   //修改为 `deploy`
    <goals>
        <goal>jar-no-fork</goal>
    </goals>
</execution>
...
```

## 注意点

当我把生命周期改为`deploy`执行后发现并为生效

后面我发现命令是在生命周期后执行的等价于`maven deploy source:jar`

**然后我将生命周期改成`verify`后成功解决,因为`verify`在`deploy`周期之前**

> 命令是在你指定的生命周期后执行,需要了解Maven的生命周期
