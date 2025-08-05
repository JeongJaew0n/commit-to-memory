# 6강. Controller

<br>

### Controller 란?

- Pod의 개수를 보장
    - API, etcd, controller가 협업한다.

<br>

## 6-1. ReplicationController

가장 기본적인 Controller.

k8s v1에서 만들어졌다.

<br>

- 요구하는 Pod의 개수를 보장하며, 파드 집합의 실행을 항상 안정적으로 유지하는 것을 목표
    - 요구하는 Pod의 개수가 부족하면 template을 이용해 Pod 추가
    - 요구하는 Pod 수 보다 많으면 최근에 생성된 Pod를 삭제
- 기본 구성
    - selector
        - Pod의 label을 기준으로 한다.
        - label: 메타데이터 같은 것. Pod의 yaml에 `metadata.labes` 에 선언할 수 있음.
        - pod의 label 개수가 많은 것은 ㄱㅊ. 그런데 selector 보다 적으면 에러.
    - replicas
        - pod 개수
    - template
        - 부족하면 생성할 pod 의 yaml
        ```yaml
        apiVersion: v1
        kind: ReplicationController
        metadata:
          name: <RC_이름>
        spec:
          replicas: <배포개수>
          selector:
            key: value
          template:
            <컨테이너 템플릿>
        ```

<br>

- pod 생성되는 이력 확인 방법
    - `kubectl describe rc <RC_이름>`  이후 Events 항목 확인
- 실행되는 pod들의 label 확인 방법
    - `kubectl get pods --show-labels` 
- RC 수정
    - yaml 수정: `kubectl edit rc <RC_이름>` 
    - scale 수정: `kubectl scale rc <RC_이름> --replicas=2` 

<br>

- template 을 수정하면 기존 pod 바뀔까?
    - 아니! RC는 selector 만 보기 때문에 변하지 않는다.
    - 기존 pod를 삭제해야 한다. 운영 중에 업데이트 하는 것을 Rolling Update 라고 한다.

<br>

### 문제

1. 다음의 조건으로 ReplicatonController를 사용하는 rc-lab.yaml 파일을 생성하고 동작시킵니다.
    - labels(name: apache, app:main, rel:stable)를 가지는 httpd:2.2 버전의 Pod를 2개 운영합니다.
        - rc name: rc-mainui
        - container: httpd:2.2
    - 현재 디렉토리에 rc-lab.yaml 파일이 생성되어야 하고, 애플리케이션 동작은 파일을 이용해 실행합니다.

<br>

2. 동작되는 http:2.2 버전의 컨테이너를 3개로 확장하는 명령을 적고 실행하세요.

<br>

정답

**1번 문제**

1. vi rc-lab.yaml

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: rc-mainui
spec:
  selector:
    name: apache
    app: main
    rel: stable
  replicas: 2
  template:
    metadata:
      labels:
          name: apache
          app: main
          rel: stable
    spec:
      containers:
        - image: httpd:2.2
          name: httpd
```
2. kubectl create -f rc-lab.yaml

<br>

**2번 문제**

kubectl scale rc rc-mainui --replicas=3

<br>

## 6-2. ReplicaSet

### <br>

### ReplicationController 와 차이점

- RC와 같은 역할을 하는 컨트롤러
- RC에 비해 풍부한 selector  

```yaml
selector:
  matchLabels:
   key: value
  matchExpressions:
   - {key: tire, operator: In, values: [cache]}
   - {key: environment, operator: NotIn, values: [dev]
```

  
- matchExpressions 연산자
    - In: Key와 values를 지정하여 key, value가 일치하는 Pod만 연결
    - NotIn: key는 일치하고 value는 일치하지 않는 Pod에 연결
    - Exists: key에 맞는 label의 Pod를 연결 - **key만 있으면 됨.**
    - DoesNotExist: key와 다른 label의 Pod를 연결 - **key만 있으면 됨.**

<br>

### 중요

- Controller 지우면 자신이 생성한 Pod까지 같이 삭제됨!
    - Controller만 지우고 Pod는 남기고 싶으면?: `kubectl delete rs <RS_이름> --cascade=false` 
    - 컨트롤러가 만들지 않은 Pod는 삭제되지 않음.
- redis, nginx, 뭐 기타 이미지 상관 없이 label 기준으로 관리하기 때문에 label을 여러 개 주는 게 좋음!

<br>

### 문제

1. 다음의 조건으로 ReplicaSet을 사용하는 rs-lab.yaml 파일을 생성하고 동작시킵니다.
    - labels(name: apache, app:main, rel:stable)를 가지는 httpd:2.2 버전의 Pod를 2개 운영합니다.
        - rs name: rs-mainui
        - container: httpd:2.2
    - 현재 디렉토리에 rs-lab.yaml 파일이 생성되어야 하고, 애플리케이션 동작은 파일을 이용해 실행합니다.

<br>

2. 동작되는 http:2.2 버전의 컨테이너를 1개로 축소하는 명령을 적고 실행하세요.

<br>

<br>

**정답**

1번 문제

1. vi rc-lab.yaml 이후 아래 yaml 입력

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rs-mainui
spec:
  selectors:
    matchLabels:
      name: apache
      app: main
      rel: stable
  replicas: 2
  template:
    metadata:
        name: apache
        app: main
        rel: stable
    spec:
      containers:
        - name: httpd
          image: httpd:2.2
```

<br>

2. kubectl create -f rc-lab.yaml

<br>

2번 문제

`kubectl scale rs rs-mainui --replicas=1` 

<br>

<br>

## 6-3. 쿠버네티스 RollingUpdate를 위한 Deployment

### 소개

- 역할: 사실은 ReplicaSet을 제어해주는 부모 역할을 함.
- Deployment가 ReplicaSet을 조절한다.
- **RollingUpdate**: 파드 인스턴스를 점진적으로 새로운 것으로 업데이트하여 디플로이먼트 업데이트가 서비스 중단없이 이뤄질 수 있도록 해준다.
- 동작방식: Deployment에 nginx1.14 버전으로 실행하라고 입력하면 얘가 ReplicaSet에게 명령함. Deployment를 nginx1.15로 바꾸면 얘가 또 명령내려서 수정함.

<br>

**yaml 예시**

- ReplicaSet과의 차이점: kind 말고는 없음.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deploy
spec:
  selector:
    matchLabels:
      app: webui
  replicas: 3
  template:
    metadata:
      labels:
        app: webui
    spec:
     containers:
       - name: web 
         image: nginx:1.14
         ports:
           - containerPort: 80
```

<br>

<br>

### Rolling Update & Rolling Backup

- ⁠Rolling Update
    - `kubectl set image deployment <deployment_name> <container_name>=<new_version_image>` 
    - 과정
        - 새로운 RS 생성
        - 새로운 RS에서 업데이트 된 Pod 생성. Running 상태가 되면 Old RS에서 Old Pod 삭제.

- Rolling Backup

        - `kubectl rollout undo deployment <deploy_name>`  
            - `--to-revision=<history_number>` : 해당 history number로 이동함.
            - history number 선언안하면 바로 직전 history 로 이동하고, 해당 history 는 사라짐.
                - 예) 1~5가 있을 때, 5가 최신. 5에서 3으로 지정해서 가면 3이 사라짐. 3에서 history number 없이 rollback 하면 5로 가고 5 사라짐.

- history 보기
    - 파일 생성, update, backup 모두 `--record`  옵션을 붙여줘야 기록됨.
    - `kubectl rollout history deployment app-deploy` 
- rollout status 보기
    - `kubectl rollout status deployment app-deploy` 
- 멈추기, 재시작
    - 멈추기: `kubectl rollout pause deployment app-deploy` 
    - 재시작: `kubectl rollout resume deployment app-deploy` 

<br>

### 관련 yaml 필드값

```yaml
... //생략
metadata:
  annotation:
  kubernetes.io/change-cause: version 1.14
spec:
  progressDeadlineSeconds: 600
  revisionHisotryLimit: 10
  startegy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  replicas: 3
.. //생략
```

<br>

- 위 값은 모두 선언하지 않았을 경우 들어가는 default 값임
- `sepc.revisionHistoryLimit: 10` : history를 10개까지만 남기겠다.
- `spec.progressDeadlineSeconds: 600` : 10분동안 업데이트가 되지 않으면 업데이트 하지 않겠다.
- `sepc.startegy.rollingUpdate.maxSurge: 25%` :
    - replicas 3의 25%는 0.75, 반올림하면 1. 3+1 = 4.
    - Running 중인 걸 4개까지만 운영하겠다라는 뜻.
- `sepc.startegy.rollingUpdate.maxUnavailable: 25%` 
    - terminating 되는 개수를 4개 까지 제한한다.
- `type: RollingUpdate` : 당장 RollingUpdate 하겠다.

<br>

### annotation  기능

쿠버네티스 동작 방식 서포트.

- `⁠kubernetes.io/change-cause: version 1.14`
    - 이 버전이름으로 history 가 남겨진다.
    - 다음에 이 yaml에서 위 버전을 수정해주면 Rolling Update가 된다. yaml로 하는 것이다.
    - 마찬가지로 history 명령어로 확인 가능.

<br>

<br>

## 6-4. DaemonSet

### 소개

- 전체 노드에 Pod가 한 개씩 실행되도록 보장.
- 로그 수집기, 모니터링 에이전트와 같은 프로그램 실행 시 적용.
- CNI는 데몬 셋으로 실행되고 있다.

<br>

**yaml 예시**

- RS와 차이점: kind랑 replicas 없는 거.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: app-deploy
spec:
  selector:
    matchLabels:
      app: webui
  template:
    metadata:
      labels:
        app: webui
    spec:
     containers:
       - name: web 
         image: nginx:1.14
         ports:
           - containerPort: 80
```

<br>

<br>

### Node 늘리기

1. `kubeadm token list` 
    - 토큰 리스트 보기. sudo 권한 필요함.
2. `kubeadm token create --ttl 1h` 
    - 1시간 짜리 토큰 만들기.
3. 늘리려는 인스턴스(노드)에 접근 하여 아래 명령어 실행
    - `kubectl reset` 
        - node 초기화.
        - 주의! 절대 master에서 실행하지 마시오

4. `kubeadm join <master_ip> --token <token>` 

<br>

### 추가 정보

- 노드 늘리면 늘어난 노드에서도 실행됨.
- Pod지우면 새로 실행 됨.
- Rolling Update 지원함.
- 요약어: `ds` 

<br>

**Rolling Update**

- `kubectl edit daemonsets.apps <daemonset_name>`  
- 컨테이너 이미지 edit으로 변경하면 업데이트가 됨.

<br>

**Rolling Backup**

- `kubectl rollout undo daemonset <daemonset_name>`

## 6-5. StatefulSet

### 소⁠개

Pod의 상태를 유지해주는 컨트롤러

- Pod 이름
- Pod의 볼륨

컨트롤러에서 만들어지는 Pod는 랜덤 해시값을 사용함.

그래서 ⁠pod-2ccm9 이런식의 이름이될 텐데, SF를 쓰면 pod-0, pod-1 이런식으로 일정하게 번호가 늘어나는 이름을 가짐.

<br>

DaemonSet과의 차이점.

- 노드 당 1개 보장은 아님.

<br>

yaml 예시

```yaml
apiVersion: apps/v1
metadata:
  name: sf-nginx
spec:
  replicas: 3
  serviceName: sf-service
# podManagementPolicy: OrderedReady
  podManagementPolicy: Parallel
  selector:
    matchLabels:
      app: webui
  template:
```

- serviceName 필수!
- podManagementPolicy: OrderedReady가 defulat, 순차적으로 실행 됨. Parallel은 0,1,2번을 동시에 실행함.

<br>

**Scale Up & Down**

`kubectl scale sf <sf_name> --replicas=<개수>` 

<br>

**Rolling Update & Rolling Backup 가능**

- `kubectl edit statefulsets.apps <sf_name>` 으로 업데이트.
    - 최신 생성된 pod부터 종료되고 새로 실행됨.
    - 예) nginx 1.14에서 1.15로 업데이트 했을 경우 pod-0,1이 있으면 1번이 종료되고 업데이트 된다음 0번이 종료되고 업데이트 됨.
- `kubectl rollout undo sf <sf_name>` 

<br>

<br>