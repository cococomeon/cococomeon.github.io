
## 一. 使用Springboot

### 1.1 Maven方式构建
#### 1.1.1 继承为工程的Parent
```xml
<!-- 从 Spring Boot 继承默认配置 -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.0.0.RELEASE</version>
</parent>
```

#### 1.1.2 不继承，导入依赖
```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <!-- 从 Spring Boot 导入依赖管理 -->
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>2.0.0.RELEASE</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### 1.1.3 使用springboot maven打包插件
```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

### 1.2 组织代码
#### 1.2.1 定位启动类
通常`@Configuration`、`@EnableAutoConfiguration`、 `@ComponentScan`等注解放到主启动类上面。
或者`@SpringBootApplication`注解。

### 1.3 配置类
Springboot支持xml与java配置。通常建议在主配置类中添加`@Configuration`注解，一个好的选择是将**主应用类**作为**主配置类**。
另外一个点是，不需要把所有的`@Configuration`放在一个配置类中，可以通过`@Import`注解导入其他配置类。还有可以通过
`@ComponentScan`注解扫描其他包，发现额外的注解。

除此之外，有一些老系统已经使用了xml的方式进行发布了配置文件的，也可以通过`@ImportResource`注解来加载xml配置文件。


*@Enable\* 注解帮忙很大的忙*

#### 1.4 自动配置
只需要在主配置类上添加一个`@EnableAutoConfiguration`注解既可开启自动配置。
