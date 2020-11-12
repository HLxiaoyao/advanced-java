# MyBatis配置文件解析源码分析
## 配置文件解析入口
```
String resource = "mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
```
首先使用MyBatis提供的工具类Resources加载配置文件，得到一个输入流。然后再通过 SqlSessionFactoryBuilder对象的build方法构建SqlSessionFactory对象。所以这里的build方法是分析配置文件解析过程的入口方法。那下面来看一下这个方法的代码：
```
// -☆- SqlSessionFactoryBuilder
public SqlSessionFactory build(InputStream inputStream) {
    // 调用重载方法
    return build(inputStream, null, null);
}

public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
    try {
        // 创建配置文件解析器
        XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
        // 调用 parse 方法解析配置文件，生成 Configuration 对象
        return build(parser.parse());
    } catch (Exception e) {
        throw ExceptionFactory.wrapException("Error building SqlSession.", e);
    } finally {
        ErrorContext.instance().reset();
        try {
        inputStream.close();
        } catch (IOException e) {
        // Intentionally ignore. Prefer previous error.
        }
    }
}

public SqlSessionFactory build(Configuration config) {
    // 创建 DefaultSqlSessionFactory
    return new DefaultSqlSessionFactory(config);
}
```
从上面的代码中，可以猜出MyBatis配置文件是通过XMLConfigBuilder进行解析的。不过目前这里还没有非常明确的解析逻辑，所以继续往下看。这次来看一下XMLConfigBuilder的parse方法，如下：
```
public Configuration parse() {
    // 配置文件已经被解析过，避免重复解析
    if (parsed) {
        throw new BuilderException("Each XMLConfigBuilder can only be used once.");
    }
    parsed = true;
    // 解析mybatis-config.xml中的各项配置，填充Configuration对象
    this.parseConfiguration(parser.evalNode("/configuration"));
    return configuration;
}
```
配置文件mybatis-config.xml以<configuration/>标签作为配置文件根节点，XMLConfigBuilder#parseConfiguration方法对配置文件的各个元素进行解析，并封装解析结果到Configuration对象中，最终返回该配置对象。方法实现如下：
```
private void parseConfiguration(XNode root) {
    try {
        // 解析 <properties/> 配置
        this.propertiesElement(root.evalNode("properties"));
        // 解析 <settings/> 配置
        Properties settings = this.settingsAsProperties(root.evalNode("settings"));
        // 获取并设置 vfsImpl 属性
        this.loadCustomVfs(settings);
        // 获取并设置 logImpl 属性
        this.loadCustomLogImpl(settings);
        // 解析 <typeAliases/> 配置
        this.typeAliasesElement(root.evalNode("typeAliases"));
        // 解析 <plugins/> 配置
        this.pluginElement(root.evalNode("plugins"));
        // 解析 <objectFactory/> 配置
        this.objectFactoryElement(root.evalNode("objectFactory"));
        // 解析 <objectWrapperFactory/> 配置
        this.objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
        // 解析 <reflectorFactory/> 配置
        this.reflectorFactoryElement(root.evalNode("reflectorFactory"));
        // 将 settings 配置设置到 Configuration 对象中
        this.settingsElement(settings);
        // 解析 <environments/> 配置
        this.environmentsElement(root.evalNode("environments"));
        // 解析 <databaseIdProvider/> 配置
        this.databaseIdProviderElement(root.evalNode("databaseIdProvider"));
        // 解析 <typeHandlers/> 配置
        this.typeHandlerElement(root.evalNode("typeHandlers"));
        // 解析 <mappers/> 配置
        this.mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
        throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
}
```
## 解析properties标签
首先看一下<properties/>标签怎么配置的，其中的配置项可以在整个配置文件中用来动态替换占位符。配置项可以从外部properties文件读取，也可以通过<property/>子标签指定。例如通过该标签指定数据源配置，如下：
```
<properties resource="datasource.properties">
    <property name="driver" value="com.mysql.jdbc.Driver"/>
    <!--为占位符启用默认值配置，默认关闭，需要采用如下方式开启-->
    <property name="org.apache.ibatis.parsing.PropertyParser.enable-default-value" value="true"/>
</properties>

文件datasource.properties内容如下：
url=jdbc:mysql://localhost:3306/test
username=root
password=123456
```
然后可以基于 OGNL 表达式在其它配置项中引用这些配置值，如下：
```
<dataSource type="POOLED"> <!--or UNPOOLED or JNDI-->
    <property name="driver" value="${driver}"/>
    <property name="url" value="${url}"/>
    <property name="username" value="${username:zhenchao}"/> <!--占位符设置默认值，需要专门开启-->
    <property name="password" value="${password}"/>
</dataSource>
```
其中，除了driver属性值来自<property/>子标签，其余属性值均是从datasource.properties配置文件中获取的。MyBatis 针对配置的读取顺序约定如下：

    （1）在 <properties/> 标签体内指定的属性首先被读取；
    （2）然后，根据 <properties/> 标签中resource属性读取类路径下配置文件，或根据 url 属性指定的路径读取指向的配置文件，并覆盖已读取的同名配置项；
    （3）最后，读取方法参数传递的配置项，并覆盖已读取的同名配置项。

<properties/>标签的解析过程，由XMLConfigBuilder#propertiesElement方法实现：
```
private void propertiesElement(XNode context) throws Exception {
    if (context != null) {
        // 获取 <property/> 子标签列表，封装成Properties对象
        Properties defaults = context.getChildrenAsProperties();
        // 支持通过 resource 或 url 属性指定外部配置文件
        String resource = context.getStringAttribute("resource");
        String url = context.getStringAttribute("url");
        // 这两种类型的配置是互斥的
        if (resource != null && url != null) {
            throw new BuilderException("The properties element cannot specify both a URL " +
                "and a resource based property file reference.  Please specify one or the other.");
        }
        // 从类路径加载配置文件
        if (resource != null) {
            defaults.putAll(Resources.getResourceAsProperties(resource));
        }
        // 从 url 指定位置加载配置文件
        else if (url != null) {
            defaults.putAll(Resources.getUrlAsProperties(url));
        }
        // 合并已有的配置项
        Properties vars = configuration.getVariables();
        if (vars != null) {
            defaults.putAll(vars);
        }
        // 填充 XPathParser 和 Configuration 对象
        parser.setVariables(defaults);
        configuration.setVariables(defaults);
    }
}

public Properties getChildrenAsProperties() {
    Properties properties = new Properties();
    for (XNode child : getChildren()) {
      String name = child.getStringAttribute("name");
      String value = child.getStringAttribute("value");
      if (name != null && value != null) {
        properties.setProperty(name, value);
      }
    }
    return properties;
}
```
## 解析settings标签
MyBatis通过<settings/>标签提供一些全局性的配置，这些配置会影响MyBatis的运行行为。
```
<settings>
    <setting name="cacheEnabled" value="true"/>
    <setting name="lazyLoadingEnabled" value="true"/>
    <setting name="multipleResultSetsEnabled" value="true"/>
    <setting name="useColumnLabel" value="true"/>
    <setting name="useGeneratedKeys" value="false"/>
    <setting name="autoMappingBehavior" value="PARTIAL"/>
    <setting name="autoMappingUnknownColumnBehavior" value="WARNING"/>
    <setting name="defaultExecutorType" value="SIMPLE"/>
    <setting name="defaultStatementTimeout" value="25"/>
    <setting name="defaultFetchSize" value="100"/>
    <setting name="safeRowBoundsEnabled" value="false"/>
    <setting name="mapUnderscoreToCamelCase" value="false"/>
    <setting name="localCacheScope" value="SESSION"/>
    <setting name="jdbcTypeForNull" value="OTHER"/>
    <setting name="lazyLoadTriggerMethods" value="equals,clone,hashCode,toString"/>
</settings>
```
首先调用XMLConfigBuilder#settingsAsProperties方法获取配置项对应的Properties对象，同时会检查配置项是否是可识别的，实现如下：
```
private Properties settingsAsProperties(XNode context) {
    if (context == null) {
        return new Properties();
    }
    // 解析 <setting/> 配置，封装成 Properties 对象
    Properties props = context.getChildrenAsProperties();
    // 构造 Configuration 对应的 MetaClass 对象，用于对 Configuration 类提供反射操作
    MetaClass metaConfig = MetaClass.forClass(Configuration.class, localReflectorFactory);
    // 遍历配置项，确保配置项是 MyBatis 可识别的
    for (Object key : props.keySet()) {
        // 属性对应的 setter 方法不存在
        if (!metaConfig.hasSetter(String.valueOf(key))) {
            throw new BuilderException(
                "The setting " + key + " is not known.  Make sure you spelled it correctly (case sensitive).");
        }
    }
    return props;
}
```
## 设置settings配置到Configuration
前面介绍了settings配置的解析过程，这些配置解析出来要有一个存放的地方，以使其他代码可以找到这些配置。这个存放地方就是Configuratio 对象，本节就来看一下这将 settings配置设置到 Configuration对象中的过程。如下：
```
private void settingsElement(Properties props) throws Exception {
    // 设置 autoMappingBehavior 属性，默认值为 PARTIAL
    configuration.setAutoMappingBehavior(AutoMappingBehavior.valueOf(props.getProperty("autoMappingBehavior", "PARTIAL")));
    configuration.setAutoMappingUnknownColumnBehavior(AutoMappingUnknownColumnBehavior.valueOf(props.getProperty("autoMappingUnknownColumnBehavior", "NONE")));
    // 设置 cacheEnabled 属性，默认值为 true
    configuration.setCacheEnabled(booleanValueOf(props.getProperty("cacheEnabled"), true));

    // 省略部分代码

    // 解析默认的枚举处理器
    Class<? extends TypeHandler> typeHandler = (Class<? extends TypeHandler>)resolveClass(props.getProperty("defaultEnumTypeHandler"));
    // 设置默认枚举处理器
    configuration.setDefaultEnumTypeHandler(typeHandler);
    configuration.setCallSettersOnNulls(booleanValueOf(props.getProperty("callSettersOnNulls"), false));
    configuration.setUseActualParamName(booleanValueOf(props.getProperty("useActualParamName"), true));
    
    // 省略部分代码
}
```
## 解析typeAliases配置
在MyBatis中，可以为有些类定义一个别名。在使用的过程中只需要输入别名，而不用输入全限定的类名。在MyBatis中，有两种方式进行别名配置。第一种是仅配置包名，让MyBatis去扫描包中的类型，并根据类型得到相应的别名。这种方式可配合Alias注解使用，即通过注解为某个类配置别名，而不是让MyBatis按照默认规则生成别名。这种方式的配置如下：
```
<typeAliases>
    <package name="xyz.coolblog.model1"/>
    <package name="xyz.coolblog.model2"/>
</typeAliases>
```
第二种方式是通过手动的方式，明确为某个类型配置别名。这种方式的配置如下：
```
<typeAliases>
    <typeAlias alias="article" type="xyz.coolblog.model.Article" />
    <typeAlias type="xyz.coolblog.model.Author" />
</typeAliases>
```
以上介绍了两种不同的别名配置方式，下面分析两种不同的别名配置是怎样解析的。代码如下：
```
private void typeAliasesElement(XNode parent) {
    if (parent != null) {
        for (XNode child : parent.getChildren()) {
        	   // 从指定的包中解析别名和类型的映射
            if ("package".equals(child.getName())) {
                String typeAliasPackage = child.getStringAttribute("name");
                configuration.getTypeAliasRegistry().registerAliases(typeAliasPackage);
                
            // 从 typeAlias 节点中解析别名和类型的映射
            } else {
            	  // 获取 alias 和 type 属性值，alias 不是必填项，可为空
                String alias = child.getStringAttribute("alias");
                String type = child.getStringAttribute("type");
                try {
                	  // 加载 type 对应的类型
                    Class<?> clazz = Resources.classForName(type);

                    // 注册别名到类型的映射
                    if (alias == null) {
                        typeAliasRegistry.registerAlias(clazz);
                    } else {
                        typeAliasRegistry.registerAlias(alias, clazz);
                    }
                } catch (ClassNotFoundException e) {
                    throw new BuilderException("Error registering typeAlias for '" + alias + "'. Cause: " + e, e);
                }
            }
        }
    }
}
```
从上述源码中可以看出，解析typeAliases配置分为两步：

    （1）首先从指定的包中解析别名和类型的映射
    （2）从typeAlias节点中解析别名和类型的映射
    
### 从指定的包中解析并注册别名
从指定的包中解析并注册别名过程主要由别名的解析和注册两步组成。下面来看一下相关代码：
```
public void registerAliases(String packageName) {
    // 调用重载方法注册别名
    registerAliases(packageName, Object.class);
}

public void registerAliases(String packageName, Class<?> superType) {
    ResolverUtil<Class<?>> resolverUtil = new ResolverUtil<Class<?>>();
    /*
     * 查找某个包下的父类为 superType 的类。从调用栈来看，这里的 
     * superType = Object.class，所以 ResolverUtil 将查找所有的类。
     * 查找完成后，查找结果将会被缓存到内部集合中。
     */ 
    resolverUtil.find(new ResolverUtil.IsA(superType), packageName);
    // 获取查找结果
    Set<Class<? extends Class<?>>> typeSet = resolverUtil.getClasses();
    for (Class<?> type : typeSet) {
        // 忽略匿名类，接口，内部类
        if (!type.isAnonymousClass() && !type.isInterface() && !type.isMemberClass()) {
            // 为类型注册别名 
            registerAlias(type);
        }
    }
}

public void registerAlias(Class<?> type) {
    String alias = type.getSimpleName();
    Alias aliasAnnotation = type.getAnnotation(Alias.class);
    if (aliasAnnotation != null) {
      alias = aliasAnnotation.value();
    } 
    registerAlias(alias, type);
}
```
上述代码总结为下面两个步骤：

    （1）查找指定包下的所有类
    （2）遍历查找到的类型集合，为每个类型注册别名
    
在这两步流程中，第一步的流程总结如下：

    （1）通过 VFS（虚拟文件系统）获取指定包下的所有文件的路径名，例如xyz/coolblog/model/Article.class
    （2）筛选以.class结尾的文件名
    （3）将路径名转成全限定的类名，通过类加载器加载类名
    （4）对类型进行匹配，若符合匹配规则，则将其放入内部集合中

### 从typeAlias节点中解析并注册别名
在别名的配置中，type属性是必须要配置的，而alias属性则不是必须的。这个在配置文件的 DTD中有规定。如果使用者未配置alias属性，则需要MyBatis自行为目标类型生成别名。对于别名为空的情况，注册别名的任务交由void registerAlias(Class<?>)方法处理。若不为空，则由void registerAlias(String, Class<?>)进行别名注册。这两个方法的分析如下：
```
private final Map<String, Class<?>> TYPE_ALIASES = new HashMap<String, Class<?>>();

public void registerAlias(Class<?> type) {
    // 获取全路径类名的简称
    String alias = type.getSimpleName();
    Alias aliasAnnotation = type.getAnnotation(Alias.class);
    if (aliasAnnotation != null) {
        // 从注解中取出别名
        alias = aliasAnnotation.value();
    }
    // 调用重载方法注册别名和类型映射
    registerAlias(alias, type);
}

public void registerAlias(String alias, Class<?> value) {
    if (alias == null) {
        throw new TypeException("The parameter alias cannot be null");
    }
    // 将别名转成小写
    String key = alias.toLowerCase(Locale.ENGLISH);
    /*
     * 如果 TYPE_ALIASES 中存在了某个类型映射，这里判断当前类型与映射中的类型是否一致，
     * 不一致则抛出异常，不允许一个别名对应两种类型
     */
    if (TYPE_ALIASES.containsKey(key) && TYPE_ALIASES.get(key) != null && !TYPE_ALIASES.get(key).equals(value)) {
        throw new TypeException(
            "The alias '" + alias + "' is already mapped to the value '" + TYPE_ALIASES.get(key).getName() + "'.");
    }
    // 缓存别名到类型映射
    TYPE_ALIASES.put(key, value);
}
```
如上，若用户未明确配置alias属性，MyBatis会使用类名的小写形式作为别名。例如全限定类名xyz.coolblog.model.Author的别名为author。若类中有@Alias注解，则从注解中取值作为别名。

### 注册MyBatis内部类及常见类型的别名
看一下一些MyBatis内部类及一些常见类型的别名注册过程。如下：
```
// Configuration
public Configuration() {
    // 注册事务工厂的别名
    typeAliasRegistry.registerAlias("JDBC", JdbcTransactionFactory.class);
    // 省略部分代码，下同

    // 注册数据源的别名
    typeAliasRegistry.registerAlias("POOLED", PooledDataSourceFactory.class);

    // 注册缓存策略的别名
    typeAliasRegistry.registerAlias("FIFO", FifoCache.class);
    typeAliasRegistry.registerAlias("LRU", LruCache.class);

    // 注册日志类的别名
    typeAliasRegistry.registerAlias("SLF4J", Slf4jImpl.class);
    typeAliasRegistry.registerAlias("LOG4J", Log4jImpl.class);

    // 注册动态代理工厂的别名
    typeAliasRegistry.registerAlias("CGLIB", CglibProxyFactory.class);
    typeAliasRegistry.registerAlias("JAVASSIST", JavassistProxyFactory.class);
}

// TypeAliasRegistry
public TypeAliasRegistry() {
    // 注册 String 的别名
    registerAlias("string", String.class);

    // 注册基本类型包装类的别名
    registerAlias("byte", Byte.class);
    // 省略部分代码，下同

    // 注册基本类型包装类数组的别名
    registerAlias("byte[]", Byte[].class);
    
    // 注册基本类型的别名
    registerAlias("_byte", byte.class);

    // 注册基本类型包装类的别名
    registerAlias("_byte[]", byte[].class);

    // 注册 Date, BigDecimal, Object 等类型的别名
    registerAlias("date", Date.class);
    registerAlias("decimal", BigDecimal.class);
    registerAlias("object", Object.class);

    // 注册 Date, BigDecimal, Object 等数组类型的别名
    registerAlias("date[]", Date[].class);
    registerAlias("decimal[]", BigDecimal[].class);
    registerAlias("object[]", Object[].class);

    // 注册集合类型的别名
    registerAlias("map", Map.class);
    registerAlias("hashmap", HashMap.class);
    registerAlias("list", List.class);
    registerAlias("arraylist", ArrayList.class);
    registerAlias("collection", Collection.class);
    registerAlias("iterator", Iterator.class);

    // 注册 ResultSet 的别名
    registerAlias("ResultSet", ResultSet.class);
}
```
## 解析typeHandlers配置
在向数据库存储或读取数据时，需要将数据库字段类型和Java类型进行一个转换。比如数据库中有CHAR和VARCHAR等类型，但Java中没有这些类型，不过Java有String类型。所以在从数据库中读取CHAR和VARCHAR类型的数据时，就可以把它们转成String。在MyBatis中，数据库类型和Java类型之间的转换任务是委托给类型处理器TypeHandler去处理的。MyBatis提供了一些常见类型的类型处理器，除此之外，还可以自定义类型处理器以非常见类型转换的需求。下面看一下类型处理器的配置方法：
```
<!-- 自动扫描 -->
<typeHandlers>
    <package name="xyz.coolblog.handlers"/>
</typeHandlers>
```
```
<!-- 手动配置 -->
<typeHandlers>
    <typeHandler jdbcType="TINYINT"
            javaType="xyz.coolblog.constant.ArticleTypeEnum"
            handler="xyz.coolblog.mybatis.ArticleTypeHandler"/>
</typeHandlers>
```
使用自动扫描的方式注册类型处理器时，应使用@MappedTypes和@MappedJdbcTypes注解配置javaType和jdbcType。下面开始分析代码。
```
private void typeHandlerElement(XNode parent) throws Exception {
    if (parent != null) {
        for (XNode child : parent.getChildren()) {
            // 从指定的包中注册TypeHandler(自动注册方式)
            if ("package".equals(child.getName())) {
                String typeHandlerPackage = child.getStringAttribute("name");
                // 注册方法
                typeHandlerRegistry.register(typeHandlerPackage);

            // 从typeHandler节点中解析别名到类型的映射（手动注册方式）
            } else {
                // 获取javaType，jdbcType和handler等属性值
                String javaTypeName = child.getStringAttribute("javaType");
                String jdbcTypeName = child.getStringAttribute("jdbcType");
                String handlerTypeName = child.getStringAttribute("handler");

                // 解析上面获取到的属性值
                Class<?> javaTypeClass = resolveClass(javaTypeName);
                JdbcType jdbcType = resolveJdbcType(jdbcTypeName);
                Class<?> typeHandlerClass = resolveClass(handlerTypeName);

                // 根据javaTypeClass和jdbcType值的情况进行不同的注册策略
                if (javaTypeClass != null) {
                    if (jdbcType == null) {
                        // 注册方法
                        typeHandlerRegistry.register(javaTypeClass, typeHandlerClass);
                    } else {
                        // 注册方法
                        typeHandlerRegistry.register(javaTypeClass, jdbcType, typeHandlerClass);
                    }
                } else {
                    // 注册方法
                    typeHandlerRegistry.register(typeHandlerClass);
                }
            }
        }
    }
}
```
上面代码中先是解析XML，然后调用了4个不同的类型处理器注册方法。这4个不同类型的注册方法互相调用，调用关系如下：

![](/images/Mybatis/typeHandlers注册方法.jpg)

在上面的调用图中，每个蓝色背景框下都有一个标签。每个标签上面都已一个编号。这里把蓝色背景框内的方法称为开始方法，红色背景框内的方法称为终点方法，白色背景框内的方法称为中间方法。下面会分析从每个开始方法向下分析，为了避免冗余分析，会按照③ → ② → ④ → ①的顺序进行分析。
###  register(Class, JdbcType, Class)方法
当代码执行到此方法时，表示javaTypeClass != null && jdbcType != null条件成立，即使用者明确配置了javaType和jdbcType属性的值。那下面来看一下该方法的分析。
```
public void register(Class<?> javaTypeClass, JdbcType jdbcType, Class<?> typeHandlerClass) {
    // 调用终点方法
    register(javaTypeClass, jdbcType, getInstance(javaTypeClass, typeHandlerClass));
}

/** 类型处理器注册过程的终点 */
private void register(Type javaType, JdbcType jdbcType, TypeHandler<?> handler) {
    if (javaType != null) {
        // JdbcType到TypeHandler的映射
        Map<JdbcType, TypeHandler<?>> map = TYPE_HANDLER_MAP.get(javaType);
        if (map == null || map == NULL_TYPE_HANDLER_MAP) {
            map = new HashMap<JdbcType, TypeHandler<?>>();
            // 存储javaType 到Map<JdbcType, TypeHandler> 的映射
            TYPE_HANDLER_MAP.put(javaType, map);
        }
        map.put(jdbcType, handler);
    }

    // 存储所有TypeHandler的class对象与TypeHandler的映射
    ALL_TYPE_HANDLERS_MAP.put(handler.getClass(), handler);
}
```
### register(Class, Class)方法
当代码执行到此方法时，表示javaTypeClass != null && jdbcType == null条件成立，即使用者仅设置了javaType属性的值。下面来看一下该方法的分析。
```
public void register(Class<?> javaTypeClass, Class<?> typeHandlerClass) {
    // 调用中间方法 register(Type, TypeHandler)
    register(javaTypeClass, getInstance(javaTypeClass, typeHandlerClass));
}

private <T> void register(Type javaType, TypeHandler<? extends T> typeHandler) {
    // 获取 @MappedJdbcTypes 注解
    MappedJdbcTypes mappedJdbcTypes = typeHandler.getClass().getAnnotation(MappedJdbcTypes.class);
    if (mappedJdbcTypes != null) {
        // 遍历 @MappedJdbcTypes 注解中配置的值
        for (JdbcType handledJdbcType : mappedJdbcTypes.value()) {
            // 调用终点方法，参考上一小节的分析
            register(javaType, handledJdbcType, typeHandler);
        }
        if (mappedJdbcTypes.includeNullJdbcType()) {
            // 调用终点方法，jdbcType = null
            register(javaType, null, typeHandler);
        }
    } else {
        // 调用终点方法，jdbcType = null
        register(javaType, null, typeHandler);
    }
}
```
上面的代码包含三层调用，其中终点方法的逻辑上一节已经分析过。上面的逻辑也比较简单，主要做的事情是尝试从注解中获取JdbcType的值。
### register(Class) 方法分析
当代码执行到此方法时，表示javaTypeClass == null && jdbcType != null条件成立，即使用者未配置javaType和jdbcType属性的值。该方法的分析如下。
```
public void register(Class<?> typeHandlerClass) {
    boolean mappedTypeFound = false;
    // 获取 @MappedTypes 注解
    MappedTypes mappedTypes = typeHandlerClass.getAnnotation(MappedTypes.class);
    if (mappedTypes != null) {
        // 遍历 @MappedTypes 注解中配置的值
        for (Class<?> javaTypeClass : mappedTypes.value()) {
            // 调用注册方法 ②
            register(javaTypeClass, typeHandlerClass);
            mappedTypeFound = true;
        }
    }
    if (!mappedTypeFound) {
        // 调用中间方法 register(TypeHandler)
        register(getInstance(null, typeHandlerClass));
    }
}

public <T> void register(TypeHandler<T> typeHandler) {
    boolean mappedTypeFound = false;
    // 获取 @MappedTypes 注解
    MappedTypes mappedTypes = typeHandler.getClass().getAnnotation(MappedTypes.class);
    if (mappedTypes != null) {
        for (Class<?> handledType : mappedTypes.value()) {
            // 调用中间方法 register(Type, TypeHandler)
            register(handledType, typeHandler);
            mappedTypeFound = true;
        }
    }
    // 自动发现映射类型
    if (!mappedTypeFound && typeHandler instanceof TypeReference) {
        try {
            TypeReference<T> typeReference = (TypeReference<T>) typeHandler;
            // 获取参数模板中的参数类型，并调用中间方法 register(Type, TypeHandler)
            register(typeReference.getRawType(), typeHandler);
            mappedTypeFound = true;
        } catch (Throwable t) {
        }
    }
    if (!mappedTypeFound) {
        // 调用中间方法 register(Class, TypeHandler)
        register((Class<T>) null, typeHandler);
    }
}

public <T> void register(Class<T> javaType, TypeHandler<? extends T> typeHandler) {
    // 调用中间方法 register(Type, TypeHandler)
    register((Type) javaType, typeHandler);
}
```
不管是通过注解的方式，还是通过反射的方式，它们最终目的是为了解析出javaType的值。解析完成后，这些方法会调用中间方法register(Type, TypeHandler)，这个方法负责解析jdbcType，该方法上一节已经分析过。一个复杂解析 javaType，另一个负责解析 jdbcType。
```
public void register(String packageName) {
    ResolverUtil<Class<?>> resolverUtil = new ResolverUtil<Class<?>>();
    // 从指定包中查找 TypeHandler
    resolverUtil.find(new ResolverUtil.IsA(TypeHandler.class), packageName);
    Set<Class<? extends Class<?>>> handlerSet = resolverUtil.getClasses();
    for (Class<?> type : handlerSet) {
        // 忽略内部类，接口，抽象类等
        if (!type.isAnonymousClass() && !type.isInterface() && !Modifier.isAbstract(type.getModifiers())) {
            // 调用注册方法 ④
            register(type);
        }
    }
}
```
## 解析environments配置
在MyBatis中，事务管理器和数据源是配置在environments中的。它们的配置大致如下：
```
<environments default="development">
    <environment id="development">
        <transactionManager type="JDBC"/>
        <dataSource type="POOLED">
            <property name="driver" value="${jdbc.driver}"/>
            <property name="url" value="${jdbc.url}"/>
            <property name="username" value="${jdbc.username}"/>
            <property name="password" value="${jdbc.password}"/>
        </dataSource>
    </environment>
</environments>
```
源码如下：
```
private void environmentsElement(XNode context) throws Exception {
    if (context != null) {
        if (environment == null) {
            // 获取 default 属性
            environment = context.getStringAttribute("default");
        }
        for (XNode child : context.getChildren()) {
            // 获取 id 属性
            String id = child.getStringAttribute("id");
            /*
             * 检测当前 environment 节点的 id 与其父节点 environments 的属性 default 
             * 内容是否一致，一致则返回 true，否则返回 false
             */
            if (isSpecifiedEnvironment(id)) {
                // 解析 transactionManager 节点，逻辑和插件的解析逻辑很相似，不在赘述
                TransactionFactory txFactory = transactionManagerElement(child.evalNode("transactionManager"));
                // 解析 dataSource 节点，逻辑和插件的解析逻辑很相似，不在赘述
                DataSourceFactory dsFactory = dataSourceElement(child.evalNode("dataSource"));
                // 创建 DataSource 对象
                DataSource dataSource = dsFactory.getDataSource();
                Environment.Builder environmentBuilder = new Environment.Builder(id)
                    .transactionFactory(txFactory)
                    .dataSource(dataSource);
                // 构建 Environment 对象，并设置到 configuration 中
                configuration.setEnvironment(environmentBuilder.build());
            }
        }
    }
}
```
## 解析plugins配置
插件是MyBatis提供的一个拓展机制，通过插件机制可在SQL执行过程中的某些点上做一些自定义操作。实现一个插件需要比简单，首先需要让插件类实现Interceptor接口。然后在插件类上添加@Intercepts和@Signature注解，用于指定想要拦截的目标方法。MyBatis允许拦截下面接口中的一些方法：

    （1）Executor: update方法，query方法，flushStatements方法，commit方法，rollback方法，getTransaction方法，close方法，isClosed方法
    （2）ParameterHandler: getParameterObject方法，setParameters方法
    （3）ResultSetHandler: handleResultSets方法，handleOutputParameters方法
    （4）StatementHandler: prepare方法，parameterize方法，batch方法，update方法，query方法
    
```
<plugins>
    <plugin interceptor="xyz.coolblog.mybatis.ExamplePlugin">
        <property name="key" value="value"/>
    </plugin>
</plugins>
```
```
private void pluginElement(XNode parent) throws Exception {
    if (parent != null) {
        for (XNode child : parent.getChildren()) {
            String interceptor = child.getStringAttribute("interceptor");
            // 获取配置信息
            Properties properties = child.getChildrenAsProperties();
            // 解析拦截器的类型，并创建拦截器
            Interceptor interceptorInstance = (Interceptor) resolveClass(interceptor).newInstance();
            // 设置属性
            interceptorInstance.setProperties(properties);
            // 添加拦截器到 Configuration 中
            configuration.addInterceptor(interceptorInstance);
        }
    }
}
```
如上，首先是获取配置，然后再解析拦截器类型，并实例化拦截器。最后向拦截器中设置属性，并将拦截器添加到Configuration中。

![](/images/Mybatis/MyBatis配置文件解析.jpg)


 