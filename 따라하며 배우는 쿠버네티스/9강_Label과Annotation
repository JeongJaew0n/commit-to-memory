## 9-1,9-2. Label 레이블

- Node를 포함하여 pod, deployment 등 모든 리소스에 할당,
- 리소스의 특성을 분류하고, Selector를 이용해서 선택,
- Key-value 한쌍으로 적용,

많은 종류의 resource들을 원하는 목적에 따라 분류하기 위해 사용.

### Label과 Selector,

- Lable,

```
metadata:
  labels:
    rel: stable
    name: mainui
```

- Selector,

```
selector:
  matchLables:
    key: value
  matchExpressions:
    - {key: name, operator: In, values: [mainui]}
    - {key: rel, operator: NotIn, values: ["beta","canary"]}
```

공식문서: \[[https://kubernetes.io/ko/docs/concepts/overview/working-with-objects/labels/\](https://kubernetes.io/ko/docs/concepts/overview/working-with-objects/labels/)](https://kubernetes.io/ko/docs/concepts/overview/working-with-objects/labels/]\(https://kubernetes.io/ko/docs/concepts/overview/working-with-objects/labels/\))  

<br>

> 레이블의 key에는 `접두사/이름`  이 올 수 있다.  
> 보통 그냥 rel: stable 하면 이름만 써준 것.  
> 이름 제약조건: 시작과 끝은 알파벳과 숫자만 가능. 대시,밑줄,점 사용 가능  
> 접두사 제약조건: DNS의 하위 도메인으로 해야 한다. 점과 슬래시로 구분되는 253자 이하.

- 레이블 보기: `kubectl get pods --show-labels`
- 레이블로 선택(l이 selector의 축약형임)
    - `kubectl get pods -l name=mainui` 
        - `kubectl get pods --selector name=mainui` 

- 레이블 할당
    - `kubectl label pod pod-demo name=test` 
        - `kubectl label pod pod-demo name=changed --overwrite` ,

- 레이블 삭제: `kubectl label pod pod-demo name-` ,

## 9-3 Annotation(애너테이션)

- Label과 차이점: label은 k8s api 에 사용됨(selector 사용 가능). 애너테이션은 관리자에게 알림용. selector 사용 불가.,
- Label과 동일하게 key-value를 통해 리소스의 특성을 기록,
- 2가지 목적으로 사용 됨.
    - K8s에게 특정 정보를 전달할 용도로 사용
        - 예를 들어 Deployment의 rolling update 정보 기록,

```
annotations:
  kubernetes.io/chage-cause: version 1.15
```

- 관리를 위해 필요한 정보를 기록할 용도로 사용
    - 릴리즈, 로깅, 모니터링에 필요한 정보들을 기록

```
annotations:
  builder: "seongmi Lee(seongmi.lee@gmail.com)"
  buildDate: "20250929"
  imageRegistry: "https://hub.docker.com/"
```

예시

```
apiVersion: v1
kind: Pod
metadata:
  name: annon-pod
  anntations:
    imageRegistry: "https://hub.docker.com/"
```

<br>

## 9-4. Canary 배포

### Pod를 배포하는 방법,

- Blue-Green: 구 버전인 Blue 버전이 돌아가는 중에, 신 버전인 Green 버전을 띄워서 Green 버전이 이상 없으면 트래픽을 Green 버전으로 돌리는 방식.
- Rolling: 구 버전과 신 버전을 동시에 띄우고, 구 버전의 트래픽을 점차 줄여 신 버전으로 옮기는 방식.
- Canary: 구 버전과 신 버전을 동시에 띄우고, “특정 사용자”의 트래픽을 조금씩 신 버전으로 옮겨 테스트 해보는 방식.

### Canary 배포,

- 기존 버전을 유지한 채로, 일부 버전만 신규 버전으로 올려서 신규 버전에 버그나 이상은 없는지 확인

**Blue-Green vs Canary** 두 방식은 유사하지만 철학적이 다름. Blue-Green 은 한 번에 확 바꾸는 방식이고, Canary는 실제 사용자를 조금씩 신 버전에 노출 시켜 테스트 해보는 방식임. **Canary vs Rolling** 이 두 방식도 유사하지만 살짝 다름. Canary는 ‘의도적으로' 구버전과 신버전을 혼용하며 각각의 트래픽을 조절함. 또한 굳이 테스트만하고 신버전으로 업데이트를 안해도 됨. 하지만 Rolling은 신버전으로 교체 하는 과정에서 어쩔 수 없이 구버전과 신버전이 혼용됨.