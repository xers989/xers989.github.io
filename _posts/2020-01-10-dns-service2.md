---
layout: post
title: dns-service
date: 2020-01-10 14:42 +0900
categories: edge dns
---
# Traffic Management
이전 DNS 관련글에서 DNS zone management 를 이용하는 도메인 관리를 소개하였다. 이를 활용 하여 지역별로 웹서버를 설정 하고 사용자 지역에 따라 가까운 서버에서 응답하도록 구성 하여 보겠다. 하나의 도메인에 여러개의 웹서버를 구성하여 트래픽이 분산 처리 되도록 구성 하여 장애에 대한 대처나 성능을 최대화 하기 위한 방안으로 활용이 가능하며 단지 DNS 을 활용한 것으로 빠르게 적용할 수 있다는 것이 장점이다.
그림을 다시 그려 보면 북미와 유럽, 아시아에 각각 웹서버를 배치하고 Edge DNS가 사용자의 위치에 따라 가까운 웹서버로 전달 되도록 경로를 조정 해 주는 것이다. 즉 사용자가 app.service.com 의 주소를 Edge DNS 에 질의하면 가장 가까운 웹서버의 주소를 반환하여 빠르게 응답을 받을 수 있게 된다. 이를 활용하여 글로벌한 Active-Active 서비스 구성을 할 수 있다. 2개 이상의 데이터 센터를 구성 하고 동일한 서비스를 Active-Active 하게 활용하기 위해 글로벌 로드 밸란서를 활용하여 트래픽을 분산하였는데 Edge DNS 를 활용한 다면 쉽게 대체 할 수 있다. 또한 트래픽이 분산되도록 결정을 담당하는 Traffic Management 가 Edge에서 서비스되어 글로벌하게 배포되어 있어 글로벌한 서비스에 활용한 다면 큰 장점을 얻을 수 있다
![](/image/dns-service2/dns-service-1.png)

# Web Server
오라클 클라우드 서비스를 활용하여 서울, 런던, 미국에 웹서버를 구성 하고 사용자별로 어디로 서비스가 전달 되는지 보도록 하겠다. 각 지역별로 웹서버를 구성 하기 위해 오라클 클라우드의 데이터 센터 리전을 확장 하도록 한다. 오라클 클라드에 로그인 하여 상단에 검색 버튼 옆에 데이터 센터를 클릭 하면 리전관리 (Manage Regions) 를 볼 수 있다. 기본 데이터 센터가 Ashburn 으로 하여 London 과 Seoul 을 추가 하도록 한다.  
![](/image/dns-service2/dns-service-2.png)

우선 미국 지역에 웹서버를 구성 하도록 한다. 지역을 US East (Ashburn) 으로 선택 한다.  
![](/image/dns-service2/dns-service-3.png)
Instance 를 생성 하기 전에 우선 Identity > Compartment 에서 테스트를 위한 Compartment 를 생성 하여 준다.
테스트용 VCN 를 구성해야 한다. 테스트 복적임으로 Quickstart 를 이용하여 바로 생성 하여 준다.
Network > Virtual Cloud Networks > Networking Quickstart > VCN with Internet Connectivity 를 선택하여 네트워크를 구성 한다.  
VCN 은 테스트를 위한 것임으로 테스트 목적에 맞게 구성하고 Compartment 를 꼭 선택 하여 준다. VCN CIDR Block 은 테스트 용도 임으로 10.0.0.0/16 으로 하고 Public Subnet 의 CIDR 은 10.0.0.0/24 하고 Private Subnet 의 CIDR dms 10.0.1.0/24 로 하여 생성 하여 준다. 생성 되는 네트워크는 기본적인 구성으로 Public subnet 와 Private subnet 으로 구성 되며 Public subnet 에 생성 되는 Instance 는 Public IP 를 가지게 된다.

Compute > Instances 에서 Create Instance 를 한다. 간단히 Docker 를 활용하여 웹서버를 구성할 것임으로 instance Shape 은 VM.Standard.E2.1 정도로 하면 된다.
Show Shape, Network and Storage Options 를 클릭 하여 상세 설정 부분에서 Shape 를 선택 하고 compartment 와 VCN / Subnet 을 선택 한다. Public IP 를 부여 받아야 함으로 Public Subnet 을 선택 한다. 또한 Assign a public IP address 를 선택 하여 준다.
SSH 접근을 위해 비대칭키의 Public key 를 등록 하여 준다.
![](/image/dns-service2/dns-service-4.png)

생성이 완료 되면 부여된 Public IP 를 이용하여 Instance 에 ssh 접근을 한다.
```bash
$ sudo yum update
$ sudo yum install docker-engine
$ sudo sudo usermod -aG docker opc
$ sudo reboot
$ sudo systemctl start docker
$ sudo firewall-cmd --add-port=80/tcp
$ mkdir html
$ echo ‘This is my Web-Server running in Ashburn’ >> ./html/index.html
$ docker run -it -d -p 80:80 -v /home/opc/html/:/usr/local/apache2/htdocs/ --name http httpd
$ docker ps 
CONTAINER ID        IMAGE               COMMAND              CREATED             STATUS              PORTS                NAMES
6beeca27476d        httpd               "httpd-foreground"   2 minutes ago       Up About a minute   0.0.0.0:80->80/tcp   http
```
80 으로 서비스를 허용해 주기 위해 Public subnet 의 security list 에 80 을 추가 하여 준다.  
![](/image/dns-service2/dns-service-5.png)
브라우저로 접근이 가능한지 테스트 하여 보자  
![](/image/dns-service2/dns-service-6.png)

동일한 작업을 직역을 변경 하여 진행 하여 준다. 
지역 : UK South (London)
VCN : EU_VCN
Docker image 를 생성할 때 웹서버를 구분 하여 보기 위해 index.html 파일을 생성 할 때 다음과 같이 하여 준다.
```bash
$ mkdir html
$ echo ‘This is my Web-Server running in London’ >> ./html/index.html
$ docker run -it -d -p 80:80 -v /home/opc/html/:/usr/local/apache2/htdocs/ --name http httpd
```
지역 : South Korea Central (Seoul)
VCN : KR_VCN
Docker image 를 생성할 때 웹서버를 구분 하여 보기 위해 index.html 파일을 생성 할 때 다음과 같이 하여 준다.
```bash
$ mkdir html
$ echo ‘This is my Web-Server running in Seoul’ >> ./html/index.html
$ docker run -it -d -p 80:80 -v /home/opc/html/:/usr/local/apache2/htdocs/ --name http httpd
```
웹으로 접근이 가능한 지 확인 하여 보자

London 에 설치한 웹서버  
![](/image/dns-service2/dns-service-7.png)
Seoul 에 설치한 웹서버  
![](/image/dns-service2/dns-service-8.png)

# Traffic Manangement
이제 도메인을 DNS 에 등록 하고 설치 한 웹서버의 IP 를 DNS Record 에 등록 하여 주면 도메인 등록이 완료될 것이다. 그러나 3대의 웹서버를 하나의 도메인으로 연결하고 사용자의 위치 정보에 따라 트래픽이 분배 되도록 조정 하겠다.
우선 이전 포스트에서 처럼 DNS Zone management 에 도메인이 등록이 사전에 필요하다. 등록이 완료 되었으면 Cloud 에 로그인 하여 Network > Traffic management 에서 새로운 Traffic Management Steering Policy 를 생성 한다.
Traffic Management Steering Policy 은 다음 순서로 등록 한다.

    1. Policy Type : 트래픽을 분배 하기 위한 규칙으로 5 가지의 타입이 있으며 이중 하나를 선택 할 수 있다. 
    2. Answer Pool : 트래픽 규칙을 생성 할 때 선택 가능 응답으로 웹트래픽이 전달 될 곳을 의미 한다.
    3. Steering Rules : Policy Type 에 따라 구성 되며 Type 에 따른 규칙을 실제로 정의 한다.
    4. Attach Health Check : 대상 시스템의 작동 유무를 판단 하기 위한 것으로 기존 등록된 것을 선택하거나 새로생성, 없음을 선택 할 수 있다.
    5. Attached Domain : Policy 를 적용할 Domain 을 선택

### Policy Type
Geolocation 을 선택 하고 Policy Name 은 WebServerGeolocationSteering 으로 하고 Policy TTL 은 60 초, Maximum Answer count 는 1 로 등록 하여 준다.
![](/image/dns-service2/dns-service-9.png)

### Answer Pool
Steering Rule 을 등록 할 때 선택 가능한 응답을 등록 하는 곳으로 등록 된 3개 서버 (Ashburn, London, Seoul) 에서 서비스를 할 것임으로 3개를 Answer Pool 로 등록 하여 준다. 
Pool Name 은 알아보기 쉽게 Ashburn 으로 하고 Answers Name 은 us, type 은 A RDATA 는 Ashburn 웹서버의 IP 를 작성 하여 준다.

Answer Pool | Answer Pool Name | Answers Name | Type | RData
Answer Pool 1 | US | us | A | IP_ADDRESS
Answer Pool 2 | EU | eu | A | IP_ADDRESS
Answer Pool 3 | ASIA | asia | A | IP_ADDRESS

### Geolocation Steering Rules
위치에 따른 응답을 선택 하는 것으로 복수개의 규칙을 등록 할 수 있으며 순서대로 실행 된다.
Geolocation 은 대륙 및 나라가 선택 가능 하며 Add Global Catch-All 이 기본으로 선택되게 된다.
시나리오는 유럽과 아프리카는 London 웹서버에서 응답을 주고 백업으로 Ashburn 에서 해주도록 구성 하며 아메리카 대륙은 Ashburn 에서 응답을 주고 London 이 백업 하며 아시아와 호주 지역은 Seoul 에서 응답하고 Ashburn 이 백업 하도록 한다. 그 외에 지역은 Ashburn, Seoul 에서 응답을 주는 것으로 구성 하도록 한다.

Rule | Geolocation | Pool Priority 1 | Pool Priority 2 
--------- | --------- | --------- | --------- 
Rule1 | North America, South America | US | EU
Rule2 | Asia, Oceania | ASIA | US 
Rule3 | Europe, Africa | EU | US
Catchall | | US | ASIA

### Attach Health Check & Attach domain
웹서버를 구성 할 때 헬스체크를 위한 서비스를 등록 하였다면 Health Check 를 등록 하여 응답 불가능 한 상태일 때 다음 서버가 응답을 주도록 구성 할 수 있다. 이번 포스팅에서는 헬스체크를 위한 서비스를 등록 하지 않았음으로 없음을 선택 한다.
Attach Domain 은 이전 포스트에서 등록 한 DNS zone 을 선택 하여 Subdomain 에 app 를 입력 해 준다. 등록이 완료 된다면 app.*.* 으로 서비스 될 것 이다.

# Geolocation Test
준비가 완료 되었음으로 브라우저의 Geolocation 을 변경 하기 위한 Proxy 를 설치 해준다. firefox 에는 Hoxx 라는 무료 plugin 이 있으니 이를 이용하면 된다. 테스트를 할 때 꼭 브라우저 캐시를 지우고 들어가도록 한다. 우선 국내에서 등록한 도메인으로 접근 하는 경우 웹서버는 Seoul 에서 응답해야 할 것이다.  
![](/image/dns-service2/dns-service-10.png)  
이제 위치를 바꾸기 위해 브라우저에 설치된 Hoxx 를 클릭 하고 독일을 선택 한다.  
![](/image/dns-service2/dns-service-11.png)  
브라우저의 캐시를 삭제하고 다시 접근 하면 London 에서 응답을 줄 것 이다.  
![](/image/dns-service2/dns-service-12.png)  
다시 Hoxx 연결을 끊고 미국을 선택 한 후 브라우저 캐시를 삭제하고 접근하면 Ashburn 웹서버에서 응답을 줄 것이다.  
![](/image/dns-service2/dns-service-13.png)  

사용자 브라우저의 위치에 따라 가까운 웹서버에서 응답을 주도록 구성을 하였으며 장애 발생시 응답을 줄 수 있는 곳까지 등록 하였음으로 안정적으로 서비스 가능한 구성이 편리 하게 되었다. 지역별 위치한 웹서버에 대한 동기화는 다루지 않았지만 동기화가 된다면 트래픽을 효과적으로 분산 할 수 있음으로 매우 효율적일 것이며 지역별 서버에 해당 지역 언어로 서비스 하로독 구성하여 보다 편리한 서비스를 구성 할 수 있을 것이다.