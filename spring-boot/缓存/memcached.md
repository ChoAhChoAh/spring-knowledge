# SpringBoot缓存
## 缓存-memcached
### 1.单体安装（基于ubuntu20.04）
#### 1.1 在线安装
如果通过linux系统在线安装可参考官方文档：https://github.com/memcached/memcached/wiki/Install  
文档中描述Debian和redhat两类系统的在线安装命令。
需要注意，memcached的服务端依赖libevent-dev库，安装时需要先将依赖库安装。
#### 1.2 源码安装
源码安装需要先下载程序包，可以在此网站先下载后上传服务器：http://memcached.org/  
也可以在服务器使用wget命令下载到指定位置：
```
wget https://memcached.org/latest
```
下载后，解压文件,配置安装路径，编译安装：
```
tar -zxf memcached-1.6.27.tar.gz
cd memcached-1.6.27/
./configure --prefix=/usr/local/memcache
make
make install
```
如果编译过程中给或者安装出现依赖libevent-dev缺失，可以通过ubuntu在线安装依赖：
```
sudo apt update
sudo apt install libevent-dev
```
也可以去libevent官网下载源码包编译运行（不推荐）,libevent官网https://libevent.org/：
```
tar -zxvf libevent-2.1.12-stable.tar.gz
cd libevent-2.1.12-stable
./configure --prefix=/usr/local/libevent
make
make install
```
#### 1.3 启动
若安装无报错，可以进入bin目录运行memcached,运行时可以指定用户、端口、ip等参数:
```
cd /usr/local/memcache/bin
./memcached -d -m 100 -u root -l 10.0.0.204 -p 11211 -c 512 -P /tmp/memcached.pid
```
运行后可以用ps -ef|grep "memcached"查看进程是否运行。

安装后如果服务器有防火墙，请配置防火墙策略配置端口访问。

#### 1.4 插件和客户端安装
如果需要按章插件和客户端，可以参考官方文档（https://github.com/memcached/memcached/wiki/Install）或者此博客：https://www.cnblogs.com/chy123/p/8616496.html

### 2.springboot集成
#### 2.1 引入依赖
java使用memcached需要引入依赖，本文以xmemcached为例：
```xml
<dependency>
    <groupId>com.googlecode.xmemcached</groupId>
    <artifactId>xmemcached</artifactId>
    <version>2.4.8</version>
</dependency>
```
#### 2.2 配置
在application.yaml中增加memcached的配置,注意ip和端口
```yaml
memcached:
  servers: 10.10.10.10:11211
  poolSize: 10
  opTimeout: 3000
```

#### 2.3 实现
由于springboot2.x版本默认没有集成memcached, 需要编码实现memcached的缓存操作工具：
```java
@Component
@Data
@ConfigurationProperties(prefix = "memcached")
public class XMemcachedProperties {

    private String servers;

    private int poolSize;

    private long opTimeout;
}

@Configuration
public class MemcacheConfig {

    @Autowired
    private XMemcachedProperties properties;
    @Bean
    public MemcachedClient getMemcachedClient() throws IOException {
        MemcachedClientBuilder memcachedClientBuilder = new XMemcachedClientBuilder(properties.getServers());
        memcachedClientBuilder.setConnectionPoolSize(properties.getPoolSize());
        memcachedClientBuilder.setOpTimeout(properties.getOpTimeout());
        MemcachedClient memcachedClient = memcachedClientBuilder.build();
        return memcachedClient;
    }

}
```

使用memcached:
```java
@SpringBootTest
public class TestMemcache {

    @Autowired
    private MemcachedClient memcachedClient;

    @Test
    public void test_memcache_set() throws InterruptedException, MemcachedException, TimeoutException {
        String key="key";
        String value ="value";
        memcachedClient.set(key,0,value);
    }

    @Test
    public void test_memcache_get() throws InterruptedException, MemcachedException, TimeoutException {
        String key="key";
        Object o = memcachedClient.get(key);
        System.out.println(o.toString());
    }

}
```

### 3.常见问题
暂无

### 4.高级配置和运维（todo）

### 5.参考
官方文档：https://github.com/memcached/memcached/wiki/Install  
扩展程序官网：https://pecl.php.net/package/memcache  
安装参考：https://www.cnblogs.com/chy123/p/8616496.html  
xmemcached java客户端：https://mvnrepository.com/artifact/com.googlecode.xmemcached/xmemcached/2.4.8  