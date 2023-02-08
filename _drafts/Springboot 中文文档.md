
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

`@EnableAutoConfiguration`隐式定义了项目**包搜索**的基点。

### 1.3 配置类
1. Springboot支持xml与java配置。通常建议在主配置类中添加`@Configuration`注解，一个好的选择是将**主应用类**作为**主配置类**。
2. 不需要把所有的`@Configuration`放在一个配置类中，可以通过**`@Import`**注解导入其他配置类。
3. 通过`@ComponentScan`注解扫描其他包，发现额外的注解。
4. 通过`@ImportResource`注解来加载xml配置文件，兼容兼容老系统已经使用了xml的方式进行发布了的配置文件。

*@Enable\* 注解帮忙很大的忙*

### 1.4 自动配置
将`@EnableAutoConfiguration`注解放在其中一个`@Configuration`注解配置类中即可开启自动配置。

#### 1.4.1 查看当前自动配置
通过启动参数--debug进行启动应用，可以了解当前正在使用的自动配置在日志中打印当前使用的自动配置

#### 1.4.2 禁用指定配置类
1. 通过`@EnableAutoConfiguration`的exclude禁用指定的自动配置类。
2. 通过`spring.autoconfigure.exclude` property控制要排除的自动配置类列表。


### 1.5 使用devtools

#### 1.5.1 引入spring-boot-devtools

devtools提供一系列开发时的便利功能，如禁用缓存，自动重启，热加载等。

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```
*将maven的依赖标记为`optional`的时候，当完全打包的时候devtools将会被禁用。同时防止项目被别的项目使用时应用到别的项目*

重新打包的归档默认情况下不包含 devtools。如果要使用某些远程 devtools 功能, 你需要禁用 excludeDevtools 构建属性以把 devtools 包含进来。该属性支持 Maven 和 Gradle 插件。


#### 1.5.2 devtools的Property默认值
主要用来禁用一些缓存操作，例如一些模版引擎的模版缓存，http静态资源的mvc http缓存头。
缓存在生产中有用，但是在开发中可能就不需要了，我们要及时看到变更后的变化，而不是之前缓存的东西。

一般可以在application.properties文件中设置缓存。而devtools则内置了一些自动屏蔽的默认值。具体见[DevToolsPropertyDefaultsPostProcessor](https://github.com/spring-projects/spring-boot/blob/v2.0.1.RELEASE/spring-boot-project/spring-boot-devtools/src/main/java/org/springframework/boot/devtools/env/DevToolsPropertyDefaultsPostProcessor.java)

#### 1.5.3 文件变更自动重启IOC容器
使用devtools的应用在classpath下的文件发生变更的时候会自动重启IOC容器。

注意的事项：
1. 如果想使用mvn命令，也就是maven来启动应用，需要将打包插件中的fork打开。应为devtool需要隔离应用类加载器才能生效(盲猜一波devtool的监听重启实现是通过自定义类加载器实现的？)
2. 自动重启和liveReload一起使用更好。如果有使用JRebel，那么devtool的自动重启将会被禁用，其他的特性liveReload，内置属性这些依然生效。
3. DevTools 依赖于应用上下文的关闭钩子，以在重启期间关闭自己。如果禁用了关闭钩子（SpringApplication.setRegisterShutdownHook(false)），它将不能正常工作。
4. 当 classpath 下的内容发生更改，决定是否触发重启时，DevTools 会自动忽略名为 spring-boot、spring-boot-devtools、spring-boot-autoconfigure、spring-boot-actuator 和 spring-boot-starter 的项目。

**【Restart和Relaod实现原理】**

Spring Boot通过使用两个类加载器来提供重启技术，不改变的类被加载到base类加载器中。经常处于开发态的类被加载到restart类加载器中。
当应用重启时，restart类加载器将会被丢弃重建。这样省去了base类的重建动作，可以复用里面的类资源，比单纯的冷重启快很多。

**【重启相关配置】**

```properties
spring.devtools.restart.log-condition-evaluation-delta=false
```
控制每次重启时，条件评估的增量报告

```properties
spring.devtools.restart.exclude=static/**,public/**
```
某些资源在更改时不一定需要触发重启。例如，Thymeleaf 模板可以实时编辑。默认情况下，更改 /META-INF/maven、/META-INF/resources、/resources、/static、/public 或者 /templates **不会触发restart，但会触发LiveReload**。如果您想自定义排除项，可以使用 spring.devtools.restart.exclude 属性

```properties
spring.devtools.restart.additional-exclude
```
如果要保留这些默认值并添加其他排除项,可用这个属性

```properties
spring.devtools.restart.additional-paths
```
添加非classpath文件重启监听器

```properties
spring.devtools.restart.enabled
```

**【禁用重启】**

您如果不想使用重启功能，可以使用 spring.devtools.restart.enabled 属性来禁用它。一般情况下，您可以在 application.properties 中设置此属性（重启类加载器仍将被初始化，但不会监视文件更改）。

如果您需要完全禁用重启支持（例如，可能它不适用于某些类库），您需要在调用 SpringApplication.run(​...) 之前将 System 属性 spring.devtools.restart.enabled System 设置为 false。例如：

```java
public static void main(String[] args) {
    System.setProperty("spring.devtools.restart.enabled", "false");
    SpringApplication.run(MyApp.class, args);
}
```

**【使用触发文件】**

如果您使用 IDE 进行开发，并且时时刻刻在编译更改的文件，或许您只是希望在特定的时间内触发重启。为此，您可以使用触发文件，这是一个特殊文件，您想要触发重启检查时，必须修改它。更改文件只会触发检查，只有在 Devtools 检查到它需要做某些操作时才会触发重启，可以手动更新触发文件，也可以通过 IDE 插件更新。

要使用触发文件，请设置 spring.devtools.restart.trigger-file 属性指向触发文件的路径。


**【自定义jar包所属加载器】**

默认情况下，IDE中打开的项目都是用restart类加载器加载。常规`jar`都使用`base`类加载器加载。如果需要自定义
的话可以创建一个META-INF/spring-devtools.properties文件。
以 restart.exclude. 和 restart.include. 为前缀的属性。include是加载到restart类加载器里面的，exclude是加载到base类加载器里面的。

```properties
restart.exclude.companycommonlibs=/mycorp-common-[\\w-]+\.jar
restart.include.projectcommon=/mycorp-myproj-[\\w-]+\.jar
```

#### 1.5.4 LiveRelaod
livereload特性主要用于web资源的重新加载，当静态资源更新的时候触发浏览器的刷新。devtool内置了一个
liveReload服务器，然后配合[livereload.com](http://livereload.com/extensions/)进行使用。

如果想禁用这个功能，可以将`spring.devtools.livereload.enabled`属性设置为false。

#### 1.5.5 全局设置

可以在`$HOME`目录中添加`.spring-boot-devtools.properties`文件来配置全局devtools配置。

