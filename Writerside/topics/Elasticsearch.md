# Elasticsearch
- Elasticsearch的官方网站如下：https://www.elastic.co/cn/elasticsearch/
- Elasticsearch是由elastic公司开发的一套搜索引擎技术，它是elastic技术栈中的一部分。完整的技术栈包括：
  - Elasticsearch：用于数据存储、计算和搜索
  - Logstash/Beats：用于数据收集
  - Kibana：用于数据可视化
- 整套技术栈被称为ELK，经常用来做日志收集、系统监控和状态分析等等：
- 整套技术栈的核心就是用来存储、搜索、计算的Elasticsearch，因此我们接下来学习的核心也是Elasticsearch。
## 安装elasticsearch
### Docker命令
```Shell
docker run -d \
  --name es \
  -e "ES_JAVA_OPTS=-Xms512m -Xmx512m" \
  -e "discovery.type=single-node" \
  -v es-data:/usr/share/elasticsearch/data \
  -v es-plugins:/usr/share/elasticsearch/plugins \
  --privileged \
  --network hm-net \
  -p 9200:9200 \
  -p 9300:9300 \
  elasticsearch:7.12.1
```
- 注意，这里我们采用的是elasticsearch的7.12.1版本，由于8以上版本的JavaAPI变化很大，在企业中应用并不广泛，企业中应用较多的还是8以下的版本。
- 如果拉取镜像困难，可以直接导入课前资料提供的镜像tar包：es.tar
### windows
- 下载 Elasticsearch 的 zip 安装包
- 下载地址：https://www.elastic.co/cn/downloads/elasticsearch
- 进入 bin 目录下，双击执行 elasticsearch.bat 文件。
- 执行文件后，可以在窗口中看到 Elasticsearch 的启动过程。
- 在 Elasticsearch 启动后，可以在浏览器的地址栏输入：http://localhost:9200/ 验证 Elasticsearch 启动情况
#### 新建系统变量
因为新版的 ElasticSearch 已经弃用了 JAVA_HOME 环境变量，转而使用了 ES_JAVA_HOME 环境变量，并且在新版的安装包中已经提供了 Java 运行环境，因此我们需要增加 ES_JAVA_HOME 这个环境变量，不然在后续配置中可能会出现“warning: usage of JAVA_HOME is deprecated, use ES_JAVA_HOME”的警告信息。
- 变量名：ES_JAVA_HOME
- 变量值：D:\Net_Program\Net_ElasticSearch\jdk
#### 修改 elasticsearch-env 文件
修改 \bin 下的elasticsearch-env文件（注意是没有后缀的这个文件），注释掉关于 JAVA_HOME 相关的部分，目的就是让 ElasticSearch 使用自带的 ES_JAVA_HOME
![es1.png](es1.png)
#### 修改 elasticsearch.yml 文件
编辑\config\elasticsearch.yml文件，在文件末尾增加如下配置
```yaml
#设置快照存储地址
path.repo: ["K:\\elasticsearch\\elasticsearch-7.17.4-windows-x86_64\\elasticsearch-7.17.4\\backup"]
 
#数据存放路径（可不设置，默认就是如下地址）
path.data: K:\elasticsearch\elasticsearch-7.17.4-windows-x86_64\elasticsearch-7.17.4/datas
#日志存放路径
path.logs: K:\elasticsearch\elasticsearch-7.17.4-windows-x86_64\elasticsearch-7.17.4/logs
 
#节点名称
node.name: node-1
#节点列表
discovery.seed_hosts: ["192.168.139.1"]
#初始化时master节点的选举列表
cluster.initial_master_nodes: ["node-1"]
 
#集群名称
cluster.name: es-main
#对外提供服务的端口
http.port: 9200
#内部服务端口
transport.port: 9300
 
#启动地址，如果不配置，只能本地访问
network.host: 192.168.139.1
#跨域支持
http.cors.enabled: true
#跨域访问允许的域名地址
http.cors.allow-origin: "*"
```
#### 修改 JVM 内存
如果你的服务器内存有限，则需要根据实际情况设置 ElasticSearch 的内存限制。
编辑 config 文件夹中的jvm.options文件，增加如下配置即可，此处我设置的是 4G 范围内。
特别说明：此步骤需要注意，建议最好根据服务器的内存情况进行设置，以免后期再来调整。同时此步骤需要在将 ElasticSearch 安装为服务前进行设置，否则安装服务后，即便是重启服务也不会生效。
```yaml
#设置最小内存
-Xms4g
#设置最大内存
-Xmx4g
```
- 在设置-Xms和-Xmx属性的时候，一定要设置为相同的值，否则在启动服务的时候出现如下的错误，倒是启动 ElasticSearch 服务失败。比如我们都可以设置为 4g，-Xms4g 和 -Xmx4g

#### 安装 Elasticsearch 服务 {id="elasticsearch_1"}
elasticsearch-service.bat install
- 安装命令执行完成后，到服务中就可以看到安装好的 Elasticsearch 服务
- 卸载服务的命令: elasticsearch-service.bat remove
- 其他操作命名： 
  - elasticsearch-service.bat install：安装Elasticsearch服务。 
  - elasticsearch-service.bat remove：删除已安装的Elasticsearch服务（如果启动则停止服务）。
  - elasticsearch-service.bat start：启动Elasticsearch服务（如果已安装）。
  - elasticsearch-service.bat stop：停止服务（如果启动）。
  - elasticsearch-service.bat manager：启动GUI来管理已安装的服务。
#### 配置 SSL 证书
- 启动刚才安装的Elasticsearch 7.17.4 (elasticsearch-service-x64)服务
- 在安装目录下新建certs文件夹，用于存放生成的 CA 证书
- CMD 定位到 bin 目录，输入如下命令 elasticsearch-certutil ca
- 接着输入 ca 证书输出地址和密码（如果设置了密码，请记住，下面会用到）
  - K:\elasticsearch\elasticsearch-7.17.4-windows-x86_64\elasticsearch-7.17.4\certs\elastic-stack-ca.p12
  - yu123456
- 输入如下命令
  - elasticsearch-certutil cert --ca K:\elasticsearch\elasticsearch-7.17.4-windows-x86_64\elasticsearch-7.17.4\certs\elastic-stack-ca.p12
  - 此集群证书的输出地址:K:\elasticsearch\elasticsearch-7.17.4-windows-x86_64\elasticsearch-7.17.4\certs\elastic-stack-ca.p12
- elasticsearch-keystore add xpack.security.transport.ssl.keystore.secure_password
  - 接着输入密码 123456（该密码为上面生成证书设置的密码）
- elasticsearch-keystore add xpack.security.transport.ssl.truststore.secure_password
- 此时，SSL 证书生成完成，我们将 certs 文件夹拷贝到 config 下
- 在 elasticsearch.yml 文件中增加如下配置
```
#开启xpack
xpack.security.enabled: true
xpack.license.self_generated.type: basic
xpack.security.transport.ssl.enabled: true
#证书配置
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.client_authentication: required
xpack.security.transport.ssl.keystore.path: certs/elastic-stack-ca.p12
xpack.security.transport.ssl.truststore.path: certs/elastic-stack-ca.p12
```
#### 设置账户密码
- 重启服务，以管理员身份运行 CMD 并定位到 ElasticSearch 的 bin 目录，执行如下命令，然后紧接着输入 y 确定，然后输入每个账户的密码和确认密码即可
- elasticsearch-setup-passwords interactive
  此时我们在浏览器中访问 http://192.168.139.1:9200/ 发现要求输入账户和密码，这是我们输入 elastic（账户）和 yu123456（密码，刚才设置的密码）即可访问成功。
## 安装 Elasticsearch 可视化工具 —— Kibana
### Docker {id="docker_1"}
```Shell
docker run -d \
--name kibana \
-e ELASTICSEARCH_HOSTS=http://es:9200 \
--network=hm-net \
-p 5601:5601  \
kibana:7.12.1
```
如果拉取镜像困难，可以直接导入课前资料提供的镜像tar包
- 安装完成后，直接访问5601端口，即可看到控制台页面
- 选择Explore on my own之后，进入主页面
- 然后选中Dev tools，进入开发工具页面
### Windows {id="windows_1"}
将下载下来的kibana-7.17.4-windows-x86_64.zip解压到K:\elasticsearch\kibana-7.17.4-windows-x86_64文件夹下
#### 修改文件
编辑 \config\kibana.yml 文件，在文件末尾增加如下配置：
##### 单机模式
```yaml
#设置中文显示
i18n.locale: "zh-CN"
 
#设置访问用户
elasticsearch.username: "elastic"
#设置访问密码
elasticsearch.password: "yu123456"
 
#ElasticSearch连接地址
elasticsearch.hosts: ["http://192.168.139.1:9200"]
 
#IP访问地址和端口号
server.host: "192.168.139.1"
server.port: 5601

```
##### 集群模式
```yaml

#设置中文显示
i18n.locale: "zh-CN"
 
#设置访问用户
elasticsearch.username: "elastic"
#设置访问密码
elasticsearch.password: "yu123456"
 
#ElasticSearch连接地址
elasticsearch.hosts: ["http://192.168.139.1:9200","http://192.168.139.1:9201","http://192.168.139.1:9202"]
 
#IP访问地址和端口号
server.host: "192.168.139.1"
server.port: 5601

```
#### 安装 Kibana 服务
- 为了避免访问 Kibana 出现server.publicBaseUrl 缺失,在生产环境中运行时应配置。某些功能可能运行不正常。的提示 请在上述配置文件中增加如下的配置
- `server.publicBaseUrl: "http://192.168.139.1:5601/"`
- 进入 bin 目录下，双击执行 kibana.bat 文件

## 倒排索引
elasticsearch采用倒排索引
- 文档（document）：每条数据就是一个文档
- 词条（term）：文档按照语义分成的词语
## IK分词器
- 中文分词往往需要根据语义分析，比较复杂， 这就需要用到中文分词
- 安装只需要将文件夹放入插件目录中即可 https://github.com/infinilabs/analysis-ik
### 测试
- 初始的分词器
```
POST /_analyze
{
  "analyzer": "standard",
  "text": ["黑马程序员学习Java太棒了"]
}
```
- ik
```
POST /_analyze
{
  "analyzer": "ik_smart", # ik_max_word 最细切分，细粒度IK分词器  ik_smart 智能切分，粗粒度
  "text": ["黑马程序员学习Java太棒了"]
}
```
### 配置扩展
- IK分词器允许配置扩展词典来增加自定义的词库
![ik1.png](ik1.png)
![ik2.png](ik2.png)
## 基础概念
- 索引（index）：相同类型的文档的集合
- 映射（mapping）：索引中文档的自段约束信息，类似于表的结构约束
![es2.png](es2.png)
## 索引库操作 
### Mapping映射属性
- mapping是对索引库中文档的约束，常见的mapping属性包括
  - type：字段数据类型，常见的简单类型
    - 字符串：text（可分词的文本）、keyword（精确值，例如：品牌、国家）
    - 数值：long、integer、short、byte、double、float
    - 布尔：boolean
    - 日期：date
    - 对象：object
  - index：是否创建索引，默认为true
  - analyzer：使用哪种分词器
  - properties：该字段的子字段
### 操作
- 提供的所有API都是Restful的接口，遵循Restful的基本规范
![es3.png](es3.png)