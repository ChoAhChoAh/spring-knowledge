# SpringBoot
## spring对bean注入的干预和注入控制
### 1.能够控制bean加载的干预
在《bean注入方式》一文中，第三部分说了能对bean干预的一些操作和接口，但在这些干预的操作中，能够控制bean注入的方式为3.3-3.6这四个方式：
- 3.3 使用AnnotationConfigApplicationContext容器的registerBean方法
- 3.4 实现ImportSelector接口
- 3.5 实现ImportBeanDefinitionRegistrar接口
- 3.6 实现BeanDefinitionRegistryPostProcessor接口
以实现ImpostSelector接口为例，下属代码实现了根据org.xxx.component.BaseComponent类是否存在控制两种不同service(DefaultService和CustomeService)的注入：
```java
/*
* ImportSelector的实现类
*/
public class ConditionImportSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        List<String> importClassList = new ArrayList<>();
        Class<?> hasBaseComponent = null;
        try{
            hasBaseComponent = Class.forName("org.xxx.bean.BaseComponent");
        }catch (Exception e){
            System.out.println("not found base component");
        }
        if(ObjectUtils.isEmpty(hasBaseComponent)){
            importClassList.add("org.xxx.bean.CustomeService");
        }else{
            importClassList.add("org.xxx.bean.DefaultService");
        }
        return importClassList.toArray(new String[importClassList.size()]);
    }
}

/*
* 配置类引入实现的ConditionImportSelector
*/
@Import({ConditionImportSelector.class})
public class AppConfig {
}

/*
* 主运行类
*/
public class Application {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(AppConfig.class);
        for (String beanDefinitionName : ac.getBeanDefinitionNames()) {
            System.out.println(beanDefinitionName);
        }
    }
}
```
上述main方法运行时，根据ConditionImportSelector的实现，通过将org.xxx.component.BaseComponent进行修改可以达到如下效果：
```
当不存在org.xxx.component.BaseComponent类时，注入customeService
appConfig
customeService

当存在org.xxx.component.BaseComponent类时，注入defaultService
appConfig
defaultService
```

### 2.通过@Conditional注解+编码进行注入控制
spring还提供了@Conditional注解， 配合Condition接口，来实现1中的效果：
```java
/*
* 配置类，配置了扫描包的路径，两个bean的注入分别使用不同的Condition实现类控制注入条件
*/
@ComponentScan("org.xxx.component")
public class AppConditionConfig {

    @Bean
    @Conditional({HaveClassCondition.class})
    public DefaultService defaultService(){
        return new DefaultService();
    }

    @Bean
    @Conditional({MissingClassCondition.class})
    public CustomeService customeService(){
        return new CustomeService();
    }
}

/*
* 存在某个bean对应类的condition
*/
public class HaveClassCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        Class<?> type = null;
        try{
            type = context.getBeanFactory().getType("baseComponent");
        }catch (Exception e){
            System.out.println("found base component failed."+e.getMessage());
        }
        if(ObjectUtils.isEmpty(type)){
            return false;
        }else{
            return true;
        }
    }
}

/*
* 缺失某个bean对应类的condition
*/
public class MissingClassCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        Class<?> type = null;
        try{
            type = context.getBeanFactory().getType("baseComponent");
        }catch (Exception e){
            System.out.println("found base component failed."+e.getMessage());
        }
        if(!ObjectUtils.isEmpty(type)){
            return false;
        }else{
            return true;
        }
    }
}

/*
* 主运行类
*/
public class ApplicationCondition {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(AppConditionConfig.class);
        for (String beanDefinitionName : ac.getBeanDefinitionNames()) {
            System.out.println(beanDefinitionName);
        }
    }
}
```
上述main方法执行时，如果容器中存在一个名称为“baseComponent”的示例，并且能获取到其类型，则注入DefaultService，反之注入CustomeService
```
缺失“baseComponent”的打印
appConditionConfig
baseComponent1
customeService

存在“baseComponent”的打印
baseComponent
baseComponent1
defaultService
```
但请注意，上述两种方式仍旧属于spring的范畴，并且对于profile的加载，spring是使用此种方式实现的，可以查看ProfileCondition类的实现
```java
/**
 * {@link Condition} that matches based on the value of a {@link Profile @Profile}
 * annotation.
 *
 * @author Chris Beams
 * @author Phillip Webb
 * @author Juergen Hoeller
 * @since 4.0
 */
class ProfileCondition implements Condition {

	@Override
	public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
		MultiValueMap<String, Object> attrs = metadata.getAllAnnotationAttributes(Profile.class.getName());
		if (attrs != null) {
			for (Object value : attrs.get("value")) {
				if (context.getEnvironment().acceptsProfiles(Profiles.of((String[]) value))) {
					return true;
				}
			}
			return false;
		}
		return true;
	}

}
```
springboot基于@Conditional默认实现了一些列的派生注解和对应的处理逻辑，更方便bean控制的加载，也因此产生了各种技术默认注入和则需而用的能力（官方实现了诸多spring-boot-starter-xxx配合spring-boot-autoconfigure）,可以查看《自动装配》下的相关说明。