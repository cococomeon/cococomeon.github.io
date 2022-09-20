---
layout: post
title: SnakeYAML
subtitle:
categories: Yaml
tags: []
banner: "/assets/images/banners/home.jpeg"

---

## **一. API入口**

```java
Yaml yaml = new Yaml()
```

线程不安全， 不可混用



## **二. 基本用法**

snakeyaml支持从stream和string中加载文档

```yaml
firstName: "John"
lastName: "Doe"
age: 20
```

```java
Yaml yaml = new Yaml();
InputStream inputStream = this.getClass()
  .getClassLoader()
  .getResourceAsStream("customer.yaml");
Map<String, Object> obj = yaml.load(inputStream);
System.out.println(obj);
```

上面的代码加载出来的就是一个map对象代表yaml中的键值对，下面就是进阶用法， 自定义类型。



## **二. 进阶用法**

### **2.1 自定义类型解析**

```yaml
firstName: "John"
lastName: "Doe"
age: 31
contactDetails:
   - type: "mobile"
     number: 123456789
   - type: "landline"
     number: 456786868
homeAddress:
   line: "Xyz, DEF Street"
   city: "City Y"
   state: "State Y"
   zip: 345657
```

```java
public class Customer {
    private String firstName;
    private String lastName;
    private int age;
    private List<Contact> contactDetails;
    private Address homeAddress;    
    // getters and setters
}

public class Contact {
    private String type;
    private int number;
    // getters and setters
}

public class Address {
    private String line;
    private String city;
    private String state;
    private Integer zip;
    // getters and setters
}
```

```java
public void whenLoadYAMLDocumentWithTopLevelClass_thenLoadCorrectJavaObjectWithNestedObjects() {
  
    Yaml yaml = new Yaml(new Constructor(Customer.class));
    InputStream inputStream = this.getClass()
      .getClassLoader()
      .getResourceAsStream("yaml/customer_with_contact_details_and_address.yaml");
    Customer customer = yaml.load(inputStream);
}
```

### **2.2 类型安全**

当一个需要load的类的一个或者多个类有泛型集合类的时候，需要TypeDescription进行指定泛型的参数类型。因为运行时的list是经过了泛型擦除了， 运行时的list中的元素是Object，snakeyaml不知道转成什么类，需要通过TypeDescription来进行指定。

```java
Constructor constructor = new Constructor(Customer.class);
TypeDescription customTypeDescription = new TypeDescription(Customer.class);
customTypeDescription.addPropertyParameters("contactDetails", Contact.class);
constructor.addTypeDescription(customTypeDescription);
Yaml yaml = new Yaml(constructor);
```

## **三. yaml转文件**

### **3.1 基本用法**

```java
public void whenDumpMap_thenGenerateCorrectYAML() {
    Map<String, Object> data = new LinkedHashMap<String, Object>();
    data.put("name", "Silenthand Olleander");
    data.put("race", "Human");
    data.put("traits", new String[] { "ONE_HAND", "ONE_EYE" });
    Yaml yaml = new Yaml();
    StringWriter writer = new StringWriter();
    yaml.dump(data, writer);
    String expectedYaml = "name: Silenthand Olleander\nrace: Human\ntraits: [ONE_HAND, ONE_EYE]\n";
 
    assertEquals(expectedYaml, writer.toString());
}
```

### **3.2 自定义java对象序列化**

```java
public void whenDumpACustomType_thenGenerateCorrectYAML() {
    Customer customer = new Customer();
    customer.setAge(45);
    customer.setFirstName("Greg");
    customer.setLastName("McDowell");
    Yaml yaml = new Yaml();
    StringWriter writer = new StringWriter();
    yaml.dump(customer, writer);        
  //实际生成的yaml文件会包含!!com.baeldung.snakeyaml.Customer
    String expectedYaml = "!!com.baeldung.snakeyaml.Customer {age: 45, contactDetails: null, firstName: Greg,\n  homeAddress: null, lastName: McDowell}\n";
 
    assertEquals(expectedYaml, writer.toString());
}
```

为了避免上面的不必要的冗余信息， 可以使用

```java
yaml.dumpAs(customer, Tag.MAP, null);
```



