# spring-RocketMQ-bus

## bus[消息总线介绍]
配置了同一个消息总线的服务，其中任意一个发送一个事件消息。如果其他也监听了该事件，都可以收到该事件消息。
下图简单的演示了当配置发现修改的时候，如果通知集群同步更新配置。serviceA-3接受通知页面更新了配置，马上发送配置修改的事件消息到bus里面。serviceA-1,serciceA-2注册了修改配置事件，收到修改配置事件。重新拉取配置。


![1202638-20180521203126866-1299643942.png](https://cdn.nlark.com/yuque/0/2021/png/1509048/1613234725918-604aa261-8ea7-4a62-baff-43db475a379f.png#align=left&display=inline&height=580&margin=%5Bobject%20Object%5D&name=1202638-20180521203126866-1299643942.png&originHeight=580&originWidth=968&size=70094&status=done&style=none&width=968)


## 安装java开发环境


## 安装RocketMQ


## 下载官方dome


## 实现流程
#### maven配置
```xml
<dependency>
      <groupId>com.alibaba.cloud</groupId>
      <artifactId>spring-cloud-starter-bus-rocketmq</artifactId>
  		<version>${revision}</version>
</dependency>
```


#### 配置
> 在spring-boot配置里面加入下面两行配置，就可以启动spring-RocketMQ-bus

```java
// web服务端口
server.port=8080
// 启动trace级别日志
spring.cloud.bus.trace.enabled=true
// spring.cloud.bus.id 是设定这个实例的在进群唯一标识符。建议使用下面的规则。${项目名}:${服务端口}
// 也可以使用dubbo的端口
// 最好是本机ip地址
spring.cloud.bus.id=${spring.application.name}-{IP}:${server.port}
// 配置spring-cloud-bus需要的stream实现，使用了rocketmq
spring.cloud.stream.rocketmq.binder.name-server=127.0.0.1:9876

```
#### 使用RemoteApplicationEventScan注解启动bus
> com.alibaba.cloud.examples.rocketmq 事件的目录

```java
@RestController
@EnableAutoConfiguration
@RemoteApplicationEventScan(basePackages = "com.alibaba.cloud.examples.rocketmq")
public class RocketMQBusApplication {

	public static void main(String[] args) {
		new SpringApplicationBuilder(RocketMQBusApplication.class).properties("IP").run(args);
	}
}
```
#### 创建消息总线事件


>  创建的事件必须继承RemoteApplicationEvent，
>  创建的事件必须继承RemoteApplicationEvent
>  创建的事件必须继承RemoteApplicationEvent

```java
package com.alibaba.cloud.examples.rocketmq;

import org.springframework.cloud.bus.event.RemoteApplicationEvent;

/**
 * {@link User} {@link RemoteApplicationEvent}.
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 0.2.1
 */
public class UserRemoteApplicationEvent extends RemoteApplicationEvent {

	/**
	 * 
	 */
	private static final long serialVersionUID = 1L;
	private User user;

	public UserRemoteApplicationEvent() {
	}

	public UserRemoteApplicationEvent(Object source, User user, String originService,
			String destinationService) {
		super(source, originService, destinationService);
		this.user = user;
	}

	public void setUser(User user) {
		this.user = user;
	}

	public User getUser() {
		return user;
	}

}
```
#### 创建消息


#### 创建发送对象
> 使用ApplicationEventPublisher对象发送事件。直接调用publishEvent方法传递需要广播的事件

```java
	@Autowired
	private ApplicationEventPublisher publisher;

	/**
	 * Publish the {@link UserRemoteApplicationEvent}.
	 * @param name the user name
	 * @param destination the destination
	 * @return If published
	 */
	@GetMapping("/bus/event/publish/user")
	public boolean publish(@RequestParam String name,
			@RequestParam(required = false) String destination) {
		User user = new User();
		user.setId(System.currentTimeMillis());
		user.setName(name);
		publisher.publishEvent(new UserRemoteApplicationEvent(this, user, originService, destination));
		return true;
	}
```




#### 接受方法
> 1. 在方法上使用EventListener注解，方法形参是接受的事件就行了。
> 1. class上必须加上Component注解或者它的子注解（Controller，Service，Repository，RestController）
> 1. AckRemoteApplicationEvent事件触发点是发送事件的服务收到消息总线广播的广播就会触发AckRemoteApplicationEvent事件，建议大家处理AckRemoteApplicationEvent事件，实现重复机制。

```java
@Server
public class EventHandler{

    @EventListener
    public void onEvent(UserRemoteApplicationEvent event) {
        System.out.printf("Server [port : %d] listeners on %s\n", localServerPort,event.getUser());
    }

    @EventListener
    public void onAckEvent(AckRemoteApplicationEvent event)throws JsonProcessingException {
        System.out.printf("Server [port : %d] listeners on %s\n", localServerPort,objectMapper.writeValueAsString(event));
    }
}
```


#### 本地测试
##### RocketMQBusApplication

