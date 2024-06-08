# 微服务保护和分布式事务 - Sentinel
## 安装
- 下载jar包
  - https://github.com/alibaba/Sentinel/releases
## 运行
`java -'Dserver.port'='8090' -'Dcsp.sentinel.dashboard.server'='localhost:8090' -'Dproject.name'='sentinel-dashboard' -jar sentinel-dashboard.jar`
- 账号/密码 sentinel/sentinel
## 微服务整合
- 依赖
```xml
 <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
        </dependency>
```
- 配置文件
```yaml
spring:
  cloud:
    sentinel:
      transport:
        dashboard: localhost:8080
      http-method-specify: true #开启请求方式作为资源名称
```
## 簇点链路
就是单机调用链路。是一次请求进入服务后经过的每一个被 Sentinel监控的资源链。默认 Sentinel会监
控 SpringMVC 的每一个 Endpoint (http 接口)。限流、熔断等都是针对簇点链路中的资源设置的。而资源名默
认就是接口的请求路径
- http-method-specify: true #开启请求方式作为资源名称
## 请求限流
![sentinel2.png](sentinel2.png)
## 线程隔离
![sentinel3.png](sentinel3.png)
## Fallback
- 将FeignClient作为Sentinel的簇点资源
```yaml
feign:
  sentinel:
    enabled: true
```
- FeignClient的Fallback有两种配置方式
  - 方式一：FallbackClass，无法对远程调用的异常做处理
  - 方式二：FallbackFactory，可以对远程调用的异常做处理
### 步骤一
自定义类 实现FallbackFactory，编写对某个FeignClient的fallback逻辑
```Java
@Slf4j
public class ItemClientFallbackFactory implements FallbackFactory<ItemClient> {
    @Override
    public ItemClient create(Throwable cause) {
        return new ItemClient() {
            @Override
            public List<ItemDTO> queryItemByIds(Collection<Long> ids) {
                log.error("查询商品失败",cause);
                return CollUtils.emptyList();
            }

            @Override
            public void deductStock(List<OrderDetailDTO> detailDTOS) {
                log.error("扣减商品库存失败",cause);
                throw new RuntimeException(cause);
            }
        };
    }
}

```
### 步骤二
将定义的factory注册为一个Bean
```Java
public class DefaultFeignConfiguration {
    @Bean
    public Logger.Level level(){
        return Logger.Level.FULL;
    }
    @Bean
    public ItemClientFallbackFactory itemClientFallbackFactory(){
        return new ItemClientFallbackFactory();
    }
    @Bean
    public UserinfoRequestInterceptor userinfoRequestInterceptor(){
        return new UserinfoRequestInterceptor();
    }
}


```
### 步骤三
在Client接口中使用Factory
```Java
@FeignClient(value = "item-service",fallbackFactory = ItemClientFallbackFactory.class)
```
## 服务熔断
- 熔断是解决雪崩问题的重要手段。思路是由断路器统计服务调用的异常比例、慢请求比例，如果超出阈值则会熔断该
  服务。即拦截访问该服务的一切请求；而当服务恢复时，断路器会放行访问该服务的请求。
![sentinel4.png](sentinel4.png)
![sentinel5.png](sentinel5.png)