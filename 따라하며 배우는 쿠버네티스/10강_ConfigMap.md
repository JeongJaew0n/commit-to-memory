### ConfigMap 생성

- ConfigMap: 컨테이너 구성 정보를 한곳에 모아서 관리
- 생성: `kubectl create configmap NAME [--from-file=source] [--from-literal=key1=value1]` 
    - 개수 제한은 없지만 파일은 1MiB를 넘을 수 없음.
- 조회: `kubectl get configmap` `kubectl describe configmap [이름]` 

<br>

### ConfigMap 일부분 적용하기.

생성한 CnofigMap의 key를 pod의 컨테이너에 적용

<br>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: genid-stone
spec:
  containers:
  - image: smlinux/genid:env
    env:
    - name: INTERVAL
      valueFrom:
        configMapKeyRef:
          name: ttabae-config
          key: INTERVAL
    name: fakeid
    volumeMounts:
    - name: html
      mountPath: /webdata
  - image: nginx:1.14
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    ports:
    - containerPort: 80
  volumes:
  - name: html
    emptyDir: {}
```

<br>

수정: `kubectl edit configmap [이름]` 

<br>

### ConfigMap 전체 적용하기.

<br>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: genid-stone
spec:
  containers:
  - image: smlinux/genid:env
    env:
    - name: INTERVAL
      envFrom:
        configMapRef: # 여기 주의!
          name: ttabae-config
    name: fakeid
    volumeMounts:
    - name: html
      mountPath: /webdata
  - image: nginx:1.14
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    ports:
    - containerPort: 80
  volumes:
  - name: html
    emptyDir: {}
```

<br>

### ConfigMap을 볼륨으로 적용하기

```yaml
spec:
  containers:
  - image: nginx:1.14
    name: web-server
    ports:
    - containerPort: 80
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
      - name: config
      mountPath: /etc/nginx/conf.d
      readOnly: true
  volumes:
  - name: config
    configMap:
      name: ttabae-config
      items:
      - key: nginx-conf.conf
        path: nginx-conf.conf # 볼륨으로 전달될 때 이름
```

- config란 이름의 볼륨을 마운트 할거다. 이 볼륨은 ttabae-config 의 nginx-conf.conf 값을 가진다. 볼륨 마운트 될 위치는 `/etc/nginx/conf.d` 이다.
- mountPath는 폴더임. 이 아래에 `nginx-conf.conf`  라는 이름으로 파일이 생성됨. 만약 item을 더 추가한다면 그만큼 파일이 더 생김.

<br>