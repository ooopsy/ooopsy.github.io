---
layout: post
title:  "Keycloak EKS 배포(Helm)"
date:   2022-08-13 22:49:20 +0900
categories: Keycloak
tags: SSO 통합인증 Keycloak 키클락 EKS Helm
---

Keycloak 19부터 Wildfy를 더 이상 사용하지 않고 Quarkus  
사용하지만 사용하는 오픈소스 Helm Chart는  17 까지만 지원하고 있어  
WildFy 버전 배포 진행. 
<br>
Helm Chart: [https://github.com/codecentric/helm-charts/tree/master/charts/keycloak](https://github.com/codecentric/helm-charts/tree/master/charts/keycloak/)


---

## 사전 준비 (Mac)
1. kubectl 설치  
``` sh
brew install kubectl
```
2. helm 설치    
``` sh
brew install helm
```
3. aws cli 설치 
``` sh
https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
```
4. aws iam authenticator 설치 
``` sh
brew install aws-iam-authenticator 
```

## AWS 관리자 계정 생성  
IAM -> Users -> Add users   Access Key도 생성 해준다.
> ![add user!](/assets/images/eks/add_user.png "add user")  

<br>EKS 사용시 여러가지 권한이 필요하여 관리자 권한을 부여한다.   
아님 중간중간 권한이 부족하다고 오류난다.  
이하 모든 작업들은 새로 생성한 관리자 권한 계정으로 진행한다.  

> ![add user admin!](/assets/images/eks/add_user_admin.png "add user admin")  

<br> Access key ID, Secret access key를 저장해둔다.  
> ![add user secret!](/assets/images/eks/add_user_secret.png "add user secret")  

## EKS용  Cluster Service Role 생성
EKS용 Cluster Service Role을 생성 해 준다.   가이드: [create-service-role](https://docs.aws.amazon.com/eks/latest/userguide/service_IAM_role.html#create-service-role)  
> ![add eks role](/assets/images/eks/add_eks_role.png "add eks role")  

## EKS Worker Node Role 생성
EKS Worker Node Role을  생성해 준다.   가이드: [Amazon EKS node IAM role](https://docs.aws.amazon.com/eks/latest/userguide/create-node-role.html#create-worker-node-role)  
> ![add eks role](/assets/images/eks/workerNode-role.png "add eks role")  


## VPC 네트워크 설정 
### Internet gateway 생성  
> ![internetgateway](/assets/images/eks/internetgateway.png "internet gateway")  

### Private/Public Route table 생성  

private route 생성 
> ![private route](/assets/images/eks/private-route.png "private route")    

public route 생성 
> ![public route](/assets/images/eks/public-route.png "public route")  

public route에 방금 생성한  Internet GateWay 추가 
> ![edit public route](/assets/images/eks/edit-public-route.png "edit public route")  

> ![attach gateway](/assets/images/eks/attach-gateway.png "attach gateway")  

### Private/Public 용  Subnet 생성  

 예: VPC  IPv4  CIDR가 171.30.0.0/16  일때 

| route table | subnet name | Ipv4 CIDR | Availability Zone |
| --- | --- | --- | --- |
| MyPrivateRouteTable | MyPrivateSubnet-01 | 171.30.1.0/24 | ap-northear-2a |
| MyPrivateRouteTable | MyPrivateSubnet-02 | 171.30.2.0/24 | ap-northear-2b |
| MyPublicRouteTable | MyPublicSubnet-01 | 171.30.3.0/24 | ap-northear-2a |
| MyPublicRouteTable | MyPublicSubnet-04 | 171.30.4.0/24 | ap-northear-2b |

#### private subnet 생성  
> ![private subnet](/assets/images/eks/private-subnet.png "private subnet")  

#### public subnet 생성 
> ![public subnet](/assets/images/eks/public-subnet.png "public subnet")  


나중에 workNode 생성할때 public subnet에  EC2를 생성하기에  
자동 아이피 할당을 꼭 enable 해야 한다. 아님 오류 발생.  

> ![enable auto ip](/assets/images/eks/enable-auto-ip.png "enable auto ip")  

#### subnet-route 매핑 
> ![mapping route](/assets/images/eks/mapping-route.png "mapping route")  


## Security Group 생성  

 EKS 인터넷 접근가능하게 해야 하고  DB는 VPC내에서만 접근 가능하게  
 Security Group을 생성한다.

### EKS용 퍼블릭 Security Group 생성  
참고: [Amazon EKS 보안 그룹 요구 사항 및 고려 사항](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/sec-group-reqs.html)
> ![eks public security group](/assets/images/eks/add_eks_security_group.png "eks public security group")  


### RDS 용 Security Group 생성  
EKS 노드는  public subnet에 배치돼고 rds는 private subnet에 배치돼기 때문에 
EKS 노드 RDS에 접근할수 있게 public subnet 아이피 3306포트만 뚫어 준다.  

> ![rds security group](/assets/images/eks/rds-security-group.png "rds security group")  

## RDS 생성  
### Subnet Group 생성 
RDS 생성 때 필요한 SubnetGroup을 생성해 준다  
앞서 ap-northear-2a, ap-northear-2b 에 Private subnet group  생성 하기 때문에 선택해 준다.  

> ![subnet group](/assets/images/eks/subnet-group.png "subnet group")  

### Mysql Instance 생성 
DB 관라자 계정 및 스팩 선택 후  
생성할때 앞서 만든 Security Group 및 Subnet Group을 선택해준다  

> ![create rds](/assets/images/eks/create-rds.png "create rds")  

### Keycloak Tablespace 생성  
RDS private subnet에 생성했기에 터널링이 필요하다.  
public subnet에 임시로 EC2 인스턴스 생성후 pem key 생성.  
pem key를 이용하여 터널링 방식으로  DB에 접근하여  schema 생성 

> ![터널링](/assets/images/eks/tunnul.png "터널링")  
<br>
> ![터널링](/assets/images/eks/create-schema.png "터널링")  


## EKS 생성  
앞서 생성한 Cluster Service Role를 선택해 준다.  
> ![chosee-eks-cluster-role.png](/assets/images/eks/chosee-eks-cluster-role.png "chosee-eks-cluster-role")  

앞서 생성한  public subnet을 선택해 준다.  
> ![eks-select-public-subnet.png](/assets/images/eks/eks-select-public-subnet.png "eks-select-public-subnet")  

앞서 생성한  public security group을 선택해 준다.  
> ![eks-select-public-security-group](/assets/images/eks/eks-select-public-security-group.png "eks-select-public-security-group")   

### Node group을 생성한다. 
앞서 생성한 Worker Node Role을 선택해 준다. 
> ![select-node-group-role](/assets/images/eks/select-node-group-role.png "select-node-group-role")   

앞서 생성한  public subnet을 선택해 준다.  
> ![node-select-public-subnet](/assets/images/eks/node-select-public-subnet.png "node-select-public-subnet")   

## Keycloak Helm 배포  
### aws cli 계정 설정 
aws configure 명령어 실행후  
앞서 저장한 관리자 계정 Access key ID, Secret access key를 입력한다.
```sh 
aws configure
```

```sh 
aws sts get-caller-identity #적용됐는지 확인 
aws eks update-kubeconfig --name MyEks --region ap-northeast-2  # .kube/config  파일 업데이트 해줌
```

### DB 연결 정보 설정 
DB 연결정보, 포트, 스크마, 계정명을 configmap에 저장한다.
```sh
kubectl apply -f configmap.yml 
```
``` yaml
apiVersion: v1
data:
  db.host: RDS앤드포인트 
  db.port: "3306"
  db.schema: keycloak
  db.user: RDS계정아이디 
kind: ConfigMap
metadata:
  annotations:
  creationTimestamp: "2022-08-15T01:09:57Z"
  name: my-config
  namespace: default
  resourceVersion: "13667"
  uid: daf633e8-5b82-4a1a-b09d-b8b11a8aa025
```
DB  계정 비밀번호 secret에 저장한다.
```sh 
kubectl create secret generic my-secret --from-literal=db.password=비밀번호
```


### Helm 생성
Helm 생성  
templates 하위에 있는 파일들 아직 안쓰니까 전부 날린다.  
Keycloak 오픈소스 Helm Chart 저장소 추가  
```sh
helm repo add codecentric https://codecentric.github.io/helm-charts # add open source helm chart repository
```
```sh 
helm create my-keycloak
cd my-keycloak 
vi Chart.yaml 
```
<br>
Chart.yaml에 keycloak dependency 추가 
``` yaml 
apiVersion: v2
name: my-keycloak
description: A Helm chart for Kubernetes
type: application
version: 0.1.0
appVersion: "1.16.0"

dependencies:
  - name: keycloak
    version: 18.3.0
    repository: "@codecentric"
```
values.yaml 정보 입력 service.type  LoadBalancer로 지정
``` yaml 
keycloak:
  fullname: my-keycloak
  cli:
    enabled: false
  ingress:
    enabled: false
  postgresql:
    enabled: false
  service:
    type: LoadBalancer
    port: 80
  extraEnv: |
    - name: DB_VENDOR
      value: mysql
    - name: DB_ADDR
      valueFrom:
        configMapKeyRef:
          name: my-config           
          key: db.host
    - name: DB_PORT
      valueFrom:
        configMapKeyRef:
          name: my-config           
          key: db.port
    - name: DB_DATABASE
      valueFrom:
        configMapKeyRef:
          name: my-config           
          key: db.schema
    - name: DB_USER
      valueFrom:
        configMapKeyRef:
          name: my-config           
          key: db.user
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: my-secret
          key: db.password
          optional: false 
```


dependency 빌드 
```sh 
helm dependency build
```
### Helm install
```sh
helm install my-keycloak my-keycloak
```

## 도메인 설정 
route 53 A record에 EKS 생성해준 ELB Endpoint 매핑 설정해 준다.
> ![route53-elb](/assets/images/eks/route53-elb.png "route53-elb")   

### ELB 용 ACM 생성  
ACM 에서 인증서 추가 
> ![add-acm](/assets/images/eks/add-acm.png "add-acm")   

route 53애 인증서 Validation CNAME 레코드 추가 
> ![cname-value](/assets/images/eks/cname-value.png "cname-value")   
<br>
<br>
> ![add-c-record](/assets/images/eks/add-c-record.png "add-c-record")   

ELB에 인증서를 걸어 준다   
> ![add-1](/assets/images/eks/add-1.png "add-1")   
> ![add-2](/assets/images/eks/add-2.png "add-2")   
> ![add-3](/assets/images/eks/add-3.png "add-3")    


## 서버 초기화 
Keycloak master 사용자 초기화 
```sh
kubectl exec -it my-keycloak-0 -- /bin/bash # POD 진입
/opt/jboss/keycloak/bin/add-user-keycloak.sh -r master -u oopsy1988 -p 비밀번호   # master 사용자 추가 
/opt/jboss/keycloak/bin/jboss-cli.sh --connect command=:reload  # 서버 재시작  
```  
























