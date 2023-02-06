

@Import 是spring 3.0引入的一个概念，语义是像代码improt那样子import进spring的bean进IOC容器中。
springboot的自动装配机制就是基于@import实现的。


@Import注解的的解析类`ConfigurationClassParser`,
```
sequenceDiagram

AbstractApplicationContext ->> AbstractApplicationContext:refresh
AbstractApplicationContext ->> AbstractApplicationContext:invokeBeanFactoryPostProcessors
AbstractApplicationContext ->> PostProcessorRegistrationDelegate:invokeBeanFactoryPostProcessors
PostProcessorRegistrationDelegate ->> PostProcessorRegistrationDelegate:invokeBeanDefinitionRegistryPostProcessors
PostProcessorRegistrationDelegate ->> ConfigurationClassPostProcessor :postProcessBeanDefinitionRegistry
ConfigurationClassPostProcessor ->> ConfigurationClassPostProcessor:processConfigBeanDefinitions
ConfigurationClassPostProcessor ->> ConfigurationClassParser:parse
ConfigurationClassParser ->> ConfigurationClassParser:processConfigurationClass
ConfigurationClassParser ->> +ConfigurationClassParser:doProcessConfigurationClass
ConfigurationClassParser ->> ConfigurationClassParser:getImports
ConfigurationClassParser ->> -ConfigurationClassParser:processImports
```

最后的processImports主要的逻辑：
1. 判断@Import的value是否是ImportSelector接口的实现类，是的话调用接口的selectImport方法，导入对应的bean
2. 判断@Import的value是否是ImportBeanDefinitionRegistrar,是的话调用接口的registerBeanDefinitions方法，注册对应的beanDefinitions

springboot的`@EnableAutoConfiguration`注解就是利用@Import注解，通过AutoConfigurationImportSelector的selectImport方法实现自动导入，
这个方法里面主要做的事情就是读取`/resources/META-INF/spring.factories`文件中读取`org.springframework.boot.autoconfigure.EnableAutoConfiguration`对应的value。
然后交给spring去处理。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {

	/**
	 * Environment property that can be used to override when auto-configuration is
	 * enabled.
	 */
	String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

	/**
	 * Exclude specific auto-configuration classes such that they will never be applied.
	 * @return the classes to exclude
	 */
	Class<?>[] exclude() default {};

	/**
	 * Exclude specific auto-configuration class names such that they will never be
	 * applied.
	 * @return the class names to exclude
	 * @since 1.3.0
	 */
	String[] excludeName() default {};

}
```

















- [Spring @Import 机制](https://www.jianshu.com/p/3c5922ec3686)
-
