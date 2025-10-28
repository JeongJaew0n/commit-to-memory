# DevOps와 CI/CD의 이해

### Waterfall vs Agile

- 상황에 맞게 각 방법을 선택해야 한다.


- 워터폴 철학
    - 개인간의 소통 향상.
    - 작동하는 소프트웨어
    - 고객과의 협업
    - 변화에 대응
- 기술의 발전. 현재는 클라우드 시대.
- 클라우드 네이티브 개념
    - 클라우드에서 빌드되고 클라우드 컴퓨팅 모델을 최대한 활용하는 워크로드를 디자인, 생성 및 운영하는 접근 방식
    - **철학: “변화에 강하고, 빠르게 적응하는 시스템을 만들자"**

> 워크로드란?  
> 개념: ‘클라우드 위에서 실제로 돌아가는 애플리케이션 단위' ‘인프라가 아닌 **실제 일을 하는 주체**'  
> 구체적으로는: 사용자가 클러스터에 배포해서 실행하는 실제 일거리. CPU, 메모리, 네트워크 등의 자원을 소비하며 동작하는 소프트웨어 단위  
> 쿠버네티스 기준: Pod, Deployment, StattefulSet, DaemonSet, Job, CronJob 등.



### Cloud Native의 요소

- Microservices
    - > MSA 서비스 운영하려면 인스턴스만 있으면 되는 게 아니고 필요한게 많네.

- Containers
- DevOps
    - PLAN & CODE
        - git, svn, jira
    - BUILD
        - maven, gradle
    - TEST
        - selenium, JUnit
    - DEPLOY & OPERATE
        - Docker, Vagrant, Ansible, Terraform
    - MONITOR
        - Nagios, Fluentd

- CI/CD
    - CI
        - compile(build), test, packaging
    - CD
        - 지속적 전달(Delivery)와 지속적 배포(Deployment) 2가지 의미를 가짐.
        - aws에서는 전달은 수동, 배포는 자동의 의미를 가진다고 정의함.

### CI/CD Flow

- IaC: Infrastructure as Code
- containerd
    - `dockerd` 라는 프로그램이 있었다. 이는 Docker 의 핵심 엔진으로, docker cli에서 명령을 받아 컨테이너 생성, 이미지 관리, 네트워크 설정 등을 수행했다.  
k8s의 초창기에는 오직 Docker만 지원했는데, 시간이 흐름에 따라 다른 컨테이너 런타임들이 생겼다.  
Kubernetes에서는 여러 종류의 컨테이너 런타임을 지원하기 위해 CRI(Container Runtime Interface)를 정의했다. 그러나 정작 docker는 이 CRI를 구현하지 않았다. 대신 `dockershim` 이라는 중간 계층(adapter)을 두어 CRI 호출을 `dockerd` 가 이해할 수 있는 형태로 변환했다.  
이후 `dockershim` 이 제거되면서, 현재 K8s는 CRI를 직접 구현한 `containerd` 혹은 `CIR-O` 같은 다른 런타임을 직접 사용하게 되었다.
    - 현재 K8s 의  기본 컨테이너 런타임은 `containerd` 이다. 또한 대부분의 K8s 배포판(Minikube, Kubeadm, GKE, EKS, AKS 등)도 이거 사용.
    - Red Hat 계열(OpenShift)는 `CRI-O` 를 기본으로 사용.



## Jenkins

> java 말고도 python, nodeJs 등 지원하는구나.  



- 단계: DEV → UAT(User Acceptance Tests) → PROD
- DSL
    - Domain Specific Language
    - jenkinsfile, dockerfile



### Jenkins 설치

구글에 jenkins 검색

docker hub: [https://hub.docker.com/r/jenkins/jenkins](https://hub.docker.com/r/jenkins/jenkins)  



실행 명령어: 

```yaml
docker run -d -v jenkins_home:/var/jenkins_home -p 8080:8080 -p 50000:50000 --restart=on-failure --name=jenkins-server jenkins/jenkins:lts-jdk21  
```

- \-d: detach 모드. 백그라운드에서 컨테이너를 실행.
- \-v: 볼륨 마운트. `jenkins_home` 이라는 도커 볼륨을 컨테이너 안의 `var/jenkins_home` 디렉터리에 연결함.
    - `docker volume ls` 를 통해서 확인 가능.
    - `docker volume inspect {volume 명}` 을 통해서 경로 확인 가능. 이 경로는 docker 가 돌아가는 vm 상의 경로임. 아래 그림 참고.
- \-p 50000:50000: Jenkins 에이전트(노드)가 서버와 통신할 때 쓰는 JNLP 포트 매핑



```yaml
┌────────────────────────────┐
│ macOS (호스트 OS)          │
│   └── Docker Desktop (VM)  │   ← 리눅스 환경 (Docker 엔진이 여기서 실행됨)
│         └── Docker Engine  │
│              └── Container │ ← Jenkins, Nginx, MySQL 등 각각 독립된 공간
└────────────────────────────┘
```



### 초기설정

- JDK 경로 잡기
    - jenkins settings → tools에서 **JDK 경로 잡아주기**
    - **만약 jdk내장 버전이라면 안잡아줘도 됨.**



### 용어

- Item: Jenkins 작업의 최소 단위.



### 한 거 정리.

- jenkins 설치 & 실행 후 docker 로 확인.
- item free style로 만든 다음, docker 명령어 통해서 workspace 찾아가기
- workspace 하위 프로젝트 폴더 하위에는 빌드 결과물이 생성된다.