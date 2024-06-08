# SpringCloud

```Javascript
docker network create hm-net
docker run -d \
--name mysql \
-p 3306:3306 \
-e TZ=Asia\Shanghai \
-e MYSQL_ROOT_PASSWORD=123 \
-v /root/mysql/data:/var/lib/mysql \
-v /root/mysql/conf:/etc/mysql/conf.d \
-v /root/mysql/init:/docker-entrypoint-initdb.d \
--network hm-net \
mysql
```
## 单体架构
- 将业务的所有功能集中在一个项目中开发，打成一个包部署
  - 优点
    - 架构简单
    - 部署成本低
  - 缺点
    - 团队协作成本高
    - 系统发布效率低
    - 系统可用性差
## 微服务
- 微服务架构，是服务化思想指导下的一套最佳实践架构方案。服务化，就是把单体架构中的功能模块拆分为多个独立项目
- 粒度小
- 团队自治
- 服务自治
## 服务拆分原则
- 高内聚：每个微服务的职责要尽量单一，包含的业务互相关联度高，完整度高
- 低耦合：每个微服务的功能要相对独立，尽量减少对其他微服务的依赖
## 拆分方式
- 纵向拆分：按照业务模块来划分
- 横向拆分：抽取公共服务，提高复用性
## 拆分遇到的问题
![error1.png](error1.png)
需要调用其他模块的service
### 传统方法
- 使用RestTemplate
![restTemplate.png](restTemplate.png)
```Java
//在没有用feign之前的方法
ResponseEntity<List<ItemDTO>> response = restTemplate.exchange(
        "http://localhost:8081/item?ids={ids}",
        HttpMethod.GET,
        null,
        new ParameterizedTypeReference<>() {
        },
        Map.of("ids", StrUtil.join(",", itemIds))
);
if(!response.getStatusCode().is2xxSuccessful()){
    //查询失败
    return;
}
List<ItemDTO> items = response.getBody();
```
#### 服务远程调用存在的问题
- 地址与端口号写死
- 服务治理问题
- <下一章：注册中心>