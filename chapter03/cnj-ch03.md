<!-- $size: 16:9 -->

3장 12요소 애플리케이션 설정
===

2018-07-13, 최범균(madvirus@madvirus.net)

---
<!-- page_number: true -->

* 스프링 프레임워크 설정 지원
* 스프링 부트의 설정 지원
* 스프링 클라우드 설정 서버

---
# 설정

* 12요소 애플리케이션의 설정
  * 실행되는 환경에 따라 달라질 수 있는 값을 의미: 예, 암호, 포트, 호스트 이름 등
  * 애플리케이션 내부 설정을 포함하지 않음
  * (주로) 환경변수에 저장

설정이 바르게 적용되어 있다면 코드베이스에서 중요한 인증 정보를 노출하거나 수정하지 않고도 아무때나 오픈소스로 공개할 수 있음

---
# 스프링 프레임워크 설정 지원: Environment, PropertyPlaceHolderConfigurer

<div style="float:left; width: 40%; font-size: 16pt">

```
# some.properties
configuration.projectName=Spring Framework
```

</div>

<div style="float:left; width:59%; margin-left: 1%; font-size: 14pt">

```java
@Configuration
@PropertySource("some.properties")
public class Application {
    @Bean
    static PropertySourcesPlaceholderConfigurer pspc() {
        return new PropertySourcesPlaceholderConfigurer();
    }

    @Value("${configuration.projectName}")
    private String fieldValue;
    @Value("${JAVA_HOME}")
    private String javaHome;

    @Autowired
    void setEnv(Environment env) {
        logger.info("env:java.version: " + env.getProperty("java.version"));
        logger.info("env:JAVA_HOME: " + env.getProperty("JAVA_HOME"));
    }

    @PostConstruct
    void afterPropertiesSet() {
        logger.info("fieldValue: {}", fieldValue);
        logger.info("JAVA_HOME: {}", javaHome);
    }

    public static void main(String[] args) {
        new AnnotationConfigApplicationContext(Application.class);
    }
```

</div>

---
# 스프링 프레임워크 설정 지원: 프로파일

* 실행 환경에 따라 빈을 다르게 설정 가능

<div style="float:left; width: 40%; font-size: 18pt">

```java
@Configuration
@Profile("prod")
public class ProdConfiguration {
  @Bean
  DataSource dataSource() { ... }
}

@Configuration
@Profile("test")
public class ProdConfiguration {
  @Bean
  DataSource dataSource() { ... }
}
```
</div>

<div style="float:left; width:59%; margin-left: 1%; font-size: 20pt">

```java
AnnotationConfigApplicationContext ctx = 
        new AnnotationConfigApplicationContext();
ctx.getEnvironment().setActiveProfiles("prod");
ctx.register(Application.class);
ctx.refresh();
```

</div>

---
# 스프링 부트 방식의 설정

우선 순위 (https://goo.gl/XAbajT)
* 명령행 인자
* SPRING_APPLICATION_JSON
* java:comp/env에 있는 JNDI 속성
* 시스템 프로퍼티
* OS 환경 변수
* jar 외부의 application-프로파일.properties (yml)
* jar 내부의 application-프로파일.properties (yml), application.properties(yml)
* jar 외부의 application.properties(yml)
* jar 내부의 application.properties(yml)
* @PropertySource
* 디폴트 프로퍼티(SpringApplication.setDefaultProperties())

---
# @ConfigurationProperties

* POJO 속성에 설정을 매핑

<div style="float:left; width: 40%; font-size: 18pt">

```java
@Component
@ConfigurationProperties("cuda")
public class CudaProperties {
    private String path;
    private int version;
    
    ... getter/setter
```

```
# application.properties
cuda.version=8

# 환경 변수
CUDA_PATH=C:\CUDA\v8.0
```
</div>

<div style="float:left; width:59%; margin-left: 1%; font-size: 17pt">

```java
@SpringBootApplication
public class ConfigApplication {
    @RestController
    static class InfoController {
        private CudaProperties cudaProp;

        @Autowired
        public InfoController(CudaProperties cudaProp) {
            this.cudaProp = cudaProp;
        }

        @GetMapping("/info")
        public CudaProperties info() {
            return cudaProp;
        }
    }

    public static void main(String[] args) {
        SpringApplication.run(ConfigApplication.class, args);
    }
}
```

</div>

<div style="clear: both"></div>

* /actuator/configprops에서 @ConfigurationProperties 목록 확인 가능


---
# 외부 설정에서 부족한 점

* 어플리케이션 설정 정보 변경시 어플리케이션 재시작
* 설정 정부 수정에 대한 추적성 없음
* 설정이 분산되어 있어 변경할 설정 정보 파악 어려움
* 손쉬운 암호화/복호화 방법 없음

스프링 클라우드 서버를 통한 중앙 집중형 설정으로 해소

---
# 스프링 클라우드 설정 서버

<div style="float:left; width: 40%; font-size: 16pt">

* 설정 제공 서버
* GIT/SVN/Vault/파일 
* spring-cloud-config-server 모듈의 @EnableConfigServer로 설정 서버 구동 가능
</div>

<div style="float:left; width:59%; margin-left: 1%; font-size: 17pt">

```java
@EnableConfigServer
@SpringBootApplication
public class ConfigServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(
            ConfigServerApplication.class, args);
    }
}
```

```
server:
  port: 8888
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/madvirus/config-server-repo
```

</div>

<div style="clear: both"></div>

---
# 스프링 클라우드 설정 서버

<div style="float:left; width: 40%; font-size: 14pt">

* 설정 매핑
  * http://localhost:8888/config-client/prod
    * 리포지토리/config-client-prod.properties
    * 리포지토리/config-client.properties

* 예
```
# 리포지토리 파일: config-client.properties
config.client.title=Config Client Title

# 리포지토리 파일: config-client-prod.properties
server.port=9999

```

</div>

<div style="float:left; width:59%; margin-left: 1%; font-size: 17pt">

<img src="config-server-02.png" />

</div>

<div style="clear: both"></div>

---
# 스프링 클라우드 설정 클라이언트

* spring-cloud-starter-config 모듈
* bootstrap.yml(properties 파일 설정)

```
spring:
  application:
    name: config-client # http://설정서버/config-client/프로필
  cloud:
    config:
      uri: http://localhost:8888 # 설정 서버
```

* @Value 애노테이션으로 설정 사용



---
# 설정 변경 반영(단일 노드)

단일 노드 변경
* 변경 대상 빈에 @RefreshScope 적용
* actuator의 /refresh 종단점 사용
  * 예: http POST http://localhost:8080/actuator/refresh

```java
@RestController
@RefreshScope
static class TitleController {
    private String title;

    @Autowired
    public TitleController(@Value("${config.client.title}") String title) {
        this.title = title;
    }

    @GetMapping("/title")
    public String title() {
        return title;
    }
```


---
# 설정 변경 반영(버스 이용)

* RabbitMQ나 Kafka를 이벤트 버스로 사용


