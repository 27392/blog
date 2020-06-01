# Mybatis配置读取之Mapper文件读取

## 创建 SqlSessionFactory

```java
class Main{

    public static void main(String[] args){
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(reader);
    }

}
```

```java
class XMLConfigBuilder extends BaseBuilder {

    public SqlSessionFactory build(Reader reader) {
        // 调用重载方法
        return build(reader, null, null);
    }

    public SqlSessionFactory build(Reader reader, String environment, Properties properties) {
        // 1. new XMLConfigBuilder(reader, environment, properties)
        // 2. parser.parse()
        // 3. build(config)
        XMLConfigBuilder parser = new XMLConfigBuilder(reader, environment, properties);
        return build(parser.parse());
    }

    public SqlSessionFactory build(Configuration config) {
        return new DefaultSqlSessionFactory(config);
    }
}
```

## XMLConfigBuilder

1. new XMLConfigBuilder(reader, environment, properties)

```java
class XMLConfigBuilder extends BaseBuilder {

    public XMLConfigBuilder(Reader reader, String environment, Properties props) {
        // 调用重载的构造器
        this(new XPathParser(reader, true, props, new XMLMapperEntityResolver()), environment, props);
    }

    private XMLConfigBuilder(XPathParser parser, String environment, Properties props) {
        // 调用父类`BaseBuilder`构造器初始化 Configuration
        super(new Configuration());
    }
}
```

```java
public abstract class BaseBuilder {
  protected final Configuration configuration;
  protected final TypeAliasRegistry typeAliasRegistry;
  protected final TypeHandlerRegistry typeHandlerRegistry;

  public BaseBuilder(Configuration configuration) {
    this.configuration = configuration;
    // 初始化类型别名注册器
    this.typeAliasRegistry = this.configuration.getTypeAliasRegistry();
    // 初始化类型处理注册器
    this.typeHandlerRegistry = this.configuration.getTypeHandlerRegistry();
  }
}
```

> `XMLConfigBuilder`类是`BaseBuilder`的子类
>
> 它在初始化的时调用父类的构造器初始化`Configuration`对象

2. XMLConfigBuilder.parse()

```java
class XMLConfigBuilder extends BaseBuilder {

  public Configuration parse() {
    // 解析<configuration> 节点
    parseConfiguration(parser.evalNode("/configuration"));
    return configuration;
  }

  private void parseConfiguration(XNode root) {
    try {

      // 解析properties配置
      propertiesElement(root.evalNode("properties"));

      // 解析 settings 配置，并将其转换为 Properties 对象
      Properties settings = settingsAsProperties(root.evalNode("settings"));

      // 加载 vfs
      loadCustomVfs(settings);

      // 加载日志实现类
      loadCustomLogImpl(settings);

      // 解析别名配置
      typeAliasesElement(root.evalNode("typeAliases"));

      // 解析插件
      pluginElement(root.evalNode("plugins"));

      // 解析对象工厂
      objectFactoryElement(root.evalNode("objectFactory"));

      // 解析 objectWrapperFactory 配置
      objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));

      // 解析 reflectorFactory 配置
      reflectorFactoryElement(root.evalNode("reflectorFactory"));

      // 将setting中的信息设置到`Configuration`对象中
      settingsElement(settings);

      // 解析 environments 配置
      environmentsElement(root.evalNode("environments"));

      // 解析 databaseIdProvider 配置
      databaseIdProviderElement(root.evalNode("databaseIdProvider"));

      // 解析 typeHandlers 配置
      typeHandlerElement(root.evalNode("typeHandlers"));

      // 解析 mappers 配置
      mapperElement(root.evalNode("mappers"));

    } catch (Exception e) {
      throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
  }
}
```

### 解析 mappers 配置

```java
class XMLConfigBuilder extends BaseBuilder {

  private void mapperElement(XNode parent) throws Exception {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        if ("package".equals(child.getName())) {
          String mapperPackage = child.getStringAttribute("name");
          configuration.addMappers(mapperPackage);
        } else {
          // 获取 resource、url、class 等属性
          String resource = child.getStringAttribute("resource");
          String url = child.getStringAttribute("url");
          String mapperClass = child.getStringAttribute("class");

          // resource 不为空,且其他两者为空,则从指定路径中加载配置
          if (resource != null && url == null && mapperClass == null) {
            ErrorContext.instance().resource(resource);

            // 加载 resource 所指定的 xml 文件
            InputStream inputStream = Resources.getResourceAsStream(resource);

            // 1. 创建`XMLMapperBuilder`对象
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());

            // 2. 解析 `Mapper.xml` 文件
            mapperParser.parse();

          } else if (resource == null && url != null && mapperClass == null) {
            ErrorContext.instance().resource(url);

            // 从网络上加载 xml 文件, 同样是使用 XMLMapperBuilder 来解析
            InputStream inputStream = Resources.getUrlAsStream(url);
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
            mapperParser.parse();
          } else if (resource == null && url == null && mapperClass != null) {
            Class<?> mapperInterface = Resources.classForName(mapperClass);
            configuration.addMapper(mapperInterface);
          } else {
            throw new BuilderException("A mapper element may only specify a url, resource or class, but not more than one.");
          }
        }
      }
    }
  }
}
```

#### XMLMapperBuilder

1. new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());

```java
class XMLMapperBuilder extends BaseBuilder {
    public XMLMapperBuilder(InputStream inputStream, Configuration configuration, String resource, Map<String, XNode> sqlFragments) {
        // 调用重载构造器
        this(new XPathParser(inputStream, true, configuration.getVariables(), new XMLMapperEntityResolver()),
            configuration, resource, sqlFragments);
    }

    private XMLMapperBuilder(XPathParser parser, Configuration configuration, String resource, Map<String, XNode> sqlFragments) {
        // 接受 XMLConfigBuilder 类的 configuration , 所以不必再次初始化
        super(configuration);
        this.builderAssistant = new MapperBuilderAssistant(configuration, resource);
        this.parser = parser;
        this.sqlFragments = sqlFragments;
        this.resource = resource; 
    }
}
```

2. XMLMapperBuilder.parse();

```java
class XMLMapperBuilder extends BaseBuilder {

  public void parse() {

    // 检查文件是否已经被解析 `Configuration`对象内的 loadedResources 是否包含
    if (!configuration.isResourceLoaded(resource)) {

      // 解析 <mapper> 节点 , -- 最后的操作就是将 将解析<mapper> 节点的数据封装在 `MappedStatement` 中并添加到 `configuration` 的 `mappedStatements` (Map, key(id[也就是namespace.id]) / value(statement对象))
      configurationElement(parser.evalNode("/mapper"));

      // 解析完成之后添加到`Configuration`对象内的 loadedResources (Set)中
      configuration.addLoadedResource(resource);

      // 通过命名空间绑定 Mapper 接口 , -- 将 Mapper接口 添加到 configuration 属性 mapperRegistry 类中的 knownMappers(map)
      bindMapperForNamespace();

    }

    parsePendingResultMaps();
    parsePendingCacheRefs();
    parsePendingStatements();

  }
}
```

#### configurationElement

```java
class XMLMapperBuilder extends BaseBuilder {
    private void configurationElement(XNode context) {
        try {
    
          // 获取 mapper 命名空间
          String namespace = context.getStringAttribute("namespace");
          if (namespace == null || namespace.equals("")) {
            throw new BuilderException("Mapper's namespace cannot be empty");
          }
    
          // 将命名空间设置在`builderAssistant`中
          builderAssistant.setCurrentNamespace(namespace);
    
          // 解析 <cache-ref> 节点
          cacheRefElement(context.evalNode("cache-ref"));
    
          // 解析 <cache> 节点
          cacheElement(context.evalNode("cache"));
    
          // 老式风格的参数映射。此元素已被废弃，并可能在将来被移除！
          parameterMapElement(context.evalNodes("/mapper/parameterMap"));
    
          // 解析 <resultMap> 节点
          resultMapElements(context.evalNodes("/mapper/resultMap"));
    
          // 解析 <sql> 节点
          sqlElement(context.evalNodes("/mapper/sql"));
    
          // 解析 <select>、<insert>、<update>、<delete> 节点
          buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    
        } catch (Exception e) {
          throw new BuilderException("Error parsing Mapper XML. The XML location is '" + resource + "'. Cause: " + e, e);
        }
    }

    private void buildStatementFromContext(List<XNode> list) {
        if (configuration.getDatabaseId() != null) {
          buildStatementFromContext(list, configuration.getDatabaseId());
        }
        buildStatementFromContext(list, null);
    }
    
    private void buildStatementFromContext(List<XNode> list, String requiredDatabaseId) {

        for (XNode context : list) {
        
          // 构建 XMLStatementBuilder 对象
          final XMLStatementBuilder statementParser = new XMLStatementBuilder(configuration, builderAssistant, context, requiredDatabaseId);
        
          try {
        
            // 解析 <select>、<insert>、<update>、<delete> 节点
            statementParser.parseStatementNode();
        
          } catch (IncompleteElementException e) {
            configuration.addIncompleteStatement(statementParser);
          }
        }
    }
}
```

##### XMLStatementBuilder

1. new XMLStatementBuilder(configuration, builderAssistant, context, requiredDatabaseId);

```java
class XMLStatementBuilder extends BaseBuilder {
    // 接受 XMLMapperBuilder 类的 configuration、builderAssistant
    public XMLStatementBuilder(Configuration configuration, MapperBuilderAssistant builderAssistant, XNode context, String databaseId) {
        super(configuration);
        this.builderAssistant = builderAssistant;
        this.context = context;
        this.requiredDatabaseId = databaseId;
    }
}
```

2. XMLStatementBuilder.parseStatementNode();

```java
class XMLStatementBuilder extends BaseBuilder {

  public void parseStatementNode() {

    // 获取 id 和 databaseId 属性
    String id = context.getStringAttribute("id");
    String databaseId = context.getStringAttribute("databaseId");

    if (!databaseIdMatchesCurrent(id, databaseId, this.requiredDatabaseId)) {
      return;
    }

    String nodeName = context.getNode().getNodeName();
    SqlCommandType sqlCommandType = SqlCommandType.valueOf(nodeName.toUpperCase(Locale.ENGLISH));
    boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
    boolean flushCache = context.getBooleanAttribute("flushCache", !isSelect);
    boolean useCache = context.getBooleanAttribute("useCache", isSelect);
    boolean resultOrdered = context.getBooleanAttribute("resultOrdered", false);

    // 解析 <include> 节点
    XMLIncludeTransformer includeParser = new XMLIncludeTransformer(configuration, builderAssistant);
    includeParser.applyIncludes(context.getNode());

    String parameterType = context.getStringAttribute("parameterType");
    Class<?> parameterTypeClass = resolveClass(parameterType);

    String lang = context.getStringAttribute("lang");
    LanguageDriver langDriver = getLanguageDriver(lang);

    // Parse selectKey after includes and remove them.
    processSelectKeyNodes(id, parameterTypeClass, langDriver);

    // Parse the SQL (pre: <selectKey> and <include> were parsed and removed)
    KeyGenerator keyGenerator;
    String keyStatementId = id + SelectKeyGenerator.SELECT_KEY_SUFFIX;
    keyStatementId = builderAssistant.applyCurrentNamespace(keyStatementId, true);
    if (configuration.hasKeyGenerator(keyStatementId)) {
      keyGenerator = configuration.getKeyGenerator(keyStatementId);
    } else {
      keyGenerator = context.getBooleanAttribute("useGeneratedKeys",
          configuration.isUseGeneratedKeys() && SqlCommandType.INSERT.equals(sqlCommandType))
          ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
    }

    SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass);

    // 解析 Statement 类型，默认为 PREPARED
    StatementType statementType = StatementType.valueOf(context.getStringAttribute("statementType", StatementType.PREPARED.toString()));

    Integer fetchSize = context.getIntAttribute("fetchSize");
    Integer timeout = context.getIntAttribute("timeout");
    String parameterMap = context.getStringAttribute("parameterMap");
    String resultType = context.getStringAttribute("resultType");
    Class<?> resultTypeClass = resolveClass(resultType);
    String resultMap = context.getStringAttribute("resultMap");
    String resultSetType = context.getStringAttribute("resultSetType");
    ResultSetType resultSetTypeEnum = resolveResultSetType(resultSetType);
    if (resultSetTypeEnum == null) {
      resultSetTypeEnum = configuration.getDefaultResultSetType();
    }
    String keyProperty = context.getStringAttribute("keyProperty");
    String keyColumn = context.getStringAttribute("keyColumn");
    String resultSets = context.getStringAttribute("resultSets");

    // 构建 MappedStatement 添加至 `Configuration` 中的 `mappedStatements` 集合中
    builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType,
        fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass,
        resultSetTypeEnum, flushCache, useCache, resultOrdered,
        keyGenerator, keyProperty, keyColumn, databaseId, langDriver, resultSets);
  }

  public MappedStatement addMappedStatement(
      String id,
      SqlSource sqlSource,
      StatementType statementType,
      SqlCommandType sqlCommandType,
      Integer fetchSize,
      Integer timeout,
      String parameterMap,
      Class<?> parameterType,
      String resultMap,
      Class<?> resultType,
      ResultSetType resultSetType,
      boolean flushCache,
      boolean useCache,
      boolean resultOrdered,
      KeyGenerator keyGenerator,
      String keyProperty,
      String keyColumn,
      String databaseId,
      LanguageDriver lang,
      String resultSets) {

    if (unresolvedCacheRef) {
      throw new IncompleteElementException("Cache-ref not yet resolved");
    }

    // 拼接 id 与 namespace (namespace + "." + base)

    id = applyCurrentNamespace(id, false);
    boolean isSelect = sqlCommandType == SqlCommandType.SELECT;

    // 构建 MappedStatement 对象
    MappedStatement.Builder statementBuilder = new MappedStatement.Builder(configuration, id, sqlSource, sqlCommandType)
        .resource(resource)
        .fetchSize(fetchSize)
        .timeout(timeout)
        .statementType(statementType)
        .keyGenerator(keyGenerator)
        .keyProperty(keyProperty)
        .keyColumn(keyColumn)
        .databaseId(databaseId)
        .lang(lang)
        .resultOrdered(resultOrdered)
        .resultSets(resultSets)
        .resultMaps(getStatementResultMaps(resultMap, resultType, id))
        .resultSetType(resultSetType)
        .flushCacheRequired(valueOrDefault(flushCache, !isSelect))
        .useCache(valueOrDefault(useCache, isSelect))
        .cache(currentCache);

    ParameterMap statementParameterMap = getStatementParameterMap(parameterMap, parameterType, id);
    if (statementParameterMap != null) {
      statementBuilder.parameterMap(statementParameterMap);
    }

    // 将MappedStatement 添加到 configuration 的 mappedStatements (Map, key(id[也就是namespace.id]) /value(statement对象))
    MappedStatement statement = statementBuilder.build();
    configuration.addMappedStatement(statement);
    return statement;
  }
}
```

#### bindMapperForNamespace

```java
class XMLMapperBuilder extends BaseBuilder {

  private void bindMapperForNamespace() {

    // 获取 namespace 命名空间
    String namespace = builderAssistant.getCurrentNamespace();

    if (namespace != null) {
      Class<?> boundType = null;
      try {

        // 根据命名空间加载类型 (mapper类型)
        boundType = Resources.classForName(namespace);

      } catch (ClassNotFoundException e) {
        //ignore, bound type is not required
      }
      if (boundType != null) {

        // 判断当前 mapper 是否在 configuration 属性 mapperRegistry 类中的 knownMappers(map) 存在
        if (!configuration.hasMapper(boundType)) {

          // Spring may not know the real resource name so we set a flag
          // to prevent loading again this resource from the mapper interface
          // look at MapperAnnotationBuilder#loadXmlResource
          configuration.addLoadedResource("namespace:" + namespace);

          // 添加到 configuration 属性 mapperRegistry 类中的 knownMappers(map)
          configuration.addMapper(boundType);
        }
      }
    }
  }
}
```

```java
class Configuration {

  protected final MapperRegistry mapperRegistry = new MapperRegistry(this);

  public <T> void addMapper(Class<T> type) {

    // 将Mapper类添加到 MapperRegistry 中
    mapperRegistry.addMapper(type);
  }
}
```

```java
class MapperRegistry {

  private final Configuration config;
  private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap<>();

  public <T> void addMapper(Class<T> type) {

    // 判断是否是接口
    if (type.isInterface()) {
      if (hasMapper(type)) {
        throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
      }
      boolean loadCompleted = false;

      try {

        // 将 mapper 类型与 MapperProxyFactory 类型绑定, MapperProxyFactory 类可以为Mapper接口生成动态代理
        knownMappers.put(type, new MapperProxyFactory<>(type));

        // 解析注解
        // It's important that the type is added before the parser is run
        // otherwise the binding may automatically be attempted by the
        // mapper parser. If the type is already known, it won't try.
        MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
        parser.parse();
        loadCompleted = true;
      } finally {
        if (!loadCompleted) {
          knownMappers.remove(type);
        }
      }
    }
  }
}
```
