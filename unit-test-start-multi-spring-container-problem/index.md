# 单元测试重复启动spring容器问题填坑


### 问题发现

* 【问题背景】 业务中心建设-库存中心压测时，发版失败，失败原因：数据库连接数超过MySQL最大连接数。
* 【问题排查】

> * 1、由于在压测环境做库存中心压测，所以第一反应时觉得压测的TPS过高造成的数据库连接不够。故找DBA将库存中心的数据库最大连接数从2048调整到3072。再次发版，数据库连接数依旧被打满。
> * 2、当时Spring配置的连接池大小：最大最小连接数都是300，尝试将数据库连接数降低到50，再次发版，数据库连接数飙升到6000
> * 3、重新调整springboot数据源配置，将最小连接降低到1，最终在压测环境发版成功。

### 问题处理

#### 【干掉@MockBean】

* @MockBean的作用：创建一个虚拟的对象替代那些不易构造会不易获取的对象。在实际开发中，我们自己的Controller，Service很可能去调用其他同事的接口或数据库，对方可能只写了一个接口，还没来得及写实现，这样时没办法进行联调测试的。此时可以通过@MockBean注入一个虚拟的Bean对象用于完成本地的单元测试。
* @MockBean 实例

```java
@MockBean
public RocketMqTempalte  rocketMqTempalte;

@MockBean
public UserIntegration  userIntegration;
//...
```

* 以上写法会造成SpringBoot启动多次，每次启动都会连接数据源，即有N个单元测试就启动了N个spring容器，就开启了300*N个数据库连接，因此导致数据库连接资源耗尽，无法发版。

#### 优化单元测试

* 通过一个公共的单元测试配置类如TestConfiguration管理需要Mock的Bean，使用Mockito.mock()方法生成Mock对象，在需要使用Mock对象的地方通过Spring依赖注入的方式注入即可。

```JAVA
@Configuration
public class TestConfiguratin
  @Bean
  private InventoryOperationDetailDao inventoryOperationDetailDao(){
    return Mockito.mock(InventoryOperationDetailDao.class);
  }

  @Bean
  private InventoryOperationDetailManage inventoryOperationDetailManage (){
    return Mockito.mock(InventoryOperationDetailManage.class);
  }
  // ...
```

### 问题分析

* [参考文档] https://docs.spring.io/spring-framework/reference/testing/testcontext-framework/ctx-management/caching.html

```
Context Caching
Once the TestContext framework loads an ApplicationContext (or WebApplicationContext) for a test, that context is cached and reused for all subsequent tests that declare the same unique context configuration within the same test suite. To understand how caching works, it is important to understand what is meant by “unique” and “test suite.”

An ApplicationContext can be uniquely identified by the combination of configuration parameters that is used to load it. Consequently, the unique combination of configuration parameters is used to generate a key under which the context is cached. The TestContext framework uses the following configuration parameters to build the context cache key:

locations (from @ContextConfiguration)

classes (from @ContextConfiguration)

contextInitializerClasses (from @ContextConfiguration)

contextCustomizers (from ContextCustomizerFactory) – this includes @DynamicPropertySource methods as well as various features from Spring Boot’s testing support such as @MockBean and @SpyBean.

contextLoader (from @ContextConfiguration)

parent (from @ContextHierarchy)

activeProfiles (from @ActiveProfiles)

propertySourceLocations (from @TestPropertySource)

propertySourceProperties (from @TestPropertySource)

resourceBasePath (from @WebAppConfiguration)

For example, if TestClassA specifies {"app-config.xml", "test-config.xml"} for the locations (or value) attribute of @ContextConfiguration, the TestContext framework loads the corresponding ApplicationContext and stores it in a static context cache under a key that is based solely on those locations. So, if TestClassB also defines {"app-config.xml", "test-config.xml"} for its locations (either explicitly or implicitly through inheritance) but does not define @WebAppConfiguration, a different ContextLoader, different active profiles, different context initializers, different test property sources, or a different parent context, then the same ApplicationContext is shared by both test classes. This means that the setup cost for loading an application context is incurred only once (per test suite), and subsequent test execution is much faster.
```

* 根据文档的描述，不难知道 application context是通过key：value方式进行缓存的，`</br>`唯一键为组合键，包含：locations、classes、contextInitiallizerClasses、contextCustomizers、contextLoader、parent、activeProfiles、propertySourceLocations、propertySourceProperties、resourceBasePath。
* 而@MockBean的使用会导致每个application context中的contextCustomizer的不同，从而导致存储在context cache中的application context 的unique key 不同，`</br>`最终导致application context 在测试类之间不能共享。虽然没有官方文档说明这一点，不过在org.springframework.boot.test.mock.mockito.MockitoContextCustomizerFactory源码中可以找到一些痕迹。

> MockitoContextCustomizerFactory源码

```JAVA
// ...
class MockitoContextCustomizerFactory implements ContextCustomizerFactory{
    @overried
    public MockitoContextCustomizerFactory createContextCustomizer(Class<?> testClass,List<ContextConfigurationAttributes> configAttributes){
        // we gather the explicit mock definitions here since they from part of the
        // MergedContextConfiguration key. Different mocks need to hava a fifferent key . !!! 看这里 ！！！
        DefinitionsParser parser = new DefinitionsParser();
        parser.parse(testClass);
        return new MockitoContextCustomizer(parser.getDefinitions());
    }
}

```

* 上面代码中注释所说的MergedContextConfiguration 就是 application context caching 的unique key。

