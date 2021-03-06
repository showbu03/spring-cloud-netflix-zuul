Spring Cloud Netflix Zuul
==============

*by S.M.Lee*

&nbsp;

# Overview

대용량 웹 서비스가 증가함에 따라 Microservice Architecture는 선택이 아니라 필수가 되어가고 있다. 기존 Monolithic Architecture와는 달리 Microservice Architecture는 작은 Microservice 단위로 나누어 시스템을 구축한다. 이러한 Microservice는 보통 하나 혹은 여러 개의 API로 개발된다. 그렇다면 Microservice가 수백 개 혹은 수천 개까지 증가할 때 수많은 Endpoint와 공통 로직 등 이를 어떻게 관리해야 할까? 

API Gateway는 수많은 백 단의 API Server들의 Endpoint들을 단일화하고, Authentication, Logging, Monitoring, Routing 등 여러 역할을 수행 할 수 있다. 물론 Netflix의 Zuul은 이러한 기능들을 전부 제공하고 있다. Netflix의 Zuul뿐만 아니라 다른 API Gateway를 사용해도 앞서 말한 기능들을 제공받을 수 있을것이다.

그러나 Zuul은 많은 트래픽과 이로 인해 발생하게 될 여러 이슈들에 대해 신속히 대응할 수 있도록 다양한 Filter를 제공한다. 
또한 Netflix의 Zuul은 다른 Netflix OSS의 Component들과 결합이 되었을 때 활용도가 증가한다. 
이 중 가장 매력적인 것은 Client Side Loadbalancer인 Ribbon + Service Discovery & Registry인 Eureka + Fault tolerance library(Circuit Breaker)인 Hystrix 조합을 활용한 Dynamic Routing이지 않을까 싶다. 그리하여 여기서는 Dynamic Routing에 중점을 맞춰서 튜토리얼을 진행해 볼 것이다.

일단 Ribbon, Eureka, Hystrix는 Zuul에 내장되어 있다. Eureka Registry에 등록된 특정 Service의 Server List들을 이용하면 Load balancer인 Ribbon에 굳이 Routing 시킬 Server List를 따로 등록해 줄 필요가 없게 된다. 이러한 특징은 MSA에서 매우 중요하게 작용한다.

도메인별 수 많은 API Server로 이루어진 시스템을 관리하기 위해 일일이 Routing 시킬 Server list를 Load balancer에 하드코딩하여 등록할 필요가 없고,
새로운 API Server가 계속해서 추가된다 하더라도 Eureka Registry에 등록된 정보를 이용하기에 새롭게 도입된 Server List를 Load balancer에 추가할 필요가 없어진다. 

또한 추후에 클러스터를 도입하여 replica를 여러 개 만들어 배포시킨다 할 때도 Dynamic Routing을 하지 않는다면 Server List를 일일이 입력해야 할 것이다.. 별로 바람직하지 않다. 어쨌든 글만으로는 이해가 부족할 수 있다. 실제로 Dynamic Routing을 시켜보자.


# Before We Start 

우리는 고정 ip가 있는 3개 혹은 4개의 Server가 필요하다. 그리고 이 Server들은 다음처럼 활용할 것이다.

1. API Gateway Zuul- (Eureka + Ribbon + Hystrix)
2. micro-service.1(Spring Boot Microservice(Eureka Client))
3. micro-service.2(Spring Boot Microservice(Eureka Client))
4. Eureka-Server

2,3번은 각자 다른 서버에 배포되는 같은 서비스이다. Eureka Server는 1,2,3번 Server 중 하나를 이용해 구축해도 된다. 

필자는 aws ec2 centos7을 활용했고, azure,gcp 등 원하는 cloud platform을 활용하던지, 물리적인 서버를 활용해도 상관없다.  

(들어가기에 앞서 다음 레퍼런스를 꼭 참고하자)
* Spring-Cloud-Netflix-Eureka-Tutorial => https://github.com/phantasmicmeans/Spring-Cloud-Netflix-Eureka-Tutorial

그럼 먼저 API Gateway를 준비해보자.

# API Gateway - Zuul 

## 1. Dependency

Zuul, Eureka-client 의존성을 추가한다. 또한 Hystrix Dashboard를 추가한다.

```xml		             
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

	<dependency>
		<groupId>org.springframework.cloud</groupId>
		<artifactId>spring-cloud-starter-netflix-zuul</artifactId>
            	<!-- <version>2.0.0.RC1</version> -->
	</dependency>
	
    	<dependency>
        	<groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        
        <dependency>
        	<groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
            <!-- <version>2.0.0.M2</version> -->
        </dependency>

	<dependency>
    		<groupId>org.springframework.boot</groupId>
            	<artifactId>spring-boot-starter-web</artifactId>
        </dependency>

	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-test</artifactId>
		<scope>test</scope>
	</dependency>
```


## 3. Include Zuul

```java
@EnableEurekaClient
@SpringBootApplication
@EnableHystrixDashboard
@EnableZuulProxy
public class ApiGatewayApplication {

	public static void main(String[] args) {
		SpringApplication.run(ApiGatewayApplication.class, args);
	}
	
}
```
EurekaClient로 만들기 위해 @EnableEurekaClient annotataion, 그리고 Hystrix Dashboard를 사용하기 위해 @EnableHystrixDashboard annotation을 추가한다.

Hystrix의 장점중 하나는 각 HystrixCommand 에 대해 metric을 수집하는 것이다. Hystrix Dashboard는 각 circuit breaker에 대한 상태를 보여준다. 이를 활용하려면 @EnableHystrixDashboard annotation을 main class에 추가하면 된다. 그리고 ~/hystrix 로 접속 시 각 Hystrix Client들의 상태를 볼 수 있다.

그리고 Zuul의 Embedded ReverseProxy를 위한 @EnableZuulProxy annotation을 추가하면 된다. 이 기능을 사용하면 local call은 적절한 service로 forwarding 된다. 예를 들어 id가 users인 service는 /users의 proxy로 부터 request를 받게 될 것이다. 그리고 이 proxy는 Ribbon을 이용해 인스턴스를 찾는다. 

위의 설명이 아직 이해가 안될 수도 있다. 아래에서 다시 나올 내용이니 일단 진행하자.


## 2. Configuration

**1. bootstrap.yml**
```yml	
spring:
    application:
        name: zuul-service
```

application.name을 정한다.

**2. application.yml**

```yml	
zuul:
    ignored-service: "*" 
    prefix: /api
    routes:
        story-service:
            path: /story/**
            serviceId: story-service
            stripPrefix: false
    ribbonIsolationStrategy: THREAD
    threadPool:
        useSeparateThreadPools: true
```
설정 파일을 잘 보면 이해할 수 있다. zuul.ribbonIsolationStrategy와 zuul.threadPool은 아래에서 설명하겠다.
그럼 먼저 ignored-service와 prefix이다.

* ignored-service => zuul의 라우팅 목록 중 story-service를 제외하고는 ignore 한다.*
* prifix => Zuul에 의해 routing 되는 모든 service의 Endpoint를 /api/~ 로 묶는다.

zuul의 routing 목록 중  /story(zuul.routes.story-service.path)로 들어오는 Http call은 story-servie(zuul.routes.story-service.serviceId)로 forwarding 된다. 이 serviceId에 우리가 routing 시킬 Eureka-Client의 serviceId를 입력하면 된다. 그렇다면 5번째 라인의 zuul.routes.story-service는 어디서 정의될까? 이제 아래를 보자.

```yml	
hystrix:
    command:
        story-service:
            execution:
                isolation:
                    strategy: THREAD
                    thread:
                        timeoutInMilliseconds: 20000
story-service:
    ribbon: 
        eureka:
            enabled: true
        NIWSServerListClassName: com.netflix.niws.loadbalancer.DiscoveryEnabledNIWSServerList
        ConnectTimeout: 5000
        ReadTimeout: 5000
        MaxTotalHttpConnections: 500
        MaxConnectionsPerHost: 100
    

```

위의 zuul.routes.story-service는 아래 story-service에서 정의된다. 그리고 이 story-service의 Server List는 Ribbon을 이용해 찾는다.  
그렇다면 이 Ribbon은 story-service가 있는 Server List들을 어디서 가져올까? 분명 yaml파일을 전부 뒤져봐도 Server List는 찾아볼 수 없다. 그렇다면 지금부터 일일이 Server List를 등록해줘야 할까? 아니다. 앞서 설명했듯이 Eureka Registry로부터 story-service가 실행 중인 Server List를 가져오면 된다. 이렇게 되면 Loadbalancer인 Ribbon에 Server List를 추가할 필요가 없다. 
**NIWSServerListClassName: com.netflix.niws.loadbalancer.DiscoveryEnabledNIWSServerList** 를 이용해 Eureka 정보를 사용하면 된다는 얘기이다. 

&nbsp;

다음은 위보다 조금 까다로운 Hystrix 부분이다. Hystrix는 Netflix의 Fault tolerance library(Circuit Breaker)라고 했다. 제일 쉽게 설명하면 "한 서비스에 장애가 일어났을때 이를 isolate 시켜 전체 시스템을 보호한다" 라고 말할 수 있다. 그러나 깊게 들어가면 조금 까다롭다.. 하지만 장시간 삽질하며 얻은 결과를 최대한 공유 해보려고 한다..

일단 Isolation을 이해해야 한다. Netflix의 Document는 다음처럼 설명한다. "Hystrix employs the bulkhead pattern to isolate dependencies from each other and to limit concurrent access to any one of them." 결국 서비스간에 의존성을 제한하고 격리하겠단 얘기로 보인다.

Hystrix는 Thread & Thread Pool, 그리고 Semaphore라는 2가지 Isolation 방법을 제공하고. 우리는 Thread & Thread Pool을 사용한다. 공식 Document에서는 Thread Isolation에 대해 "Clients (libraries, network calls, etc) execute on separate threads. This isolates them from the calling thread (Tomcat thread pool) so that the caller may “walk away” from a dependency call that is taking too long." 이라고 말한다.

예를들어 어떠한 API Server를 호출하는 Thread는 따로 격리된다는 얘기이다. 즉 실행중인 Tomcat의 Thread Pool와 API Server를 호출하는 thread는 격리된다. 따라서 API 호출 thread에서 지연이 일어나더라도 'Tomcat의 Thread Pool과는 격리되어져 있으므로 다른 thread 작업을 실행 할 수 있다' 는 의미로 이해하면 될 것 같다. 

어쨌든 본론으로 돌아가 우리가 사용할 앞에서 나왔던 zuul.ribbonIsolationStrategy와 zuul.threadPool을 보자. 
우리는 앞서 말했듯이 Hystrix의 Isolation Strategy를 Thread & Thread Pool로 사용한다. 따라서 ribbonIsolationStrategy: THREAD로 설정하자.

문제는 zuul의 IsolationStrategy를 Thread로 설정 시, 모든 라우팅 목록의 서비스들이 같은 Hystrix Thread Pool을 사용하게 된다는 것이다. 이렇게 진행한채로 Hystrix Dashboard의 Thread Pool 목록을 확인하면 "RibbonCommand" 라는 default Thread Pool을 볼 수 있을 것이다.

뭐 지금은 story-service 하나뿐이라 딱히 상관은 없지만.. 추후에 service를 추가하기 위해 서라도 Thread Pool을 서비스 별로 나눠야 한다.
따라서 threadPool.useSeparateThreadPools의 옵션을 true로 주자.

```yml
zuul:
    ribbonIsolationStrategy: THREAD
    threadPool:
        useSeparateThreadPools: true
```

위에도 나와있지만 이런 형식이다. 이렇게 진행하면 HystrixThreadPoolKey는 serviceId가 default로 붙게 된다. 우리는 story-service로 Thread Pool이 생성될 것이고, 이를 변경하고 싶다면 threadPoolKeyPrefix를 설정하면 된다. 

아래의 hystrix.command.story-service..부분도 같은 내용이므로 위 내용을 잘 파악하고 넘어가면 될 것 같다. 

**참고**

hystrix.command...timeoutInMilliseconds는 Ribbon의 각 timeout보다 커야 잘 동작한다. 
(RibbonHystrixTimeoutException, Ribbon의 TimeooutException에 대해서는 더 알아봐야 한다.. 다음을 참고하자)
* "Zuul, Ribbon and Hystrix timeout confusion" => https://github.com/spring-cloud/spring-cloud-netflix/issues/2606

아래의 Hytrix의 wiki에는 Isonlation Strategy와 관련하여 Thread & Thread Pool의 장단점, Semaphore의 특징 등 자세히 설명 되있다. 꼭 참고하자!
* "Netflix/Hystrix" => https://github.com/Netflix/Hystrix/wiki/How-it-Works#isolation


```yml	
eureka:
    client:
        healthcheck: true 
        fetch-registry: true
        serviceUrl:
            defaultZone: ${vcap.services.eureka-service.credentials.uri:http://{Your-Eureka-Server-Address}:8761}/eureka/
        instance:
            instance-id: ${spring.application.name}:${spring.application.instance_id:${random.value}}
            perferIpAddress: true
```

이 부분은 Zuul 또한 Eureka Client로 등록하는 부분이다.


# Microservice 

이제 소스코드에서 spring-cloud-netflix-eureka-client directory에 담겨있는 Microservice(Spring boot application)를 남은 2개의 서버로 각각 가져가서 실행하자. 이 Microservice는 REST API Server로 구축된다. Service에 대한 명세는 다음과 같다. 

## 1. REST API Server 

**REST API**

METHOD | PATH | DESCRIPTION 
------------|-----|------------
GET | /storys | Hostname Check(Loadbalancing이 되고 있음을 확인하기 위함)
GET | /story | 전체 stroy 중 최근 10개 제공 
GET | /story/{id} | 해당 id를 가진 user에 대한 story 제공 
POST | /story | story 정보 입력
DELETE | /story/{id} | 해당 id를 가진 user에 대한 story 삭제 

&nbsp;


**Table(table name = story) description**

| Field       | Type        | Null | Key | Default | Extra          |
--------------|-------------|------|-----|---------|----------------|
| story_id    | int(11)     | NO   | PRI | NULL    | auto_increment |
| ID          | varchar(20) | NO   |     | NULL    |                |
| message     | varchar(300) | NO   |     | NULL    |                |

*5.6.40 MySQL Community Server*

&nbsp;

DB Server 세팅은 Microservice가 배치 될 어느곳에 해도 상관 없다. 이 외에도 REST API Server 구축, 실행 방법에 관한 정보는 다음을 참고하면 된다.

* Spring-Boot-Microservice-with-Spring-Cloud-Netflix => https://github.com/phantasmicmeans/Spring-Boot-Microservice-with-Spring-Cloud-Netflix/

&nbsp;

# Dynamic Routing 

Eureka Server, API Gateway, Microservice가 전부 준비 되면 실제로 Dynamic Routing, Client Side Loadbalancing 이 되는지 확인해야 한다.
일단 Eureka Registry에 Gateway와 2개의 Microservice가 잘 등록 되었는지 보자.

## Check Eureka

* https://{Your-Eureka-Server-Address}:8761/
&nbsp;

![image](https://user-images.githubusercontent.com/20153890/41764741-043a23ec-763d-11e8-9550-32bb5896b549.png)

Zuul 그리고 application.name이 story-service인 Microservice가 2개가 Registry에 등록된 것을 볼 수 있다.

&nbsp;

이번에는 Eureka Registry에 등록된 Client들의 정보를 확인해보자. 

* https://{Your-Eureka-Server-Address}:8761/eureka/apps
&nbsp;

![image](https://user-images.githubusercontent.com/20153890/41764911-7db6ba0a-763d-11e8-8240-1f8155160ada.png)

STORY-SERVICE인 instance가 2개 존재한다. 같은 service이지만 instanceId가 다르다. hostname도 각각의 ip Address를 사용하고 있다. 


## Curl

우리의 Loadbalancer인 Ribbon에는 이 story-service에 대한 Server List가 없다. 하지만 Eureka Registry에는 story-service에 대한 정보가 들어있다. 준비는 모두 끝났고 Client Side에서 Loadbalancing을 진행하며 Dynamic하게 Routing 할 일만 남았다.

그럼 local에서 API Gateway가 실행중인 Server에 curl을 날려보자
&nbsp;

```sh	
sangmin@Mint-SM ~ $ curl -X GET http://13.125.247.1**:4000/api/story-service/storys
micro-service1

sangmin@Mint-SM ~ $ curl -X GET http://13.125.247.1**:4000/api/story-service/storys
micro-service2

sangmin@Mint-SM ~ $ curl -X GET http://13.125.247.1**:4000/api/story-service/storys
micro-service1

sangmin@Mint-SM ~ $ curl -X GET http://13.125.247.1**:4000/api/story-service/storys
micro-service2
```

Dynamic Routing 뿐만 아니라 Microservice가 실행되고 있는 각 Server의 hostname이 Load balancing되며 출력 되는 모습을 볼 수 있다. 

&nbsp;

## Hystrix Dashboard 

그럼 앞에서 설명했던 Hystrix Dashboard를 들어가보자.
&nbsp;

* https://{Your-Zuul-Address}:4000/hystrix
![image](https://user-images.githubusercontent.com/20153890/41768328-db7c95de-7646-11e8-82f8-8a5295ed5c61.png)




위 처럼 Hystrix Dashboard 하단의 Input box에 https://{Your-Zuul-Address}:4000/hystrix.stream, 그리고 아래 Delay를 입력하자.

```java
2018-06-22 08:27:51.905  INFO 1423 --- [io-4000-exec-10] ashboardConfiguration$ProxyStreamServlet : 

Proxy opening connection to: http://13.125.247.1**:4000/hystrix.stream?delay=2000
```
zuul이 실행중인 application의 log를 확인해 보면 proxy가 opening 되었단 로그를 볼 수 있다. 이 상태로 다시 curl을 날려보면 다음처럼 story-service의 Hystrix Thread Pool에 속한 HystrixCommand에 대해 metric을 수집 하고 있는 것을 볼 수 있다.

![image](https://user-images.githubusercontent.com/20153890/41793198-5024648a-7696-11e8-92ab-13d2325d94c7.png)

## Conclusion ## 

이상으로 Netflix Zuul(Eureka+Ribbon+Hystrix)을 활용해 Dynamic Routing, 그리고 Client Side에서의 Loadbalncing까지 진행해 보았다.
다음은 Docker Swarm(Docker Container Clustering Tool)와 Spring Cloud Netflix의 Component를 결합하여 Netflix OSS의 장점을 살리고 Clustering까지 할 수 있는 튜토리얼을 진행해 보려한다.

## References

* Router and Filter Zuul : https://cloud.spring.io/spring-cloud-netflix/multi/multi__router_and_filter_zuul.html
* 우아한형제들 기술 블로그(배민 API GATEWAY - spring cloud zuul 적용기) : http://woowabros.github.io/r&d/2017/06/13/apigateway.html
* spring-cloud-netflix : https://github.com/spring-cloud/spring-cloud-netflix
* "Netflix/Hystrix" => https://github.com/Netflix/Hystrix/wiki/How-it-Works#isolation
