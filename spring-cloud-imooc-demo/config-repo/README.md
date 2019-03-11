# springcloud-study-config-center
cpringcloud配置中心:用来存放git配置文件

# spring-cloud 学习总结笔记

### 环境
* spring-boot: 2.0.0.M3
* spring-cloud: Finchley.M2
* jdk: 1.8
* mavne: 3.5.4
* linux: centos 7
* docker: 17.05.0-ce
* rabbitmq: 3.7.12
* mysql

## Eureka
### Eureka Server
-   部分依赖
```xml
<dependency>
     <groupId>org.springframework.cloud</groupId>
     <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```
* 启动类添加注解: @EnableEurekaServer
```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaApplication.class, args);
    }
}

```
* application.yml配置
```yaml
server:
  port: 8761
eureka:
  client:
    service-url:
     defaultZone: http://localhost:8761/eureka/
#    defaultZone: http://localhost:8761/eureka/,http://localhost:8763/eureka/
#    可以多个服务之间互相注册
#    在默认设置下，Eureka服务注册中心也会将自己作为客户端来尝试注册它自己，所以我们需要禁用它的客户端注册行为。
    register-with-eureka: false
    fetch-registry: false
  server:
    enable-self-preservation: false

# 当前应用名称
spring:
  application:
    name: eureka-server

```
>   配置完成，我们就可以访问到：http://localhost:8761。
Eureka注册中心完成。

### Eureka Client
#### 商品服务
-   部分依赖
```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
        </dependency>

```
-   application.yml
```yaml
server:
  port: 8082
eureka:
  client:
    service-url:
      defaultZone: http://192.168.193.132:8761/eureka
# 我这里地址为虚拟机地址，我把eureka-server放在linux中自动运行的。      
  instance:
    hostname: product-service
    
spring:
  application:
    name: product-service
#    数据源，使用的jpa
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    username: root
    password: 123456
    url: jdbc:mysql://192.168.193.132/demo_sell?characterEncoding=utf-8&useSSL=false
  jpa:
    show-sql: true  
```
-   同样需要在启动类上添加@EnableDiscoveryClient

#### Eureka总结
-   添加依赖、yml配置
-   服务端：@EnableEurekaServer     
客户端：@EnableEurekaClient
-   心跳检查、健康检查、负载均衡登功能
-   Eureka的高可用，多节点注册中心
-   分布式系统中，服务注册中心是最重要的基础部分

### 康威定律
-   任何组织在设计一套系统（广义概念上的系统）时，所交付的设计方案在结构上都与该组织的沟通结构保持一致。

### 应用间通信

#### 客户端负载均衡器：Ribbon
>   Eureka属于客户端发现，它的负载均衡是软负载，也就是客户端会向服务器比如Eureka Server拉取已注册的
> 
>  服务信息,然后根据负载均衡策略直接选择哪一台服务器发送请求。这整个过程都是在客户端完成，并不需要
>
>服务器的参与，spring cloud中的客户端负载均衡的组件就是Ribbon，基于netflix ribbon实现，所以通过spring
>
>cloud的封装可以轻松的面向服务的restful模板请求，自动转换成客户端负载均衡的服务调用。

-   ribbon实现负载均衡核心：
>   -   服务发现：发现服务依赖的列表
>   -   服务选择规则：依据规则策略如何从多个服务中选择有效的服务
>   -   服务监听：检测失效的服务做到高效剔除

-   主要组件：
>   -   ServerList
>   -   IRule
>   -   ServerListFilter
>
>流程：  
1.首先会通过ServerList获取所有的可用服务列表
>
>2.然后ServerListFilter过滤掉不需要的服务地址
>
>3.最后从剩下的服务中通过IRule选择一个实例作为最终目标结果


#### Spring Cloud服务间两种restful通信方式

##### TestTemplate的三种方式
-   商品服务接口
```java
@RestController
@RequestMapping("/product-server")
public class ServerController {

    @GetMapping("/msg")
    public String msg() {
        return "this is product 'msg'";
    }
}
```
-   订单服务调用
```java
@Slf4j
@RestController
@RequestMapping("/order-client")
public class ClientController {

    // 第一种方式    直接使用RestTemplate调用路由, url写死
    @GetMapping("/product-msg-query-1")
    public String getProductMsg1() {
        RestTemplate restTemplate = new RestTemplate();
        // 路由接口-url， 返回类型
        String response = restTemplate.getForObject("http://localhost:8082/product-server/msg", String.class);
        log.warn("response: {}", response);
        return response;
    }
    
    
    // 第二种方式  org.springframework.cloud.client.loadbalancer.LoadBalancerClient获取url及端口，然后再使用第一种方法，需注入LoadBalancerClient
    @Autowired
    private LoadBalancerClient loadBalancerClient;
    
    @GetMapping("/product-msg-query-2")
    public String getProductMsg2() {
        RestTemplate restTemplate = new RestTemplate();
        ServiceInstance choose = loadBalancerClient.choose("PRODUCT-SERVICE");
        String url = String.format("http://%s:%s", choose.getHost(), choose.getPort() + "/product-server/msg");
        String response = restTemplate.getForObject(url, String.class);
        log.warn("response: {}", response);
        return response;
    }
    
    
    // 第三种方法 (使用注解@LoadBalanced， 可在restTemplate里使用应用名字)，
    @Autowired
    private RestTemplate restTemplate;
    
    @GetMapping("/product-msg-query-3")
    public String getProductMsg3() {
        String response = restTemplate.getForObject("http://PRODUCT-SERVICE/product-server/msg", String.class);
        log.warn("response: {}", response);
        return response;
    }
}

// 使用第三种RestTemplate通信时
@Component
public class RestTemplateConfig {

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

```

#### Feign