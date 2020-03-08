---
layout: post
title:  Container Repository
date:   2020-03-06 16:57:53 +0900
categories: OKE Container Respository
---
# Container Repository
Oracle Kubernetes Engine 는 Docker 를 기반으로 하는 서비스로 컨테이너 이미지를 보관하고 공유 하기 위한 서비스가 필요하다. Oracle cloud를 사용 한 다면 무료로 Registry를 사용 할 수 있으며 Registry 접근을 위한 사용자 관리가 제공 되니 활용 하는 것도 좋을 것이다.
Cloud 계정이 있다면 OCI Console의 개발자 메뉴에서 OCIR (Oracle Cloud Infrastructure Registry) 을 볼 수 있다. Oracle Cloud를 사용 하면 홈 리전을 선택 하여 기본 데이터 센터를 선택 하게 되는데 이때 Tenancy의 Object Storage Namespace가 생성 된다. Registry는 해당 Object Storage Namespace (이를 tenancy namespace 라고도 한다)를 이용하여 접근 한다.
우선 이미지가 저장될 Repostory 를 생성 하여 보자.   
![](/image/ocir/ocir-1.png)

Create Repository를 클릭 하고 이름을 지정하여 준다. 접근 정책은 외부 사용을 제한 하기 위해 Private 로 선택 한다.  
![](/image/ocir/ocir-2.png)

Registry의 사용자는 Cloud의 IAM에서 관리되는 사용자로 접근이 가능하다. 로그인 정보는 파일에 기록 되어 해당 PC 에서는 로그인 된 상태가 저장 되게 된다. 안전한 접근 권한 및 보안을 위한 IAM은 AuthToken을 제공 하고 있다. 즉, 패스워드를 넣지 않고 임의 생성된 Toke을 이용하여 접근 하도록 하여 패스워드의 유출을 방지 하는 것이다. AuthToken은 사용자 정보에서 생성이 가능하며 한번 생성 된 AuthToken은 이후로 다시 조회가 불가능 하다.   
![](/image/ocir/ocir-3.png)

콘솔에서 다음 명령어를 이용하여 생성한 Registry에 로그인을 한다.
iad.ocir.io는 로그인 하기 위한 registry를 지정 하는 것으로 iad (Ashburn region)의 ocir (Registry)에 로그인을 하는 것을 의미 한다. 참고로 서울은 icn.ocir.io로 로그인 할 수 있다.
사용자는 "Object Namespace/사용자이름" 으로 IAM 의 사용자 정보에서 볼 수 있다. 

```bash
[jakekim@JakeKim]$ docker login iad.ocir.io
Username (idcollxtpgky/okeuser): idbkzk6hw0ua/oracleidentitycloudservice/****
Password: 
WARNING! Your password will be stored unencrypted in /home/jakekim/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store
Login Succeeded
```

로그인이 완료 되었음으로 이제 업로드할 Docker image를 만들도록 한다. 간단한 웹서버를 다음 명령어로 만들도록 한다.
다음과 같이 dockerfile을 만들어서 nginx image를 만들도록 한다. 이미지를 생성하기 전에 dockerfile이 있는 폴더에 index.html을 간단하게 생성 하여 준다. 

```bash
[jakekim@JakeKim dockerfile]$ cat Dockerfile 
FROM nginx:latest
MAINTAINER Jake Kim
COPY ./index.html /usr/share/nginx/html
CMD ["nginx","-g","daemon off;"]
EXPOSE 80
EXPOSE 443

[jakekim@JakeKim dockerfile]$ docker build -t my-nginx:1.0 ./
Sending build context to Docker daemon  3.072kB
Step 1/6 : FROM nginx:latest
 ---> 6678c7c2e56c
Step 2/6 : MAINTAINER Jake Kim
 ---> Running in 0a6b329a0499
Removing intermediate container 0a6b329a0499
 ---> 6dc3ec92b539
Step 3/6 : COPY ./index.html /usr/share/nginx/html
 ---> eafee9a486ce
Step 4/6 : CMD ["nginx","-g","daemon off;"]
 ---> Running in 32a176f4966a
Removing intermediate container 32a176f4966a
 ---> a4624c382cfb
Step 5/6 : EXPOSE 80
 ---> Running in 986bd43c69dc
Removing intermediate container 986bd43c69dc
 ---> 0e615bf6918f
Step 6/6 : EXPOSE 443
 ---> Running in 95afcc7af235
Removing intermediate container 95afcc7af235
 ---> 5bf5271f828a
Successfully built 5bf5271f828a
Successfully tagged my-nginx:1.0
[jakekim@JakeKim dockerfile]$ docker images
REPOSITORY                                       TAG                 IMAGE ID            CREATED             SIZE
my-nginx                                         1.0                 5bf5271f828a        6 seconds ago       127MB
```

이미지가 생성 되었음으로 이를 registry로 업로드 하도록 한다.
업로드를 위해서는 먼저 tag를 작성 한 후 push 할 수 있다. tag 작성은 "registry 주소/object namespace/registry 이름:tag" 으로 작성 하여 준다.

```bash
[jakekim@JakeKim dockerfile]$ docker tag my-nginx:1.0 iad.ocir.io/idbkzk6hw0ua/privateimages:my-nginx
[jakekim@JakeKim dockerfile]$ docker push iad.ocir.io/idbkzk6hw0ua/privateimages:my-nginx
The push refers to repository [iad.ocir.io/idbkzk6hw0ua/privateimages]
bc5633c583e9: Pushed 
55a77731ed26: Layer already exists 
71f2244bc14d: Layer already exists 
f2cb0ecef392: Layer already exists 
my-nginx: digest: sha256:b96c59a22ddb2f739ec749f1bbae16407bf923ee97abdc87f798c8baecef8cc8 size: 1155
[jakekim@JakeKim dockerfile]$
```

Push 된 이미지는 Registry 에서 확인 할 수 있다.   
![](/image/ocir/ocir-5.png)

# Container 생성
이제 Registry를 이용하여 OKE 에 Container를 생성 해 보도록 하겠다. 우선 Registry 접근을 위해 Kubernetes의 secret을 생성 하여 준다.
docker-username은 앞서 사용한 object namespace/사용자이름 으로 작성 하고 docker-passwor는 authtoken 을 넣어준다. 앞서 사용한 authtoken을 알고 있다면 그대로 사용 하면 되고 메모 해 놓은 것이 없다면 추가로 생성 하여 주면 된다.

```bash
[jakekim@JakeKim oke]$ kubectl create secret docker-registry ocirsecret --docker-server=iad.ocir.io --docker-username='idbkzk6hw0ua/oracleidentitycloudservice/****' --docker-password='*****' --docker-email='***@email.com'
secret/ocirsecret created
[jakekim@JakeKim oke]
```

다음과 같이 Pod를 생성 하는 deployment Yaml 파일 (mynginx-deploy.yaml) 을 생성 한다.
container를 생성 하기 위한 image에 Registry 를 지정 하였으며 접근을 위한 Secret 를 작성 하였다.
이미지 이름은 Registry 정보 중 Full Path 정보를 사용 하면 된다.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
 name: mynginx-deployment
spec:
 selector:
  matchLabels:
   app: mynginx
 replicas: 2
 template:
  metadata:
   labels:
    app: mynginx
  spec:
   containers:
   - name: my-nginx
     image: iad.ocir.io/idbkzk6hw0ua/privateimages:my-nginx
     ports:
     - containerPort: 80
   imagePullSecrets:
   - name: ocirsecret
```

Load Balancer 를 이용한 Service 를 위해 다음과 같이 Service Yaml (mynginx-service.yaml) 파일도 생성 하여 준다.
```yaml
apiVersion: v1
kind: Service
metadata:
 name: mynginx-service
spec:
 type: LoadBalancer
 ports:
 - port: 80
   protocol: TCP
   targetPort: 80
 selector:
  app: mynginx
```

Pod를 배포 하기 위해 다음과 같이 실행 하여 준다

```bash
[jakekim@JakeKim oke]$ kubectl apply -f mynginx-deploy.yaml 
deployment.apps/mynginx-deployment created
[jakekim@JakeKim oke]$ kubectl apply -f mynginx-service.yaml 
service/mynginx-service created
[jakekim@JakeKim oke]$ kubectl get service
NAME              TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)        AGE
kubernetes        ClusterIP      10.96.0.1       <none>            443/TCP        61m
mynginx-service   LoadBalancer   10.96.131.136   129.213.177.226   80:32605/TCP   40s
[jakekim@JakeKim]$
```

외부 IP가 생성 되었음으로 해당 IP 로 접근 하여 보면 다음과 같이 정상 동작 하는 것을 볼 수 있다.   
![](/image/ocir/ocir-6.png)
