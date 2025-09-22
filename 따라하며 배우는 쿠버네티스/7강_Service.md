# 7강. Service

- 서비스 개념
- 서비스 타입
- 서비스 사용하기
- 헤드리스 서비스
- kube-proxy

<br>

## 7-1. 쿠버네티스 Service 개념과 종류

<br>

 Service API는 로드밸런서 처럼 각 pod에 가는 요청을 분산시켜줌.

왜냐면 하나의 pod에만 일이 집중될 수 있기 때문.

<br>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webui-svc
sepc:
  clusterIP: 10.96.100.100 // 생략 많이 함
  selector:
    app: webui
  ports:
  - protocol: TCP
    port: 80 // clusterIp의 포트
    targetPort: 80 // pod의 포트
```

<br>

### 4가지 Type 지원

- ClusterIP
    - 단일 진입점 VIP를 만드는 가장 기본 타입.
- NodePort
    - ClusterIP 생성 + 각 노드의 포트도 같이 개방
- LoadBalancer
    - ClusterIP 생성 + NodePort 개방 + 로드밸런서를 통해서 각 노드에 가는 요청 분산
    - 실제 LB 장비가 설치되어야 함.
    - AWS, AZure 와 같은 특정 플랫폼 내에서만 쓸 수 있음
- ExternalName
    - DNS 서비스 같은 거임.
    - 클러스터 내부에서 특정 이름으로 도메인쓸 수 있게 해주는 것

<br>

## 7-2. 쿠버네티스 Service 4가지 종류 실습해보기

### ClusterIP

```yaml
apiVersion: v1
kind: Service
metadata:
  name: clusterip-service
spec:
  type: ClusterIP
  clusterIP: 10.100.100.100
  selector:
    app: webui
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

<br>

- selecotr의 label 이 동일한 파드들을 그룹으로 묶어 단일 진입점(VIP)을 생성.
- 클러스터 내부에서만 사용가능
- type 생략 시 default 값으로 10.96.0.0/12 범위에 할당됨
    - 실무에서는 생략해서 랜덤하게 쓰는 경우가 많은데, 이래야 중복을 피하기 때문이다.

<br>

### NodePort

- 모든 노드를 대상으로 외부 접속 가능한 포트를 예약
- Default NodePort 범위: 30000-32767
- ClusterIP 를 생성 후 NodePort를 예약

<br>

`ClusterIP:포트번호`  이렇게 접근하면 pod 중 하나에 랜덤 접근.

<br>

### LoadBalancer

- Public Cloud(AWS, Azure, GCP 등)에서 운영가능
- LB를 자동으로 구성 요청
- NodePort를 예약 후 해당 nodeport로 외부 접근을 허용

<br>

LB장비가 있고, 여기에 setting을 요청하는 거임.

설정이 잘 되면, `kubectl get svc`   했을 때 , LB장비 ip가 나옴.

<br>

### ExternalName

- 클러스터 내부에서 External(외부)의 도메인을 설정

<br>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: externalname-svc
spec:
  type: ExternalName
  externalName: google.com
```

<br>

다른 애들이랑 다름. DNS 서비스를 지원해줌.

`etc/hosts`  파일과 같은 역할.

etcd에 저장됨.

<br>

`curl externalname-svc.default.svc.clutser.local` 

- `default.svc.cluster.local` : k8s가 사용하는 내부적으로 사용하는 도메인 명.

### <br>

## 7-3. Headless Service, Kube Proxy

### Headless Service

- ClusterIP가 없는 서비스로, **단일 진입점이 필요 없을 때** 사용
- Service와 연결된 Pod의 endpoint로 DNS 레코드가 생성됨
- Pod의 DNS 주소: `pod-ip-addr.namesapce.pod.cluster.local` 
    - 이 레코드 정보는 `coreDNS` 에 저장됨.

<br>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: headless-svc
spec:
  type: ClusterIP
  clusterIP: None # 이게 핵심
  selector:
    app: webui
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

<br>

<br>

`/etc/resolv.conf` : dns 서버 위치가 저장되어 있음.

pod로 띄워진 리눅스에서 저기에 접근하면 master가 제공하는 coreDNS 주소가 들어감.

<br>

### Kube Proxy

- Kubernetes Service의 backend 구현
- endpoint 연결을 위한 iptables 구성
- nodePort로의 접근과 Pod 연결을 구현(iptables 구성)

<br>

총 3가지 모드가 있음.

- userspace
    - k8s 초기 버전에서 잠깐 사용됐었음.
    - 클라이언트의 요청이 NodePort로 들어오면 그걸 iptables가 해석하고 kube-proxy로 보냄.
- iptables
    - 현재 디폴트 모드임.
    - kube-proxy는 service API 요청 시 iptables rule 생성함.
    - kube-proxy가 NodePort에 대해 Listen하고 있다가 클라이언트 요청오면 받아서 iptables의 룰을 참고해서 연결.
- IPVS