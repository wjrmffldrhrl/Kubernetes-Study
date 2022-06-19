# 2주차!
## 컨테이너 이미지 생성, 실행, 공유
### docker run image

busybox를 이용하여 hello world 컨테이너 실행함

'docker run busybox echo "Hello world" 명령어에 대한 이해
1. busybox가 로컬 이미지로 존재하지 않기 때문에 도커 허브 레지스트리에서 다운로드 해옴
2. 이미지가 다운 완료되면 이미지로부터 도커가 컨테이너를 생성해줌
3. 컨테이너 내부에서 명령어를 실행

실행돼야 할 명령어는 보통 이미지를 생성할때 내부에 넣어 패키징, 명령어 오버라이드가능(명령어를 재정의) -> docker file?

p 76 <요청 핸들러는 나중에 필요한 경우를 위해 클라이언트 ip 주소를 표준 출력에 로깅한다?> -> 나중에 ip는 다른데 고정ip로 접근하는걸 알려주기위해?

#### dockerfile
Dockerfile 에는 도커가 이미지를 생성하기 위해 수행해야할 지시 사항이 담겨있음, app.js 파일과 동일한 디렉터리에 있어야함
from줄에는 컨터이너 이미지를 정의

### docker build
docker build 명령어를 사용하면, 도커 클라이언트가 디렉토리에 있는 dockerfile과 app.js를 도커데몬에 업로드
도커데몬은 dockerfile을 확인해서 node:7.0 이미지가 없기때문에 레지스토리에서 이미지 pull

이미지 빌드 명령어 : docker build -t kubia . 

디렉터리 콘텐츠 전체가 도커 데몬에 업로드 되고 거기서 이미지 빌드됨

이미지는 레이어라 공유가능, 도커는 저장되어있지 않은 레이어만 다운로드

이미지 수동 생성 하는방법: 기존 이미지에서 컨테이너 실행, 컨테이너 내부에서 명령어 수행, 최종 상태를 새로운 이미지로 커밋

### docker run --name kubia-container -p 8080:8080 -d kubia
-d는 백그라운드에서 실행

실행할떄 원래는 http://localhost로 애플리케이션에 접근하는데 로컬머신에 도커 데몬이 실행중이 아닌경우 (맥이나 윈도우 -> 가상머신) localhost대신 가상머신 호스트이름이나 ip를 사용해야됨

내부탐색방법
bash 셀은 컨테이너의 메인 프로세스와 동일한 리눅스 네임스페이스를 가짐 -> 내부를 볼수 있게됨

쿠버네티스 클러스터?
쿠버네티스 클러스터 내에서 실행되는 모든 컨테이너가 동일한 플랫 네트워킹 공간을 통해 서로 연결되도록 네트워크를 올바르게 설정해야한다 (플랫 네트워킹 공간?)

### docker stop ~ , docker rm ~

이미지를 푸시할떄 태그를 지정해줘야됨 (도커허브규칙)
p86  한 이미지에 새로운 태그가 추가됬다고 했는데 예제 2.9 tag에는 둘다 latest이다. 태그를 지정해줄때 ::tag 방식으로 앞에서나왔는데 ::latest를 하면 어느 레포에서 이미지를 갖고오는가..?

## 쿠버네티스 클러스터 설치
(1) 로컬에서 단일 노드 쿠버네티스 클러스터 실행 (minikube) 
(2) 구글 쿠버네티스 엔진(GKE)에 실행중인 클러스터에 접근

minikube 설치한후, minikube start로 시작
그리고 kubectl를 다운로드

gke에서 노드 세개의 클러스터 생성 명령어
- gcloud container clusters create kubia --num-nodes 3

각 노드는 도커, kubelet, kube-proxy를 실행

kubectl 명령어를 사용하면 로컬에서 rest 요청을 마스터노드로 보낸다 
kubectl의 여러 명령어들 나옴 


### 파드란?
쿠버네티스는 개별 컨테이너들을 직접 다루지 않는다
대신 함께 배치된 다수의 컨테이너라는 개념을 사용한다 -> 파드
파드는 같은 워커노드에서 같은 리눅스 네임스페이스로 실행
-> 논리적으로 분리되어있다

다른 파드에 실행중인 컨테이너는 같은 워커노드에서 실행중이라 할지라도 다른머신에서 실행중으로 나타남

백그라운드에서 일어나는일
-> kubectl이 rest http요청으로 클러스터에 새로운 레플리케이션컨트롤러 오브젝트가 생성된다, 레플리케이션컨트롤러는 새파드를 생성하고, 워커노드에 스케줄링된다, 이미지가 로컬에 없으면 pull해온다, 그다음에 이미자로 컨테이너 생성, 실행

외부에서 파드에 접근을 가능하게 하려면 서비스 오브젝트를 통해 노출해야함 load balancer가 필요함
load balancer의 public ip를 이용하면 파드와 연결가능

p101 레플리케이션컨트롤러를 노출하는게 새로운 서비스 오브젝트를 만드는건가?

kubectl run 명령을 수행하면 레플리케이션컨트롤러가 파드를 생성하는데 이때 파드안에 몇개의 컨테이너가 들어갈지도 정하는지?


kubia 레플레케이션 컨트롤러는 항상 하나의 파드 인스턴스를 실행하도록 지정
모니터링  : 레플리케이션컨트롤러  외부노출 : 서비스
파드가 사라지면 새로운 파드를 생성해줌
서비스는 정적 ip로 연결해줌

desired state를 알려주면 항상 그 state를 유지하려고 조정함
