<!-- $size: 16:9 -->

# 7장 라우팅

---

## 클라우드 네이티브 시스템
- 기존 인스턴스가 내려가고 새 인스턴스가 올라오더라도 이를 이용하던 서비스는 문제 없이 서비스를 이용할 수 있어야 함
- IP주소는 클라이언트와 서버 사이를 고정적으로 결합하므로 정적 IP 주소에 의존하는것은 적합하지 않음
- 경로를 동적으로 찾을 수 있는 간접화가 필요

## DNS?

- 캐시때문에 오히려 DNS 레코드를 받아오는데 시간을 더 사용하게 됨
- 시스템 상태나 토폴로지 같은 정보 없음
- 로드 밸런싱을 적용하려면 라우팅 고민

---

# DiscoveryClient 추상화

```java
public interface DiscoveryClient {

	String description();

	List<ServiceInstance> getInstances(String serviceId);

	List<String> getServices();

}
// https://github.com/spring-cloud/spring-cloud-commons/commit/89170d127d4929bb0d0ca3ae69257b1fca67ac00#diff-27f0fb833ff90de5e39f258a4b527d74
```
> 서비스 레지스트리 종류는 다양한데 여기선 넷플릭스의 유레카를 살펴봄

---

# 유레카 서비스 레지스트리

```
org.springframework.cloud:spring-cloud-starter-netflix-eureka-server
```

```java
@EnableEurekaServer
@SpringBootApplication
public class EurekaServiceApplication { ... }
```
> `@EnableEurekaServer` 로 Eureka Server로 만들 수 있음

```
server.port=${PORT:8761}
# 서비스 레지스트리는 관례적으로 `8761`포트를 사용
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
# 서버 자신은 레지스트리에 등록하지 않는다
eureka.server.enable-self-preservation=false
# 자기 보호(self-preservation): 등록된 인스턴스중 많은 수가 정해진 시간 간격 내에 
# 생존 신호(heartbeat)을 보내오지 않으면 네트워크 문제라고 간주하고 등록을 유지
# 여기선 예제를 위해 인스턴스가 해제되는것을 보기 위해 false
```

---

# 유레카 클라이언트

```
org.springframework.cloud:spring-cloud-starter-netflix-eureka-client
```
> spring-cloud-starter-eureka (deprecated, please use spring-cloud-starter-netflix-eureka-client)
```java
@EnableDiscoveryClient
@SpringBootApplication
public class CloudNativeStudyCh7ClientApplication { ... }

@RestController
class GreetingRestController {
  @GetMapping("/hi/{name}")
  Map<String, String> hi(
    @PathVariable String name, 
    @RequestHeader(value = "X-CNJ-Name", required = false) Optional<String> cn) {
    String resolvedName = cn.orElse(name);
    return Collections.singletonMap("greeting", "Hello, " + resolvedName + "!");
  }
}
```

---

# 유레카 클라이언트

* bootstrap.yml
```
spring:
  application:
    name: greetings-service

server:
  port: ${PORT:0} # 랜덤 포트

eureka:
  instance:
    instance-id: ${spring.application.name}:${spring.application.instance_id:${random.value}}
    # hostname: ${vcap.application.uris[0]:localhost}
    # nonSecurePort: 80

#  client:
#    serviceUrl:
#      defaultZone: <service>.cfapps.io/eureka/
```

---

client
![](./스크린샷%202018-07-16%20오후%207.07.18.png)

server
![](./스크린샷%202018-07-16%20오후%207.09.07.png)


---

# 엣지(Edge) 서비스 (간단한 클라이언트)

> 시스템에 들어오는 요청이 처음 거쳐가는 서비스

서비스 레지스트리에서 `greetings-service`를 찾고 `greetings-service`와 통신

---

```java
@Component
public class DiscoveryClientCLR implements CommandLineRunner {
    private final DiscoveryClient discoveryClient;
    private final Registration registration;

    @Override
    public void run(String... args) throws Exception {
        log.info("localServiceInstance");
        logServiceInstance(registration);

        final String serviceId = "greetings-service";
        log.info(String.format("registered instance of '%s'", serviceId));
        discoveryClient.getInstances(serviceId).forEach(this::logServiceInstance);
    }

    private void logServiceInstance(Registration r) {
        log.info(String.format("\thost = %s, port = %s, service ID = %s", 
        	r.getHost(), r.getPort(), r.getServiceId()));
    }

    private void logServiceInstance(ServiceInstance s) {
        log.info(String.format("\thost = %s, port = %s, service ID = %s", 
        	s.getHost(), s.getPort(), s.getServiceId()));
    }
}
```

---

`DiscoveryClient` 로 서비스 레지스트리에 등록된 인스턴스 정보를 얻을 수 있다.
> `DiscoveryClient`의 `getLocalServiceInstance()` 매소드는 deprecated,
> 대신 `Registration` 빈을 주입받아 사용

---

## Ribbon

*클라이언트 로드밸런싱* 라이브러리
라운드로빈, 응답가중치 등 다양한 로드밸런싱 전략을 지원

> 클라이언트 로드밸런싱
> 등록된 서비스의 인스턴스 중 한버전을 선택 or 
> 라운드로빈 로드밸런싱처럼 임의로 하나를 선택 or
> 애플리케이션에 대한 정보를 더 확보 할 수 있다면 응답 가중치(weight-response) 등등 다양한 전략 존재 
> 코드에서는 단 한번만 기술하고 이후의 서비스 호출 시 재사용

---

* RibbonCLR
```java
List<Server> servers = discoveryClient.getInstances(serviceId).stream()
        .map(s -> new Server(s.getHost(), s.getPort()))
        .collect(Collectors.toList());

IRule roundRobinRule = new RoundRobinRule();

BaseLoadBalancer loadBalancer = LoadBalancerBuilder.newBuilder()
        .withRule(roundRobinRule)
        .buildFixedServerListLoadBalancer(servers);

IntStream.range(0, 10).forEach(i -> {
    Server server = loadBalancer.chooseServer();
    URI uri = URI.create("http://" + server.getHost() + ":" + server.getPort() + "/");
    log.info("resolved service " + uri.toString());
});
```
---

```
localServiceInstance
	host = 10.0.1.5, port = 8080, service ID = unknown
registered instance of 'greetings-service'
	host = 10.0.1.5, port = 49834, service ID = GREETINGS-SERVICE
	host = 10.0.1.5, port = 49821, service ID = GREETINGS-SERVICE
	host = 10.0.1.5, port = 49716, service ID = GREETINGS-SERVICE
Client: default instantiated a LoadBalancer: {NFLoadBalancer:name=default,current list of Servers=[],Load balancer stats=Zone stats: {},Server stats: []}
resolved service http://10.0.1.5:49821/
resolved service http://10.0.1.5:49716/
resolved service http://10.0.1.5:49834/
resolved service http://10.0.1.5:49821/
resolved service http://10.0.1.5:49716/
resolved service http://10.0.1.5:49834/
resolved service http://10.0.1.5:49821/
resolved service http://10.0.1.5:49716/
resolved service http://10.0.1.5:49834/
resolved service http://10.0.1.5:49821/
```

---

# Customizing the Ribbon Client by Setting Properties

```
<ServiceName>:
  ribbon:
    NIWSServerListClassName: 
    	org.springframework.cloud.netflix.ribbon.eureka.DomainExtractingServerList
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.WeightedResponseTimeRule
    NFLoadBalancerPingClassName: com.netflix.niws.loadbalancer.NIWSDiscoveryPing
```
https://cloud.spring.io/spring-cloud-netflix/multi/multi_spring-cloud-ribbon.html

---

# @LoadBalanced

> 스프링 클라우드가 자동 설정

`RestTemplate`에 `@LoadBalanced`를 사용해서 스프링 클라우드에게 리본을 인식할 수 있는 로드밸런싱 인터셉터를 `RestTemplate`에 적용하라고 알려줄 수 있다.

```java
Map<String, String> variables = Collections.singletonMap("name", "Cloud Natives!");

IntStream.range(0, 10).forEach(i -> {
    ResponseEntity<JsonNode> response = restTemplate
      .getForEntity("//greetings-service/hi/{name}", JsonNode.class, variables);

    JsonNode body = response.getBody();
    String greeting = body.get("greeting").asText();
    log.info("greeting: " + greeting);
});
```
> 호출시 `"//greetings-service/hi/{name}"` 이렇게 서비스ID를 사용한다

---
```
greeting: Hello, Cloud Natives!! from greetings-service(10.0.1.5:50315)
greeting: Hello, Cloud Natives!! from greetings-service(10.0.1.5:50322)
greeting: Hello, Cloud Natives!! from greetings-service(10.0.1.5:50289)
greeting: Hello, Cloud Natives!! from greetings-service(10.0.1.5:50315)
greeting: Hello, Cloud Natives!! from greetings-service(10.0.1.5:50322)
greeting: Hello, Cloud Natives!! from greetings-service(10.0.1.5:50289)
greeting: Hello, Cloud Natives!! from greetings-service(10.0.1.5:50315)
greeting: Hello, Cloud Natives!! from greetings-service(10.0.1.5:50322)
greeting: Hello, Cloud Natives!! from greetings-service(10.0.1.5:50289)
greeting: Hello, Cloud Natives!! from greetings-service(10.0.1.5:50315)
```

---

# 클라우드 파운드리 라우트 서비스

## 클라이언트 로드밸런싱
 - 강력한 경로 결정 기능
 - 라우팅 관련 코드를 쉽게 작성
 - 라우팅 로직이 변경시 다른 클라이언트에 영향 X

하지만 모든 서드파티 클라이언트가 넷플릭스 리본을 탑재할것이라고 가정하기 힘듦
(iOS 클라이언트가 아파치 주키퍼, 넷플릭스 리본을 사용하진 않을테니...)

이럴땐 클라이언트 요청, 라우팅, 리다이렉팅을 담당하는 **중간 서버**를 두는것이 좋다.

라우트 서비스는 범용 프록시 역할을 담당. HTTP, HTTPS 둘다 지원해야함.

---

# 예제 7-10, 11) 라우트 서비스

> 로깅 기능이 있는 라우트 서비스

downstream-service 에 접속하면
리퀘스트를 route-service가 받아서 로그를 남기고
downstream-service를 호출

```
cf create-user-provided-service route-service -r <라우트 서비스의 URL>
# 사용자 정의 서비스 생성
cf bind-route-service cfapps.io route-service --hostname <바인딩할 URL>
# 라우트 서비스를 "<바인딩할 URL>.cfapps.io"를 거치는 모든 애플리케이션에 바인딩
```

---

# PWS 사용해보기

> 프리티어: 2GB 메모리 + 86달러의 트라이얼 크레딧
> `service-registry` 와 2개의 `greetings-service` 인스턴스 띄워보기

--- 

## manifest.yml 작성

* service-registry
```
applications:
- name: service-registry
  memory: 682M
  instances: 1
  random-route: true
```

* downstream-service (greetings-service)
```
applications:
- name: downstream-service
  memory: 682M
  instances: 2
  random-route: true
  path: build/libs/cloud-native-study-ch7-client-0.0.1-SNAPSHOT.jar
  env:
    SPRING_PROFILES_ACTIVE: prod
```
> path를 지정해주면 `cf push`로 쉽게 배포 가능
> env로 유저 환경변수도 줄 수 있음

---

![](./스크린샷%202018-07-16%20오후%206.05.14.png)

---

유레카 서비스 레지스트리에 등록된 인스턴스
![](./스크린샷%202018-07-16%20오후%207.12.04.png)

---

라우트 서비스를 이용해 `downstream-service` 호출시 로깅
![](./스크린샷%202018-07-16%20오후%207.22.54.png)

```
> curl https://downstream-service-relaxed-ratel.cfapps.io/hi/test
incoming request: <GET https://route-service-zany-crocodile.cfapps.io/, {
 host=[route-service-zany-crocodile.cfapps.io],
 user-agent=[curl/7.49.0],
 accept=[*/*],
 accept-encoding=[gzip],
 x-b3-spanid=[c0b0eb9dd8dbc93d],
 x-b3-traceid=[c0b0eb9dd8dbc93d],
 x-cf-applicationid=[b17fb651-ad1e-4a92-970e-6625df30a9ec],
 x-cf-forwarded-url=[https://downstream-service-relaxed-ratel.cfapps.io/hi/test],
 x-cf-instanceid=[74884991-4476-48af-64af-0f6e],
 x-cf-instanceindex=[0],
 x-cf-proxy-metadata=[eyJub25jZSI6IkNwTmhQSGd5M295U1Y0Ym8ifQ==],
 x-cf-proxy-signature=[yomevnEJDNHCTVmwhSYwk-9bwYCB52bpjBDra1qudk1xS-Iq_m30KJpvlxuc8pTyPI9aJec-8JqNGdNeGFEeD-Lz_RUyPhxcmVqIq38zDcx3Ud4Ckth2PgOZP8ZHgXL2MJcJCj7bZpuTAPOXyx7-H67MFvoN4-6xE4T29q9Ivb0cAUwv_mDHdkNHALz4aJG4],
 x-vcap-request-id=[c725e980-6fa8-4970-437e-d20f06edc799],
 x-forwarded-port=[443],
 x-forwarded-proto=[https],
 x-request-start=[1531736553801]}>
```
---

![](./스크린샷%202018-07-16%20오후%207.46.55.png)

기본적으로 약 640M 이상을 주어야 띄울 수 있지만 `JAVA_OPTS=-Xss256k`를 주면 512M 으로도 띄울 수 있음