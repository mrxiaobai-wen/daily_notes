微服务注册中心nacos学习：先尝试使用它，然后撸它源码搞懂它。

在这里整理一下自己之前集成nacos的内容。

我的github地址：https://github.com/mrxiaobai-wen/springcloud_study.git

前置条件：下载nacos并安装启动。

## 服务提供者集成

创建一个Spring Cloud项目，即nacos-server-spring-cloud。

### 引入Nacos的依赖

~~~xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
~~~

### 配置nacos连接

bootstrap.yml配置内容：

~~~yaml
server:
  port: 8021

spring:
  application:
    name: nacos-server-spring-cloud
  cloud:
    nacos:
      discovery:
        server-addr: http://localhost:8848
      config:
        server-addr: http://localhost:8848
        file-extension: yaml
~~~

在application.yml中添加一个ceshi.version配置，用于后面测试nacos配置中心：

~~~yaml
ceshi:
  version: dev
~~~

## 创建启动类

~~~java
@SpringBootApplication
@EnableDiscoveryClient
@EnableConfigurationProperties
public class NacosServerSpringCloudApplication {

    @Resource
    private ConfigBean configBean;

    public static void main(String[] args) {
        SpringApplication.run(NacosServerSpringCloudApplication.class, args);
    }

    @RestController
    class EchoController {
        @RequestMapping(value = "/server/echo/{string}", method = RequestMethod.GET)
        public String echo(@PathVariable String string) {
            return "Hello Nacos Discovery " + string + " 当前版本：" + configBean.getVersion();
        }
    }

}
~~~

~~~java
@ConfigurationProperties("ceshi")
@Component
public class ConfigBean {

    public String version;

    public String getVersion() {
        return version;
    }

    public void setVersion(String version) {
        this.version = version;
    }
}
~~~

就这样，启动上面的NacosServerSpringCloudApplication后，就可以在本地nacos服务列表中查看到当前服务了，然后在nacos配置中心里面新建一个nacos-server-spring-cloud.yml文件，变更发布ceshi.version的值，然后访问localhost:8021/server/echo/{string}就可以看到变更的内容了。

这样就简单的完成了服务注册和配置动态管理。

### 采坑

我在最开始的时候，将bootstrap.yml的内容放在application.yml中，然后在配置中心一起发布，但是配置变更一直没有生效。然后经过一番摸索后,拆成了bootstrap.yml和application.yml两个配置文件后，配置动态变更生效了。

Spring Cloud获取数据的时候，其dataId的拼接格式为：${prefix} - ${spring.profiles.active} . ${file-extension}。其中prefix默认为spring.application.name的值，如果配置了多环境，spring.profiles.active即为配置的环境的值。

## 服务消费者集成

创建一个Spring Cloud项目，即nacos-consumer-spring-cloud。

依赖与上面一致。application.yml配置也与上面一致。

### 创建启动类

~~~java
@SpringBootApplication
@EnableDiscoveryClient
public class NacosConsumerSpringCloudApplication {

    public static void main(String[] args) {
        SpringApplication.run(NacosConsumerSpringCloudApplication.class, args);
    }

    @LoadBalanced
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

    @RestController
    public class TestController {

        private final RestTemplate restTemplate;

        @Autowired
        public TestController(RestTemplate restTemplate) {this.restTemplate = restTemplate;}

        @RequestMapping(value = "/consumer/echo/{str}", method = RequestMethod.GET)
        public String echo(@PathVariable String str) {
            return restTemplate.getForObject("http://nacos-server-spring-cloud/server/echo/" + str, String.class);
        }
    }
}
~~~

然后将上面的nacos-server-spring-cloud和这个消费服务一起启动。然后访问当前的消费服务，访问/consumer/echo/{str}接口，可以看到请求最终转到了上面的那个服务中。

而我们的请求地址是http://nacos-server-spring-cloud/server/echo/，这就是注册中心的作用，我们不用关注server服务的具体地址，只是请求nacos-server-spring-cloud即可。

