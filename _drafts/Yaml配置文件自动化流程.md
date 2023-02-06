

写一个树形配置类自动解析的小框架，配置文件有多个，配置文件是yaml格式。

总体思路
通过借助snakeyaml进行解析yaml, 但是snakeyaml一次只能解析一个自定义配置类，
```java
//snakeyaml解析映射成类的时候，通过Constructor进行传递自定义的解析规则以解析出复杂javabean配置类。
Yaml yaml = new Yaml(new Constructor(配置类.class));
```
将每个配置文件读取成一个模块配置类, 然后再通过一定的规则，通过反射装配的方式将这个配置文件注入到总的配置文件类中。

## 配置类
```java
import java.lang.reflect.Constructor;
import java.security.cert.X509Certificate;

//主配置类，存在多个模块配置类
public class AllConfig {

  private ModuleOneConfig one;

  private ModuleTwoConfig two;

  private ModuleThreeConfig three;

  //略...
}

//子模块配置类
@ModuleConfig(parent = Allconfig.class, file = "module-one.yaml")
public class ModuleOneConfig {

  private String xx1;

  private List<String> xx2;

  //略...
}

@ModuleConfig(parent = Allconfig.class, file = "module-two.yaml")
public class ModuleTwoConfig {
  //略...
}

```

## 解析规则
将自定义的配置类解析规则封装成一个一个个的`ConfigRule`
```java
//配置类解析规则接口
public interface ConfigRule {

  public Constructor constructor(); //每个配置类构建的时候需要的解析规则

  public List<TypeDescription> typeDescriptiuon();//主要用来设置上面的Constructor的类型描述

}

public class AllConfigRule implements ConfigRule {

}

//子模块解析规则
public class ModuleOneConfigRule implements ConfigRule {

  @Override
  public Constructor constructor() {
    Constructor constructor = new Constructor(ModuleOneConfig.class);
    constructor.setPropertyUtils(new PropertyUtils());
    typeDescription().foreach(a -> constructor.addTypeDescription(a));
  }

  @Override
  public List<TypeDescription> typeDescription() {
    //模块解析时需要自定义的解析规则描述，具体支持哪些参考snakeYaml官网文档描述
    List<TypeDescription> result = new ArrayList<TypeDescription>();

  }
}



```

## 解析工厂

```java

import java.io.File;
import java.io.FileInputStream;

//解析工厂，解析工具类
public class ConfigFactory {

  public static <T> T getConfig(String fileName, Class<? extends T> configClass) {
    Yaml yaml = new Yaml(getConstructor(configClass));
    FileInputStream fis = new FileInputStream(new File(fileName));
    return yaml.loadAs(fis, configClass);
  }

  public static Constructor getConstructor(Class clazz){
      //根据class获取上面封装的construct， 可以将class和对应的configrule封装成Enum
    return ConfigStrage.parse(clazz).configRule().constructor();

  }

}

//借助枚举类型，将每个configrule封装成一个个策略，消除根据clazz返回不同的configrule的if/else或者switch。
public enum ConfigStrage {

    ALL_CONFIG(AllConfig.class){
      @Override
      public ConfigRule configRule(){
          return new AllConfigRule();
      }
    },

    MODULE_ONE_CONFIG(ModuleOneConfig.class){
      @Override
      public ConfigRule configRule(){
          return new ModuleOneConfigRule();
      }
    },

    MODULE_TWO_CONFIG(ModuleTwoConfig.class){
      @Override
      public ConfigRule configRule(){
        return new ModuleOneConfigRule();
      }
    };

    private Class clazz;

    private ConfigStrage (Class clazz) {
        this.clazz = clazz;
    }



    public ConfigStrage parse(Class clazz){
        //根据class解析出上面对应的configrule
    }

    public abstract ConfigRule configRule();

}

```


## 配置类加载器

```java

public abstract class ConfigLoader<T> {

  //初始化，将所有的配置类扫描出来进行缓存
  static {

  }

  public abstract T load();

  public <T> T getConfig(String fileName, Class<? extends T> configClass) {
    return ConfigFactory.getConfig(fileName, configClass);
  }
}

public class AllConfigLoader extends ConfigLoader<AllConfig> {

  public AllConfig load() {
      //根据主配置(根)类加载配置类，然后再加载主(根)类进行组装配置类
    Allconfig all = getconfig("all.yaml", AllConfig.class);

    //根据扫描缓存出来的配置类找到属于allconfig的子模块，然后同样的getconfig，再设置进去。
  }

}



```
