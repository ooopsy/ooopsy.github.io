---
layout: post
title:  "SpringSecurity5.7 Keycloak 연동"
date:   2022-10-09 22:49:20 +0900
categories: Keycloak
tags: SpringSecurity 5.7 Keycloak 
---

SpringSecurity 5.7 버전부터 OAuth2.0 및 SAML2.0에 대한 지원이 추가되면서   
Keycloak 홈페이지에서도 기존에 제공하는 Springboot Keycloak Adapter보다    
SpringSecurity 5.7을 사용하는 것을 추천하고 있다.   

이 문서는 SpringSecurity5.7 기반이므로 이전 버전은   
Keycloak Adapter를 사용하는 이하 링크를 참조하면 될거 같다.   
[Vue,SpringBoot 프로젝트 Keycloak 연동](https://www.sad-waterdeer.com/keycloak/2022/09/11/Vue,SpringBoot-%ED%94%84%EB%A1%9C%EC%A0%9D%ED%8A%B8-Keycloak-%EC%97%B0%EB%8F%99.html)



## Maven dependency 

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.7.4</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>
	<groupId>com.example</groupId>
	<artifactId>demo</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>demo</name>
	<description>Demo project for Spring Boot</description>
	<properties>
		<java.version>11</java.version>
		<spring-security.version>5.7.3</spring-security.version>
	</properties>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-security</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		
		<dependency>
		    <groupId>org.springframework.boot</groupId>
		    <artifactId>spring-boot-starter-thymeleaf</artifactId>
		</dependency>
		
		<dependency>
	      <groupId>org.springframework.security</groupId>
	      <artifactId>spring-security-oauth2-client</artifactId>
	    </dependency>
		
		<dependency>
	      <groupId>org.springframework.security</groupId>
	      <artifactId>spring-security-oauth2-jose</artifactId>
	    </dependency>
	    
	     <dependency>
	      <groupId>org.springframework.security</groupId>
	      <artifactId>spring-security-oauth2-resource-server</artifactId>
      	</dependency>
		
		<dependency>
		  <groupId>org.springframework.boot</groupId>
		  <artifactId>spring-boot-starter-test</artifactId>
		  <scope>test</scope>
		</dependency>
		<dependency>
		  <groupId>org.springframework.security</groupId>
		  <artifactId>spring-security-test</artifactId>
		  <scope>test</scope>
		</dependency>
	</dependencies>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>

</project>

```



## Two Tier 구조 
진행 해봤던  Two Tier 구조 프로젝트들은 사용자에 대한 관리 기능을 전부 가지고 있고   
SpringSecurity를 이용하여 로그인 기능을 구현하고  서버 메모리 또는  Redis를 통해 웨서버 새션을 관리하여  
Keycloack은 단순 통합용으로 Thrid Party(간편 로그인)으로 사용했다.   
> ![login_page.png](/assets/images/security/login_page.png "login_page.png")   

### Security 설정  

SecurityConfig  설정 
``` java
@Configuration
public class SecurityConfig{

	@Bean
	public SecurityFilterChain filterChain(HttpSecurity http) throws Exception{
		http
			.formLogin() // 로그인 페이지 추가 
			.and()
			.oauth2Login() // OAuth2.0 로그인 기능 활성화 
			.and()
			.authorizeHttpRequests()
			.anyRequest()
			.authenticated();
		
		return http.build();
	}
	
```

Keycloak 관련 정보 application.yml에 추가 
``` yaml
spring:
  thymeleaf:
    cache: false
  security:
    oauth2:
      client:
        registration:
          keycloak:  
            client-id: springsecurity
            client-secret: exbrjy60WcnbTh4Slwk9FZ4NHlTnS13S
            authorization-grant-type: authorization_code
            redirect-uri: http://localhost:8081/login/oauth2/code/keycloak 
        provider:
          keycloak: 
            authorization-uri: http://localhost:8080/realms/MY/protocol/openid-connect/auth
            token-uri: http://localhost:8080/realms/MY/protocol/openid-connect/token
            user-info-uri: http://localhost:8080/realms/MY/protocol/openid-connect/userinfo
            jwk-set-uri: http://localhost:8080/realms/MY/protocol/openid-connect/certs
            user-name-attribute: sub

```









 