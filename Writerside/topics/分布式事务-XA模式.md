# 分布式事务 - XA模式
- XA 规范 是 X/Open 组织定义的分布式事务处理（ DTP ， Distributed Transaction Processing ）标准， XA 规
范 描述了全局的 TM 与局部的 RM 之间的接口，几乎所有主流的关系型数据库都对 XA 规范 提供了支持。
![xa1.png](xa1.png)
## 优点
- 事物的强一致性，满足ACID原则
- 常用数据库都支持，实现简单，并且没有代码侵入
## 缺点
- 因为一阶段需要锁定数据库资源，等待二阶段结束才释放，性能较差
- 依赖关系型数据库实现事务
## 使用
- seata的starter已经完成了XA模式的自动装配，实现非常简单
### 修改yaml文件 开启XA模式
```yaml
seata:
  data-source-proxy-mode: XA
```
### 给入口方法添加注解
`@GlobalTransactional`
- 同时也需要其他被调用方法添加注解`@Transactional`