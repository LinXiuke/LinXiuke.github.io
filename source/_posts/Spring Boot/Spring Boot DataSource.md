---
title: Spring Boot 配置多数据源和动态数据源

date: 2019-11-19

categories: 
- Spring Boot

tags:
- MySQL
---

### 多数据源

在项目中有时候需要同时连接多个数据库，这些数据库可能是各不相同的，这个时候就要去配置多数据源。

用spring boot 配置jpa的多数据源，首先要在配置文件中添加相关数据库配置

<!--more-->

```
spring:
    datasource:
        # 主要数据库
        primary:
            type: com.zaxxer.hikari.HikariDataSource
            driver-class-name: com.mysql.jdbc.Driver
            url: jdbc:mysql://localhost:3306/base1?autoReconnect=true&useUnicode=true&characterEncoding=utf8&useSSL=false
            username: root
            password: 123456
        secondary:
            # 次要要数据库
            type: com.zaxxer.hikari.HikariDataSource
            driver-class-name: com.mysql.jdbc.Driver
            url: jdbc:mysql://localhost:3306/base2?autoReconnect=true&useUnicode=true&characterEncoding=utf8&useSSL=false
            username: root
            password: 123456
```

然后是DataSource配置类

```
@Configuration
public class DataSourceConfig {

    @Primary
    @Bean("primaryDataSource")
    @Qualifier("primaryDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.primary")
    public DataSource primaryDS() {
        return DataSourceBuilder.create().build();
    }


    @Bean("secondaryDataSource")
    @Qualifier("secondaryDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.secondary")
    public DataSource secondaryDS() {
        return DataSourceBuilder.create().build();
    }
}
```
@ConfigurationProperties注解的prefix对应的就是配置文件里各自的数据库

DataSource配置好后还有有关于单个数据库的配置

```
@Configuration
@EnableTransactionManagement  //开启数据库事务
@EnableJpaRepositories(entityManagerFactoryRef = "primaryEntityManagerFactory", transactionManagerRef =
        "primaryTransactionManager", basePackages = {"com.xxx.primary.dao  （DAO文件目录） "})
public class PrimaryRepositoryConfig {

    @Autowired
    private JpaProperties jpaProperties;

    @Autowired
    @Qualifier("primaryDataSource")
    private DataSource primaryDataSource;

    @Bean(name = "primaryEntityManager")
    public EntityManager entityManager(EntityManagerFactoryBuilder builder) {
        return entityManagerFactoryPrimary(builder).getObject().createEntityManager();
    }

    @Bean(name = "primaryEntityManagerFactory")
    public LocalContainerEntityManagerFactoryBean entityManagerFactoryPrimary(EntityManagerFactoryBuilder builder) {
        return builder.dataSource(primaryDataSource).properties(getVendorProperties(primaryDataSource))
                .packages("com.xxx.primary.entity") // 实体类目录
                .persistenceUnit("persistenceUnit").build();
    }

    private Map<String, String> getVendorProperties(DataSource dataSource) {
        return jpaProperties.getHibernateProperties(dataSource);
    }


    // jpa事务配置
    @Primary
    @Bean(name = "primaryTransactionManager")
    PlatformTransactionManager transactionManagerPrimary(EntityManagerFactoryBuilder builder) {
        return new JpaTransactionManager(entityManagerFactoryPrimary(builder).getObject());
    }

}
```

primary数据库的配置就这样完成了，secondary数据库的配置和primary是一样的。这样一来spring boot就能根据实体类和DAO所在目录自动连接数据库

如果要使用mybatis作为持久化层框架的话，只需要额外配置mybatis的SQLSession

```
    @Primary
    @Bean("primarySqlSessionFactory")
    public SqlSessionFactory sqlSessionFactory() {
        org.apache.ibatis.session.Configuration configuration = new org.apache.ibatis.session.Configuration();
        //驼峰自动转换
        configuration.setMapUnderscoreToCamelCase(true);

        SqlSessionFactoryBean bean = new SqlSessionFactoryBean();
        bean.setDataSource(primaryDataSource);
        bean.setConfiguration(configuration);

        PageInterceptor pageInterceptor = new PageInterceptor();
        Properties properties = new Properties();
        properties.setProperty("helperDialect", "mysql");
        pageInterceptor.setProperties(properties);
        Interceptor[] interceptors = {pageInterceptor};
        bean.setPlugins(interceptors);
        //添加XML目录
        ResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
        try {
            bean.setMapperLocations(resolver.getResources("classpath*:mybatis/primary/**/*.xml")); // xml文件目录
            return bean.getObject();
        } catch (Exception e) {
            e.printStackTrace();
            throw new RuntimeException(e);
        }
    }
```

配置完后在改配置类加上mybatis扫描注解
@MapperScan(basePackages = {"com.xxx.primary.mapper"}, sqlSessionFactoryRef = "primarySqlSessionFactory")


### 动态数据源
多数据源对实体类和DAO层所在目录有要求，而动态数据源则是根据你代码里做的选择在对数据库操作时动态地去切换数据库。一般来说使用动态数据源的各个数据库表大体一致，逻辑可以公用。比如一套系统多租户数据物理隔离使用不同数据库。

配置文件和多数据源一样，配置类会有些不同



```
spring:
    datasource:
        # 主要数据库
        1:
            type: com.zaxxer.hikari.HikariDataSource
            driver-class-name: com.mysql.jdbc.Driver
            url: jdbc:mysql://localhost:3306/base1?autoReconnect=true&useUnicode=true&characterEncoding=utf8&useSSL=false
            username: root
            password: 123456
        2:
            # 次要要数据库
            type: com.zaxxer.hikari.HikariDataSource
            driver-class-name: com.mysql.jdbc.Driver
            url: jdbc:mysql://localhost:3306/base2?autoReconnect=true&useUnicode=true&characterEncoding=utf8&useSSL=false
            username: root
            password: 123456
```




```
@ServletComponentScan
@Configuration
@EnableTransactionManagement // 开启事务
public class DataSourceConfig {
    @Primary
    @Bean(name = "DataSource1")
    @ConfigurationProperties(prefix = "spring.datasource.1")
    public DataSource primaryDataSource() {
        DruidDataSource dataSource = DruidDataSourceBuilder.create().build();
        dataSource.setName("userDataSource");
        return dataSource;
    }

    @Bean(name = "Datasource2")
    @ConfigurationProperties(prefix = "spring.datasource.2")
    public DataSource secondaryDatasource() {
        return DruidDataSourceBuilder.create().build();
    }

    

    @Bean("dynamicDataSource")
    public DataSource dynamicDataSource() {
        DynamicDataSource dynamicDataSource = new DynamicDataSource();
        //默认数据源
        dynamicDataSource.setDefaultTargetDataSource(primaryDataSource());
        Map<Object, Object> dataSourceMap = new HashMap<>();
        dataSourceMap.put("1", primaryDataSource());
        dataSourceMap.put("2", secondaryDatasource());
        dynamicDataSource.setTargetDataSources(dataSourceMap);
        return dynamicDataSource;
    }

    @Bean(name = "sqlSessionFactory")
    @Qualifier("sqlSessionFactory")
    @ConfigurationProperties(prefix = "mybatis")
    public SqlSessionFactory sqlSessionFactory() throws Exception {
        SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
        org.apache.ibatis.session.Configuration configuration = new org.apache.ibatis.session.Configuration();
        //驼峰自动转换
        configuration.setMapUnderscoreToCamelCase(true);
        sqlSessionFactoryBean.setConfiguration(configuration);
        sqlSessionFactoryBean.setTypeAliasesPackage("type-aliases-package");
        sqlSessionFactoryBean.setMapperLocations(new PathMatchingResourcePatternResolver().getResources("mapping/*.xml"));
        sqlSessionFactoryBean.setDataSource(dynamicDataSource());

        return sqlSessionFactoryBean.getObject();
    }

    @Bean
    public SqlSessionTemplate sqlSessionTemplate() throws Exception {
        return new SqlSessionTemplate(sqlSessionFactory());
    }
    
    // 事务
    @Bean
    public PlatformTransactionManager platformTransactionManager(
            @Qualifier("dynamicDataSource") DataSource myDataSource) {
        return new DataSourceTransactionManager(myDataSource);
    }

}

```


Spring boot提供了AbstractRoutingDataSource 根据用户定义的规则选择当前的数据源，这样我们可以在执行查询之前，设置使用的数据源。实现可动态路由的数据源，在每次数据库查询操作前执行。它的抽象方法 determineCurrentLookupKey() 决定使用哪个数据源。

在程序运行时，把数据源数据源通过 AbstractRoutingDataSource 动态织入到程序中，灵活的进行数据源切换


```
public class DynamicDataSource extends AbstractRoutingDataSource {

    private static final Logger logger = LoggerFactory.getLogger(DynamicDataSource.class);

    @Override
    protected Object determineCurrentLookupKey() {
        logger.info("-------------------Current DataSource is [{}]----------------", DataSourceHolder.getDataSource());
        return DataSourceHolder.getDataSource();
    }
}


public class DataSourceHolder {


    private static final ThreadLocal<String> dataSources = new ThreadLocal<>();

    public static void setDataSource(String customerType) {
        dataSources.set(customerType);
    }

    public static String getDataSource() {
        return dataSources.get();
    }

    public static void clearDataSource() {
        dataSources.remove();
    }
}

```

这样一来，就可以在代码中通过修改DataSourceHolder的dataSources来动态地修改数据源
