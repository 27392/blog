# Maven插件 - versions-maven-plugin

> versions-maven-plugin插件主要可以用来管理项目版本

## 资料

[versions-maven-plugin 官方页面](https://www.mojohaus.org/versions-maven-plugin/)

## 起因

最近有个需求是拓展以前的功能(加字段的那种).由于改动的接口是以`jar`的方式提供给其他服务来调用

我在开发完成之后,很习惯的就更新了`jar`,准备在开发环境联调测试.就在这时我忽略了一点

忽略了一旦更新了`jar`所有有依赖的项目都会在下次构建时将会引用还没有调试好的接口,这就会(任何环境,包括生产)在构建时会发生错误

还好立刻就意识到了问题的严重性,然后立马还原代码将旧的代码打包后重新上传到私服

## 解决办法

其实想了一下解决办法很简单,我们只需要修改版本号即可,这样原有的项目也不会受到影响

在网上搜索后可以通过`maven`的有提供插件管理版本

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.codehaus.mojo</groupId>
            <artifactId>versions-maven-plugin</artifactId>
            <version>2.7</version>
        </plugin>
    </plugins>
</build>
```

在使用插件后可以直接在idea的图形化界面完成操作

当然也可以使用命令行完成操作

```
mvn versions:set -DnewVersion=版本号
```

例如我们现在版本是`0.0.1-SNAPSHOT`我们想把改为`0.0.2-SNAPSHOT`

```
mvn versions:set -DnewVersion=0.0.2-SNAPSHOT
```

当然在改变后是可以还原的

```xml
mvn versions:revert
```

如果确定修改可以执行下面这条命令,提交我们的修改

```xml
mvn versions:commit
```

至于更多的操作可以参考上面给出的官方地址

## 注意

> **切勿修改当前生产等其他环境所依赖的`jar`文件的等,如有修改务必提升版本**

**如果使用命令行操作的话必须在`pom.xml`所在的目录进行操作**




