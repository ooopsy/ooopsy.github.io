---
layout: post
title:  "CodeBuild pnpm 설정"
date:   2022-09-09 22:49:20 +0900
categories: AWS
tags: AWS CodeBuild pnpm 
---

github에 있는  keywind라는 Keycloak 테마가 pnpm을 통해 빌드 하기에 설정을 진행했다   
최종 목표는  keywind 소스를 빌드후  S3에 업로드 하는 것이다   
나중에  Keycloak  이미지를 빌드 할때 Dockerfile에서 S3에 올라간  
pnpm 빌드 결과물 ftl 파일을 다운로드해서 사용한다.

keywind Keycloak 테마:  [https://github.com/lukin/keywind/](https://github.com/lukin/keywind/)  

## keywind 소스를 다운로드후 codecommit에 올린다  
> ![source!](/assets/images/cb/source.png "source")  

***

## buildspec.yml 파일을 소스 루트에 추가한다 
buildspec 파일은 CodeBuild에서 디폴트로 사용하는 파일이다   
npm은 CodeBuild에 제공한  Environment Image에 기본으로 설치돼 있지만  
pnpm은 추가로 설치해야 한다  

``` yaml
version: 0.2
phases:
  install:
    commands:
      - npm install -g pnpm #pnpm 설치 
      - pnpm install #pnpm통해 dependency 설치 
  build:
    commands:
      - n latest  #최신 node 버전 사용 
      - pnpm build #소스 빌드 
artifacts:
  base-directory: theme #build 결과물만 s3에 업로드할꺼니까 결과물 풀더를 지정한다 
  files:
    - '**/*'   # theme 폴드 아래에 있는 모든 파일을 올린다.
```
***  

## CodeBuild build project를 만든다.
### 소스 지정 
> ![choose_source!](/assets/images/cb/choose_source.png "choose_source")  

### CodeBuild용 이미지 선택 
인터넷에서 찾은 글들은 nodejs 이미지를 선택하라고 했지만 AWS에서 여러 가지 이미지를 통합해서 
제공하는 걸로 변경했기에  standard를 선택해 주면 됀다.
> ![env!](/assets/images/cb/env.png "env")  

### buildspec 지정  
디폴트인 소스에 추가한 buildspec.yml를 사용한다.
> ![buildspec!](/assets/images/cb/buildspec.png "buildspec")  

### 빌드결과를 저장할 S3를 지정한다 
> ![s3!](/assets/images/cb/s3.png "s3")  

## 설정 완료 파일 확인.  
설정 완료 후 빌드하면  S3에 파일 올라간 것을 확인할 수 있다.  
> ![result!](/assets/images/cb/result.png "result")  

 ***