# Kubernetes Service

파드 집합에서 실행중인 애플리케이션을 노출시키는 추상화 방법이다.  

파드들은 동적으로 생성되고 제거되는데 어떻게 일부 파드 집합이 다른 파드 집합의 주소를 찾아서 연결을 할 수 있는가?  

# 서비스 리소스
서비스가 대상으로 하는 파드 집합은 일반적으로 셀렉터가 결정한다.  

프론트엔드는 백엔드를 신경쓰지 않아야 한다.  
백엔드를 구성하는 실제 파드는 변경될 수 있지만 프론트엔드는 그것을 인식할 필요가 없다.  

이를 위해 서비스 추상화를 이용하여 디커플링을 수행한다.  

# 서비스 정의
서비스는 파드와 비슷한 REST 오브젝트이다 (?)  

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```  

위와 같은 명세는 `my-service`라는 서비스 오브젝트를 생성하고, `app=MyApp` 레이블을 가진 파드의 TCP 9376 포트를 대상으로 한다.  
- 쿠버네티스는 이 서비스에 서비스 프록시(?)가 사용하는 IP 주소(Cluster IP)를 할당한다.  


```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app.kubernetes.io/name: proxy
spec:
  containers:
  - name: nginx
    image: nginx:11.14.2
    ports:
      - containerPort: 80
        name: http-web-svc
        
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app.kubernetes.io/name: proxy
  ports:
  - name: name-of-service-port
    protocol: TCP
    port: 80
    targetPort: http-web-svc
```
파드의 포트 정의에서 특정한 이름을 지정하여 서비스의 targetPort 속성에서 이 이름을 참조할 수 있다.  
- 이렇게 포트 이름을 사용하면 클라이언트를 변경하지 않고 백엔드 파드의 포트 번호를 변경할 수 있다.  

# 셀렉터가 없는 서비스  
서비스 구성시 셀렉터 대신 매칭되는 엔드포인트와 함께 사용하면 다른 종류의 백엔드도 추상화할 수 있다.
- 클러스터 외부에서 실행되는 것도 포함된다.  
- 프로덕션과 테스트 환경에서 사용하는 데이터베이스를 나누는 경우
- 한 서비스에서 다른 네임스페이스 또는 다른 클러스터의 서비스를 지정하려는 경우
  
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```
셀렉터 없이 서비스를 구성하면 엔드포인트 오브젝트가 자동으로 생성되지 않는다.  
따라서 아래와 같이 엔드포인트 오브젝트를 수동으로 추가하여 매핑할 수 있다.  

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: my-service
subsets:
  - addresses:
      - ip: 192.0.2.42
    ports:
      - port: 9376
```

셀렉터가 없는 서비스에 접근하면 셀렉터가 있는 것처럼 동작한다.  
- 위 경우 트래픽은 192.0.2.42:9376으로 라우팅된다.  


# 가상 IP와 서비스 프록시 ????
쿠버네티스 클러스터의 모든 노드는 kube-proxy(?)를 실행한다.    

## 라운드-로빈 DNS를 사용하지 않는 이유
왜 쿠버네티스가 인바운드 트레픽을 백엔드로 전달하기 위해 프록시에 의존하는가?  

그 이유는 아래와 같다. ?????
- 레코드 TTL을 고려하지 않고, 만료된 이름 검색 결과를 캐싱하는 DNS 고현에 대한 오래된 역사가 있다.
- 일부 앱은 DNS 검색을 한 번만 수행하고 결과를 무기한으로 캐시한다.
- 앱과 라이브러리가 적절히 재확인을 했다고 하더라도, DNS 레코드의 TTL이 낮거나 0이면 DNS에 부하가 높아 관리하기가 어려워 질 수 있다.   

# 멀티 포트 서비스
서비스는 둘 이상의 포트를 노출해야하는 경우도 지원한다. 
- 이 경우 모든 포트 이름을 명확하게 지정해야 한다.  

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 9376
    - name: https
      protocol: TCP
      port: 443
      targetPort: 9377
```


# 트래픽 정책
## 외부 트래픽 정책
`spec.externalTrafficPolicy`필드를 통해 외부 소스에서 오는 트래픽이 어떻게 라우트될지 제어할 수 있다.  
Cluster 또는 Local로 설정할 수 있다. 
- Cluster: 외부 트래픽을 준비 상태인 모든 엔드포인트로 라우팅
- Local: 준비 상태의 노드 로컬 엔드포인트로만 라우팅

## 내부 트래픽 정책
`spec.internalTrafficPolicy` 필드를 설정하여 내부 소스에서 오는 트래픽이 어떻게 라우트될지 제어할 수 있다.
Cluster 또는 Local로 설정할 수 있다. 
- Cluster: 내부 트래픽을 준비 상태인 모든 엔드포인트로 라우팅
- Local: 준비 상태의 노드 로컬 엔드포인트로만 라우팅

# 서비스 디스커버리 하기  
## 환경변수
파드가 노드에서 실행될 때, kubelet은 각 활성화된 서비스에 대해 환경 변수 세트를 추가한다.  
`{SVCNAME}_SERVICE_HOST` 및 `{SVCNAME}_SERVICE_PORT`환경 변수가 추가된다.  

## DNS
애드온을 사용하여 쿠버네티스 클러스터의 DNS 서비스를 설정할 수 있다.  

CoreDNS와 같은, 클러스터-인식 DNS 서버는 새로운 서비스를 위해 쿠버네티스 API를 감시하고 각각에 대한 DNS 레코드를 세팅한다.  
- 모든 파드는 DNS 이름으로 서비스를 자동으로 확인할 수 있어야 한다.   

`my-ns` 네임스페이스에 `my-service`라는 서비스가 있는 경우 컨트롤 플레인과 DNS 서비스가 함께 작동하여 `my-service.my-ns`에 대한 DNS 레코드를 만든다.  

# 헤드리스 서비스 
명시적으로 클러스터 IP (`spec.clusterIP`)에 `None`을 지정한다.  

다른 서비스 디스커버리 메커니즘과 인터페이스할 수 있다.  

헤드리스 서비스의 경우, 클러스터 IP가 할당되지 않고, kube-proxy가 이러한 서비스를 처리하지 않으며, 플랫폼에 의해 로드 밸런싱 또는 프록시를 하지 않는다.  

## 셀렉터가 있는 경우
셀렉터를 정의하는 헤드리스 서비스의 경우, 엔드포인트 컨트롤러는 엔드포인트 레코드를 생성하고, DNS 구성을 수정하여 서비스를 지원하는 파드를 직접 가리키는 A 레코드를 반환한다.  

## 셀렉터가 없는 경우
셀렉터를 정의하지 않는 헤드리스 서비스의 경우, 엔드포인트 컨트롤러는 엔드포인트 레코드를 생성하지 않는다.  

다음 중 하나를 찾고 구성한다.
    - ExternalName 유형 서비스에 대한 CNAME 레코드
    - 다른 모든 유형에 대해, 서비스의 이름을 공유하는 모든 엔드포인트에 대한 레코드  

# **서비스 퍼블리싱**
ServiceTypes는 원하는 서비스 종류를 지정할 수 있도록 해준다. 
기본값은 ClusterIp 이다.

- ClusterIP
  - 서비스를 클러스터 내부 IP에 노출시킨다.
  - 클러스터 내에서만 서비스에 도달할 수 있다.
- NodePort
  - 고정 포트로 각 노드의 IP에 서비스를 노출시킨다.
  - NodePort 서비스가 라우팅되는 ClusterIP 서비스가 자동으로 생성된다.
  - NodeIP:NodePort를 요청하여 클러스터 외부에서 NodePort 서비스에 접근할 수 있다.
- LoadBalancer
  - 클라우드 공급자의 로드 밸런서를 사용하여 서비스를 외부에 노출시킨다.
  - 외부 로드 밸런서가 라우팅되는 NodePort와 ClusterIP 서비스가 자동으로 생성된다. 
- ExternalName  
  - 서비스를 ExternalName 필드의 콘텐츠 (ex: foo.bar.example.com)에 매핑한다.

