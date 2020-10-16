---
title: SpringCloudAlibaba服务注册新发现
date: 2020-06-02 09:13:17
tags: SpringCloudAlibaba
---

# @(SpringCloudAlibaba)
2018.09.21「小马哥技术周报」- 第一期《Spring Cloud 服务发现新选择 - Alibaba Nacos Discovery》基本是刚出来的时候就已经讲了，现在都2020了
![Alt text](http://47.105.116.133:8080/upload/1.png)
![Alt text](http://47.105.116.133:8080/upload/2.png)
![Alt text](http://47.105.116.133:8080/upload/3.png)
![Alt text](http://47.105.116.133:8080/upload/6.png)
![Alt text](http://47.105.116.133:8080/upload/4.png)
![Alt text](http://47.105.116.133:8080/upload/5.png)
不太适合大规模的分布式服务发现ZAB算法
内存型，有内存限制
![Alt text](http://47.105.116.133:8080/upload/7.png)
3000-5000节点会出现问题
![Alt text](http://47.105.116.133:8080/upload/8.png)
https://start.spring.io/线上生成
![Alt text](http://47.105.116.133:8080/upload/9.png)
Eureka30K个实例后出现GC,以及实例之间的复制，中小型企业不会达到这么高服务
除了ZK都是AP
![Alt text](http://47.105.116.133:8080/upload/10.png)
![Alt text](http://47.105.116.133:8080/upload/11.png)

# Springcloudcommons as servicediscovery load balancing  circuit break
EnableDiscoveryClient是通用API

```java
public interface DiscoveryClient extends Ordered {
    int DEFAULT_ORDER = 0;

    String description();

    List<ServiceInstance> getInstances(String serviceId);

    List<String> getServices();

    default int getOrder() {
        return 0;
    }
}
```
实现类的其中一种
```java
public class EurekaDiscoveryClient implements DiscoveryClient {
    实现方法的返回值的ServiceInstance源码
    public interface ServiceInstance {
    default String getInstanceId() {
        return null;
    }
    zk是唯一标识；eureka是ip+服务名
    String getServiceId();

    String getHost();

    int getPort();
    //是否是HTTPs协议or not
    boolean isSecure();

    URI getUri();
    源信息  zk是又专门存储字段  补充信息
    Map<String, String> getMetadata();

    default String getScheme() {
        return null;
    }
}
}
```
在Eureka传递序列化或反序列化通过metadata传递，相关的开销比较大
超类接口Registration  cloud commons 继承了上面的serviceInstance
现在的位置
![Alt text](http://47.105.116.133:8080/upload/12.png)
空继承，为了扩展
```java
public interface ServiceRegistry<R extends Registration> {
    注册
    void register(R registration);
    de解除注册
    void deregister(R registration);

    void close();

    void setStatus(R registration, String status);

    <T> T getStatus(R registration);
}
```
# Nacos的实现注册
 com.alibaba.nacos.api.naming;NamingService
现在我下载源码总是下不下来，以为是maven出问题，忘记因为其他项目公用maven，导致maven配置的是私服地址，改下maven就行了
```java
F:\cloud2020>mvn dependency:resolve -Dclassifier=sources
[INFO] Scanning for projects...
Downloading from nexus-server: xxx
ependencies/2.2.2.RELEASE/spring-boot-dependencies-2.2.2.RELEASE.pom
Downloading from nexus-server: xxx
-dependencies/Hoxton.SR1/spring-cloud-dependencies-Hoxton.SR1.pom
Downloading from nexus-server: xxx
-dependencies/2.1.0.RELEASE/spring-cloud-alibaba-dependencies-2.1.0.RELEASE.pom
```
namingSpace有很多重载方法
```java
public interface NamingService {

    /**
     * register a instance to service
     *
     * @param serviceName name of service
     * @param ip          instance ip
     * @param port        instance port
     * @throws NacosException
     */
    void registerInstance(String serviceName, String ip, int port) throws NacosException;

    /**
     * register a instance to service
     *
     * @param serviceName name of service
     * @param groupName   group of service
     * @param ip          instance ip
     * @param port        instance port
     * @throws NacosException
     */
    void registerInstance(String serviceName, String groupName, String ip, int port) throws NacosException;

返回的是Instance
/**
 * get all instances of a service
 *
 * @param serviceName name of service
 * @return A list of instance
 * @throws NacosException
 */
 服务名   是否健康
List<Instance> getAllInstances(String serviceName) throws NacosException;
```
Ribbon-->server  
Eureka==>ServiceInstance
![Alt text](http://47.105.116.133:8080/upload/13.png)
```java
namingSpace以及instance都是Nacos的Api,而registration是Spring 为了适配实现
public void register(Registration registration) {
    if (StringUtils.isEmpty(registration.getServiceId())) {
        log.warn("No service to register for nacos client...");
    } else {
        String serviceId = registration.getServiceId();
        //this.getNacosInstanceFromRegistration  get  set
        Instance instance = this.getNacosInstanceFromRegistration(registration);

        try {
            放入nacos  注册中心持久化  可以连db
            this.namingService.registerInstance(serviceId, instance);
            log.info("nacos registry, {} {}:{} register finished", new Object[]{serviceId, instance.getIp(), instance.getPort()});
        } catch (Exception var5) {
            log.error("nacos registry, {} register failed...{},", new Object[]{serviceId, registration.toString(), var5});
        }

    }
}
```
放入nacos  注册中心持久化  可以连db
this.namingService.registerInstance(serviceId, instance);
Nacos  =Naming +config server
上面的实现类图可以看到ZK的实现
org.springframework.cloud.zookeeper.serviceregistry;
ZK的注册 反注册就跟现在的Nacos有一定程度的相似
![Alt text](http://47.105.116.133:8080/upload/14.png)
![Alt text](http://47.105.116.133:8080/upload/15.png)
![Alt text](http://47.105.116.133:8080/upload/16.png)


注解驱动
依赖注入
外部化配置
事件驱动
github.com/nacos-group/nacos-spring-project
github.com/nacos-group/nacos-spring-boot-project
![Alt text](http://47.105.116.133:8080/upload/17.png)
![Alt text](http://47.105.116.133:8080/upload/18.png)
![Alt text](http://47.105.116.133:8080/upload/20.png)
![Alt text](http://47.105.116.133:8080/upload/19.png)
gateway 是Http协议的一个转换
