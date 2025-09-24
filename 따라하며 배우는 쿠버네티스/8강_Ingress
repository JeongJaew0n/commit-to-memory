# 8강 Ingress (인그레스)

## 8-1. Ingress, Ingress Controller

- HTTP나 HTTPS를 통해 클러스터 내부의 서비스를 외부로 노출
- 기능
    - Service에 외부 URL을 제공
    - 트래픽을 로드 밸런싱
    - SSL인증서 처리
    - Virtual hosting을 지정

<br>

### Ingress 설정 방법

1. Ingress Controller를 설치함.
    - k8s에서 공식적으로 지원하는 것도 있고 외부 구현체들도 많음.
    - k8s가 공식 서포트하는 것은 AWS, GCE, nginx
2. Ingress Controller에 Ingress Rule을 설정.
    - 이 rule을 통해서 virtual hosting, 로드 밸런싱 등이 가능함.
3. rule을 참조하여 요청을 각 Service로 요청 보냄.

<br>

### Ingress Controller 설치

- k8s 공식 사이트에서 설치하고자 하는 ingress controller 설치.
    - nginx 같은 경우 yml파일을 다운받을 수 있게 제공하고 있음.
        - [https://kubernetes.github.io/ingress-nginx/deploy/](https://kubernetes.github.io/ingress-nginx/deploy/)
        - kubectl apply -f [https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.13.2/deploy/static/provider/cloud/deploy.yaml](https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.13.2/deploy/static/provider/cloud/deploy.yaml)

- nginx yml 파일 뜯어보기
    - pod 1개 띄워짐.(ingress controller)
    - 외부에서 연결되기 위해 Service가 제공되는데, NodePort 로 제공됨.

<br>

## 8-2. Ingress를 이용한 웹서비스 운영

IngressRule이 곧 kind가 Ingress 인 k8s 객체임.

해당 객체의 yaml 파일에서 설정가능.

```yaml
apiVersion: v1
kind: Ingress
metadata:
  name: marvel-ingress
sepc:
  rules:
  - http:
      paths:
      - path: /
        backend:
          serviceName: marvel-service
          servicePort: 80
      - path: /pay
        backend:
          serviceName: pay-service
          servicePort: 80
```

<br>

- 인증서 만들어서 집어 넣을 수도 있음.
- 세션 어피니티 기능도 넣을 수 있음.

<br>

### 정리

Ingress는 Service의 NodePort로 동작함. 이 Service는 `Ingress Controller` 파드 안에 있음.

그리고 설정된 포트로 들어오면 Ingress Controller의 Service가 받아서 Ingress 규칙에 따라 처리함.