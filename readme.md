# spring cloud
本项目使用

spring-boot     2.0.3.RELEASE版本

spring-cloud    Finchley.RELEASE

### 一 eureka-server
eureka-server 是 服务注册中心。

配置文件application.properties，提供 eureka 的相关信息。

1. hostname=localhost 表示主机名称。
2. register-with-eureka=false. 表示是否注册到服务器。 
    --- 因为它本身就是服务器，所以就无需把自己注册到服务器了。
3. fetch-registry=false. 表示是否获取服务器的注册信息，和上面同理，这里也设置为 false。
4. service-url.defaultZone=http://${eureka.instance.hostname}:${server.port}/eureka/ 
   自己作为服务器，公布出来的地址。 
   比如后续某个微服务要把自己注册到 eureka server, 那么就要使用这个地址： http://localhost:8761/eureka/
5. name=eureka-server 表示这个微服务本身的名称是 eureka-server

### 二 product-data-service
product-data-service 是 一个微服务。

1. 启动多次该服务，分配不一样的端口号8001，8002...
2. 在服务层service中加入了 一个port变量，可以显现出用了哪个端口，从不同的微服务得到的数据。
3. 配置文件中需要 配置：
    1. 注册中心的地址 http://localhost:8761/eureka/
    2. spring.application.name=product-data-service 表示 该数据服务在 eureka 注册中心的名称。
   

### 三 product-view-service-ribbon
访问已经注册好的数据微服务。 spring-cloud 提供了两种方式，一种是 Ribbon，一种是 Feign。
#### 理论
1. Ribbon 是使用 RestTemplate 进行调用，并进行客户端负载均衡。
2. 客户端负载均衡：在前面 注册数据微服务 里，注册了8001和8002两个微服务， Ribbon 会从注册中心获知这个信息，
   然后由 Ribbon 这个客户端自己决定是调用哪个，这个就叫做客户端负载均衡。
3. Feign 是对 Ribbon的封装。使用注解的方式，调用起来更简单，也是主流的方式。
4. 为什么不用前后端分离呢？ 干嘛要用 thymeleaf 做服务端渲染呢？
   
    原因如下：
    1. 使用前后端分离，站长多半会用 vue.js + axios.js来做，就像 springboot 天猫教程那样。 如果学习者没有这个基础，就会加重学习的负担。
    2. 使用前后端分离，是走的 http 协议， 那么就无法演示重要的 微服务端调用了，所以站长这里特意没有用前后端分离，以便于大家观察和掌握微服务的彼此调用

### 三 product-view-service-feign
Feign 是对 Ribbon 的封装，使用注解的方式，调用起来更简单，也是主流的方式。

pom需要添加 spring-cloud-starter-openfeign 依赖，用来支持 Feign 方式的

client客户端，Ribbon和Feign的不同之处：

Ribbon：
```java
@Component public class ProductClientRibbon {
   @Autowired
   private RestTemplate restTemplate;
   public List<Product> listProduct(){
      return restTemplate.getForObject("http://PRODUCT-DATA-SERVICE/products",List.class);
   }

}
```
Feign：

```java
@FeignClient(value = "PRODUCT-DATA-SERVICE")
public interface ProductClientFeign {
    @GetMapping("/products")
    List<Product> listProduct();
}
```

#### 实操
1. ribbon 需要 配置一个类 ProductClientRibbon：

   Ribbon 客户端， 通过 RestTemplate 访问（getForObject/postForObject） http://PRODUCT-DATA-SERVICE/products，
   而 product-data-service 既不是域名也不是ip地址，而是 数据服务在 eureka 注册中心的名称。
   注意看，这里只是指定了要访问的 微服务名称，但是并没有指定端口号到底是 8001, 还是 8002

2. ribbon 和 service 有相同实体类、service层，略微不同的controller，但是ribbon有web层（试图）
3. 启动类 配置： 
    
    @EnableDiscoveryClient， 表示用于发现eureka 注册中心的微服务；
    @Bean @LoadBalanced RestTemplate 表示用 restTemplate 这个工具来做负载均衡。
4. 配置文件

product-view-service-ribbon：该数据服务在 eureka 注册中心的名称。

### 四 zipkin 服务链路

1. 什么是服务链路

   我们有两个微服务，分别是数据服务和视图服务，随着业务的增加，就会有越来越多的微服务存在，他们之间也会有更加复杂的调用关系。
   这个调用关系，仅仅通过观察代码，会越来越难以识别，所以就需要通过 zipkin 服务链路追踪服务器 这个东西来用图片进行识别了。
   
2. 这里使用 zipkin-server-2.10.1-exec.jar（zipkin目录下）

   1. 启动命令 java -jar zipkin-server-2.10.1-exec.jar(地址栏就可输入)/在idea直接右键run
   2. 启动所有需要的服务系统
   3. 执行一次 http://127.0.0.1:8012/products
   4. 访问链路追踪服务器 http://localhost:9411/zipkin/dependency/ 就可以看到 视图微服务调用数据微服务 的图形
   
3. 改造：product-data-service和product-view-service-xxx
   
   1. 都增加pom依赖 spring-cloud-starter-zipkin
   2. 启动类都配置 Sampler 抽样策略： ALWAYS_SAMPLE 表示持续抽样
      ```
      @Bean
      public Sampler defaultSampler(){ return Sampler.ALWAYS_SAMPLE;}
      ```
   3. 配置文件都增加： spring.zipkin.base-url: http://localhost:9411

### 配置服务

有时候，微服务要做集群，这就意味着，会有多个微服务实例。
在业务上有时候需要修改一些配置信息，比如说 版本信息吧~ 倘若没有配置服务， 那么就需要挨个修改微服务，挨个重新部署微服务，这样就比较麻烦。
为了偷懒， 这些配置信息就会放在一个公共的地方，比如git, 然后通过配置服务器把它获取下来，然后微服务再从配置服务器上取下来。
这样只要修改git上的信息，那么同一个集群里的所有微服务都立即获取相应信息了，这样就大大节约了开发，上线和重新部署的时间了。

见图（repos/ConfigServer.png）
我们先在 git 里保存 version 信息， 然后通过 ConfigServer 去获取 version 信息， 接着不同的视图微服务实例再去 ConfigServer 里获取 version.

### 启动：
1. 先启动注册中心 EurekaServerApplication
2. 启动ConfigServerApplication，访问 http://localhost:8030/version/dev
   ```
   {"name":"version","profiles":["dev"],"label":null,"version":"5046c9520e739312615997563357740456cc5756","state":null,"propertySources":[]}
   ```
3. 然后启动两次服务 ProductDataServiceApplication， 分别输入 8001和8002.
4. 然后运行Ribbon ProductViewServiceRibbonApplication 以启动 微服务，然后访问地址：
   http://127.0.0.1:8010/products

   或者运行Feign  ProductViewServiceFeignApplication 已启动 微服务，访问
   http://127.0.0.1:8012/products
5. 执行一次 http://127.0.0.1:8012/products（启用Feign，不使用Ribbon）
6. 访问链路追踪服务器 http://localhost:9411/zipkin/dependency/ 就可以看到 视图微服务调用数据微服务 的图形
---