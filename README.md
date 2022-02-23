# eks-cluster-dev-systems

### NGINX Ingress Controller Load Blancing
NGINX Ingress Controller를 사용해도 외부에서 오는 트래픽을 적절히 분배해 줄 외부 로드밴러서는 필요하다. AWS 로드 밸런서는 Classic ELB, ALB, NLB가 있는데 ALB는 gRPC를 지원하지 않는다. 
gRPC를 지원하려면 Classic ELB를 TCP 모드로 사용하거나 NLB를 사용해야 한다. Classic ELB는 동시에 많은 연결을 처리하려면 웜업이 필요한 단점이 있어  NLB 사용이 적절하다. 
   
Traffic 흐름   
  NLB -> NodePort -> NGINX Ingress Controller -> 내부 서비스 
  
## Nginx Ingress Controller 설치

### Mginx-ingress controller download
AWS에서 Type=LoadBlancer의 Service 뒤에 NGINX Ingress Controller를 노출하기 위해서 Network Load blancer(NLB)를 사용한다.    

#### Network Load Blancer (NLB)
```shell
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.1/deploy/static/provider/aws/deploy.yaml
```
#### TLS Termination AWS Load Balancer(NLB)
디폴트로서, TLS는 Ingress Controller 내에서 terminate 된다. 그러나 Load Balancer내에서도 TLS를 terminate도 가능하다.  
아래 내용은 NLB를 사용해서 AWS 상에서 어떻게 하는지 설명한다.   
1. Download the deploy.yaml template
```shell
   wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.1/deploy/static/provider/aws/nlb-with-tls-termination/deploy.yaml
```
2. 파일을 열어서 Kubernetes cluster에서 사용하도록 VPC CIDR을 수정한다. 
```shell
   proxy-real-ip-cidr: xxx.xxx.xxx/xx
```
3. AWS Certificate Manager (ACM) ID를 수정한다.
```shell
   arn:aws:acm:ap-northeast-2:XXXXXX:certificate/xxxxxx-xxxxxxx-xxxxxxx-xxxxxxx
```
4. kubernetes cluster에 적용한다.
```shell
   kubectl apply -f deploy.yaml
```

#### NLB IDLE TIMEOUTS
TCP flows의 Idle Timeout은 350초이며 수정할 수 없다.  
이러한 이유로 keepalive_timeout 값이 예상대로 작동하도록 350초 미만으로 구성되었는지 확인해야 한다. 
디폴트로 NGINX의 keepalive_timeout은 75초이다. 

### 설치 확인 
기본적으로 설치가 되면 ingress-nginx 네임스페이스 안에 ingress controller와 관련된 객체들이 설치된다. 
```shell
#> kubectl get svc --namespace=ingress-nginx
NAME                                TYPE       CLUSTER-IP        EXTERNAL-IP     PORT(S)                       ACE
ingress-nginx-controller            NodePort   10.103.10.114     <none>          80:32289/TCP,443:30975/TCP    102s
ingress-nginx-controller-admission  ClusterIP  10.99.113.142     <none>          443/TCP                       102s
```
Ingress NGINX Controller가 NodePort 타입으로 노출되었고 NodePort는 32289로 정해졌다.  노드 포트는 ingress를 통해 노툴되는 서비스기 해당 포트로 노출된다. 

## Ingress 생성 
Ingress Controller는 Ingress 규칙을 관리하기 위한 서버이므로 실제 Ingress 규칙을 생성해야 한다.  Ingress 규칙은 Layer 7 라우팅을 의미한다. 
    
Ingress 규칙은 경로로 들어온 요청을 서비스 객체에 연결하는 규칙을 설정한다.   
즉 도메인에서 뒤에 붙은 경로로 들어온 요청을 어떤 서비스 객체에 연결할 것인지에 대한 부분을 정의하는 규칙이다.
파일명 : ingress_rule.yaml
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
   name: ingress-systems
   annotaions:
      nginx.ingress.kubernetes.io/rewrite-target: /
spec:
   rule:
   - host: dev.bithumbsumsystems.com
     http:
        paths:
        - path: /auth
          backend:
             serviceName: authService
             servicePort: 80
        - path: /member
          backend:
             serviceName: memberService
             servicePort: 80
```
### Ingress 배포 
Ingress 배포시 Ingress Controller가 설치된 동일한 네임스페이스를 사용해야 한다. 
```shell
  root@minsoub:~#> kubectl apply -f ingress_rule.yaml --namespace=ingress-nginx
  ingress.extensions/ingress-systems created
```
정상적으로 배포 되었는지 확인한다. 배포가 완료되기 전까지는 Address 값이 노출되지 않는다.  노출될 때까지 잠시 기다리다 다시 수행한다.   
```shell
  root@minsoub:~#> kubectl get ing --namespace=ingress-nginx
  NAME             HOSTS                         ADDRESS        PORTS    ACE
  ingress-systems  dev.bithumbsumsystems.com     172.27.0.111   80       58s
```

## Service 생성 
ingress 규칙이 생성되면 ingress 규칙에 맞게 동작을 할 서비스를 생성해야 한다. 위에서 정의한 ingress_rule.yaml 파일에서 서비스 객체 2개를 정의했다. 
서비스를 생성하기 전에 서비스에 노출될 POD들이 이미 배포되어 있어야 한다. POD의 배포는 Deployment를 사용해서 배포되어야 한다. 
deployment가 되었다면 해당 deployment를 위에서 생성한 서비스로 노출시킨다. 
서비스 노출시 각 서비스는 Nodeport 형태로 노출시켜야 한다. 
```shell
  root@minsoub:~#> kubectl expose deployment auth-dev --name=authService --port=80 --type=NodePort --namespace=ingress-nginx
  service/authService exposed
```
다른 서비스도 같은 방법으로 노출시킨다.
```shell
  root@minsoub:~#> kubectl expose deployment member-dev --name=memberService --port=80 -type=NodePort --namespace=ingress-nginx
```

모두 다 정상적으로 배포되었는지 아래와 같이 확인한다.
```shell
  root@minsoub:~#> kubectl get all --namespace=ingress-nginx
```

## 결과 확인
ingress 규칙이 제대로 동작하는지 확인해보려면 도메인의 IP를 호스트파일에 등록해야 한다. NodePort 타입이므로, 클러스터 내 아무 VM의 IP를 등록해도 된다. 
master노드의 IP를 도메인으로 등록해본다.
$vi /etc/hosts
파일 끝에 xxx.xxx.xxx.xxx dev.bithumbsystems.com

