# 컨피그맵과 시크릿
### 다루는 내용
- 컨테이너의 주 프로세스 변경
- 애플리케이션에 명령줄 옵션 전달
- 애플리케이션에 노출되는 환경변수 설정
- 컨피그맵으로 애플리케이션 설정
- 시크릿으로 민감한 정보 전달

# 컨테이너화된 애플리케이션 설정

컨테이너화된 애플리케이션에 설정 데이터를 전달하는 방법은 아래와 같다. 
- 모든 설정을 애플리케이션에 포함
- 명령줄 인수로 애플리케이션에 전달
- 설정을 파일에 저장하고 사용
- 환경변수 사용

컨테이너에서는 환경변수를 널리 사용한다.  
- 도커 컨테이너 내부에 있는 설정 파일을 사용하려면 설정 파일을 컨테이너 이미지에 포함해야 하거나 볼륨을 마운트해야 하기 때문  

간단한 방법으로 설정 데이터를 쿠버네티스 리소스에 저장하고 제공하는 것을 컨피그맵이라고 한다.  

# 컨테이너에 명령줄 인자 전달

## 도커에서 명령어와 인자 정의

- ENTRYPOINT : 컨테이너가 시작될 때 호출될 명령어
- CMD : ENTRYPOINT에 전달할 인자 정의
올바른 방법은 ENTRYPOINT로 명령어를 실행하고 기본 인자를 정의하려는 경우에만 CMD를 지정하는 것이다.  

- shell 형식 : ENTRYPOINT node app.js
- exec 형식 : ENTRYPOINT ["node", "app.js"]
두 형식의 차이점은 정의된 명령을 셸로 호출하는지 여부다.  
shell 형식을 사용하면 메인 프로세스는 node 프로세스가 아닌 shell 프로세스다.  
- shell 프로세스는 불필요하므로 ENTRYPOINT 명령에서 exec 형식을 사용하는게 옳다.  



## 쿠버네티스에서 명령과 인자 재정의
쿠버네티스에서 컨테이너를 정의할 때, ENTRYPOINT와 CMD 둘 다 재정의할 수 있다.  

```yaml
kind: Pod
spec: 
  containers:
  - image: some/image
    command: ["/bin/command"] # ENTRYPOINT와 대응
    args: ["args1", "args2", "args3"] # CMD와 대응
```  

# 컨테이너의 환경변수 설정
쿠버네티스는 아래 그림과 같이 파드의 각 컨테이너를 위한 환경변수 리스트를 지정할 수 있다.  

![image_1](../images/chapter_7_1.png)  

## 컨테이너 정의에 환경변수 지정
```yaml
kind: Pod
spec:
  containers:
  - image: luksa/fortune:env
    env:
    - name: INTERVAL # 환경 변수 추가
      value: "30"
    name: html-generator

```  

## 변숫값에서 다른 환경변수 참조
위 예시에서는 고정된 환경변수 값을 설정했지만 `$(VAR)` 구문을 사용해 이미 정의된 환경변수나 기타 기존 변수를 참조할 수 있다.  

```yaml
env:
- name: FIRST_VAR
  value: "foo"
- name: SECOND_VAR
  value: "$(FIRST_VAR)bar"
```

## 하드코딩된 환경변수의 단점
여러 환경에서 동일한 파드 정의를 재사용하려면 파드 정의에서 설정을 분리하는 것이 좋다.  

value 필드 대신 valueFrom으로 환경변숫값의 원본 소스로 사용할 수 있다.  


# 컨피그맵으로 설정 분리  
애플리케이션 구성의 요점은 환경에 따라 다르거나 자주 변경되는 설정 옵션을 애플리케이션 소스 코드와 별도로 유지하는 것이다.


## 컨피그맵 소개
쿠버네티스에서는 설정 옵션을 컨피그맵이라 부르는 별도 오브젝트로 분리할 수 있다.  
- 컨피그맵은 짧은 문자열에서 전체 설정 파일에 이르는 값을 가지는 키/값 쌍으로 구성된 맵이다.  

맵의 내용은 컨테이너의 환경변수 또는 볼륨 파일로 전달된다.  

또한 환경변수는 `$(ENV_VAR)` 구문을 사용해 명령줄 인수에서 참조할 수 있기 때문에, 컨피그맵 항목을 프로세스의 명령줄 인자로 전달할 수 도 있다.
![image_2](../images/chapter_7_2.png)  

파드는 컨피그맵을 이름으로 참조하기 때문에, 모든 환경에서 동일한 파드 정의를 사용하면서 서로 다른 설정을 사용할 수 있다.  

![image_3](../images/chapter_7_3.png)  

## 컨피그맵 생성
`k create -f`명령어로 YAML 파일을 게시하는 대신 `k create configmap` 명령으로 컨피그맵을 생성할 수 있다.  

```sh
> k create configmap fortune-config --from-literal=sleep-interval=25
```  
여러 문자열 항목을 가진 컨피그맵을 생성하려면 여러 개의 --from-literal 인자를 추가한다.  
```sh
> k create configmap myconfigmap \
  --from-literal=foo=bar --from-literal=bar=baz --from-literal=one=two
```  

### 파일 내용으로 컨피그맵 생성  
컨피그맵에는 전체 설정 파일 같은 데이터를 통째로 저장하는 것도 가능하다. 
```sh
> k create configmap my-config --from-file=config-file.conf
```  

기본적으로 파일 이름을 키 이름으로 지정하지만 직접 이름을 지정할 수 있다. 
```sh
> k create configmap my-config --from-file=customkey=config-file.conf
```

각 파일을 개별적으로 추가하는 대신 디렉터리 내부의 모든 파일을 가져올 수 있다.
```sh
> k create configmap my-config --from-file=customkey=/path/to/dir
```
지정한 디렉터리 안에 있는 각 파일을 개별 항목으로 작성한다.  


305 - 352