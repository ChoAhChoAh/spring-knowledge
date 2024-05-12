# SpringBoot缓存
## 缓存-ehcache
### 1.spring-boot集成
#### 1.1 引入依赖
springboot默认集成了ehcache的依赖，开启依赖引入即可：
```xml
<dependency>
    <groupId>net.sf.ehcache</groupId>
    <artifactId>ehcache</artifactId>
</dependency>
```

#### 1.2 增加ehcache配置文件
使用ehcache需要配置其配置文件ehcache.xml（默认名称），可以参考如下配置：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="http://ehcache.org/ehcache.xsd">

    <!-- 磁盘缓存位置 -->
    <diskStore path="C:\Users\liecho\IdeaProjects\taskprocessor\cachedata"/>

    <!-- 默认缓存 -->
    <defaultCache
            eternal="false"
            diskPersistent="false"
            maxElementsInMemory="1000"
            overflowToDisk="false"
            timeToIdleSeconds="60"
            timeToLiveSeconds="60"
            memoryStoreEvictionPolicy="LRU"/>

    <cache
            name="dataCache"
            eternal="false"
            diskPersistent="false"
            maxElementsInMemory="1000"
            overflowToDisk="false"
            timeToIdleSeconds="60"
            timeToLiveSeconds="60"
            memoryStoreEvictionPolicy="LRU"/>
</ehcache>
```
配置文件中指定了缓存数据的磁盘位置，各个缓存配置（包括名称、是否磁盘存储、存活时间、过期策略等）。
具体的配置可以参考官方文档：https://www.ehcache.org/generated/2.10.4/html/ehc-all/#page/Ehcache_Documentation_Set%2Fco-cfgbasics_xml_configuration.html%23wwconnect_header
也可以参考官方定义文件：http://ehcache.org/ehcache.xsd
#### 1.3 配置并使用
在springboot启动类增加@EnableCaching：
```java
@SpringBootApplication
@EnableCaching
public class XXXXApplication {

    public static void main(String[] args) {
        XXXXApplication.run(XXXXApplication.class, args);
    }
}
```

在application.yml文件中增加springboot的缓存配置，并指定使用ehcache和配置文件:
```yaml
spring:
  cache:
    type: ehcache
    ehcache:
      config: ehcache.xml
```
使用springboot的@Cacheable注解即可使用ehcache,请注意@Cacheable使用的存储名要和ehcache.xml中cache配置的name一致：
```java
@Service
public class DataService {

    @Autowired
    private DataDao dataDao;

    @Cacheable(value = "dataCache",key="#id")
    public Data getDataById(String id){
        Data data = dataDao.selectById(id);
        return data;
    }
}
```

### 2.常见问题
暂无

### 3.高级配置和运维（todo）


### 4.参考
官方文档：https://www.ehcache.org/documentation/3.10
springboot官方文档：https://docs.spring.io/spring-boot/docs/2.7.18/reference/pdf/spring-boot-reference.pdf#page=406&zoom=100,0,650
配置springboot+ehcache：https://www.cnblogs.com/fishpro/p/spring-boot-study-ehcache.html
springboot的缓存注解：https://blog.csdn.net/dreamhai/article/details/80642010