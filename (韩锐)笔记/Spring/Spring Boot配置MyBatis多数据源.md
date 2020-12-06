`mybatis-spring-boot-starter`提供了在Spring Boot之上快速构建MyBatis应用程序的能力，但它目前还不支持多数据源场景。  

为了使用多数据源，一种方式是为每个`DataSource`注册对应的`SqlSessionFactory`，并使用`@MapperScan`注解来明确指定需要使用的`SqlSessionFactory`。  
以下将以查询`CMS2015`库的`News`表和`VideoBase2013`库的`Video`表为示例来实现这种方式：

### 1. 添加POM依赖
``` xml
<parent>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-parent</artifactId>
	<version>1.5.9.RELEASE</version>
	<relativePath/>
</parent>

...

<dependencies>
    <dependency>
        <groupId>com.microsoft.sqlserver</groupId>
        <artifactId>mssql-jdbc</artifactId>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>org.mybatis.spring.boot</groupId>
        <artifactId>mybatis-spring-boot-starter</artifactId>
        <version>1.3.1</version>
    </dependency>
</dependencies>
```
由于将要连接的2个库均是sqlserver，所以需要添加`mssql-jdbc`依赖，如果您使用的是mysql，需要将依赖改成：
``` xml
<dependency>
	<groupId>mysql</groupId>
	<artifactId>mysql-connector-java</artifactId>
</dependency>
```
这里无需指定版本号，因为使用的parent`spring-boot-starter-parent`已经定义了默认版本号。

### 2. 添加数据源配置
在`application.yml`添加以下配置：
``` yml
demo:
  datasource:
    news:
      driver-class-name: com.microsoft.sqlserver.jdbc.SQLServerDriver
      url: jdbc:sqlserver://192.168.0.128:1433;database=CMS2015
      username: cms
      password: abc.123
    video:
      driver-class-name: com.microsoft.sqlserver.jdbc.SQLServerDriver
      url: jdbc:sqlserver://192.168.0.128:1433;database=VideoBase2013
      username: cms
      password: abc.123
```
这些均是测试环境的连接字符串和账号，如果连接不上，还请使用最新配置。

### 3. 配置数据源Bean
新建java类`DatabaseConfiguration.java`，并添加以下信息：
``` java
@Configuration
public class DatabaseConfiguration {
    @Bean
    @ConfigurationProperties("demo.datasource.news")
    public DataSourceProperties newsDataSourceProperties() {
        return new DataSourceProperties();
    }

    @Bean(name = {"newsDataSource"})
    public DataSource newsDataSource(){
        return newsDataSourceProperties().initializeDataSourceBuilder().build();
    }

    @Bean
    @ConfigurationProperties("demo.datasource.video")
    public DataSourceProperties videoDataSourceProperties() {
        return new DataSourceProperties();
    }

    @Bean(name = {"videoDataSource"})
    public DataSource videoDataSource(){
        return videoDataSourceProperties().initializeDataSourceBuilder().build();
    }
}
```
以上代码分别为news和video注册了`DataSourceProperties`和`DataSource`，并分别为这2个`DataSource`定义了`name`属性，以便稍后注入使用。

### 4. 配置SqlSessionFactory
新建java类`DatabaseConfiguration.java`，并添加以下信息：
``` java
@Configuration
public class MyBatisConfiguration {

    @Configuration
    @MapperScan(basePackages = {"com.example.news.dao"}, sqlSessionFactoryRef = "newsSqlSessionFactory")
    static class AggrConfiguration {

        @Bean(name = "newsSqlSessionFactory")
        @Primary
        public SqlSessionFactory newsSqlSessionFactory(@Qualifier("newsDataSource") DataSource aggrDataSource)
                throws Exception {
            return createSessionFactory(aggrDataSource, "classpath:mapper/news/*.xml");
        }
    }

    @Configuration
    @MapperScan(basePackages = {"com.example.video.dao"}, sqlSessionFactoryRef = "videoSqlSessionFactory")
    static class VideoConfiguration {

        @Bean(name = "videoSqlSessionFactory")
        public SqlSessionFactory videoSqlSessionFactory(@Qualifier("videoDataSource") DataSource newsDataSource) throws Exception {
            return createSessionFactory(newsDataSource, "classpath:mapper/video/*.xml");
        }
    }

    private static SqlSessionFactory createSessionFactory(DataSource dataSource, String xmlResourceLocation) throws Exception {
        final SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
        sqlSessionFactoryBean.setDataSource(dataSource);
        sqlSessionFactoryBean.setMapperLocations(new PathMatchingResourcePatternResolver()
                .getResources(xmlResourceLocation));

        return sqlSessionFactoryBean.getObject();
    }
}
```
由于注解不能同时使用多次，所以这里定义了2个静态内部类，并添加`@MapperScan`注解，其中`basePackages`申明类
mapper接口所在的包（稍后将创建），`sqlSessionFactoryRef`表示要使用的SessionFactory，而SessionFactory在内内部定义并注册为Bean。在定义SessionFactory时，使用了注解`@Qualifier`引用不同的`DataSource`，同时配置了xml文件路径（稍后将创建）。  

### 5. 添加MyBatis文件
创建包`com.example.news.dao`、新建文件`NewsDao.java`，并添加以下内容：
``` java
public interface NewsDao {
    String getTitle(int newsId);
}
```
创建包`com.example.video.dao`、新建文件`VideoDao.java`，并添加以下内容：
``` java
public interface VideoDao {
    String getTitle(int videoId);
}
```
在resources下创建目录`mapper/news`，新建文件`news.xml`，并添加以下内容：
``` xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="com.example.news.dao.NewsDao">
    <select id="getTitle" resultType="String">
        SELECT Title FROM dbo.News WHERE NewsId=#{newsId}
    </select>
</mapper>
```
在resources下创建目录`mapper/video`，新建文件`video.xml`，并添加以下内容：
``` xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="com.example.video.dao.VideoDao">
    <select id="getTitle" resultType="String">
        SELECT Title FROM dbo.Video WHERE VideoId=#{videoId}
    </select>
</mapper>
```
至此所有的配置都已完成，项目的结构将类似于：
```
├── src/main/
    ├── java/
    │   ├── com.example.demo/
    │   │   ├── DatabaseConfiguration.java
    │   │   ├── DemoApplication.java
    │   │   └── MyBatisConfiguration.java
    │   ├── com.example.news.dao/
    │   │   └── NewsDao.java
    │   └── com.example.video.dao/
    │       └── VideoDao.java
    └── resources/
        └── mapper/
            ├── news/
            │   └── news.xml
            ├── video/
            │   └── video.xml
            └── application.yml
```

### 5. 使用
新建示例文件`Runner.java`，并添加以下内容：
``` java
@Component
public class Runner implements CommandLineRunner {
    @Autowired
    private NewsDao newsDao;
    @Autowired
    private VideoDao videoDao;

    @Override
    public void run(String... strings) {
        String newsTitle = newsDao.getTitle(6573559);
        System.out.printf("News Title: %s \n", newsTitle);

        String videoTitle = videoDao.getTitle(257364);
        System.out.printf("Video Title: %s \n", videoTitle);
    }
}
```
由于`Runner`实现了`CommandLineRunner`接口，Spring Boot将在应用程序启动后运行它，详情见[官方文档](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-spring-application.html#boot-features-command-line-runner)。  
运行应用程序后，将如期看到类似以下输出：
```
News Title: 广汽传祺“王者SUV”茂名万丰震撼上市会 
Video Title: 群主三分球22222222222222222222222222
```

示例完整代码地址：http://gitlab.bitautotech.com/hanrui3/mybatis-multi-datasource-demo

### 6. 参考
https://github.com/mybatis/spring-boot-starter/issues/78
