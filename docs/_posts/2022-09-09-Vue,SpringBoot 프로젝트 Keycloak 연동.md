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


### Vue 
#### Keycloak client 등록  
Keycloak 콘설에서  Vue프로젝트를 Client로 등록한다.  
Vue 프로젝트는 단순하게 화면표시만 하고 token을 검증하거나 token에 있는 정보로  
데이터를 조회하지 않기에 Access Type을  public으로 해야한다.   
또한 Vue와 같은 frontend는 사용자에게 완전히 노출돼기 때문에 client_secret을 가지면 안됀다.   
> ![client_setting.png!](/assets/images/vue/client_setting.png "client_setting.png")

#### Vue 소스 설명  
keycloak javascript adpater가 프로젝트에 추가 돼 있어야 한다  
[https://www.npmjs.com/package/keycloak-js](https://www.npmjs.com/package/keycloak-js)

Keycloak 인증서버 URL 및 등록한 Client의 Id 및 Realm 정보로  keycloak 객체를 초기화 한다 .
``` javascript 
//https://github.com/ooopsy/kanban-frontend/blob/master/src/main.js
var keycloak = new Keycloak({ 
    url: 'https://sso.sad-waterdeer.com/auth', 
    realm: 'TEST', 
    clientId: 'kanban_local' 
});
```

프로젝트 특성상  로그인 안한 사용자는 사용할 수가 없기에  
onLoad: 'login-required'로  설정 하여  사용가 로그인한 상태에만   
화면을 표시하게 설정했다 (app.mount("#app")).

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
``` javascript 
  axios.interceptors.request.use(
    async config => {
        await keycloak.updateToken(60) 
        config.headers.Authorization = `Bearer ${keycloak['token']}`
      return config;
    }
  );
```
backend 서버로 요청하여  401 또는 403을 받았는다는 것은  token이 더 이상 유효 하지  
않거나 불법 URL을 방문 했다는것으로 바로 logout시켜서 다시 인증게 한다.  

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


To Be continued...
 ****