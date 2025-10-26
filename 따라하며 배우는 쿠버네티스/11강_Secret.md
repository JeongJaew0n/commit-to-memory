### ConfigMap과 차이점

- Key-Value 로 저장되는 건 동일.
- Secret은 중요 정보를 저장.
    - auth token, password, ssh key 등
    - base64로 인코딩 됨.

<br>

### Secret 만드는법

- `kubectl create secret <Available Commands> name [flags] [options]` 
    - docker-registry
        - Dodcker 이미지 레지스트리에 접근할 때 인증 정보를 저장해두는 시크릿
        - ```yaml
            kubectl create secret docker-registry my-registry-secret \
            --docker-server=https://index.docker.io/v1/ \
            --docker-username=myuser \
            --docker-password=mypassword \
            --docker-email=myemail@example.com
            ```

    - generic 
        - 로컬 파일, 디렉토리, 평문. 가장 범용적인 타입.
        - `kubectl create secret generic ttabae-secret --from---literal=INVERAL=2 --from-file=./my-dir/` 
    - tls
        - TLS(SSL) 인증서 쌍을 저장할 때 사용.
        - Ingress 리소스 등에서 HTTPS용 인증서를 제공할 때 필요.
        - `kubectl create secret tls my-tls-secret --cert=./tls.crt --key-./tls.key` 

## <br>

### Secret 의 타입

|     |     |
| --- | --- |
| type | 의미  |
| Opaque | 임의의 사용자 정의 데이터 |
| kubernetes.io/service-account-token | 서비스 어카운트 토큰 |
| kubernetes.io/dockercfg | 직렬화된 `~/.dockercfg` 파일 |
| kubetnetes.io/dockerconfigjson | 직렬화된 `~/.docker/config.json` 파일 |
| kubernetes.io/basic-auth | 기본 인증을 위한 자격 증명(credential) |
| kubernetes.io/ssh-auth | SSH를 위한 자격 증명 |
| kubernetes.io/tls | TLS 클라이언트나 서버를 위한 데이터 |
| bootstrap.kubernetes.io/token | 부트스트랩 토큰 데이터 |

<br>

### Secret 사용하기

1. 컨테이너 환경변수(environment variable)로 전달
2. volume mount로 전달
3. 커맨드 라인으로 전달

<br>

1. 컨테이너 환경변수로 전달

```yaml
spec:
  containers:
  - image: image명
    env:
    - name: INTERVAL
      valueFrom:
        secretKeyRef:
          name: ttabae-secret
          key: INTERVAL
... 생략
```

<br>

2. volumne mount로 전달.

```yaml
... 생략
  containers:
  - image: nginx:1.14
    name: web-server
    volumeMouns:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    - name: config
      mountPath: /etc/nginx/conf.d
      readOnly: true
...생략
  volumes: # contianers와 같은 depth
  - name: html
    emptyDir: {}
  - name: config
    secret:
      secretName: ttabae-secret
      items:
      - key: nginx-config.conf
        path: nginx-config.conf
```

<br>

<br>

<br>

### Secret의 용량제한

- secret은 etcd안에 평문 형태로 저장됨. etcd가 메모리를 사용하므로 secret value 가 커지면 메모리 용량을 많이 사용하게 됨.
- secret의 최대 크기는 1MB

<br>