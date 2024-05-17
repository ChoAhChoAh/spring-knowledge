# SpringBoot数据源
## 非关系数据库-elasticsearch
### 1.单体安装（基于ubuntu20.04）
#### 1.1 下载程序包
可以在elastic的官网下载程序包，本文安装的版本为：8.13.3。
https://www.elastic.co/cn/downloads/elasticsearch
下载前建议明确系统和架构，本文下载的linux（x86）包，没有下载debain ubuntu的deb程序包。
elasticsearch-8.13.3-linux-x86_64.tar.gz
也可以通过wget命令直接下载到服务器上：
```
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.13.3-linux-x86_64.tar.gz
```

#### 1.2 解压程序包移动到安装目录
tar.gz程序包可以使用tar命令解压，并建议通过mv命令移动到统一的软件路径，可以参考如下命令：
```
tar -zxf elasticsearch-8.13.3-linux-x86_64.tar.gz
mv elasticsearch-8.13.3 /usr/local/elasticsearch8.13.3
```

#### 1.3 修改配置文件
##### 1.3.1 jvm文件配置
进入config目录，使用vim命令修改jvm.options文件：
```
cd /usr/local/elasticsearch8.13.3/config
vim jvm.options
```
启动的基础配置主要修改内存分配参数，elasticsearch建议分配4g内存,修改后保存：
```shell script
################################################################
## IMPORTANT: JVM heap size
################################################################
##
## The heap size is automatically configured by Elasticsearch
## based on the available memory in your system and the roles
## each node is configured to fulfill. If specifying heap is
## required, it should be done through a file in jvm.options.d,
## which should be named with .options suffix, and the min and
## max should be set to the same value. For example, to set the
## heap to 4 GB, create a new file in the jvm.options.d
## directory containing these lines:
##
-Xms4g
-Xmx4g
##
## See https://www.elastic.co/guide/en/elasticsearch/reference/8.13/heap-size.html
## for more information
##
################################################################
```
##### 1.3.2 修改elasticsearch配置文件
修改jvm.options同目录下的elasticsearch配置文件（elasticsearch.yml）:
```
cd /usr/local/elasticsearch8.13.3/config
vim elasticsearch.yml
```
基础的配置主要关注如下配置，具体的配置信息可以参考官方文档：https://www.elastic.co/guide/en/elasticsearch/reference/index.html
```shell script
# 节点的描述名称:
node.name: node-base
# 数据存储的目录路径（用逗号分隔多个位置）:
path.data: /usr/local/elasticsearch8.13.3/data
# 日志文件路径:
path.logs: /usr/local/elasticsearch8.13.3/logs
# 节点对外暴露的ip地址:
network.host: 10.10.10.10
# 默认情况下，Elasticsearch 在找到的第一个空闲端口上监听 HTTP 流量，
# 从 9200 端口开始。在此处设置一个特定的 HTTP 端口：
http.port: 9200
# 使用初始的一组主节点来引导集群的启动:
cluster.initial_master_nodes: ["node-base"]
```
修改后保存。
注意:elasticsearch还有一个节点的通讯服务端口9300

#### 1.4 安装分词插插件
可以在github上下载分词插件：https://github.com/infinilabs/analysis-ik/releases
下载后放到elasticsearch程序的plugins目录下：
```
cd /usr/local/elasticsearch8.13.3/plugins/
unzip -d ik elasticsearch-analysis-ik-8.13.3.zip
```
解压后建议删除elasticsearch-analysis-ik-8.13.3.zip文件，否则启动时elasticsearch会误将压缩文件识别成插件导致无法启动。

#### 1.5 创建运行用户和数据目录
elasticsearch不允许使用root用户启动，因此需要创建运行用户，并切换到改用户后执行运行命令：
```
useradd elastic
passwd elastic
```
创建数据目录：
```
cd /usr/local/elasticsearch8.13.3/
mkdir data
chown root data
```
给elastic用户赋予elasticsearch程序目录的权限：
```
chown -R elastic:elastic /usr/local/elasticsearch8.13.3/
chmod +rwx -R /usr/local/elasticsearch8.13.3/
```

#### 1.6 启动elasticsearch
进入bin目录，启动程序
```
cd /usr/local/elasticsearch8.13.3/bin
./elasticsearch
```
注意：
1. 如果服务器有配置防火墙，请开启端口访问策略。
2. elasticsearch默认用户随机生成的密码需要去日志文件中查找

### 2.spring-boot集成
#### 2.1 集成java版本客户端
在新的elasticsearch版本中，已经不推荐使用spring-boot-starter-data-elasticsearch中自带的客户端以及三方的高版本客户端，推荐使用官方最新客户端。
elasticsearch官网更新了java客户端链接程序，可以引入如下依赖(https://www.elastic.co/guide/en/elasticsearch/client/java-api-client/current/getting-started-java.html):
```xml
<project>
  <dependencies>
    ... ...
    <dependency>
      <groupId>co.elastic.clients</groupId>
      <artifactId>elasticsearch-java</artifactId>
      <version>8.13.4</version>
    </dependency>

    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-databind</artifactId>
      <version>2.17.0</version>
    </dependency>
    ... ...
  </dependencies>
</project>
```
jackson-databind可以不使用2.17版本，实测2.13版本也可以使用，但建议跟随官方版本使用。

#### 2.2 使用客户端操作elasticsearch
```java
@SpringBootTest
@ContextConfiguration(classes = {XXXApplication.class} )
public class XXXApplicationTest {

    private RestClient restClient = null;
    // Create the transport with a Jackson mapper
    private ElasticsearchTransport transport = null;
    // And create the API client
    private ElasticsearchClient esClient = null;

    @BeforeEach
    public void setUp(){
        String serverUrl = "http://10.10.10.10:9200";
        RestClient restClient = RestClient
                .builder(HttpHost.create(serverUrl))
                .build();
        transport = new RestClientTransport(
                restClient, new JacksonJsonpMapper());
        esClient = new ElasticsearchClient(transport);
    }

    @Test
    public void get_documents() throws IOException {
        GetResponse<Config> response = esClient.get(g -> g
                        .index("config")
                        .id("2"),
                Config.class
        );
        if (response.found()) {
            Config config = response.source();
            System.out.println(config);
        }
    }

    @AfterEach
    public void clear() throws IOException {
        if(esClient!=null){
            esClient.shutdown();
        }
        if(restClient!=null){
            restClient.close();
        }
    }

}
```

### 3.常见问题
#### 3.1 报错max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
原因elasticsearch用户拥有的内存权限太小，至少需要262144
在 /etc/sysctl.conf文件最后添加一行
```
vm.max_map_count=262144
```
添加保存后执行命令：
```
sysctl -p
```

#### 3.2 访问管理页面报错
出现类似报错：received plaintext http traffic on an https channel, closing connection Netty4HttpChannel{localAddress=/[0:0:0:0:0:0:0:1]:9200
elasticsearch默认开启了认证，在安装配置阶段可以先关闭授权(将true改为false)：
```
xpack.security.enabled: false
```

#### 3.3 使用客户端程序链接报错
出现类似报错信息：NoSuchMethodError RequestOptions$Builder.removeHeader when creating a client
说明elasticsearch-rest-client的版本过低，参考官网说明更新版本：
https://www.elastic.co/guide/en/elasticsearch/client/java-api-client/7.17/no-such-method-request-options.html

### 4.高级配置和运维（todo）
#### 4.1 安装kibana
可以到官网下载kibana：https://www.elastic.co/cn/downloads/kibana

### 5.参考
访问elasticsearch服务报错：https://blog.csdn.net/qq_25086397/article/details/124749039  
客户端链接报错：https://www.cnblogs.com/rong-xu-drum/p/18133925  
其他参考：https://blog.csdn.net/qq_45036013/article/details/129448347  
安装配置：https://blog.csdn.net/cool_sec/article/details/124532757  
安装配置：https://blog.csdn.net/2401_83946224/article/details/137654056  
官网客户端链接报错参考：https://www.elastic.co/guide/en/elasticsearch/client/java-api-client/7.17/no-such-method-request-options.html  


### 6.附录
#### 6.1 elasticsearch基本操作（基于官方api）
##### 6.1.1 创建（修改）索引
使用PUT请求http://10.10.10.10:9200/config
参数如下,properties中每个键为数据键,type指定数据值类型，analyzer指定分词类型，copy_to可以配置全文检索：
```json
{
    "mappings":{
        "properties":{
            "id":{
                "type":"keyword"
            },
            "name":{
                "type":"text",
                "analyzer":"ik_max_word",
                "copy_to":"all"
            },
            "type":{
                "type":"keyword"
            },
            "description":{
                "type":"text",
                "analyzer":"ik_max_word",
                "copy_to":"all"
            },
            "all":{
                "type":"text",
                "analyzer":"ik_max_word"
            }
        }
    }
}
```
##### 6.1.2 获取索引
使用GET请求http://10.10.10.10:9200/config

##### 6.1.3 删除索引
使用delete请求http://10.10.10.10:9200/config
返回值：
```json
{
    "acknowledged": true
}
```

##### 6.1.4 创建文档（请求体中带id）
使用POST请求 http://10.10.10.10:9200/config/_doc
注意此种方式请求提中的id尽量使用数字型，否则会被识别为普通检索字段，会自动生成随机id
请求体：
```json
{
    "id":1,
    "name":"dataconfig",
    "type":"dataconfig",
    "description":"dataconfig"
}
```
响应体：
```json
{
    "_index": "config",
    "_id": "1",
    "_version": 1,
    "result": "created",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "_seq_no": 0,
    "_primary_term": 1
}
```

##### 6.1.5 创建文档（请求path中带id）
使用POST请求 http://10.10.10.10:9200/config/_create/3
请求体：
```json
{
    "name":"dataconfig book",
    "type":"dataconfig success",
    "description":"dataconfig fail"
}
```
响应体：
```json
{
    "_index": "config",
    "_id": "3",
    "_version": 1,
    "result": "created",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "_seq_no": 2,
    "_primary_term": 1
}
```
##### 6.1.6 获取文档
使用GET请求 http://10.10.10.10:9200/config/_doc/3
响应体：
```json
{
    "_index": "config",
    "_id": "3",
    "_version": 1,
    "_seq_no": 2,
    "_primary_term": 1,
    "found": true,
    "_source": {
        "name": "dataconfig book",
        "type": "dataconfig success",
        "description": "dataconfig fail"
    }
}
```
##### 6.1.7 查询文档（全量）
使用GET请求 http://10.10.10.10:9200/config/_search

##### 6.1.8 查询文档（关键字匹配）
使用GET请求 http://10.10.10.10:9200/config/_search?q=name:book
?q表示查询条件
响应体：
```json
{
    "took": 9,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 1,
            "relation": "eq"
        },
        "max_score": 0.60996956,
        "hits": [
            {
                "_index": "config",
                "_id": "3",
                "_score": 0.60996956,
                "_source": {
                    "name": "dataconfig book",
                    "type": "dataconfig success",
                    "description": "dataconfig fail"
                }
            }
        ]
    }
}
```

##### 6.1.9 删除文档
使用DELETE请求 http://10.10.10.10:9200/config/_doc/1
响应体：
```json
{
    "_index": "config",
    "_id": "1",
    "_version": 2,
    "result": "deleted",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "_seq_no": 3,
    "_primary_term": 1
}
```

##### 6.1.10 修改文档（全量）
使用PUT请求 http://10.10.10.10:9200/config/_doc/2
请求体：
```json
{
    "name":"dataconfig spring",
    "type":"dataconfig elastic",
    "description":"dataconfig elastic"
}
```
响应体：
```json
{
    "_index": "config",
    "_id": "2",
    "_version": 4,
    "result": "updated",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "_seq_no": 6,
    "_primary_term": 1
}
```

##### 6.1.11 修改文档（增量）
使用PUT请求 http://10.10.10.10:9200/config/_doc/2
请求体：
```json
{
    "doc":{
        "name":"spring boot"
    }
}
```
响应体：
```json
{
    "_index": "config",
    "_id": "2",
    "_version": 5,
    "result": "updated",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "_seq_no": 7,
    "_primary_term": 1
}
```