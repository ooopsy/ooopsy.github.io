---
layout: post
title:  "Vue,SpringBoot 프로젝트 Keycloak 연동"
date:   2022-09-11 22:49:20 +0900
categories: Keycloak
tags: Vue SpringBoot Keycloak 
---

## 소스 및 데모 사이트   

데모 사이트: [https://kanban.sad-waterdeer.com/](https://kanban.sad-waterdeer.com/){:target="_blank"}  
Vue 소스: [https://github.com/ooopsy/kanban-frontend](https://github.com/ooopsy/kanban-frontend){:target="_blank"}   
SpringBoot 소스: [https://github.com/ooopsy/kanban-backend](https://github.com/ooopsy/kanban-backend){:target="_blank"}

Keycloak의  Oauth2.0 프로토컬을 이용하여  Kanban 샘플 프로젝트를 만들었다  
Oauth 2.0 은 인터넷에 잘 풀어서 설명하는 글이 많아 포로토콜에 대한 설명은 생략한다.  

****  

### Vue 
#### Keycloak client 등록  
Keycloak 콘솔에서  Vue프로젝트 URL를 Client로 등록한다.  
Vue 프로젝트는 단순하게 화면표시만 하고 token을 검증하거나 token에 있는 정보로  
데이터를 조회하지 않기에 Access Type을  public으로 해야한다.   
또한 Vue와 같은 frontend는 사용자에게 완전히 노출돼기 때문에 client_secret을 가지면 안됀다.   
> ![client_setting.png!](/assets/images/vue/client_setting.png "client_setting.png")

#### Vue 소스 설명  
keycloak javascript adpater가 프로젝트에 추가 돼 있어야 한다  
[https://www.npmjs.com/package/keycloak-js](https://www.npmjs.com/package/keycloak-js)

Keycloak 인증서버 URL 및 등록한 Client의 Id 및 Realm 정보로  keycloak 객체를 초기화 한다
``` javascript 
//https://github.com/ooopsy/kanban-frontend/blob/master/src/main.js
var keycloak = new Keycloak({ 
    url: 'https://sso.sad-waterdeer.com/auth', 
    realm: 'TEST', 
    clientId: 'kanban_local' 
});
```
<br>
<br>
<br>
프로젝트 특성상  로그인 안한 사용자는 사용할 수가 없기에  
onLoad: 'login-required'로  설정 하여  사용가 로그인한 상태에만   
화면을 표시하게 설정 (app.mount("#app"))  
.then() 블록에 진입시  aurhozation code flow 전부 진행 완료하여  
token 발급 까지 완료한 상태다 그 과정이 궁금하다면 keycloak.js를 분석하면 됀다  

``` javascript
keycloak.init({
    flow:  'standard',
    onLoad: 'login-required',
    checkLoginIframe: false, 
}).then((auth) => {
    app.mount("#app");
})

```
<br>


aioxs 요청 발생전 사용자 인증이 완료한 상태 이기에  
keycloak 객체에 무조건 token이 담겨져 있을거다.  
backend 서버(Springboot)에 요청 할때  발급 받은 token을  해더에 적재 한다.
서버에 요청하기전에  refresh token으로 신규 token을 갱신한다(필요시)
파라미터 60은 token의 유효시간이 60초 이하 또는 이미 유효시간 지났을 때 token을 갱신하겠다는 뜻이다. 
``` javascript 
  axios.interceptors.request.use(
    async config => {
        await keycloak.updateToken(60) 
        config.headers.Authorization = `Bearer ${keycloak['token']}`
      return config;
    }
  );
```
<br>
<br>
backend 서버로 요청하여  401 또는 403을 받았는다는 것은  token이 더 이상 유효 하지  
않거나 불법 URL을 방문 했다는것으로 바로 logout시켜서 다시 인증하게 한다.  

``` javascript 
  axios.interceptors.response.use((response) => {
    return response
  }, function (error) {
    if (error.response.status === 401 || error.response.status === 403) {
        VueCookies.remove("token", '/', getTopDomain());
        keycloak.logout()
    }
    return Promise.reject(error.response)
  });

```

****  

### SpringBoot
#### Springboot Client로 등록 
Springboot 토큰을 검증만 하기에  bearer-only로 등록한다. 
> ![springboot.png!](/assets/images/vue/springboot.png "springboot.png")
<br>
<br>

Credentials 탭에서 Generate Secret로 secret를 생성한다.
> ![credential.png!](/assets/images/vue/credential.png "credential.png")


#### dependency 추가 
pom.xml에 필요한 dependency를 추가한다  
keycloak adapter는  spring security에 dependency가 있어  spring security도 추가 해준다  
``` xml
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
  </dependency>

  <dependency>
    <groupId>org.keycloak</groupId>
    <artifactId>keycloak-spring-boot-starter</artifactId>
    <version>17.0.1</version>
  </dependency>
```
#### keycloak 관련 설정을 추가한다
resource:  client_id 입력  
bearer-only: true 로 설정한다 true로 설정하면 token이 유하지 않을때 401을 리턴한다
``` yml
keycloak:
  auth-server-url: https://sso.sad-waterdeer.com/auth 
  realm: TEST
  bearer-only: true
  resource: kanban-backend
  ssl-required: none
  credentials:
    secret: tOZGJnUHBJULRdCuAOZ5lmPAzBMqnQMK
```
#### 필요 소스 추가 
KeycloakSpringBootConfigResolver를 추가하여 application.yml로 keycloak 정보를 관리한다   
KeycloakWebSecurityConfigurerAdapter 내용을 확인하면 디폴트로  keycloak.json 파일을 찾는다  
json 파일은 개발, 운영 환경 profile별로 관리하기가 어렵다.(profile에 맞게 json파일을 찾는 나만의 KeycloakConfigResolver을 구현해야 함 )  

```java
package com.example.kanban.config;

import org.keycloak.adapters.springboot.KeycloakSpringBootConfigResolver;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class KeycloakConfig {

    @Bean
    public KeycloakSpringBootConfigResolver keycloakConfigResolver() {
        return new KeycloakSpringBootConfigResolver();
    }
}
```
<br>
<br>

Keycloak Spring security 관련 소스 추가  
- SessionAuthenticationStrategy
  - NullAuthenticatedSessionStrategy를 선언한다  token의 유효시간 체크만 의존하고 session을 관리할 필요가 없다.(token 시간을 짧게 가져가면 되기에)
- KeycloakAuthenticationProvider
  - spring security 관련해여 공부 해보면 기본적으로 AuthenticationProvider를 통해 인증기능을 구현한다
  - 최소 하나는 필요하여  Keycloak의 기본 Provider를 선언한다 
  - 인증 체계에 맞게 해당 Class를 상속하여 인증로직을 추가해야 한다  
- 상위인 KeycloakWebSecurityConfigurerAdapter 에서 token에 대한 유현성체크  filter 들을 대신 선언 해줬다 
  - 핵심 로직은 KeycloakAuthenticationProcessingFilter 에서 확인 할 수가 있다 
  - 실제 token 체크는 BearerTokenRequestAuthenticator.authenticate 에서 진행한다.
  - AdapterTokenVerifier에서  keycloak 메타 정보에 있는 public key로 token이 위조 됐는지를 확인한다  
  - AuthenticatedActionsHandler에서 추가로 keycloak에 요청하여 인가를 체크를 할 수도 있다. 인가는 굉장히 큰 주제로.....   

```java
@KeycloakConfiguration
public class SecurityConfig extends KeycloakWebSecurityConfigurerAdapter {

	
	@Override
	protected SessionAuthenticationStrategy sessionAuthenticationStrategy() {
		return new NullAuthenticatedSessionStrategy();
	}
	
	@Bean
	public KeycloakAuthenticationProvider getKeycloakAuthenticationProvider() {
		return new KeycloakAuthenticationProvider();
	}
	
	
	@Override
    protected void configure(HttpSecurity http) throws Exception {
        super.configure(http);
        http.cors().and().csrf().disable()
        	.authorizeHttpRequests()
        	.anyRequest().authenticated();	
    }
	
	@Bean
    public CorsConfigurationSource corsConfigurationSource() {
        final CorsConfiguration configuration = new CorsConfiguration();
        configuration.setAllowedOrigins( Arrays.asList("http://kanban.sad-waterdeer.com", 
        		"https://kanban.sad-waterdeer.com", 
        		"http://localhost:5173"));
        configuration.setAllowedMethods(Arrays.asList("HEAD",
                "GET", "POST", "PUT", "DELETE", "PATCH"));
        configuration.setAllowCredentials(true);
        configuration.setAllowedHeaders(Arrays.asList("Authorization", "Cache-Control", "Content-Type"));
        final UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", configuration);
        return source;
    }
	
}
```  

<br>
<br>

token 체크를 완료하면 token에 해당하는 정보들이 SpringSecurity의 Context(Threadlocal)에 저장 됀다.  
Service 또는 Controller에서 꺼내서 쓰면 됀다.  
```java
public class TokenUtil {
	public static AccessToken getAccessToken() {
		Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
		KeycloakAuthenticationToken token = (KeycloakAuthenticationToken) authentication;
		@SuppressWarnings("unchecked")
		KeycloakPrincipal<KeycloakSecurityContext> keycloakPrincipal = (KeycloakPrincipal<KeycloakSecurityContext>)token.getPrincipal();
        KeycloakSecurityContext context = keycloakPrincipal.getKeycloakSecurityContext();
		return context.getToken();
	}
}
```

 