# 엣지 서비스의 보안

- 엣지 서비스는 보안 같은 공통 횡단 관심사를 처리할 수 있는 적임자
    ```text
    요청 -> [인증 필터] -> 리소스 반환
    ```
- 어떤 클라이언트에서 접근했는지 확인이 필요함
    
    사용자의 역할 별로 권한을 부여했다면 클라이언트 별로도 권한을 조정할 수 있어야함.
    
    ex) 페이스북 공식 모바일 웹과 서드파티 앱들의 권한 차등 부여

## OAuth 기본

- Open Authorization
- 토큰 기반의 인가`authorization` 프로토콜
- 인증`authentication`은 OAuth와는 별개로 다른 곳에서 처리되어야 함
- 인증이 처리된 후에 OAuth가 제어권을 넘겨받음
- OAuth는 데이터의 소유자를 대신하여 서비스에 접근할 수 있는 보안 위임 접근`secure delegated access`을 클라이언트에게 제공


## OAuth 용어 정리

### 자원 서버 Resource Server
- 보호되는 자원을 호스팅하는 서버
- access token을 포함하는 보호받는 요청에 대해서 응답할 수 있음
- ex) 페이스북 사용자의 프로파일, 친구관계 그래프 등 자원에 대한 접근 통제를 지원하는 페이스북 API

### 자원 소유자 Resource Owner
- 자신이 소유한 정보에 대한 접근 권한을 부여하는 최종 사용자
- ex) 페이스북의 일반 유저들

### 클라이언트 Client
- 자원서버에 있는 보호되는 자원들에 대한 접근이 필요한 애플리케이션
- 보통 모바일 앱, 또는 웹서비스, 최근에는 시계 혹은 기타 IOT 기기들도 가능
- 사용자가 없는 (사용자 컨텍스트가 없는) 앱들도 가능. 예컨데 배치 프로그램 등
- ex) 페이스북 앱을 포함한 여러거지 서드파티 앱들

### 인가서버 Authorization Server
- 인증 과정을 통과하고 인가(허가)를 획득한 자원소유자의 클라이언트에게 access token을 발부
- 페이스북에서는 인가서버와 자원서버가 같음, 그러나 항상 같을 필요는 없음


### [추가] 인증서버
- 일반적으로 아이디와 패스워드를 가지고 사용자를 인증해주는 서버
- ex) 페이스북 로그인 서버

## 페이스북으로 보는 OAuth의 절차

1. 서드파티 웹사이트(클라이언트)에서 페이스북 로그인으로 리다이렉트
    - 브라우저 주소창에는 HTTPS를 통해서 현재 사이트가 인증된 사이트임을 표시
2. 사용자가 아이디, 패스워드 입력을 통해 인증받음
3. 인증 후에 일부 권한을 클라이언트에게 부여하는 데 동의하느냐는 알림 표시되면 동의
4. 페이스북은 요청에 인가코드를 추가하여 다시 서드파티 웹사이트로 리다이렉트
5. 서드파티 서버에서 페이스북 인가서버를 통해서 클라이언트 아이디, 시크릿, 인가코드를 가지고 access token을 얻음
6. 클라이언트는 접근토큰을 획득하고 데이터베이스에 저장한 후 사용자를 대신해서 페이스북 자원 서버에 원하는 자원 요청

이 전체 흐름을 인가 코드 부여`authorization code grant`라고 한다. 이 흐름대로 하면 접근 코드의 유출을 막을 수 있따.

## 인증을 담당하는 Spring Security




## 메모
유레카 먼저 키고

그리팅 서비스 먼저 키고