
## 生成的代码样例

```java
@FeignClient("userservice")
public interface UserClient {
    @RequestMapping(path="/user/{id}", method=RequestMethod.GET, params={"KEY1=VALUE1","..."},
                    headers={"",""}, consumer={"", ""}, produces={"", ""})
    User findById(@PathVariable("id") Long id);

    @RequestMapping(path="/user/name")
    User findByName(String name);
}
```

## fegin输入场要素

1. 服务目录输入场：选择服务目录中的服务，作为注解@FeignClient的value值
2. api输入场：选择返回结果类型，当前映射方法名称
3. header输入场(RequestMapping注解参数)：
   1. path(必选):访问路径。(路径中需要识别`{}`，在生成的方法中通过添加@PathVariable注解进行传参)
   2. params(可输入，多项): key-value键值对
   3. method(必选一个)：GET, HEAD, POST, PUT, PATCH, DELETE, OPTIONS, TRACE
   4. consumer(可选，多选)：{"text/plain", "application/*", 补全...}
   5. produces(可选，多选)：{"text/plain;charset=UTF-8", "text/plain", "application/*", 补全...}
   6. header(可输入，多项)：key-value键值对

4. body输入场，作为上面的方法入参，在页面输入的一般为string







