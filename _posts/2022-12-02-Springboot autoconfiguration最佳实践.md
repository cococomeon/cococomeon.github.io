---
layout: post
title: Springboot autoconfiguration最佳实践
subtitle:
categories: Spring
tags: []
banner: "/assets/images/banners/home.jpeg"

---


## **简介**
在spring初始化阶段，主要是要根据某些Resource资源找到对应的BeanDefinition，然后再初始化，注入依赖。
BeanDefinition生成主要有两种方式：
第一种: 通过Resource读取对应的资源，再配合对应的BeanDefinitionReader将这些资源解析成BeanDefinition注册到BeanDefinitionRegistry中。
第二种: 直接通过操作BeanDefinitionRegistry，不通过BeanDefinitionReader而通过自定义操作手动将资源映射成BeanDefinition注册进BeanDefinitionRegistry中。

springboot autoconfiguration主要走的是第一种分两部分：
1. 封装各个配置类，这些配置类通过一系列的condition来描述自己在什么情况下需要自动配置。
2. 如何找到配置类，主配置类中通过指定包扫描、显式import、spring.factories机制的方式，找到封装的@Configuration和@Bean，自动注入到IOC容器中。


## **一. 封装各个配置类**

### **1.1 自动配置基本条件**
1. 类条件，@ConditionOnClass, @ConditionalOnMissingClass
2. bean条件,@ConditionOnBean, @ConditionalOnMissingBean
3. property条件, @ConditionOnProperty
4. 资源条件, @ConditionOnResource
5. web条件, @ConditionOnWebApplication, @ConditionalOnNotWebApplication
6. SpEL条件, @ConditionalOnExpression

### **1.2 自动配置进阶条件**

#### **1.2.1 复合条件匹配**
需求：某一个功能的启用条件往往是多个条件符合在一起的，他们之间的关系可能是`与`关系，也可能是`或`关系。
可以通过springboot condition机制中提供的， 通过继承`AllNestedConditions`，`AnyNestedCondition`，`NoneNestedConditions`来实现condition之间的逻辑关系。
可以在继承类中定义多个内部静态class，每个class中的条件便是这个configuration启用的其中一个条件。

```java
//使用封装的condition限定配置类的启用条件
@Configuration
@ConditionalOnAfaBootApplication
public class XxxConfiguration {
  @Bean
  public ReferenceManager referenceManager() {
    return new ReferenceManager();
  }
}

//封装condition，通过@Conditional指定复合条件
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(HasBootMonitor.class)
public @interface ConditionalOnAfaBootApplication {

}

//封装复合条件类
public static class HasBootMonitor extends AllNestedConditions {

        public HasBootMonitor() {
            super(ConfigurationPhase.PARSE_CONFIGURATION);
        }

        @ConditionalOnProperty(value = "xxxxxx", matchIfMissing = true,
                               havingValue = Constraints.TRUE)
        static class OnEnabled {

        }

        @ConditionalOnClass(name = "cn.com.xx.xx.xx.xx.xxclass")
        static class OnClass {

        }

    }
```

#### **1.2.2 自定义条件匹配**
上面的基础匹配和进阶匹配都是通过已经定义好的conditionOnxxxx进行匹配，限定条件说明配置类可不可以启用。
需求：需要通过编程，根据代码执行结果作为条件来决定是否启用此配置类。

```java
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(BootVersionCtrl.class)
public @interface SupportedVersion{

}

public static class BootVersionCtrl extends SpringBootCondition {

        @Override
        public ConditionOutcome getMatchOutcome(ConditionContext context, AnnotatedTypeMetadata metadata) {
            MultiValueMap<String, Object> attributes = metadata.getAllAnnotationAttributes(
                ConditionalOnBootVersion.class.getName(), true);
            if (attributes == null) {
                return null;
            }
            BootVersion min = (BootVersion) attributes.get("minor").get(0);
            BootVersion max = (BootVersion) attributes.get("major").get(0);
            BootVersion[] in = (BootVersion[]) attributes.get("in").get(0);
            String version = SpringBootVersion.getVersion();
            if (!StringUtils.hasText(version)) {
                return ConditionOutcome.noMatch("SpringBoot version not found");
            }
            BootVersion actual = BootVersion.parse(version);
            if (actual == null) {
                return ConditionOutcome.noMatch(String.format("SpringBoot version(%s) unKnown.", version));
            }
            if (in.length > 0) {
                return checkInVersion(actual, in);
            }
            return checkBetweenVersion(actual, min, max);
        }

        private ConditionOutcome checkBetweenVersion(BootVersion actual, BootVersion min, BootVersion max) {
            boolean found = actual.isBetween(min, max);
            if (found) {
                return ConditionOutcome.match(
                    String.format("SpringBoot version(%s) supported. available:{ %s<=\"%s\"<=%s }\"", actual.getOrigin(),
                        min.getOrigin(),actual.getOrigin(), max.getOrigin()));
            }
            return ConditionOutcome.noMatch(
                String.format("SpringBoot version(%s) not supported. available:{ %s<=\"%s\"<=%s }", actual.getOrigin(),
                    min.getOrigin(),actual.getOrigin(), max.getOrigin()));
        }

        private ConditionOutcome checkInVersion(BootVersion actual, BootVersion[] in) {
            boolean found = Arrays.stream(in).allMatch(v -> v == actual);
            if (found) {
                return ConditionOutcome.match(String.format("SpringBoot version(%s) supported.", actual.getOrigin()));
            }
            return ConditionOutcome.noMatch(String.format("SpringBoot version(%s) not supported.", actual.getOrigin()));
        }
    }

```


## **二. 如何找到配置类**

主要由以下注解组成：
1. @EnableAutoConfiguration，在@Configuration主配置类中启动springboot自动配置机制，主要是去读取rsources/META-INF/spring.factories
2. @ComponentScan
3. @Configuration
4. @Import，显式导入对应的configurationbean


## **三. 手动注册配置类**

除了上面的通过编写各种的条件来触发是bean的自动装配外，还可以跳过这些条件的判断，直接操作BeanDefinitionRegistrar的方式进行注册BeanDefinition。
springboot 中获取beandefinition主要有两种方式触发：
1. 条件触发的被动装配bean的流程，通过各种条件(配置文件，bean，class等)被动装配对应的bean进容器中
2. 编码触发的主动装配bean的流程，通过各种的`@EnableXxx`注解进行自动装配，这个功能模块并不是通过条件触发的(配置文件,class,bean)等，
   而是用户要使用的时候就需要在代码中主动使用这个注解，通过这个注解显式启动这个模块



```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import({ SelectImport.class,xxxClientRegister.BeanDefinitionRegistrar.class })
public @interface EnableXxxClient {

    boolean template;

    boolean client;
}

public static class SelectImport implements ImportSelector {
  @Override
  public String[] selectImports(AnnotationMetadata importingClassMetadata) {
    List<String> result = new ArrayList<>(3);
    result.add(xxxConfiguration.class.getName());
    result.add(xxxClientConfiguration.class.getName());
    return StringUtils.toStringArray(result);
  }
}

public static class xxxClientRegister implements ImportBeanDefinitionRegistrar, ResourceLoaderAware,
  EnvironmentAware {
  private ResourceLoader resourceLoader;
  private Environment environment;
  @Override
  public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
    BeanDefinitionRegistry registry) {
    if(!Ctrl.isEnabled(environment)){
      logger.info("Xxx has been disabled.");
      return;
    }
    if (importingClassMetadata.hasAnnotation(EnableXxxClient.class.getName())) {
      boolean useTemplate = (boolean) importingClassMetadata
        .getAnnotationAttributes(EnableXxxClient.class.getName()).get("template");
      if (useTemplate) {
        registerSingleton(AbusTemplate.class, registry, builder -> {});
      } else {
      }
      boolean useClient = (boolean) importingClassMetadata
        .getAnnotationAttributes(EnableXxxClient.class.getName()).get("client");
      if (useClient) {
        registerAsfClients(importingClassMetadata,registry);
      } else {
      }
    }
  }

  @Override
  public void setEnvironment(Environment environment) {
    this.environment = environment;
  }

  @Override
  public void setResourceLoader(ResourceLoader resourceLoader) {
    this.resourceLoader = resourceLoader;
  }

  private void registerAsfClients(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
    AbusClientRegister.registerFeignClients(importingClassMetadata,registry,resourceLoader,environment);
  }

  private static void registerSingleton(Class<?> beanClass, BeanDefinitionRegistry registry,
    @Nullable
      Consumer<BeanDefinitionBuilder> builderConsumer) {
    if (registry.containsBeanDefinition(beanClass.getName()))
      return;
    BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition();
    builder.getRawBeanDefinition().setBeanClass(beanClass);
    builder.getRawBeanDefinition().setScope(BeanDefinition.SCOPE_SINGLETON);
    if (builderConsumer != null) {
      builderConsumer.accept(builder);
    }
    registry.registerBeanDefinition(beanClass.getName(), builder.getBeanDefinition());
  }

}
```







- [Bean与BeanDefinition的理解——理解spring的基础和关键](cnblogs.com/jing-yi/p/14242016.html)
- [Spring详解（五）——Spring IOC容器的初始化过程](cnblogs.com/tanghaorong/p/13497223.html)
- [SpringBoot启动原理与Spring核心内容](https://www.cnblogs.com/wk-missQ1/p/14330919.html)
