# SpringBoot
## spring的bean注入方式
从spring到spring-boot，bean的加载方式发生了不小的变化，本文通过对bean注入方式的变化对spring及springboot的实现进行理解和整理
### 1.配置文件注入bean
早期spring时代（ssm时期），spring只提供容器和相关的逻辑工具，对于其他技术的集成没有进行太多的关注，因此配置相对零散且繁琐。  
由于当时配置文件盛行，大部分场景会使用配置文件方式进行bean的注入。例如：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean id="data" class="org.xxx.bean.DataProcess" >
        <property name="id" value="1111"/>
    </bean>
    <bean class="org.xxx.bean.DataProcess"/>
    <bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource"/>
</beans>
```
可以通过bean标签将bean加入spring容器中,此种方式不会对代码本身存在侵入，因此可以通过该种方式加载自定义和三方技术的实力作为spring容器中的bean。  
但由于单个定义bean可能需要大批量的编写bean标签，不利于开发和维护，因此spring同时支持了context命名空间，允许通过扫描的方式自动注入。  
举例如下，开启context命名空间，配置扫描基包路径：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="
       http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       http://www.springframework.org/schema/context/spring-context.xsd">
    
    <context:component-scan base-package="org.xxx.bean,org.xxx.config"/>
</beans>
```
但注意，通过context配置包扫描路径的方式，需要给路径下的类，配置@Component及其派生注解，否则spring无法识别。  
此种方式可以减轻bean的配置，但是引入可一个新的问题，即如果配置了包扫描，实际上配置文件的适用显得没有必要，因此spring又支持了注解的方式来实现bean的加载，可以不在编写xml的容器bean配置文件。  
此外，使用context配置扫描路径+@Component开始对代码有了一定程度的入侵。

### 2.注解式注入bean
#### 2.1 @ComponentScan+配置类
在1中，通过ClassPathXmlApplicationContext上下文容器读取xml文件进行配置文件加载的方式由于存在维护和开发的便利性问题，因此spring又支持了通过注解的方式注入bean。  
对应的上下文容器为AnnotationConfigApplicationContext，该容器通过读取配置类上的注解，解析出需要注入的bean信息：
```java
@Configuration
@ComponentScans({
        @ComponentScan(basePackages = "org.xxx.bean"),
        @ComponentScan(basePackages = "org.xxx.service")
        })
//@ComponentScan("org.xxx.bean")
public class AppScan {
}
```
上述实例代码中，可以通过@ComponentScan或者@ComponentScans+@Configuration对需要加载到spring容器中的路径进行解析。  
注意，该种方式下，配置扫描路径下需要加载的类，仍需要增加@Component及其派生注解，并且对类的侵入依然存在。  

#### 2.2 @ImportResrouce读取已有的配置类
对于已有的容器xml配置，一般不会手动修改对应类逐一增加@Component注解，spring提供了@ImportResource读取已经存在的xml配置：
```java
/*
* 容器启动类
*/
public class ApplicationImportResource {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(App5ImportResource.class);
    }

}
/*
* 配置类使用@ImportResource注解读取已有配置文件进行加载（此时容器上下文类依旧为AnnotationConfigApplicationContext）
*/
@ImportResource({"applicationContext1copy.xml","applicationContext1.xml"})
public class App5ImportResource {
}
```
注意，此种使用方式中，xml配置文件的加载顺序会影响bean的生成，如果applicationContext1copy.xml和applicationContext1.xml存在同名的bean，书写在后侧的xml配置会覆盖前侧的xml配置。  

#### 2.3 @Import注解配置注入类
使用@Import导入需要注入的bean：
```java
/*
* 配置类
*/
@Import({ExecuteProcess.class,DataSourceConfig2.class})
public class App7ImportConfig {
}

/*
* 配置类导入的数据源配置类
*/
public class DataSourceConfig2 {

    @Bean
    public DataSource dataSource(){
        DataSource dataSource = new DruidDataSource();
        return dataSource;
    }
}

/*
* 容器启动类
*/
public class Application7Import {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(App7ImportConfig.class);
        String[] beanDefinitionNames = ac.getBeanDefinitionNames();
        for (String beanDefinitionName : beanDefinitionNames) {
            System.out.println(beanDefinitionName);
        }
    }
}
```
启动类运行后打印如下,包括配置类本身、@Import中设置的类、@Import设置类中使用了@Bean等spring可解析的注解的类，均被注入到容器中：
```
app7ImportConfig
org.xxx.bean.ExecuteProcess
org.xxx.config.DataSourceConfig2
dataSource
```
此种方式对代码的入侵小，spring内部也较多使用此种方式，被注入的类无需增加其他spring的注解，但注意，此种方式需要有默认的空参构造器。

### 3.干预bean注入
#### 3.1 工厂bean
注解的方式虽然省去了xml配置文件，但因为需要侵入式开发，对于三方技术的实例加载，无法直接通过增加@Component及其派生注解的方式实现。  
spring提供了一种工厂bean的方式，可以在非侵入的情况下对bean进行加载：
```java
/*
* 容器启动类
*/
public class ApplicationFactoryBean {
    public static void main(String[] args) {
        ApplicationContext ac = new AnnotationConfigApplicationContext(App4Config.class);
    }
}
/*
* 配置类
*/
public class App4Config {
    @Bean
    public DataProcessFactoryBean dataProcess(){
        return new DataProcessFactoryBean();
    }
}

/*
* 工厂bean
*/
public class DataProcessFactoryBean implements FactoryBean<DataProcess> {
    @Override
    public DataProcess getObject() throws Exception {
        DataProcess dataProcess = new DataProcess();
        // do something...
        return dataProcess;
    }

    @Override
    public Class<?> getObjectType() {
        return DataProcess.class;
    }

    @Override
    public boolean isSingleton() {
        return false;
    }
}
```
注意，上述代码没有配置包扫描的相关注解，是因为容器类直接加载了配置类，配置类内部的方法因为被@Bean修饰也会被读取识别并进行加载，在实际开发中，工厂bean类可能分散在很多包，仍旧需要配置包扫描。  
此种方式在一定程度上避免了代码的侵入，因为无需对需要加载的类进行@Component修饰，且生成实例的操作可以通过编码的方式进行干预，提高灵活度。

#### 3.2 @Configuration注解的proxyBeanMethods属性
@Configuration注解中的属性proxyBeanMethods需要注意，该属性默认值为true，true的情况下，spring会使用cglib对期望加载的bean进行代理增强。  
在配合@Bean注解使用时：
1. 如果proxyBeanMethods=true,则每次调用@Configuration修饰的配置类，其创建bean的方法时（下实例代码的jobProcess方法），都返回容器内创建好的bean,类似单例作用域@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
2. 如果proxyBeanMethods=false，则每次会创建新的对象，类似原型模式作用域@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
注意：如果proxyBeanMethods = true，在创建Bean的方法上，使用作用域可以控制使用单例还是原型，但是如果proxyBeanMethods = false,则只会是原型模式
```java
/*
 * 配置类
 */
//@Configuration(proxyBeanMethods = true)
@Configuration(proxyBeanMethods = false)
public class App6ProxyMethodBean {

    @Bean
    //@Scope(ConfigurableBeanFactory.SCOPE_SINGLETON)
    public JobProcess jobProcess(){
        return new JobProcess();
    }
}
```
启动代码如下：
```java
/*
* 容器启动类
*/
public class ApplicationProxyBeanMethod {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(App6ProxyMethodBean.class);
        App6ProxyMethodBean app6ProxyMethodBean = ac.getBean("app6ProxyMethodBean",App6ProxyMethodBean.class);
        System.out.println(app6ProxyMethodBean.getClass());

        System.out.println(app6ProxyMethodBean.jobProcess());
        System.out.println(app6ProxyMethodBean.jobProcess());
        System.out.println(app6ProxyMethodBean.jobProcess());
    }
}
```
使用上述代码每次调用app6ProxyMethodBean的jobProcess创建DataProcess时，在不使用@Scope注解的情况下分别打印
```
proxyBeanMethods=false的打印
class org.xxx.config.App6ProxyMethodBean
org.xxx.bean.JobProcess@6121c9d6
org.xxx.bean.JobProcess@87f383f
org.xxx.bean.JobProcess@4eb7f003

proxyBeanMethods=true的打印
class org.xxx.config.App6ProxyMethodBean$$EnhancerBySpringCGLIB$$4914d540
org.xxx.bean.JobProcess@3b2c72c2
org.xxx.bean.JobProcess@3b2c72c2
org.xxx.bean.JobProcess@3b2c72c2
```
一般情况下，无需将proxyBeanMethods设置为false，因为如果设置了，在配置注入mq连接或者数据源时，每次都会创建新的对象，方法运行可能出现问题,  
如下代码，rabbitmq在绑定topic时，会出现每次绑定一个新的topicExhange，这在一般的开发规范中是不允许的
```java
@Configuration(proxyBeanMethods = false)
public class RabbitConfigTopic {

    @Bean
    public Queue topicQueue(){
        return new Queue("xxx_topic");
    }
    @Bean
    public Queue topicQueue2(){
        return new Queue("xxx_topic_2");
    }
    @Bean
    public TopicExchange topicExhange(){
        return new TopicExchange("xxxTopicExchange");
    }
    @Bean
    public Binding bindTopic(){
        return BindingBuilder.bind(topicQueue()).to(topicExhange()).with("topic.*.data");
    }
    @Bean
    public Binding bindTopic2(){
        return BindingBuilder.bind(topicQueue2()).to(topicExhange()).with("topic.#");
    }
}
```

#### 3.3 使用容器的registerBean方法
可以通过获取容器实例后，通过调用其registerBean方法进行bean的注册。
```java
/*
* 容器启动类
*/
public class ApplicationRegister {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(App6ProxyMethodBean.class);
        ac.registerBean("execute", ExecuteProcess.class,1);
        ac.registerBean("execute", ExecuteProcess.class,2);
        ac.registerBean("execute", ExecuteProcess.class,3);
        String[] beanDefinitionNames = ac.getBeanDefinitionNames();
        for (String beanDefinitionName : beanDefinitionNames) {
            System.out.println(beanDefinitionName);
        }
        System.out.println("================");
        ExecuteProcess execute = ac.getBean("execute", ExecuteProcess.class);
        System.out.println(execute);
        System.out.println(execute.getId());
        System.out.println(execute.getClass());
    }
}

public class ExecuteProcess {
    private Integer id;
    public ExecuteProcess(){
    }
    public ExecuteProcess(Integer id){
        this.id=id;
    }
    public void setId(Integer id){
        this.id=id;
    }
    public Integer getId(){
        return this.id;
    }
}
```
通过registerBean方法注入的类，会根据编码的顺序覆盖同名bean的内部属性，靠后的代码覆盖靠前代码的设置。
注意此种方式仅适用于AnnotationConfigApplicationContext类型的容器，并且实在容器初始化后进行的操作。

#### 3.4 实现ImportSelector接口
spring支持实现ImportSelector接口，通过代码的方式对bean的注入进行干预：
```java
/*
* 配置类使用@Import注解引入ImportSelector的实现类
*/
@Import({CustomeSelector.class})
public class AppConfig8Selector {
}

/*
* ImportSelector的实现类
*/
public class CustomeSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        String className = importingClassMetadata.getClassName();
        System.out.println("selectImports:"+className);
        System.out.println("selectImports:"+importingClassMetadata.isAnnotated("org.springframework.stereotype.Component"));
        return new String[]{"org.xxx.bean.ExecuteProcess"};
    }
}

/*
* 容器启动类
*/
public class ApplicationSelector {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext annotationConfigApplicationContext = new AnnotationConfigApplicationContext(AppConfig8Selector.class);
        String[] beanDefinitionNames = annotationConfigApplicationContext.getBeanDefinitionNames();
        for (String beanDefinitionName : beanDefinitionNames) {
            System.out.println(beanDefinitionName);
        }
    }
}
```
上述代码中的CustomeSelector实现了ImportSelector，并在返回值中返回了一些类的全路径类名，执行后结果如下：
```
selectImports:org.xxx.config.AppConfig8Selector
selectImports:false
appConfig8Selector
org.xxx.bean.ExecuteProcess
```
通过实现ImportSelector接口的方法返回的类名创建的bean，默认bean的名称也为全路径类名。  
除了可以通过在返回值中增加需要注入容器的bean全路径类名外，ImportSelector实现的方法selectImports还带有一个AnnotationMetadata类型参数，  
该参数包含了一些元数据信息主要是被@Import注解注释的类的一些信息，通过这些元数据信息，可以在方法中编写逻辑，对bean的注入进行干预，例如满足一些特定条件，才注入某些bean等。  

#### 3.5 实现ImportBeanDefinitionRegistrar接口
spring支持实现ImportBeanDefinitionRegistrar接口，通过代码的方式对bean的注入进行干预：
```java
/*
* ImportBeanDefinitionRegistrar接口
*/
public interface ImportBeanDefinitionRegistrar {

	default void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry,
			BeanNameGenerator importBeanNameGenerator) {

		registerBeanDefinitions(importingClassMetadata, registry);
	}

	default void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
	}
}
```
ImportBeanDefinitionRegistrar接口有两个默认实现方法，其中一个多了BeanNameGenerator参数，BeanNameGenerator参数也是spring中控制bean命名的参数。  
registerBeanDefinitions方法包含主要的两个参数AnnotationMetadata和BeanDefinitionRegistry，AnnotationMetadata在ImportSelector接口的方法入参中有提及，  
相较于ImportSelector的方法，ImportBeanDefinitionRegistrar的registerBeanDefinitions方法没有返回值，而是通过第二个BeanDefinitionRegistry类型参数干预bean的注入，相当于将bean的管理机制开放共开发者自行实现。  
```java
/*
* 配置类
*/
@Import({CustomeRegister.class})
public class AppConfig9CustomeRegister {
}

/*
* 自定义实现类
*/
public class CustomeRegister implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        BeanDefinition beanDefinition = BeanDefinitionBuilder.genericBeanDefinition(DataProcess.class).getBeanDefinition();
        registry.registerBeanDefinition("registerDataProcess",beanDefinition);
    }
}

/*
* 容器启动类
*/
public class Application9CustomeRegister {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(AppConfig9CustomeRegister.class);
    }

}
```
实现ImportBeanDefinitionRegistrar接口的方法更进一步的开放了元数据操作+bean定义的注册，可以进行更多的注入逻辑编码。

#### 3.6 实现BeanDefinitionRegistryPostProcessor接口
spring支持bean定义注册后置处理器接口，在容器初始化后进行后置处理，对bean的注入进行干预。  
接口如下：
```java
/*
* 接口
*/
@FunctionalInterface
public interface BeanFactoryPostProcessor {
	void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {
	void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;
}
```
BeanDefinitionRegistryPostProcessor继承自BeanFactoryPostProcessor，实际也可以通过实现BeanFactoryPostProcessor接口实现对bean注入的干预。
```java
/*
* 自定义实现类
*/
public class CustomePostProcessor implements BeanDefinitionRegistryPostProcessor {
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
        BeanDefinition beanDefinition = BeanDefinitionBuilder.rootBeanDefinition(DataJobProcessor2.class).getBeanDefinition();
        registry.registerBeanDefinition("dataJobProcessor",beanDefinition);
    }
    
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        // ... ...
    }
}
/*
* DataJobProcessor2类的代码
*/
public class DataJobProcessor2 implements Processor {
    public void process() {
        System.out.println("----------process2----------");
    }
}
```
通过实现postProcessBeanDefinitionRegistry或者postProcessBeanFactory方法，允许通过bean定义注册器或者bean工厂操作容器中的bean。
```java
/*
* 配置类，导入了自定义的后置处理器和一个类DataJobProcessor，并且DataJobProcessor类使用@Service("dataJobProcessor")标注名称为dataJobProcessor，和上方通过后置处理器添加的bean同名。
*/
@Import({CustomePostProcessor.class,DataJobProcessor.class})
public class AppConfig10PostProcessor {
}

@Service("dataJobProcessor")
public class DataJobProcessor implements Processor {

    public void process() {
        System.out.println("----------process----------");
    }
}

/*
* 容器启动类
*/
public class Application10CustomePostProcessor {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(AppConfig10PostProcessor.class);
        for (String beanDefinitionName : ac.getBeanDefinitionNames()) {
            System.out.println(beanDefinitionName);
        }
        System.out.println("================");
        Processor dataJobProcessor = (Processor)ac.getBean("dataJobProcessor");
        dataJobProcessor.process();
    }
}
```
启动类运行后打印如下,可见后置处理器总会在bean注入操作后对bean进行处理：
```
appConfig10PostProcessor
org.xxx.postprocessor.CustomePostProcessor
dataJobProcessor
================
----------process2----------
```

### 4.总结
spring在迭代过程中，bean的自动注入（ioc/di）从配置文件到配置类，从代码侵入到非侵入的支持，都在极力简化spring的使用,同时又不断的加强对bean注入干预能力的扩展和开放。  
这种改进也为后续springboot的拆包即用、则需而用的特性提供了实现基础。