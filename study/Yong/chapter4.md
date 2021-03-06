# 4 레플리케이션과 그 밖의 컨트롤러: 관리되는 파드 배포
## 파드를 안정적으로 유지
파드가 노드에 스케줄링 되는 즉시 노드의 kubelet이 컨테이너를 실행하고 유지하려 한다 크래시가 발생할 경우에는 컨테이너를 다시 시작한다

크래시없이 작동이 중단된 경우도 있는데 이런 경우에도 다시 자동으로 시작하게 하려면 외부에서 상태를 체크해주어야한다

쿠버네티스는 라이브니스 프로브로 컨테이너가 살아있는지 확인한다
라이브니스 프로브가 실패할 경우 컨테이너를 다시 시작한다
- HTTP GET
- TCP 소켓
- Exec


종료코드는 128+x인데 x는 프로세스에 전송된 시그널 번호이다 (왜 128일까..?)

라이브니스 프로브에는 추가적인 매개변수도 따로 지정가능하다
http요청 응답여부를 확인하는 라이브니스 프로브도 있지만, 특정 url 경로에 요청하도록 하는 라이브니스 프로브가 더 효율적이다 재시도 루프를 구현하지 않는것도 효율성을 높이는 방법중 하나이다

## 레플리케이션컨트롤러
레플리케이션컨트롤러는 파드가 항상 실행되도록 한다
파드가 사라지면 이를 감지해서 새 파드를 생성해준다

실행중인 파드 목록을 지속적으로 모니터링하고, 의도하는 실행중인 파드수가 맞는지 확인해준다
레플리케이션컨트롤러는 파드 유형이 아니라 특정 레이블 셀렉터와 일치하는 파드 세트에서 작동한다

레플리케이션컨트롤러의 세가지 요소
- 레이블 셀렉터
- 레플리카 수
- 파드 템플릿

레플리케이션컨트롤러의 장점
- 파드가 사라지면 새로운 파드를 만들어 파드가 항상 실행하게 해준다
- 클러스터 노드에 장애가 발생하면 장애가 발생한 노드의 파드를 모두 복제해준다
- 수평으로 확장가능하다

레플리케이션 컨트롤러의 범위에 들어왔다 나갔다 하려면 레이블을 변경하면된다

레플리케이션 컨트롤러 수평 확장은 replicas로 값을 변경해주면된다

## 레플리카셋
레플리카셋은 레플리케이션 컨트롤러보다 더 풍부한 표현식을 사용하는 파드 셀렉터를 가지고있다
in, notin, exists, doesnotexists 사용가능

## 데몬셋
레플리케이션컨트롤러와 레플리카셋은 한 노드에 여러 파드들이 실행될 수 있다. 하지만 데몬셋은 한 노드당 하나의 파드만 실행하길 원할때 사용하면 된다 (인프라 관련 파드)

## 완료 가능한 단일 태스크를 수행하는 파드 (잡)
완료됬을때 파드가 종료되길 원하면 잡 리소스를 사용하면 된다 잡 리소스는 컨테이너 내부에서 실행중인 프로세스가 완료되면 컨테이너를 다시 시작하지 않는다
장애가 발생한 경우에는 다른 노드에 다시 스케줄링 해준다

한 프로세스를 두번 이상 완료시키고 종료되길 원하는 경우나 여러 프로세스를 동시에 실행시키고 싶은 경우에는 completions과 parallelism을 사용하면된다.

## 잡을 주기적, 한번 실행 스케줄링
특정시간, 또는 지정된 간격으로 잡이 실행되길 원하면 크론잡 리소스를 사용하면된다