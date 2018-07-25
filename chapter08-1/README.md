예제 소스 - https://github.com/inaz1502/cloud-native-java

<br><br><br><br>

<!-- $theme: gaia -->
<!-- template: invert -->
<!-- page_number:true -->
<!-- $size: 16:9 --> 
  
# Ch 8. 엣지 서비스

----

<div style="font-size:75%">

## 엣지 서비스 ( API Gateway )

* 여러 Client와 Backend 사이의 추가 계층
* Client 입장에서는 하나의 Endpoint
* 보안, 요청 제한, 계측, 라우팅 등의 횡단 관심사 처리
* Technically, an API Gateway is the API exposed to the public (REST, etc.), and an Edge Service is a service running on the API resolving the proxying, routing, etc. There could be many edge services on the Gateway. But practically there is usually only one service, logic, on the Gateway thus API Gateway = Edge Service. (https://stackoverflow.com/questions/46002727/what-is-the-difference-between-an-api-gateway-and-an-edge-service)
	
</div>

----

<div style="font-size:75%">

<img src="apigateway.jpg" width=800px />

</div>

----

<div style="font-size:75%">

## Netflix Feign

<br>

* 인터페이스와 몇 가지 규약을 정의하여 서비스 클라이언트를 생성
* spring-cloud-starter-openfeign 의존성 추가
* @EnableFeignClients
* @FeignClient(name = "", (optional)url = "", ...)


</div>

-----

<div style="font-size:75%">
  
## Spring Cloud Feign


* build.gradle
~~~
compile('org.springframework.cloud:spring-cloud-starter-openfeign:2.0.0.RELEASE')
~~~
<br>

* GreetingClient.java
~~~
@FeignClient(name = "greeting-service", url = "http://localhost:8081")
public interface GreetingClient {

    @RequestMapping(method = RequestMethod.GET, value = "/greet/{name}")
    Map<String, Object> greet(@PathVariable("name") String name);

    @RequestMapping(method = RequestMethod.POST, value = "/greet")
    Map<String, Object> greetPost(Map<String, Object> map);

}
~~~

</div>

-----

<div style="font-size:75%">
 

* FeignController.java
~~~
@RestController
public class FeignController {

    private GreetingClient greetingClient;

    @Inject
    FeignController(GreetingClient greetingClient) {
        this.greetingClient = greetingClient;
    }

    @GetMapping("/api/{name}")
    public Map<String, Object> api(@PathVariable String name) {
        return greetingClient.greet(name);
    }

    @PostMapping(value = "/api2")
    public Map<String, Object> api2(@RequestBody Map<String, Object> map) {
        return greetingClient.greetPost(map);
    }
}
~~~
  
</div>

----

<div style="font-size:75%">
  
## Feign Only

<br>

* GreetingClient2.java
~~~
public interface GreetingClient2 {
    @RequestLine("GET /greet/{name}")
    @Headers({
            "Content-Type: application/json"
    })
    Map<String, Object> greet(@Param("name") String name);

    @RequestLine("POST /greet")
    Map<String, Object> greetPost(@HeaderMap Map<String, Object> headers,
                                    Map<String, Object> map);
}
~~~
  
</div>

----

<div style="font-size:75%">


* FeignController.java
~~~
@RestController
public class FeignController {

    private GreetingClient2 greetingClient2;

    @Inject
    FeignController(GreetingClient greetingClient) {
        ConfigurationManager.getConfigInstance()
		            .setProperty("greeting-service" + 
                    			".ribbon.listOfServers", 
	                    		"http://localhost:8082,http://localhost:8081");
                        
        LoadBalancingTarget<GreetingClient2> target = 
        	LoadBalancingTarget.create(GreetingClient2.class, 
            					"http://greeting-service");
                                
        System.out.println(target.lb().getLoadBalancerStats());
        
        greetingClient2 = Feign.builder()
                                .encoder(new GsonEncoder())
                                .decoder(new GsonDecoder())
                                .target(target);
    }

    @GetMapping("/api3/{name}")
    public Map<String, Object> api3(@PathVariable String name) {
        return greetingClient2.greet(name);
    }

    @PostMapping(value = "/api4")
    public Map<String, Object> api4(@RequestBody Map<String, Object> map) {
        Map<String, Object> headers = new HashMap<>();
        headers.put("Content-Type", "application/json");
        return greetingClient2.greetPost(headers, map);
    }
}
~~~
  
</div>

----

<div style="font-size:75%">
  
## Netflix Zuul

<br>

* Client로부터 들어와서 서비스에 전달되는 모든 요청에 공통적으로 해당되는 이슈들이 존재
* Netflix Zuul을 사용하면 일정량 이상의 요청 제한, 프록시 등 여러 문제를 해결할 수 있는 필터를 적용 가능
* 인증 및 보안
* 모니터링과 분석
* 동적 라우팅
* 트래픽 조정 등등...
* 서비스들의 Cross Cutting Concern 처리에 적합
  
</div>

----

<div style="font-size:75%">
  
<br>
  
* build.gradle
~~~
compile('org.springframework.cloud:spring-cloud-starter-netflix-zuul:2.0.0.RELEASE')
~~~
 
<br>

* Main
~~~~
@EnableZuulProxy
@EnableDiscoveryClient
@SpringBootApplication
public class ProxyServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(ProxyServiceApplication.class, args);
    }

}
~~~~
 
</div>

----

<div style="font-size:75%">

## Zuul Configuration Base Route

<br>

* application.yml

~~~~~

zuul:
  routes:
    greeting-service:
      path: /greet/**
      // url이 없으면 아래 ribbon 설정으로 server를 discovery
      url: http://localhost:8081
      // path에 Match된 url을 떼고 전달 하고 싶으면 true
      stripPrefix: false
      

// default
greeting-service:
  ribbon:
    NIWSServerListClassName: com.netflix.loadbalancer.ConfigurationBasedServerList
~~~~~

</div>

----

<div style="font-size:75%">

## Zuul Eureka Base Route

* build.gradle

~~~~
// com.netflix.niws.loadbalancer packege가 아래 의존성을 추가해야 사용 가능
compile('com.netflix.ribbon:ribbon-eureka:2.3.0')
~~~~

<br>

* application.yml

~~~~~
greeting-service:
  ribbon:
    eureka:
      enabled: true
    NIWSServerListClassName: com.netflix.niws.loadbalancer.DiscoveryEnabledNIWSServerList
    ConnectTimeout: 1000
    ReadTimeout: 3000
    MaxTotalHttpConnections: 500
    MaxConnectionsPerHost: 100
~~~~~


</div>

-----

<div style="font-size:75%">
  
## Zuul Filter

* Pre Filter - 요청이 라우팅 되기 전에 실행
* Routing Filter - 요청의 실제 라우팅을 처리
* Post Filter - 요청이 라우팅 된 후에 실행
* Error Filter - 요청 처리중 에러가 발생하면 실행


## Zuul 2.0

* Pre Filter -> InBound Filter
* Routing Filter -> EndPoint Filter
* Post Filter -> OutBound Filter


</div>

-----

## Zuul 1.x Filter Lifecycle
<br>

<img src="zuul 1.x.png" width=800px />


-----

## Zuul 2.x Filter Lifecycle
<br>

<img src="zuul 2.x.png" width=600px />

-----


<div style="font-size:75%">

<br><br>

## 그러나...

* spring-cloud-starter-netflix-zuul은 zuul-core 1.3 버전을 사용
* 현재 버전의 spring zuul integration에서는 2.x 사용 불가능
* Spring + Zuul 2.x 를 사용 하려면 spring-cloud-starter-netflix-zuul 의존성 없이 사용해야 가능

</div>

----

<div style="font-size:75%">

## Custom Zuul Filter

<br><br>

* ZuulConfiguration.java

~~~
@Configuration
public class ZuulConfiguration {
    @Bean
    RateLimiter rateLimiter() {
        return RateLimiter.create(1.0D / 5.0D);
    }
}
~~~

</div>

----

<div style="font-size:75%">


* RateLimiterZuulInboundFilter.java

~~~
@Service
@RefreshScope
public class RateLimiterZuulInboundFilter extends ZuulFilter {

    @Value("${zuul.RateLimiterFilter.enabled:true}")
    private Boolean filterEnabled;

    private final HttpStatus tooManyRequests = HttpStatus.TOO_MANY_REQUESTS;

    private final RateLimiter rateLimiter;

    @Inject
    public RateLimiterZuulInboundFilter(RateLimiter rateLimiter) {
        this.rateLimiter = rateLimiter;
    }

    @Override
    public String filterType() {
        return FilterConstants.PRE_TYPE;
    }

    @Override
    public int filterOrder() {
        return Ordered.HIGHEST_PRECEDENCE;
    }

    @Override
    public boolean shouldFilter() {
        return filterEnabled && RequestContext.getCurrentContext()
        		.getRequest().getRequestURI().contains("greet");
    }

    @Override
    public Object run() throws ZuulException {
        try {
            RequestContext currentContext = RequestContext.getCurrentContext();
            HttpServletResponse response = currentContext.getResponse();

            if(!rateLimiter.tryAcquire()) {
                response.setContentType(MediaType.APPLICATION_JSON_UTF8_VALUE);
                response.setStatus(tooManyRequests.value());
                //response.getWriter().append(tooManyRequests.getReasonPhrase());

                currentContext.setSendZuulResponse(false);

                throw new ZuulException(tooManyRequests.getReasonPhrase(), 
                			tooManyRequests.value(), 
                            		tooManyRequests.getReasonPhrase());
            }
        } catch (Exception e) {
            ReflectionUtils.rethrowRuntimeException(e);
        }
        return null;
    }
}
~~~

</div>
