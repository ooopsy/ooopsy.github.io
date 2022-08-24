---
layout: post
title:  "Keycloak EKS Ingress 적용"
date:   2022-08-23 22:49:20 +0900
categories: Keycloak
tags: SSO 통합인증 Keycloak 키클락 EKS Ingress
---

참조: [https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/alb-ingress.html](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/alb-ingress.html)  

## 사전 준비 사항 
1. eksctl 설치 
```sh
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl
```

## subnet tag 추가  
Keycloak을 외부에서 사용하기에 Eks의 worker node는 사용하는  Subnet은 Public접근 허용해야 하며   
ALB를 생성할수 있게  Subnet은 AWS에 정의한 Tag가 추가돼어야 한다. 
```
kubernetes.io/role/elb
```  
> ![tag subnet!](/assets/images/ingress/subget_tag.png "tag subnet")  

## IAM OIDC 자격 증명 공급자  

<pre><code>
eksctl utils associate-iam-oidc-provider --cluster <mark>MyEks</mark> --approve
</code></pre>

## AWS Load Balancer Controller 추가 기능 설치

AWS Load Balancer Controller의 IAM 정책을 다운로드  
```sh
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.3/docs/install/iam_policy.json
```
다운로드한 정책 JSON 파일 적용 
```sh
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document iam_policy.json
```  
클러스터의 OIDC 공급자 URL을 확인
<pre><code>
aws eks describe-cluster --name <mark>MyEks</mark> --query "cluster.identity.oidc.issuer" --output text
</code></pre>
<br>
출력한 메세지 /id/뒤  <mark>EXAMPLED539D4633E53DE1B71EXAMPLE</mark>를 복사한다.
<pre><code>
oidc.eks.ap-northeast-2.amazonaws.com/id/<mark>EXAMPLED539D4633E53DE1B71EXAMPLE</mark>
</code></pre>
<br>
<mark style='background-color: orange' >1111122223333</mark>을 자신의  VPC ID로 바꿔주고 노락색으로 표시한부분 위에서 복사한 내용을 붙여준다.  
<pre><code>
cat >load-balancer-role-trust-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::<mark style='background-color: orange'>111122223333</mark>:oidc-provider/oidc.eks.ap-northeast-2.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "oidc.eks.ap-northeast-2.amazonaws.com/id/<mark>EXAMPLED539D4633E53DE1B71EXAMPLE</mark>:aud": "sts.amazonaws.com",
                    "oidc.eks.ap-northeast-2.amazonaws.com/id/<mark>EXAMPLED539D4633E53DE1B71EXAMPLE</mark>:sub": "system:serviceaccount:kube-system:aws-load-balancer-controller"
                }
            }
        }
    ]
}
EOF
</code></pre>
<br>
load-balancer-role-trust-policy.json 파일을 통해 IAM 역할을 생성한다.
<pre><code>
aws iam create-role \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --assume-role-policy-document load-balancer-role-trust-policy.json
</code></pre>
<br>
<mark style='background-color: orange' >1111122223333</mark>을 자신의  VPC ID로 바꿔준다. IAM 약할과 적책을 연결해준다.
<pre><code>
aws iam attach-role-policy \
  --policy-arn arn:aws:iam::<mark style='background-color: orange'>111122223333</mark>:policy/AWSLoadBalancerControllerIAMPolicy  \
  --role-name AmazonEKSLoadBalancerControllerRole
</code></pre>

<mark style='background-color: orange'>1111122223333</mark>을 자신의  VPC ID로 바꿔준다. loadbalancer yaml 파일을 생성한다  
<pre><code>
cat >aws-load-balancer-controller-service-account.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/name: aws-load-balancer-controller
  name: aws-load-balancer-controller
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<mark style='background-color: orange'>111122223333</mark>:role/AmazonEKSLoadBalancerControllerRole
EOF
</code></pre>
<br>
적용하여 Eks에 service account을 생성해 준다.
<pre><code>
kubectl apply -f aws-load-balancer-controller-service-account.yaml
</code></pre>
<br>

## Helm을 이용하여 AWS Load Balancer Controller 배포
eks-charts 리포지토리를 추가
```sh
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

aws ecr에 있는 이미지를 통해 ALB Controller를 배포 한다 
<pre><code>
helm install aws-load-balancer-controller eks/aws-load-balancer-controller   -n kube-system \
--set clusterName=<mark>MyEks</mark> \
--set serviceAccount.create=false \
--set serviceAccount.name=aws-load-balancer-controller \
--set image.repository=602401143452.dkr.ecr.ap-northeast-2.amazonaws.com/amazon/aws-load-balancer-controller
</code></pre>

## Keycloak Ingress를 적용한다 
https로 접근해야 하기에 alb에 사용할 인증서를  ACM에서 미리 만들어 줘야 한다.  
소스: [https://github.com/ooopsy/MyEks.git](https://github.com/ooopsy/MyEks.git)
<pre><code>
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-northeast-2:<mark>1111122223333</mark>:certificate/<mark>aaaaaaab-bbbb-4ae8-90e6-447d9c880523</mark>
    alb.ingress.kubernetes.io/ssl-redirect: '443'
spec:
spec:
  rules:
  - host: sso.sad-waterdeer.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-keycloak-http
            port:
              number: 80
</code></pre>
정상 적용 됐으면  ALB 생성됀것을 확인 할 수가 있다.  

> ![tag subnet!](/assets/images/ingress/confirm_alb.png "tag subnet")  























