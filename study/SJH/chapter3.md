# k8s 스터디 3주차
## 파드가 필요한 이유
왜 단일 컨테이너를 사용하지 않는가?
왜 모든 프로세스를 단일 컨테이너에 넣지 않는가?
왜 파드가 필요한가?

 컨테이너는 단일 프로세스를 실행하는것을 목적으로 설계되었다
->  자식 프로세스를 실행하는 경우를 제외
->  모든 프로세스가 표준 출력으로 로그를 기록하여 로그 추적이 어렵다
->  개별 프로세스의 실패 시 재시작 메커니즘을 포함하여야 함

그렇기 때문에 단일 프로세스의 여러 컨테이너를 함께 묶는 단위가 필요
-> 파드


## 파드 내부 컨테이너 간 격리
K8s는 파드 안의 모든 컨테이너가 동일한 리눅스 네임 스페이스를 공유하도록 설정한다
-> 같은 파드 내부의 컨테이너는 동일한 네임스페이스에서 실행된다
-> IPC(Inter-Process Communication)으로 통신 가능

그러나 파일 시스템은 컨테이너 이미지에서 나오므로 다른 컨테이너와 완전히 분리된다
이를 공유하기 위해서는 쿠버네티스 볼륨이 필요하다


## 컨테이너가 동일한 IP와 포트 공간을 공유하는 방법
파드 안의 컨테이너는 동일한 네트워크 네임스페이스에서 실행된다
-> IP 주소, 포트 공간 공유
-> 프로세스간 충돌 가능성이 있음
-> 로컬 호스트를 통해 통신 가능

## 파드간 네트워크
K8s 클러스터의 모든 파드는 Aclass private network에 상주한다
-> NAT 없이 서로 통신이 가능
	* 같은 노드 위에 올라가 있는지는 상관 없다



## 파드 컨테이너의 적절한 구성
파드는 상대적으로 가볍고 스케일링이 자유롭기 때문에
애플리케이션을 여러개의 파드로 구성하는것이 유리하다

### 파드 분할의 장점
* 한 노드에 집중되지 않아 컴퓨팅 리소스의 절약 가능
* 스케일링, FailOver 용이


### 그럼에도 여러 컨테이너를 사용하는 경우
![](chapter3/A73BF73B-A46E-43E9-85A1-49FFA2263CC2.png)
위와 같이 하나의 프로세스와 하나 이상의 보완 프로세스로 구성된 경우

의사 결정
* 컨테이너가 함께 실행되어야 하는가?
* 서로 다른 호스트에서 실행될수 있는가?
* 여러 컨테이너가 모여 하나의 구성요소를 나타내는가?
* 컨테이너가 함께, 또는 개별적으로 스케일링 되어야 하는가?


## 파드 생성
파드를 포함한 k8s 리소스는 일반적으로 Json, YAML 매니페스트를 통해 생성한다
-> 파일로 관리하면 버전 관리가 가능

```
kubectl get pods {pod_name} -o yaml // 매니페스트 확인 명령어
```

### 디스크립터
* Metadata
	* 이름, 네임스페이스, 레이블 및 파드에 관한 기타 정보 포함
* Spec
	* 파드 컨테이너, 볼륨, 기타 데이터 등 파드 자체에 관한 실제 명세
* Status
	* 파드 상태, 각 컨테이너 상태, 파드 내부 IP 등의 파드 기본 정보


![](chapter3/D47053B5-D70B-43A7-B902-F8D009EA0519.png)

포트 설정을 명시적으로 해주지 않아도 0.0.0.0 파드에 접근할 수 있으나
매니페스트에 명시할 경우 포트에 이름을 지정하여 편리하게 사용할 수 있다


### yaml을 통한 파드 만들기
```
kubectl create -f {manifest_file_name}
```

### 애플리케이션 로그
```
kubectl logs {pod_name}
kubectl logs {pod_name} -c {container_name} // 다중 컨테이너 파드일 경우
```
컨테이너 로그는 하루 단위, 10MB에 도달할 때마다 순환된다
파드가 삭제되면 로그도 같이 삭제된다. 이를 방지하기 위해서는 클러스터 전체의 로그를 수집하는 중장입중식 로깅을 설정해야한다


## 레이블
파드를 비롯한 모든 k8s 리소스를 그룹핑할수 있는 기능
Key-value pair로 이루어져 있으며 레이블 셀렉터를 이용해 리소스를 선택할 수 있다
레이블 값은 수정, 추가가 가능하다


![](chapter3/A7E7A2CF-A51F-45FA-A0C2-C20E18D6F272.png)
Key = app, rel
Value = ui, as, pc, sc, os, stable, beta, canary


### 레이블 지정 및 수정

![](chapter3/900620BE-9B4D-4F9A-BB57-1A7E94C5048D.png)

```
kubectl label pods {pod_name} {label_key}={label_value}  // 수정 시에는 --overwrite
```


### 레이블 셀렉터
특정 레이블로 태그된 파드의 부분집합을 선택해 원하는 작업을 수행할 수 있는 기능
* 특정 키를 포함, 포함하지 않는 레이블
* 특정 키와 값을 가진 레이블
* 특정 키를 갖고있지만 값은 다른 레이블

```
-l {key}={value}  // key, value 일치
-l {key}  // key만 일치
-l '!env'  // key가 다른 것
-l {key} in (value1,value2)  // key가 value1 또는 value2
```


## 레이블과 셀렉터 응용
### 노드 분류에 레이블 사용
```
kubectl label node {node_name} {label_key}={value}
```

### 특정 노드에 파드 스케줄링

![](chapter3/F4967A25-4B6D-4848-8066-ACF0DC6F2FE5.png)

노드에는 kubernetes.io/hostname=hostname인 레이블이 존재하여 이 레이블을 사용하여 스케줄링 할수 있으나 노드가 오프라인일 경우 파드가 스케줄링 되지 않는 문제가 발생할 수 있다
-> 노드 또한 개별 노드가 아닌 논리적인 그룹으로 생각해야 한다


## 파드 어노테이션
* key-value pair이지만 식별정보를 갖지 않아 오브젝트 그룹핑 불가
* 레이블보다 상대적으로 큰 정보 저장 가능 (~256KB)
* 특정 어노테이션은 k8s에 의해 자동으로 추가되지만 나머지는 사용자에 의해 수동으로 추가된다
* K8s 오브젝트에 설명을 추가하는데 유용하게 사용된다

주로 k8s에 새로운 기능을 추가할때 흔히 사용된다
Ex) 신기능의 알파, 베타 버전을 위한 어노테이션으로 기능을 테스트하고 도입될 때 어노테이션 -> 파라미터로 변경하는 방식


### 어노테이션 조회

![](chapter3/E3C9C94F-7A71-4406-BC36-F59211A3A032.png)

### 어노테이션 추가 및 수정
```
kubectl annotate pod {pod_name} {key}={value}
```

레이블과 다르게 override 파라미터 없이도 덮어버릴 수 있으므로 주의


## 네임스페이스
각 오브젝트는 여러 레이블을 가질 수 있으므로 셀렉터 만으로 오브젝트 그룹을 완벽히 나눌 수 없다
이를 위해 k8s에서는 쿠버네티스 네임스페이스를 통해 오브젝트를 격리할 수 있다
-> 리소스는 네임스페이스 안에서 고유하므로 같은 리소스를 다른 네임 스페이스에서 가질 수 있다

대부분의 k8s리소스는 네임스페이스에 속하나 노드는 예외이다

### 네임스페이스 리소스 접근
```
kubectl get ns  // name space 리스트
kubectl get pod -n {name_space}  // 해당 name space에 해당하는 pod list
```


### 네임스페이스 생성
네임스페이스 또한  k8s 리소스이므로 YAML 파일을 통해 생성 할 수 있다
![](chapter3/127E4923-0ACC-4FFA-8C93-8C0C20BAC355.png)
또는
```
kubectl create namespace {name_space}
```


### 오브젝트 관리
다른 네임스페이스 안에 있는 오브젝트에 접근하기 위해서는 --namespace 또는 -n 파라미터를 반드시 사용해야 한다
-> 그렇지 않으면 default context로 동작


### 네임스페이스의 격리 수준
* 오브젝트를 분리하여 작업할 수 있게 한다
-> 그렇다고 다른 네임스페이스의 파드와 통신이 불가한것은 아님


## 파드 중지와 제거
```
kubectl delete pod {pod_name}
kubectl delete po -l {lable_key}={value}
kubqctl delete ns {name_space}  // 네임스페이스 내부의 파드까지 삭제
```

파드를 삭제하면 k8s는 파드 내의 모든 컨테이너를 종료하도록 지시한다

1. SIGTERM 신호를 프로세스에 보내고 30초 대기
2. SIGKILL 신호를 통해 종료




## 질문 정리
### NAT
Network Address Translation

### UTS namespace
Unix Time Sharing namespace
* 한 시스템 내에서 host, domain name을 분리하는 namespace
-> 말 그대로 다른 사용자 이름을 가진 시스템에서 실행중인것 처럼 동작

### YAML vs JSON
yaml이나 json 이나 결국 파라미터 집합을 표현하기위한 포맷에 불과하므로 차이점은 없을것으로 예상됨, 간단히 검색해봐도 뚜렷한 차이점을 찾을 수 없으나
일반적으로 JSON processing이 yaml processing보다 빠른 속도를 가지고 있다고 합니다

### run=kubia label
왜 그러지

### manifest를 통한 파드 생성시 파드 유지 문제
Json, yaml을 통해 생성한 파드에 문제가 있는게 아니라
실행시켰던 manifest는 오직 pod의 명세가 적힌 manifest이기 때문에 그런것이고 RC같은 경우 pod의 상태를 유지해주는 역할의 리소스이기 때문에 RC, RS, Deployment가 존재할 경우 pod의 상태가 유지됨

### IPC
Inter process Communication
실행중인 프로세스 간에 통신을 하기위한 인터페이스

* pipe
* named pipe
* socket
* signal
* shared memory
등의 구현체 존재

### Flat Network
모든 시스템이 같은 주소 영역에 존재하여 public network 접근 없이 통신이 가능한 network

### node label을 통한 파드 생성이 좋지 못한 이유
해당 기능은 어떤 파드가 반드시 특정 노드 위에서 동작할것을 보장해야할 때 사용해야되는 기능이라고 생각.

### SIGTERM, SIGKILL
signal은 process에게 특정 event가 발생했음을 알려주는 IPC의 한 종류이다

SIGNAL+TERMINATION, SIG+KILL 신호로
둘 다 process의 종료를 전달하는 시그널이지만
SIGTERM의 경우 graceful shutdown을,
SIGKILL의 경우 가차없이 강제종료를 시켜버린다