### 1. 不使用Spring Boot提供的Parent POM
如果不使用`spring-boot-starter-parent`，那么需要将`scope`修改为`import`：
``` xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-dependencies</artifactId>
      <version>1.5.8.RELEASE</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
    <!-- ... -->
  </dependencies>
</dependencyManagement>
```
并且需要引用`spring-boot-maven-plugin`，否则提示“xxxx.jar 中没有主清单属性”：
``` xml
<build>
  <plugins>
    <plugin>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-maven-plugin</artifactId>
      <version>1.5.8.RELEASE</version>
      <executions>
        <execution>
          <goals>
            <goal>repackage</goal>
          </goals>
        </execution>
      </executions>
      <configuration>
          <mainClass>${start-class}</mainClass>
      </configuration>
    </plugin>
  </plugins>
</build>
```

### 2. 将项目打包成war包
首先需要将启动类修改为继承自`SpringBootServletInitializer`：
``` java
@SpringBootApplication
public class Application extends SpringBootServletInitializer {

	@Override
	protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
		return application.sources(Application.class);
	}

	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}
}
```
同时修改POM文件的打包方式为war，如果需要将war部署到外部容器，那么需要将内嵌的tomcat修改为`provided`：
``` xml
<packaging>war</packaging>
<!-- ... -->
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-tomcat</artifactId>
    <scope>provided</scope>
  </dependency>
  <!-- ... -->
  </dependencies>
```
最后，需要引用plugin，否则在maven打包的时候提示错误：
> Failed to execute goal org.apache.maven.plugins:maven-war-plugin:2.2:war (default-war) on project springcloud-config-sampleclient: Error assembling WAR: webxml attribute is required (or pre-existing WEB-INF/web.xml if executing in update mode) -> [Help 1]
``` xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-war-plugin</artifactId>
  <version>3.2.0</version>
  <configuration>
    <failOnMissingWebXml>false</failOnMissingWebXml>
  </configuration>
</plugin>
```

### 3. 将项目配置为打包成war包后，在Idea中运行出错，但maven一切正常
如果出现类似一下错误：  
> java.lang.IllegalStateException: ApplicationEventMulticaster not initialized - call 'refresh' before multicasting events via the context: org.springframework.context.annotation.AnnotationConfigApplicationContext@24dbe3f: startup date [Mon Nov 27 11:31:37 CST 2017]; parent: org.springframework.context.annotation.AnnotationConfigApplicationContext@704d1261


那么需要在Idea中将`spring-boot-starter-tomcat`和`tomcat-embed-core`配置成Compile，具体步骤：
Project structure -> Modules -> 要配置的模块 -> Dependencies -> 找到依赖项并在Scope列选择Compile，具体参考这个[SO回答](https://stackoverflow.com/a/43472825/1965211)

### 4. 在CentOS6.9上将项目部署为System V服务时出错
提示以下错误
> invalid file (bad magic number): Exec format error

需要在POM中将`executable`配置为`true`：
``` xml
<build>
	<finalName>eureka-server</finalName>
	<plugins>
		<plugin>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-maven-plugin</artifactId>
			<version>1.5.8.RELEASE</version>
			<executions>
				<execution>
					<goals>
						<goal>repackage</goal>
					</goals>
				</execution>
			</executions>
			<configuration>
				<mainClass>${start-class}</mainClass>
				<executable>true</executable>
			</configuration>
		</plugin>
	</plugins>
</build>
```

### 5. 在Tomcat下部署多个Spring Boot应用出错
提示以下错误
> org.springframework.jmx.export.UnableToRegisterMBeanException: Unable to register MBean [org.springframework.cloud.context.environment.EnvironmentManager@bad5693] with key 'environmentManager'; nested exception is javax.management.InstanceAlreadyExistsException: org.springframework.cloud.context.environment:name=environmentManager,type=EnvironmentManager

对于此种情况，[需要分隔MBean名称空间或者直接禁用MBean](https://github.com/spring-cloud/spring-cloud-config/issues/118)
``` yaml
#方法一 禁用jmx
spring.jmx.enabled=false
#方法二
spring.jmx.default-domain=my-app-name
```

## 参考
https://docs.spring.io/spring-boot/docs/current/reference/html/using-boot-build-systems.html  
https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#build-tool-plugins-maven-packaging
https://stackoverflow.com/questions/30237768/run-spring-boots-main-using-ide