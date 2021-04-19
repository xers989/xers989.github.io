---
layout: post
title: Kubernetes-오케스트레이션과 Pod배포
date: 2021-03-16 16:05 +0900
categories: OKE kubernetes
---
# Kubernetes 구조 및 배포
#### Kubernetes 구조
Kubernetes는 컨테이너를 멀티 호스트 환경에서 효율적으로 운영하기 위한 도구로 컨테이너의 시작, 정지 및 네트워크, 컨테이너 구동에 대한 스케쥴링을 제공하는 툴(컨테이너 오케스트레이션)이다.
중요한 개념은 desired state 즉 원하는 상태를 유지시켜주는 개념이다. 주기적으로 사용자가 원하는 상태 값고 현재 운영되고 있는 값을 체크 하여 그 차이를 비교 하고 차이 나는 부분을 처리 해주는 것이다. 즉 컨테이너가 3개를 생성 운영 하도록 하였을 때 어떠한 사유에 의해 컨테이너 1개가 정지 되어 현재 2개 컨테이만 운영 중이라면 Kubernetes는 차이나는 1개에 대해 처리하여 줌으로 목표 하고 있던 3개 컨테이너를 유지 시켜 주는 것이다.  
이러한 컨테이너 관리를 위해 기본 구조는 관리 부분인 Master (혹은 Control Plane) 와 컨테이너가 실행 되어 운영되는 Node (혹은 Data Plane)로 기본 구성이 된다. Master 의 경우 모든 요청을 처리하기 위한 API Server (kube-apiserver), 데이터를 저장 하고 있는 저장소 (ETCD), Pod의 배치를 제어하는 스케쥴러와 클러스터를 감시 하는 컨트롤러로 구성 된다.   
![](/image/kubernetes2/kubernetes02.png){: width="90%" height="90%"}   

#### API Server
리소스 관리를 위한 서버로 REST API를 제공 한다. 사용자가 접근 할 수 있는 유일한 창구로 모든 명령은 API Server를 통해서만 가능 하다. 사용자의 요청을 받아서 유효성을 검증하며 수평 확장이 가능 한 구조이다.    
제공되는 API는 /{group}/{version}/namespaces/{namespace}/{resource} 형태이다. 
API가 동작 하는 구조는 다음과 같으며 사용자 요청을 받아서 인증 및 인가 처리를 한 후 Admission 처리, Validation이 처리 된다.   
![](/image/kubernetes2/kubernetes03.png){: width="100%" height="100%"}   
Mutation Admission은 사용자 요청을 Hooking하여 플러그 인을 이용한 확장 기능을 적용 할 수 있게 한다. 요청한 API의 매니페스트 내용 중 특정한 값으로 변경하는 작업을 수행하한다. 간단한 예가 Pod를 배포 할 때 사용 할 수 있는 리소스에 제한을 두기 위해 LimitRange를 줄 수 있는데 이를 적용 하지 않은 요청이 온 경우 중간에 이를 가로 채서 기본값으로 채워 주는 역할을 할 수 있다.
Object Schema Validation은 객체의 값이 올바르게 입력 되었는지 검증하게 된다. Validation Admission은 사용자 지정 정책에 따라 사용자 요청을 최종 허용/거부를 결정하여 주는 역할을 한다. Admission의 경우 클러스터 버전이 1.16 이상에서 사용하는 것을 권장한다.  
모든 과정이 통과된 경우 데이터는 데이터 베이스에 저장 된다. 데이터 베이스 영역은 가상화 레이어로 제공됨으로 다양한 종류의 데이터베이스를 사용 할 수 있다. 데이터는 Key-Value 형태로 저장 되며 ETCD를 사용 하는 경우가 많다.  

#### ETCD
고가용성을 제공하는 Key-Value 저장소로 Kubernetes에서 필요한 데이터를 저정하는 데이터 베이스이다. 분산형 합의 기반 시스템으로 클러스터 형태로 복수개로 인스턴스를 구성하거나 API Server와 분리된 노드에 설치 할 수 있다.


#### Scheduler
클러스터 내부에 자원 할당이 가능한 노드 중 알맞은 노드를 선택 하여 자원을 배포해 주는 역할을 한다.


#### Kube-Controller-Manager
노드에 실행되고 있는 Pod를 관리하는 컨트롤러로 노드 컨트롤러, 레플리케이션 컨트롤러, 엔드포인트 컨트롤러, 서비스 어카운트, 토큰 컨트롤러를 포함하고 있다. 

#### Cloud-Controller-Manager
클라우드 자원를 사용하기 위한 컨트롤러로 클라우드 플랫폼과 상호 작동하기 위해 클라우드 서비스 제공자 전용 컨트롤러로 Kube-Controller-Manager와 구분되어 사용 된다. 노드, 라우드, 서비스 컨트롤러가 있다. (서비스 제공자에 따라 다를 수 있다)

#### Pod
Pod는 Kubernetes가 유지 시켜 주기 위한 기본 단위로 최소 1개의 컨테이너를 포함하는 구조로 되어 있다. 아래 그림 처럼 Pod는 한개 이상의 컨테이너와 스토리지, 네트워크로 구성된 단위라고 보면 된다.   
![](/image/kubernetes2/kubernetes01.png){: width="70%" height="70%"}   
Pod를 배포 하기 위한 매니페스트는 다음 내용을 포함 한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
 name: www
 labels:
  name: nginx
  method: Pod
spec:
 containers:
 - name: nginx
   image: nginx
   ports:
    - containerPort: 80
   volumeMounts:
    - mountPath: /etc/www
      name: www-data
   resources:
    requests:
     cpu: 800m
     memory: 8G
 restartPolicy: OnFailure
 volumes:
  - name: www-data
```
metadata 항목은 이름과 레이블을 지정 할 수 있다.(레이블은 이후 Pod를 선택 하기 위한 검색값으로 사용된다) spec 항목에 Pod에 들어갈 컨테이너에 대한 정보로 nginx (dockerhub에서 검색 하며 버젼을 명기 할 수 있다), 오픈하는 포트 정보 및 마운트 관련 정보를 가지고 있다.
resources 항목에 컨테이너에 할당할 리소스를 설정 하는 것으로 최대 CPU, Memory 요청하는 CPU, Memory 값을 설정 할 수 있다.
요청 하는 리소스가 배치 가능한 노드가 선택 되어 배치 되며 노드별 가능한 용량을 넘어서는 자원이 요청 되는 경우 오류가 발생 된다.
Pod 가 배치된 노드의 터미널에 접속 하여 컨테이너 프로세스 정보(docker ps)를 보면 다음 과 같은 내용을 볼 수 있다.
```bash
[opc@oke ~]$ docker ps -a
CONTAINER ID  IMAGE   COMMAND                 CREATED         STATUS   PORTS NAMES
115555731b61  nginx   "/docker-entrypoint.…"  21 hours ago    Up 21 hours    k8s_nginx_www_default_35ca8888-0f55-4af3-98ef-7d1d787be985_0
71f8d996b150  ap-seoul-1.ocir.io/axoxdievda5j/oke-public-pause-amd64  "/pause"  22 hours ago  Up 22 hours    k8s_POD_www_default_35ca8888-0f55-4af3-98ef-7d1d787be985_0
...
```
Pod 가 배치 되면 지정된 컨테이너(nginx) 뿐만 아니라 Pause 컨테이너가 하나 추가 배포된 것을 볼 수 있다. Pause 컨테이너는 Pod내에 존재하는 컨테이너의 부모와 같은 역할을 한다. 해당 Pod의 환경 정보를 컨테이너와 공유하여 주며 라이프 사이클에 대한 관리 등을 하여 준다. Pod가 배포 될 때 Pause 컨테이너가 배포되면서 노드 머신에 네임스페이스를 생성하고 mnt, pid,net,ipc 등 네임스페이스 정보를 이후 생성 될 컨테이너와 공유 한다. 이러한 이유로 Pod내의 컨테이너는 다른 노드에 분리되어 설치 될 수 없다.
Pod의 구조는 다음과 같다고 볼 수 있다.    
![](/image/kubernetes2/kubernetes04.png){: width="70%" height="70%"} 
