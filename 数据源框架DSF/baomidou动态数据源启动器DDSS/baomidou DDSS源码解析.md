# Baomidou 动态数据源启动器源码解析

<!-- vscode-markdown-toc -->
* 1. [工程结构](#)
* 2. [注解设计](#-1)
* 3. [配置文件](#-1)
	* 3.1. [外层配置 Model——_DynamicDataSourceProperties_](#Model_DynamicDataSourceProperties_)
	* 3.2. [内层配置 Model——_DataSourceProperty_](#Model_DataSourceProperty_)
* 4. [数据源创建器](#-1)
	* 4.1. [顶级接口 _DataSourceCreator_](#_DataSourceCreator_)
	* 4.2. [默认数据源创建器 _DefaultDataSourceCreator_](#_DefaultDataSourceCreator_)
	* 4.3. [JNDI数据源创建器 _JndiDataSourceCreator_](#JNDI_JndiDataSourceCreator_)
	* 4.4. [Hikari数据源创建器 _HikariDataSourceCreator_](#Hikari_HikariDataSourceCreator_)
	* 4.5. [Druid数据源创建器 _DruidDataSourceCreator_](#Druid_DruidDataSourceCreator_)

<!-- vscode-markdown-toc-config
	numbering=true
	autoSave=true
	/vscode-markdown-toc-config -->
<!-- /vscode-markdown-toc -->

##  1. <a name=''></a>工程结构

##  2. <a name='-1'></a>注解设计

dynamic-datasource-springboot-starter 基于注解的方式，能够实现数据源的动态配置，提供的注解主要有 4 类:

1. **`@DS`** :数据源注解，该注解可以作用于类和方法上，该注解的`value`字段将指明数据源的名称
2. **`@DSTransactional`** : 多数据源事务管理，可作用于类和方法上，含有该注解的方法将会开启事务管理
3. **`@Master`** :主数据源注解，可作于于类和方法上，添加了该注解之后，将会默认使用名为"master"的数据源
4. **`@Slave`** :从数据源注解，可作用于类和方法上，添加了该注解之后，将会默认使用"salve"的数据源

##  3. <a name='-1'></a>配置文件

dynamic-datasource-springboot-starter 的配置文件大致如下:

```yml
spring:
  datasource:
    dynamic:
      primary: master #设置默认的数据源或者数据源组,默认值即为master
      strict: false #设置严格模式,默认false不启动. 启动后在未匹配到指定数据源时候会抛出异常,不启动则使用默认数据源.
      datasource:
        master:
          url: jdbc:mysql://xx.xx.xx.xx:3306/dynamic
          username: root
          password: 123456
          driver-class-name: com.mysql.jdbc.Driver # 3.2.0开始支持SPI可省略此配置
        slave_1:
          url: jdbc:mysql://xx.xx.xx.xx:3307/dynamic
          username: root
          password: 123456
          driver-class-name: com.mysql.jdbc.Driver
        slave_2:
          url: ENC(xxxxx) # 内置加密,使用请查看详细文档
          username: ENC(xxxxx)
          password: ENC(xxxxx)
          driver-class-name: com.mysql.jdbc.Driver
          schema: db/schema.sql # 配置则生效,自动初始化表结构
          data: db/data.sql # 配置则生效,自动初始化数据
          continue-on-error: true # 默认true,初始化失败是否继续
          separator: ";" # sql默认分号分隔符
```

###  3.1. <a name='Model_DynamicDataSourceProperties_'></a>外层配置 Model——_DynamicDataSourceProperties_

`DynamicDataSourceProperties`对应配置文件中的外层配置，即 prefix 为:**`spring.datasource.dynamic`** ，配置参数对应的实体类定义如下:

```java
@Slf4j
@Getter
@Setter
@ConfigurationProperties(prefix = DynamicDataSourceProperties.PREFIX)
public class DynamicDataSourceProperties {

    public static final String PREFIX = "spring.datasource.dynamic";
    public static final String HEALTH = PREFIX + ".health";
    public static final String DEFAULT_VALID_QUERY = "SELECT 1";

    //必须设置默认的库,默认master
    private String primary = "master";
    //是否启用严格模式,默认不启动. 严格模式下未匹配到数据源直接报错, 非严格模式下则使用默认数据源primary所设置的数据源
    private Boolean strict = false;
    //是否使用p6spy输出，默认不输出
    private Boolean p6spy = false;
    //是否使用开启seata，默认不开启
    private Boolean seata = false;
    //seata使用模式，默认AT
    private SeataMode seataMode = SeataMode.AT;
    //是否使用 spring actuator 监控检查，默认不检查
    private boolean health = false;
    //监控检查SQL
    private String healthValidQuery = DEFAULT_VALID_QUERY;
    //每一个数据源
    private Map<String, DataSourceProperty> datasource = new LinkedHashMap<>();
    //多数据源选择算法clazz，默认负载均衡算法
    private Class<? extends DynamicDataSourceStrategy> strategy = LoadBalanceDynamicDataSourceStrategy.class;
    //aop切面顺序，默认优先级最高
    private Integer order = Ordered.HIGHEST_PRECEDENCE;
    //Druid全局参数配置
    @NestedConfigurationProperty
    private DruidConfig druid = new DruidConfig();
    // HikariCp全局参数配置
    @NestedConfigurationProperty
    private HikariCpConfig hikari = new HikariCpConfig();
    //全局默认publicKey
    private String publicKey = CryptoUtils.DEFAULT_PUBLIC_KEY_STRING;
    //aop 切面是否只允许切 public 方法
    private boolean allowedPublicOnly = true;
}
```

###  3.2. <a name='Model_DataSourceProperty_'></a>内层配置 Model——_DataSourceProperty_

`DataSourceProperty`对应的是配置文件中的内层参数，即分别对应的是每一个数据源的配置参数，例如`master`数据源,`slave1`数据源，`slave2`数据源，定义内容如下:

```java
@Slf4j
@Data
@Accessors(chain = true)
public class DataSourceProperty {
    //加密正则
    private static final Pattern ENC_PATTERN = Pattern.compile("^ENC\\((.*)\\)$");
    //连接池名称(只是一个名称标识)</br> 默认是配置文件上的名称
    private String poolName;
    //连接池类型，如果不设置自动查找 Druid > HikariCp
    private Class<? extends DataSource> type;
    //JDBC driver
    private String driverClassName;
    //JDBC url 地址
    private String url;
    //JDBC 用户名
    private String username;
    //JDBC 密码
    private String password;
    //jndi数据源名称(设置即表示启用)
    private String jndiName;
    //自动运行的建表脚本
    private String schema;
    //自动运行的数据脚本
    private String data;
    //启用seata
    private Boolean seata = true;
    //启用p6spy
    private Boolean p6spy = true;
    //错误是否继续 默认 true
    private boolean continueOnError = true;
    //分隔符 默认 ;
    private String separator = ";";
    //Druid参数配置
    @NestedConfigurationProperty
    private DruidConfig druid = new DruidConfig();
    //HikariCp参数配置
    @NestedConfigurationProperty
    private HikariCpConfig hikari = new HikariCpConfig();
}
```

##  4. <a name='-1'></a>数据源创建器

数据源创建模块位于`com.baomidou.dynamic.datasource.creator`包下。数据源创建器会根据配置参数(来自配置文件或者默认参数)创建不同类型的数据源。

**在创建数据源的时候，需要判别当前配置参数是否支持(support)相应的数据源的创建(后续代码会讲解)**

本项目一共实现了 5 种数据源创建器，分别为:

1. **`DefaultDataSourceCreator`**
2. **`JndiDataSourceCreator`**
3. **`HikariDataSourceCreator`**
4. **`DruidDatSourceCreator`**
5. **`BasicDataSourceCreator`**

5 类数据源创建器的继承关系如下:

<div align=center><img src="/素材/dsc.jpg"></div>

###  4.1. <a name='_DataSourceCreator_'></a>顶级接口 _DataSourceCreator_

`DataSourceCreator`为数据源创建器的顶层接口，定义了**如何创建数据源**，源码如下:

```java
public interface DataSourceCreator {

    /**
     * 通过属性创建数据源
     *
     * @param dataSourceProperty 数据源属性
     * @return 被创建的数据源
     */
    DataSource createDataSource(DataSourceProperty dataSourceProperty);

    /**
     * 通过属性创建数据源
     *
     * @param dataSourceProperty 数据源属性
     * @param publicKey          解密公钥
     * @return 被创建的数据源
     */
    DataSource createDataSource(DataSourceProperty dataSourceProperty, String publicKey);

    /**
     * 当前创建器是否支持根据此属性创建
     *
     * @param dataSourceProperty 数据源属性
     * @return 是否支持创建该数据源
     */
    boolean support(DataSourceProperty dataSourceProperty);
}
```

###  4.2. <a name='_DefaultDataSourceCreator_'></a>默认数据源创建器 _DefaultDataSourceCreator_

`DefaultDataSourceCreator`类成员有两个，分别为`DynamicDataSourceProperties`和`creators`，前者为配置的一些参数封装的 Model，后者为创建器的列表，**在创建数据源之前，需要调用`support`方法判断是否支持创建此数据源**

```java
@Slf4j
@Setter
public class DefaultDataSourceCreator implements DataSourceCreator {

    private DynamicDataSourceProperties properties;
    private List<DataSourceCreator> creators;
```

数据源的创建是根据`DataSourceProperty`，即内置的单个数据源配置参数(配置文件配置或者默认值)，以及解密用的公钥`publicKey`，因为密码等隐私信息不能在配置文件中明文显示出来，需要进行加密解密操作保证数据安全。创建行为发生在`createDataSource`方法上，源码如下:

```java
public DataSource createDataSource(DataSourceProperty dataSourceProperty, String publicKey) {
    DataSourceCreator dataSourceCreator = null;
    //遍历数据源创建器列表，对于每一种创建器，尝试是否能够创建
    for (DataSourceCreator creator : this.creators) {
        if (creator.support(dataSourceProperty)) {
            dataSourceCreator = creator;
            break;
        }
    }
    //创建器获取失败，返回异常
    if (dataSourceCreator == null) {
        throw new IllegalStateException("creator must not be null,please check the DataSourceCreator");
    }
    //开始创建
    DataSource dataSource = dataSourceCreator.createDataSource(dataSourceProperty, publicKey);
    //如果配置有SQL脚本，执行脚本，初始化数据库
    this.runScrip(dataSource, dataSourceProperty);
    //进一步封装数据源，以便支持事务
    return wrapDataSource(dataSource, dataSourceProperty);
}
```

Java原生支持的`DataSource`需要进一步封装以支持一些高级特性，比如Seata事务和P6spy等，封装动作由`wrapDataSource`完成，最后返回封装好的`com.baomidou.dynamic.datasource.ds.ItemDataSource`

```java
private DataSource wrapDataSource(DataSource dataSource, DataSourceProperty dataSourceProperty) {
    String name = dataSourceProperty.getPoolName();
    DataSource targetDataSource = dataSource;

    Boolean enabledP6spy = properties.getP6spy() && dataSourceProperty.getP6spy();
    if (enabledP6spy) {//支持p6spy
        targetDataSource = new P6DataSource(dataSource);
        log.debug("dynamic-datasource [{}] wrap p6spy plugin", name);
    }

    Boolean enabledSeata = properties.getSeata() && dataSourceProperty.getSeata();
    SeataMode seataMode = properties.getSeataMode();
    if (enabledSeata) {//支持seata
        if (SeataMode.XA == seataMode) { //seata模式为XA模式
            targetDataSource = new DataSourceProxyXA(dataSource);
        } else { //seata模式为其他模式
            targetDataSource = new DataSourceProxy(dataSource);
        }
        log.debug("dynamic-datasource [{}] wrap seata plugin transaction mode [{}]", name, seataMode);
    }
    return new ItemDataSource(name, dataSource, targetDataSource, enabledP6spy, enabledSeata, seataMode);
}
```

###  4.3. <a name='JNDI_JndiDataSourceCreator_'></a>JNDI数据源创建器 _JndiDataSourceCreator_

JDNI数据源由`JndiDataSourceLookup`创建，配置时仅需要`jndi-name`即可

```java
@Override
public DataSource createDataSource(DataSourceProperty dataSourceProperty, String publicKey) {
    return LOOKUP.getDataSource(dataSourceProperty.getJndiName());
}
```

###  4.4. <a name='Hikari_HikariDataSourceCreator_'></a>Hikari数据源创建器 _HikariDataSourceCreator_

Hikari数据源在创建之前会首先尝试加载hikari驱动，如果加载驱动成功，那么就把`hikariExists`设置为true

```java
private static Boolean hikariExists = false;

static {
    try {
        Class.forName(HIKARI_DATASOURCE); //尝试加载驱动
        hikariExists = true;
    } catch (ClassNotFoundException ignored) {
        //加载失败，什么也不做，也不会创建
    }
}
```

加载时依旧从配置文件中读取参数，配置`HikariDataSource`

```java
@Override
public DataSource createDataSource(DataSourceProperty dataSourceProperty, String publicKey) {
    //配置文件是否配置了公钥，如果没有，使用默认公钥
    if (StringUtils.isEmpty(dataSourceProperty.getPublicKey())) {
        dataSourceProperty.setPublicKey(publicKey);
    }
    //Hikari配置参数Model
    HikariConfig config = dataSourceProperty.getHikari().toHikariConfig(hikariCpConfig);
    //参数设置
    config.setUsername(dataSourceProperty.getUsername());
    config.setPassword(dataSourceProperty.getPassword());
    config.setJdbcUrl(dataSourceProperty.getUrl());
    config.setPoolName(dataSourceProperty.getPoolName());
    String driverClassName = dataSourceProperty.getDriverClassName();
    if (!StringUtils.isEmpty(driverClassName)) {
        config.setDriverClassName(driverClassName);
    }
    //封装好的数据源
    return new HikariDataSource(config);
}
```

###  4.5. <a name='Druid_DruidDataSourceCreator_'></a>Druid数据源创建器 _DruidDataSourceCreator_

Druid数据源的创建过程和Hikari基本无区别，但是Druid的配置参数相较之下更复杂一些，多了一些过滤器的设置和高级特性的设置，例如:

```java
//stat filter是检查数据库健康状态的过滤器
if (!StringUtils.isEmpty(filters) && filters.contains("stat")) {
    StatFilter statFilter = new StatFilter();
    statFilter.configFromProperties(properties);
    proxyFilters.add(statFilter);
}
```

```java
//wall filter是保护数据库安全的防火墙拦截器
if (!StringUtils.isEmpty(filters) && filters.contains("wall")) {
    WallConfig wallConfig = DruidWallConfigUtil.toWallConfig(dataSourceProperty.getDruid().getWall(), gConfig.getWall());
    WallFilter wallFilter = new WallFilter();
    wallFilter.setConfig(wallConfig);
    proxyFilters.add(wallFilter);
}
```

```java
//日志拦截器
if (!StringUtils.isEmpty(filters) && filters.contains("slf4j")) {
    Slf4jLogFilter slf4jLogFilter = new Slf4jLogFilter();
    // 由于properties上面被用了，LogFilter不能使用configFromProperties方法，这里只能一个个set了。
    DruidSlf4jConfig slf4jConfig = gConfig.getSlf4j();
    slf4jLogFilter.setStatementLogEnabled(slf4jConfig.getEnable());
    slf4jLogFilter.setStatementExecutableSqlLogEnable(slf4jConfig.getStatementExecutableSqlLogEnable());
    proxyFilters.add(slf4jLogFilter);    
}
```

### 基础数据源创建器 _BasicDataSourceCreator_

`BasicDataSourceCreator`是为了适配spring1.5和2.x创建的。创建数据源的行为基于反射，源码如下:

```java
@Data
@Slf4j
public class BasicDataSourceCreator extends AbstractDataSourceCreator implements DataSourceCreator {

    private static Method createMethod;
    private static Method typeMethod;
    private static Method urlMethod;
    private static Method usernameMethod;
    private static Method passwordMethod;
    private static Method driverClassNameMethod;
    private static Method buildMethod;

    static {
        //to support springboot 1.5 and 2.x
        Class<?> builderClass = null;
        try {
            builderClass = Class.forName("org.springframework.boot.jdbc.DataSourceBuilder");
        } catch (Exception ignored) {
        }
        if (builderClass == null) {
            try {
                builderClass = Class.forName("org.springframework.boot.autoconfigure.jdbc.DataSourceBuilder");
            } catch (Exception e) {
                log.warn("not in springBoot ENV,could not create BasicDataSourceCreator");
            }
        }
        if (builderClass != null) {
            try {
                createMethod = builderClass.getDeclaredMethod("create");
                typeMethod = builderClass.getDeclaredMethod("type", Class.class);
                urlMethod = builderClass.getDeclaredMethod("url", String.class);
                usernameMethod = builderClass.getDeclaredMethod("username", String.class);
                passwordMethod = builderClass.getDeclaredMethod("password", String.class);
                driverClassNameMethod = builderClass.getDeclaredMethod("driverClassName", String.class);
                buildMethod = builderClass.getDeclaredMethod("build");
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
```

