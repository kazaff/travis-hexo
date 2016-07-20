title: disconf+spring boot+mybatis
date: 2016-07-15 09:37:12
tags:
- 配置中心
- datasource
- 实时同步
- 自动更新
- disconfig
- spring boot
- mybatis
categories: 运维
---

最近在调研“配置中心”这一块儿，用于解决大量子系统的部署问题。目前系统一多，每次一个配置项目的变更老麻烦了，流程是：

> 修改项目的配置文件 --> 重新打包 --> FTP上传服务器 --> 解压替换现有项目 --> 重启项目

<!-- more -->
由于目前项目处于开发阶段，更多的场景是变更代码，所以上述流程并没有显得特别的笨重，但每次线上调试时就显得有些不合理了。再加上日后打算做**基于配置的功能开关**，所以说配置中心是一个不错的解决方案。

没有花太多的时间去调研各个开源配置中心解决方案，有兴趣的推荐看看这篇[博客](http://vernonzheng.com/2015/02/09/%E5%BC%80%E6%BA%90%E5%88%86%E5%B8%83%E5%BC%8F%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83%E9%80%89%E5%9E%8B/)，这里我选择的是百度的disconf。


### disconf环境搭建

这里我们直接走个捷径，使用[docker-disconf](http://git.oschina.net/gongxusheng/docker-disconf)来完成环境搭建。

关于docker的用法，可以参考我前面的一些文章，我就不啰嗦了。


### disconf测试环境

在仔细阅读完[官方wiki](https://github.com/knightliao/disconf)后，我们就可以clone其提供的[demo](https://github.com/knightliao/disconf-demos-java)代码到本地，进行测试了。

这部分基本上只要文档读的仔细，根据自己环境来调整几个参数，基本上就都可以顺利跑起来。

而demo中不仅提供了spring boot的用法，连dubbo都有，果然是国产良心。而我们可以直接使用它的spring boot例子来继续我们的任务。

例子其实比较简单，可能出于某种目的，并没有根据配置文件创建对应的redis连接，只是通过一个定时器不停的打印配置项，供我们来测试配置中心的实时同步和自动更新功能。

而我们接下来要使用disconf的配置变更来做到自动更新mybatis使用的数据源。


### 与mybatis结合

老实说，我并不是java高手，对spring boot的研究也很浅薄。基于我的同事搭建好的一个项目基础代码来完成我们的目标。

要完成我们的目标，我们可能需要做到以下几点：

1. disconf和spring boot完美结合（官方demo已经给了方法）
2. spring boot和mybatis结合（我同事的代码已经做好）
3. disconf和mybatis结合（我们要做的事儿）

代码我最终会上传到我的[github](https://github.com/kazaff/disconfDemo)上，下面我们来详细说说这个第三步。

要做到disconf和mybatis结合，其实说白了就是两件事儿：

1. 让mybatis使用的datasource配置项走disconf
2. 当disconf发布相关配置项的变更事件后，mytabais能感知到

第一步比较好搞：

```java
		@Bean(name = "dataSource")
    public DataSource dataSource(DBConfig dbConfig){
        return  DataSourceBuilder.create()
                .url(dbConfig.getUrl())
                .username(dbConfig.getUsername())
                .password(dbConfig.getPassword())
                .driverClassName(dbConfig.getDriverClassName())
                .build();
    }

		@Bean(name = "sqlSessionFactory")
    public SqlSessionFactoryBean sqlSessionFactory(DataSource dataSource) {
        SqlSessionFactoryBean factoryBean = new SqlSessionFactoryBean();
        factoryBean.setDataSource(dataSource);
        factoryBean.setTypeAliasesPackage("xyz.uutech.www.opencartservice.model");

        PageHelper pageHelperPlugin = new PageHelper();
        Properties properties = new Properties();
        properties.setProperty("dialect", "mysql");
        pageHelperPlugin.setProperties(properties);

        Interceptor[] plugins = new Interceptor[] {pageHelperPlugin};
        factoryBean.setPlugins(plugins);

        return factoryBean;
    }

		@Bean
    public MapperScannerConfigurer mapperScannerConfigurer() {
        MapperScannerConfigurer mapperScannerConfigurer = new MapperScannerConfigurer();
        mapperScannerConfigurer.setBasePackage("xyz.uutech.www.opencartservice.repository");
        mapperScannerConfigurer.setSqlSessionFactoryBeanName("sqlSessionFactory");
        return mapperScannerConfigurer;
    }
```

就是自己创建datasource覆盖spring boot默认的即可，上面的例子中使用到的`DBConfig`类正是与disconf结合的点：

```java
@DisconfFile(filename = "db.properties")
public class DBConfig {
    private String url;
    private String username;
    private String password;
    private String driverClassName;

    @DisconfFileItem(name = "url")
    public String getUrl() {
        return url;
    }

    public void setUrl(String url) {
        this.url = url;
    }

    @DisconfFileItem(name = "username")
    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    @DisconfFileItem(name = "password")
    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    @DisconfFileItem(name = "driver-class-name")
    public String getDriverClassName() {
        return driverClassName;
    }

    public void setDriverClassName(String driverClassName) {
        this.driverClassName = driverClassName;
    }
}
```

和官方demo基本上一样。

这样我们运行项目，即可看到会先和disconf通信并下载我们需要的配置文件到本地。

关键就是第二步，对我这种半吊子选手就有点麻烦了。我们要想做到动态变更datasource，需要借助` AbstractRoutingDataSource`这个类，网上有不少讨论spring boot下为mybatis配置多个数据源的文章，都是推荐使用这个抽象类来搞的。

依葫芦画瓢，我们也这么做：

```java
public class DisconfDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey(){
        return "TARGET";
    }
}
```

注意，我们的场景其实并非是多个数据源，我们的目的是替换旧的数据源，so，这里我们直接在`determineCurrentLookupKey`方法中固定返回一个标识位。

然后我们将第一步中定义的`dataSource`修改一下：

```java
	 @Bean(name = "dataSource")
	 public DataSource dataSource(DBConfig dbConfig){
			 DataSource ds =  DataSourceBuilder.create()
							 .url(dbConfig.getUrl())
							 .username(dbConfig.getUsername())
							 .password(dbConfig.getPassword())
							 .driverClassName(dbConfig.getDriverClassName())
							 .build();

			 Map<Object, Object> dss = new HashMap<>();
			 dss.put("TARGET", ds);

			 DisconfDataSource dds = new DisconfDataSource();
			 dds.setTargetDataSources(dss);

			 return dds;
	 }
```

注意这里我们要保持那个自定义的标识位一致。做到这里，我们其实已经让mybatis使用我们指定的数据源了，根据disconf官方的[Tutorial 14 ](https://github.com/knightliao/disconf/wiki/Tutorial14-bean-setter-mode)，我们还需要为对应的变更事件绑定回调：

```java
@Service
@DisconfUpdateService(classes = {DBConfig.class})
public class sqlSessionFactoryUpdateCallback implements IDisconfUpdate {

    @Autowired
    private DataSource dds;

    @Autowired
    private DBConfig dbConfig;

    @Override
    public void reload() throws Exception{
        DisconfDataSource targetDds =((DisconfDataSource) dds);

        //根据更新后的配置重建数据源
        DataSource dataSource = DataSourceBuilder.create()
                .url(dbConfig.getUrl())
                .username(dbConfig.getUsername())
                .password(dbConfig.getPassword())
                .driverClassName(dbConfig.getDriverClassName())
                .build();
        Map<Object, Object> dss = new HashMap<>();
        dss.put("TARGET", dataSource);
        targetDds.setTargetDataSources(dss);
        targetDds.afterPropertiesSet();
    }
}
```

注意`reload`方法的**最后一行**，由于我们的目的是替换旧的数据源（而非在多个数据源之间切换），所以我们必须避免使用`AbstractRoutingDataSource`为我们缓存起来的数据源，我们可以看一下这个`afterPropertiesSet`方法的实现细节：

```java
public void afterPropertiesSet() {
		if (this.targetDataSources == null) {
			throw new IllegalArgumentException("Property 'targetDataSources' is required");
		}
		this.resolvedDataSources = new HashMap<Object, DataSource>(this.targetDataSources.size());
		for (Map.Entry<Object, Object> entry : this.targetDataSources.entrySet()) {
			Object lookupKey = resolveSpecifiedLookupKey(entry.getKey());
			DataSource dataSource = resolveSpecifiedDataSource(entry.getValue());
			this.resolvedDataSources.put(lookupKey, dataSource);
		}
		if (this.defaultTargetDataSource != null) {
			this.resolvedDefaultDataSource = resolveSpecifiedDataSource(this.defaultTargetDataSource);
		}
	}
```

所以，这么做，我们才可以让项目立刻切换到新的数据源配置上。


### 遗留问题

我们虽然快草猛的做到了我们想要的效果，但是这里面有个疑问，由于spring boot默认会使用数据库连接池来提升性能，我们目前的这种切换datasource的方式，是否会造成一些无法察觉的bug或性能问题，希望有这方面研究的朋友可以给我留言解答。
