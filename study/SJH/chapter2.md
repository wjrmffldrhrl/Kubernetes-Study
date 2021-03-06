# k8s 스터디 2주차
## 도커 background 실행 순서
![](./image/chapter2.1.png)
	1. Busybox:latest 이미지가 로컬에 존재하는지 체크
	2. 없을 경우 도커 허브 레지스트리에서 이미지 다운로드
	3. 컨테이너 생성
	4. 컨테이너 내부에서 명령어 실행
![](./image/chapter2.2.png)

## 이미지 버전 지정
* 도커는 동일한 이미지에 여러개의 버전을 가질 수 있다
* 각 버전은 고유한 태그를 가진다
* 태그가 명시적이지 않을 경우 latest를 default로 참조한다

## Dockerfile
도커가 이미지를 생성하기 위해 수행할 지시사항이 담긴 파일
 
```
FROM node:7  // node 이미지의 태그7를 베이스 레이어로 사용 
ADD app.js /app.js  // 로컬 디렉터리의 app.js파일을 
                    // 컨테이너 루트 디렉터리에 추가
ENTRYPOINT ["node", "app.js"]  // 이미지 실행 시 수행될 명령어
```


## 이미지 빌드
명령어 docker build -t {image_name}

![](./image/chapter2.3.png)

빌드 프로세스는 도커 클라이언트가 아닌 데몬에서 진행된다
그렇기 때문에 데몬이 로컬에 실행 중이지 않을경우 빌드 디렉토리의 크기에 따라 업로드 시간에 차이가 발생한다

### 이미지 레이어 
![](./image/chapter2.4.png)

## 이미지 실행
명령어 

```
docker run --name {container_name} -p 8080:8080 -d {image_name}
```

* -d : 백그라운드 실행
* -p : 포트 매핑

## 컨테이너 중지와 삭제
```
docker stop {container_name}  // 컨테이너의 메인 프로세스 중지
docker rm {container_name}  // 컨테이너 삭제
```

## 이미지 푸시
외부 이미지 저장소에 이미지를 푸시할 수 있다
레지스트리의 규칙에 따라 이미지 태그 지정이 필요할 수 있다


## 쿠버네티스 클러스터
* 단일 노드 클러스터
	* 책에서는 minikube
* 다중 노드 클러스터
	* GCE, ACE, EKS등

![](./image/chapter2.5.png)


### 쿠버네티스 앱 실행
```
kubectl run {app_name} --image={image_path} --port=8080 --generator-run/v1
```

	* --image : 이미지 명시
	* -port : 컨테이너의 메인 프로세스가 대기할 port
	* --generator : 디플로이먼트 대신 레플리케이션 컨트롤러(레플리카셋) 생성
![](./image/chapter2.6.png)


### 파드
* 쿠버네티스는 개별 컨테이너를 직접 다루지 않고 컨테이너를 그룹으로 묶어 파드라는 개념을 통해 사용한다
* 하나 이상의 밀접한 컨테이너 그룹으로 같은 워커 노드에서 같은 네임스페이스로 함께 실행된다
* 각 파드는 자체 IP, 호스트 이름, 프로세스 등이 있는 논리적으로 분리된 머신

![](./image/chapter2.7.png)

### 파드 접근
 파드는 각각 private ip 를 갖기 때문에 외부 접근을 위해서는 서비스 오브젝트를 통해 노출해야한다.
이때 생성되는 서비스 또한 쿠버네티스 오브젝트이기 때문에 외부에 노출되지 않으므로 public ip를 통해 외부에 노출되는 LB 타입의 서비스를 생성해야 한다

#### 서비스 오브젝트 생성
```
kubectl expost rc {pod_name} --type=LoadBalancer --name {service_name}  // rc = replicationcontroller
```

![](./image/chapter2.8.png)
서비스 오브젝트가 생성되면 External-IP를 통해 파드와 통신할 수 있다


### k8s 오브젝트 이해
* pod
	* 파드는 k8s의 가장 중요한 구성 요소이다
	* 원하는 만큼의 컨테이너를 갖는다
	* 자체 private ip와 host name을 갖는다
	* 일시적이다
* Replication controller (replica set으로 대체)
	* pod가 항상 지정된 만큼 실행되도록 하는 오브젝트
* 서비스
	* pod는 일시적이며 개별 파드의 주소는 바뀔수 있기 때문에 여러개의 파드를 대표하는 주소를 갖는 오브젝트

### 애플리케이션 수평 확장
K8s에서는 pod의 갯수를 자유롭게 조정할 수 있다
pod의 갯수를 지정하여 k8s에게 원하는 상태를 선언하면 k8s는 자동으로 해당 상태로 시스템을 조정한다