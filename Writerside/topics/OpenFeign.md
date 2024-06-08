# OpenFeign
### 入门
- 引入依赖 包括OpenFeign和负载均衡组件SpringCloudLoadBalancer
```xml
<!--        openfeign-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>
<!--        负载均衡器-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-loadbalancer</artifactId>
        </dependency>
```
- 通过注解 启用OpenFeign功能
   @EnableFeignClients
- 编写FeignClient
```Java
@FeignClient("item-service")
public interface ItemClient {
    @GetMapping
    List<ItemDTO> queryItemByIds(@RequestParam("ids") List<Long> ids);
}

```
- 使用FeignClient 实现远程调用
```Java
 private final ItemClient itemClient;
 List<ItemDTO> items = itemClient.queryItemByIds(itemIds);
 
```
## 连接池
- OpenFeign对http请求做了优雅的伪装，不过其底层发起http请求，依赖于其他的框架。这些框架可以自己选择
  - HttpURLConnection 默认实现 不支持连接池
  - Apache HttpClient 支持连接池
  - OKHttp 支持连接池
### 整合OKHttp
```xml
 <dependency>
            <groupId>io.github.openfeign</groupId>
            <artifactId>feign-okhttp</artifactId>
        </dependency>
```
```yaml
feign:
  okhttp:
    enabled: true
```
## 最佳实践
![拆分1](拆分1)
![拆分2](拆分2)
- 如果报错 则是实体类没有被扫描到
`@EnableFeignClients(basePackages = "com.hmall.api.client")`
## 日志
- OpenFeign 只会在 FeignClient 所在包的日志级别为 DEBUG 时，才会输出日志。而且其日志级别有4级:
  - NONE:不记录任何日志信息，这是默认值
  - BASIC:仅记录请求的方法，URL以及响应状态码和执行时间
  - HEADERS :在 BASIC的基础上，额外记录了请求和响应的头信息
  - FULL:记录所有请求和响应的明细，包括头信息、请求体、元数据
- 要自定义日志级别需要声明一个类型为Logger.Level的Bean
```Java
public class DefaultFeignConfiguration {
    @Bean
    public Logger.Level level(){
        return Logger.Level.FULL;
    }
}

```
`@EnableFeignClients(defaultConfiguration = DefaultFeignConfiguration.class, basePackages = "com.hmall.api.client")`

