# 5강 Pod

학습 내용

- 개념 및 사용하기
- livenessProbe를 사용한 self-healing Pod
- init container
- infra container(pause) 이해하기
- static pod 만들기
- Pod에 resource 할당하기
- 환경변수를 이용해 컨테이너에 데이터 전달하기
- pod 구성 패턴의 종류

<br>

## 5-1-1.

- docker에서 컨테이너 실행 방법
    - app 제작
    - docker file을 만들고 그 안에 필요한(다운받을) 패키지 정보,  app을 copy해서 집어 넣기 위한 app의 경로, 컨테이너가 실행될 때 무슨 명령어 실행할지 등을 작성.
        - `FROM` : 베이스 이미지 설정
        - `COPY` : 필요 파일 복사
        - `CMD` : 실행할 명령어
    - 도커 빌드: `docker build` 이렇게 하면 Dockerfile 기반으로 이미지를 빌드함
        - `docker build -t myapp:latest` 
            - `-t` : tag  
빌드한 이미지에 태그(이름:버전)를 붙여준다.  
태그 안하면 자동으로 latest가 붙는다.

    - 허브에 저장: `docker push` 
        - `docker push jjw/myapp:latest` 

- k8s 의 실행단위인 pod
    - 컨테이너가 최소 1개 들어감. 2개 이상들어갈 수도 있음.

### pod 생성법

1. run 명령어  
`kubectl run web1 --image=nginx:1.14`   
kubectl run \[Pod 이름\] --image=이미지:버전

<br>

2. create - yaml 파일 사용  
`kubectl create -f pod.yaml`   
kubectl create -f \[yaml파일명\].yaml

<br>

### pod 보기

1. yaml 형태로 보기`kubectl get pods web1 -o yaml   `json도 가능.
2. 동작상태 보기  
`watch kubectl get pods -o wide`   
watch는 리눅스에서 제공하는 명령어임 default 는 2초 간격.

<br>

### multiple container pod 만들기

multiple container: pod 안에 여러 개의 컨테이너를 집어 넣는 경우.

<br>

```yaml
apiVersion: v1
kind: Pod
metadata:
 name: multipod
spec:
 containers:
 - name: nginx-container
   image: nginx:1.14
   ports:
   - containerPort: 80
 - name: centos-container
   image: centos:7
   command:
   - sleep
   - "100000"
```

<br>

### multi container 컨테이너 접속

multi container일 경우 `kubectl exec`  명령에 container name을 입력해서 접근할 수 있다.

예시: `kubectl exec multipod -c nginx-container -it — /bin/bash` 

it뜻: 

- `-i`  = interactive  
표준 입력(STDIN)을 열어둔다.  
즉, 컨테이너 쉘에 명령어를 입력할 수 있게 한다.
- `t` = tty(teletype)터미널 화면을 만들어서 사람이 쉘처럼 다룰 수 있게 해준다.

<br>

위 yaml로 pod를 실행하고 centos를 들어가서

`curl localhost:8080` 을 하면, nginx 서버가 응답한다.

왜냐면 ip가 같기 때문.

<br>

### log 보기

멀티 pod 예시:⁠`kubectl logs multipod -c nginx-container` 

싱글 pod 예시: `kubectl logs multipod` (container 이름 필요 없음)

<br>

<br>

## 5-1-2. Pod 동작 flow

<br>

1. 컨트롤 플레인의 API가 명령을 받음 → 문법이 맞는지 검사.
2. API는 etcd 정보를 꺼내서 scheduler에게 보냄. → 그러면 scheduler 가 적절한 노드 찾아서 API로 보냄 → API가 kubelete에 전달 → 여기까지가 Pending 상태.
3. 워커 노드에서 파드가 실행되면 → Running | Failed | Succeeded

<br>

**pod 삭제**

- 전부 삭제: `kubectl delete pod --all` 
- 단일 삭제: `kubectl delete pod [pod명]` 

<br>

**동작중인 파드 정보 주기적으로 보기**

`kubectl get pods -o wide --watch` 

### <br>

**동작 중인 파드 수정**

`kubectl edit pod [pod명]` 

<br>

**전체 namespace에서 동작중인 파드 보기**

`kubectl get pods --all-namespaces` 

<br>

describe: 리소스 상태, 이벤트 로그

get ~~ -o yaml: 리소스의 선언 상태를 yaml로.

<br>

1. ==kubectl get pods==
2. ==kubectl get pods --all-namespaces==
3. ==kubectl run nginx-pod --image=nginx:1.14==
4. kubectl get pod nginx-pod -o yaml → descirbe
5. kubectl describe pod nginx-pod || get pods
6. kubectl get pods || describe →running 개수
7. kubectl get pods -o wide ||  describe
8. describe
9. 실행 대기 중인 컨테이너 개수 → Ready인 컨테이너 수 / 전체 컨테이너 수
10. kubectl delete pod nginx-pod
11. kubectl run redis --image=redis123 --dry-run -o yaml > redis.yaml

<br>

<br>

## 5-2. livenessProbe를 이용해서 Self-healing Pod 만들기

**livenessProbe**: 건강검진 같은 것.

probe: 조사(탐색)

<br>

| 일반 pod | livenessProbe 포함 pod |
| --- | --- |
| ```yaml<br>apiVersion: v1<br>kind: Pod<br>metadata:<br>  name: basic-pod<br>spec:<br>  containers:<br>  - name: app<br>    image: nginx<br>```<br><br><br> | ```yaml<br>apiVersion: v1<br>kind: Pod<br>metadata:<br>  name: liveness-pod<br>spec:<br>  containers:<br>  - name: app<br>    image: nginx<br>    livenessProbe:<br>      httpGet:<br>        path: /<br>        port: 80<br>``` |

  

### 매커니즘 3가지

- httpGet
    - 지정한 IP주소, port, path에 HTTP Get 요청을 보내서 해당 컨테이너가 응답하는지 확인한다.반환코드가 200이 아니면 오류. 컨테이너를 다시 시작한다.
- tcpSocket
    - 지정된 포트에 TCP 연결을 시도하여 연결되지 않으면 컨테이너를 다시 시작한다.
    - ```yaml
      livenessProbe:
        tcpSocket:
        port: 22
      ```

- exec
    - exec 명령을 전달하고 명령의 종료코드가 0이 아니면 컨테이너를 다시 시작한다.
    - ```yaml
       livenessProbe:
         exec:
           command:
        - ls
          - /data/file
       ```

### 매개변수

- initialDelaySeconds
    - default: 0
    - n 초만큼 기다렸다가 검사를 시작하겠다.
- timeouts
    - default: 1
    - n 초만큼 응답이 안오면 에러
- periodSecdons
    - default: 10
    - 검사 간격.
- successThreshold
    - default: 1
    - n 번 연속 성공하면 성공으로 보겠다.
- failureThreshold
    - defulat: 1
    - n 번 연속 실패하면 에러로 보겠다.

<br>

**이거 어케 외워서 침?**

→ 동작 중인 pod를 `get pod pod명 -o yaml`  명령어로 yaml 을 보면 위 매개변수를 입력하지 않아도 들어있다.

<br>

**재시작된 거 확인 방법**

`kubectl describe pod [pod명]` 

을 통해서 로그를 보고 아 새 이미지를 다운로드 받아서 컨테이너를 다시 실행했구나 하고 알 수 있다.

# <br>

## 5-3. init container

**init container 예시**

- db에서 데이터를 가져오는 init container를 설정하면, nodejs, spring boot 와 어플리케이션이 돌아가는 main container 는 돌아가지 않게 할 수 있음.
- init container에서 network 상태를 점검하여 정상일 때만 main container 실행.

<br>

**init container 소개**

- 앱 컨테이너 실행 전에 미리 동작시킬 컨테이너
- 초기화 컨테이너가 모두 실행된 후 앱 컨테이너 실행함.

<br>

- initContainer가 2개면 2개 모두 실행되어야 메인 컨테이너가 실행된다.  
아래 이미지는 init 컨테이너가 실행되지 않아 메인 컨테이너가 실행되지 않는 모습  
![](Files/Screenshot%202025-07-23%20at%2012.28.04%E2%80%AFAM.png)  

<br>

<br>

<br>

**예시 yaml**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-container-pod
spec:
  containers:
    name: main-container
    image: nginx
  initContainers:
  - name: init-myservice
    image: busybox
    command: ['sh', '-c', 'echo Init container 실행 중... && sleep 5']
```

<br>

<br>

## 5-4. infra container

**안 보이는 진실**

- pod 생성 시 pause 라는 이름의 컨테이너가 안보이게 1개 만들어진다.
- ip 혹은 hostname을 관리하는 컴포넌트.

<br>

**진짜 만들어지는 거 맞아? 확인 방법.**

1. `kubectl run webserver --image=nginx --port=80` 
2. `docker ps` : 현재 동작 중인 컨테이너 정보를 출력해줘.  
![](Files/Screenshot%202025-07-23%20at%2012.39.07%E2%80%AFAM.png)  
같은 시간대에 같이 만들어진 것 확인 가능.
3. `kubectl delete pod webserver`
    - 이후 다시 docker ps 를 해보면 pause 컨테이너도 같이 사라진 것 확인 가능.

<br>

## 5-5. static Pod(feat. kubelet daemon)

### 일반 Pod와 다른점

1. API에게 직접 요청 보내지 않음.
    - kubelet 이 관리하는 static pod 디렉토리가 있음. 거기에 yaml을 저장하면 kubelet daemon에 의해서 알아서 실행/삭제 됨.

<br>

### static 디렉토리

`/var/lib/kubelet/config.yaml` 

모든 노드는 해당 파일을 가지고 있음. 해당 파일을 cat 해서 보면 ⁠`staticPodPath` 가 있음.

kubelet daemon이 해당 path에 yaml이 있는지 검사해서 실행시켜줌. yaml 삭제하면 pod도 삭제 됨.

- yaml 넣으면 바로 실행
- yaml 삭제하면 바로 삭제

<br>

- `config.yaml`  수정하면 kubelet daemon restart 필수임!! 
    - 재시작 명령어(환경에 따라 다름): `sudo systemctl restart kubelet` 

<br>

### control plane(마스터 노드)에도 있다.

- `etcd.yaml` 
- `kube-apiserver.yaml` 
- `kube-controller-manager.yaml` 
- `kube-scheduler.yaml` 

k8s가 동작하는데 필수인 애들이 static pod로 실행된다.

<br>

### 기타

- CKA에 static pod 문제가 자주 나옴.
    - 특정 경로에 디렉토리를 만들고 해당 디렉토리를 `staticPodPath` 를 지정하라.

<br>

## 5-6. Pod에 Resource 할당하기 (CPU/momory requests, limits)

- k8s가 관리하는 리소스: cpu, memory

<br>

### 소개

**limits**

node 들은 리소스가 제한된 환경이다.

여러 pod들이 뜰 수 있는데 제한을 걸지 않으면 하나의 pod가 리소스를 모두 다 써버릴 수 있다.

그러면 2번째 파드는 쓸 수 있는 리소스가 하나도 남지 않게 됨.

그러니 하나의 pod가 얼마나 리소스를 쓸지 limit을 걸어줘야 함.

결론: Pod가 사용할 수 있는 최대 리소스양을 제한

<br>

limit을 넘겨서 리소스를 사용하면 컨테이너를 restart 해버린다. 그러니 너무 타이트한 설정은 하지 말자.

<br>

**requests**

pod 에 필요한 용량을 지정하는 것을 requests 라고 한다.

예) mongo DB 띄울 건데 cpu 500mc, memory 100mi 필요해요!

결론: 파드를 실행하기 위한 최소 리소스 양을 요청.

<br>

k8s에서는 limits와 requests를 걸어서 리소스 제한할 수 있다.

<br>

### 사용법

예시 yaml

```yaml
... 생략
sepc:
 containers:
 - name: nginx
   image: nginx:1.14
   ports:
   - containerPort: 80
     protocol: TCP
   resources:
    requests:
      cpu: 200m
      memory: 250Mi
     limits:
      cpu: 1
      memory: 500Mi
```

<br>

- limits만 걸면 request는 limits와 동일한 값으로 같이 들어간다.
- requests 가 리소스 제한량보다 많으면 **Pending** 상태로 대기하게 됨.

<br>

<br>

## 5-7. Pod의 환경변수 설정하기

### 환경변수

- PoD내의 컨테이너가 실행될 때 필요로 하는 변수
- 컨테이너 제작 시 미리 정의
    - Nginx Dockerfile 예
        - ENV NGINX\_VERSION 1.19.2
        - ENV NJS\_VERSION 0.4.3

- Pod 실행 시 미리 정의된 컨테이너 환경변수를 변경할 수 있다.
    - 예) 1.19.2를 1.20.3으로 바꿀 수 있나?
    - yaml 에서 `sepc.containers.env` 를 설정하면 된다.
    -  ```yaml
        apiVersion: v1
        kind: Pod
        metadata:
          name: env-pod
        spec:
          containers:
          - name: app
            image: nginx
            env:
            - name: MY_VAR
              value: "hello"
        ```

<br>

- 환경 변수 확인 방법: `exec` 로 들어간 다음, `env`  명령어 쓰면 환경변수 볼 수 있음.

<br>

## 5-8. Pod 실행 패턴

### 1\. Sidecar

웹서버가 로그를 만들고 그 로그를 분석과 같은 작업을 함.

혼자서는 움직이지 않고 컨테이너 2가지가 함께 동작해야 하는 패턴.

<br>

### 2\. Adapter

실제 데이터는 외부 시스템이 가지고 있고, Adapter 컨테이너가 데이터를 App 에 전달해줌. App은 UI만 표출.

<br>

### 3\. Ambassador

App 컨테이너에서 데이터를 만들면, Ambassador 컨테이너가 해당 데이터를 받아서 외부 시스템으로 내보내 주는 것.

<br>

## 5강 문제 풀이

- Create a static pod on node01 called mydb with image redis.Create this pod on node01 and make sure that it is recreated/restarted automatically in case of a failure.
    - Use `/etc/kubernetes/manifests`  as the Static Pod path of example.
    - Kubelet Configured for Static Pods
    - Pod mydb-node01 is Up and running

<br>

- 다음과 같은 조건에 맞는 Pod를 생성하시오.
    - Pod name: myweb, image: nginx:1.14
    - CPU 200m, Memory 500Mi 를 요구하고, CPU 1core, Memory 1Gi 제한 받는다.
    - Application 동작에 필요한 환경변수 DB=mybe를 포함한다.
    - namespace product 에서 동작되어야 한다.  
  

첫 번째 문제

1. static Pod 경로 위치 확인
    - `/var/lib/kubelet/config.yaml`  여기 확인하고 경로를 문제 경로로 변경.
2. 문제 경로 하위에 아래 yaml 파일 생성.  

```yaml
apiVersion: v1
kind: Pod
metadata: 
  name: mydb
spec:
  containers:
    - name: app
      image: redis
```

  

두 번째 문제

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myweb
  namespace: product
sepc:
  containers:
  - name: app
    image: nginx:1.14
    env:
    - name: DB
      value: "mydb"
    resources:
      requests: 
        cpu: 200m
        memory: 500Mi
      limits:
        cpu: 1
        memory: 1Gi
```
