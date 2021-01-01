---
layout: post
title:  WAF 등록
date:   2020-03-08 19:46:00 +0900
categories: Edge WAF
---
# Web Application firewall
웹애플리케이션 방화벽(이하 WAF)은 기존 방화벽이 네트워크의 레이어 3,4 에서 동작하는 방화벽을 개선 한 것이다. 이름에서 예측 할 수 있듯이 레이어 7 방화벽으로 웹 트래픽 내부에 데이터까지 스캐닝 하여 악성 코드나 인젝션 공격 등을 사전에 방어 하는 서비스이다.  증가되고 있는 DDoS 공격 등을 감지하고 방어하여 도입이 늘어나는 추세라고 볼 수 있다.  
WAF는 크게 두 가지 형태로 구분해 볼 수 있는데 데이터 센터내에 설치 구현하는 방법과 클라우드에 설치 하는 방법으로 나눌 수가 있다.
첫번 째로 데이터 센터내에 설치 하는 것은 인터넷을 거쳐 유입되는 트래픽을 웹서버로 전달 하기 전에 검사하는 것으로 검사 장비가 데이터 센터내에 설치 되는 것이다. 이 경우 DDoS 공격을 받게 되면 WAF 자체도 부하를 받게 되어 효과적인 방어가 어려울 수 있으며 늘어나는 공격방법에 대한 업데이트를 주기적으로 해주어야 하는 단점이 있다.  
두번째는 클라우드에 설치되는 WAF서비스를 클라우드에 배포 하고 처리 하는 것으로 클라우드 특성을 활용하여 DDoS 공격 등에 자원을 동적으로 증가 시켜 대응 하여 공격을 방어 한다. 또한 서비스 형태의 WAF는 증가되는 공격 방법을 주기적으로 업데이트 하여 주어 관리 측면에 장점이 있다.  
오라클 WAF는 클라우드 기반의 방화벽 서비스로 사용한 만큼만 지불 하기 때문에 도입비용이 낮고 적용이 매우 간단하다는 장점이 있다.   
![](/image/waf-register/waf-01.png)

그림과 같이 WAF는 데이터 센터 혹은 애플리케이션이 설치된 네트워크 외부에 배치 되어 있어 애플리케이션에 영향을 영향을 주지 않는다. 동작은 Proxy서버 처럼 모든 트래픽을 앞단에서 처리 하는 역할을 한다. WAF를 거치도록 처리하는 것은 DNS를 이용하여 간단하게 처리 될 수 있으며 애플리케이션에 전혀 수정이 없이 빠르게 처리 될 수 있다.  
방법은 DNS 제공자, 예를 들어 goDaddy,에 CNAME유형으로 도메인을 등록 하고 지시 방향을 WAF주소로 설정하면 도메인 접근시 지시방향인 WAF로 트래픽이 전달되어 WAF서비스가 이를 받아 처리하고 정상 트랙픽만을 웹애플리케이션(통산 Origin 이라고 한다)으로 전달 한다. 오라클 WAF는 Edge에서 제공하는 서비스로 오라클이 제공하는 모든 Edge에 설정한 WAF정보가 배포된다. 만약 유럽의 사용자가 웹애플리케이션을 접근 하는 경우 유럽에 설치된 Edge의 WAF에 트래픽이 전달 되어 처리 된다. 한국 사용자의 경우 한국에 설치된 Edge의 WAF에 트래픽이 전달된다.    
![](/image/waf-register/waf-02.png)

그림 처럼 글로벌 하게 분포된 Edge가 사용자의 트래픽을 받고 그 중 정상 트래픽만을 웹애플리케이션(origin)으로 전달한다. DDoS를 포함한 외부 공격을 앞단에서 방어 하여 웹애플리케이션에 영향을 주지 않는 구조이다.

# WAF 설정
WAF 설정을 위해서는 웹애플리케이션의 도메인명이 등록 되어 있어야 한다. DNS를 이용하여 WAF를 경유 하도록 하는 구조임으로 테스트등을 하기 위해서는 도메인을 보유 하고 있어야 한다. korwaf.site라는 도메인을 보유하고 있음으로 waf.korwaf.site 도메인을 생성하고 웹어플리에션을 생성하는 방법을 소개 하도록 하겠다.   
전체 구조는 다음과 같다.   
![](/image/waf-register/waf-04.png)
Origin에 Web Application을 구성 하고 WAF에 Origin을 등록 하여 주면 고유한 주소가 생성이 되며 이를 DNS에 등록 하여 주면 설정이 완료 된다. 사용자가 도메인 (waf.korwaf.site)을 접근 하려 할때 DNS는 WAF의 주소를 반환하게 되며 WAF는 사용자 트래픽을 설정된 Origin으로 전달하는 구조이다.   

#### Web application 설치
애플리케이션의 취약점을 테스트 하기 위한 웹애플리케이션을 설치하도록 한다. 취약점을 테스트하기 위한 Docker image DVWA를 사용하도록 한다.   
(https://hub.docker.com/r/vulnerables/web-dvwa)    
공용 IP를 가지고 있는 인스턴스를 생성 한 후에 Docker engine을 설치 한다. 로컬 환경에 머신과 공용 IP가 있다면 로컬에 구성 하여도 상관없으나 빠르게 환경을 구성하기 위해서는 클라우드 환경을 추천 한다. Linux 환경을 기준으로 Docker engine 설치와 docker image 를 구동 하는 것 명령어 이다. 
```bash
[root@wafapplication]# yum install docker-engine -y  

====================================================================================================================================================
 Package                             Arch                     Version                                            Repository                    Size
====================================================================================================================================================
Installing:
 docker-engine                       x86_64                   19.03.1.ol-1.0.0.el7                               ol7_addons                    24 M
Installing for dependencies:
 container-selinux                   noarch                   2:2.77-5.el7                                       ol7_addons                    37 k
 containerd                          x86_64                   1.2.0-1.0.5.el7                                    ol7_addons                    21 M
 criu                                x86_64                   3.12-2.el7                                         ol7_latest                   452 k
 docker-cli                          x86_64                   19.03.1.ol-1.0.0.el7                               ol7_addons                    40 M
 libnet                              x86_64                   1.1.6-7.el7                                        ol7_latest                    57 k
 protobuf-c                          x86_64                   1.0.2-3.el7                                        ol7_latest                    27 k
 runc                                x86_64                   1.0.0-19.rc5.git4bb1fe4.0.4.el7                    ol7_addons                   1.9 M

Transaction Summary
====================================================================================================================================================
Install  1 Package (+7 Dependent packages)

Total download size: 88 M
Installed size: 370 M
[root@wafapplication]# usermod -aG docker opc
[root@wafapplication]# reboot  
[opc@wafapplication]$ sudo systemctl start docker
[opc@wafapplication]$ sudo firewall-cmd --permanent --add-port=80/tcp
[opc@wafapplication]$ sudo firewall-cmd --reload
[opc@wafapplication]$ docker -it -d --name waf -p 80:80 vulnerables/web-dvwa
[opc@wafapplication]$ docker ps -a
CONTAINER ID        IMAGE                  COMMAND             CREATED             STATUS              PORTS                NAMES
9414f813e5e2        vulnerables/web-dvwa   "/main.sh"          3 minutes ago        Up 3 minutes          0.0.0.0:80->80/tcp   waf
```
애플리케이션은 80 포트로 오픈됨으로 ACL 에 80포트가 접근 가능하도록 허용해준 후에 공용 IP로 접속 한다.
최초로 접속 하면 Database를 생성하라는 메시지를 볼 수 있다. MySQL로 생성 되며 이후 자동으로 로그인 페이지로 접속하게 된다. 접근 ID는 admin이다.   

#### WAF Policy 생성
오라클 클라우드의 OCI 콘솔에 로그인 한 후 Security탭에 WAF Policies를 선택 한다. 등록한 WAF Policy가 Compartment에 소속됨으로 적절한 Compartment를 선택하여 준다.
Create WAF Policy를 클릭 하면 다음과 같은 화면을 볼 수 있다.    
![](/image/waf-register/waf-05.png)

Policy Name을 입력 하여 주고 Domains의 Primary Domain에 도메인(waf.korwaf.site)를 등록 하여 준다.
WAF Origin에는 Origin을 구분 하기 위한 이름을 입력 하고 URI 부분에 Origin의 공용 IP를 입력 하여 준다.   
![](/image/waf-register/waf-06.png)

저장 후 몇 분간 Edge에 등록한 WAF Policy를 배포 하기 시작 한다. 전체 Edge에 배포되기 때문에 약간의 시간이 소요 된다. 등록이 완료 되면 다음과 같이 세부 정보 및 메시지를 볼 수 있다. Policy 정보중 CNAME Target을 복사하여 준다.   
![](/image/waf-register/waf-07.png)

#### DNS 변경
Domain인 제공자가 제공하는 도메인 관리 페이지에서 도메인을 수정하여 주어야 한다. Godaddy에서 관리하는 도메인임으로 godaddy에 로그인 하여 DNS 관리를 선택 한다. 도메인의 레코드 관리에 들어가서 레코드를 추가 하여 준다. 유형은 CNAME을 선택 하고 호스트는 waf를 입력하여 준다.(waf.korwaf.site 라는 도메인을 서비스 하기 위한 것) 지시 방향에 WAF Policy 정보의 CNAME Target을 복사하여 준다. TTL 은 1시간 혹은 1/2 시간을 선택 하여 준다. (해당 시간 안에 변경 사항이 적용 된다)    
![](/image/waf-register/waf-08.png)

설정이 완료 되면 nslookup 으로 WAF 를 거쳐 가는지 점검하여 본다.

```bash
Jakes-MacBook-Pro:~ jakekim$ nslookup waf.korwaf.site
Server:		168.126.63.1
Address:	168.126.63.1#53

Non-authoritative answer:
waf.korwaf.site	canonical name = waf-korwaf-site.o.waas.oci.oraclecloud.net.
waf-korwaf-site.o.waas.oci.oraclecloud.net	canonical name = tm.inregion.waas.oci.oraclecloud.net.
tm.inregion.waas.oci.oraclecloud.net	canonical name = apac-seoul.inregion.waas.oci.oraclecloud.net.
Name:	apac-seoul.inregion.waas.oci.oraclecloud.net
Address: 192.29.19.39
Name:	apac-seoul.inregion.waas.oci.oraclecloud.net
Address: 192.29.18.3
Name:	apac-seoul.inregion.waas.oci.oraclecloud.net
Address: 192.29.19.178
```

명령어 결과에서 볼 수 있 듯이 lookup 결과   
apac-seoul.inregion.waas.oci.oraclecloud.net 으로 트래픽이 가는 것을 볼 수 있다. nslookup 하는 장소에 따라 서울 혹은 일본 등으로 경로가 설정 될 수 있다.
등록이 완료 되었음으로 도메인 주소로 들어가면 정상적으로 웹애플리케이션이 오픈을 확인 하여 준다.    
![](/image/waf-register/waf-09.png)

#### 기본 WAF 설정
OCI의 WAF Policies에 들어가면 등록되어진 WAF Policy를 선택 할 수 있다. 초기는 아무런 설정이 없기 때문에 모든 트래픽이 그대로 Origin으로 전달 되는 형태 이다. 메뉴 옵션을 보면 Origin Management 에서 입력한 Origin에 대한 정보를 볼 수 있으며 IP가 변경 되거나 할 경우 이를 수정하여 주면 된다.
Settings 에서는 도메인을 http 혹은 https 로 할지를 선택 할 수 있다. WAF는 애플리케이션 레벨의 방화벽임으로 암호화 되어 전달 되는 데이터를 복호화 할 필요가 있기 때문에 https 로 하는 경우 인증서를 등록 해 주어야 한다. 현재 Web application 이 http 만 서비스 하기 때문에 수정 할 필요는 없다. (https를 설정할 경우 https enable 을 선택)   
![](/image/waf-register/waf-10.png)

다음은 WAF 의 핵심인 Protection Rules 이다. 외부 공격을 방어하기 위한 패턴 규칙이 약 450여개가 등록 되어 있으며 선별하여 적용이 가능하다. 각 패턴별로 대응하기 위한 액션을 선택 할 수 있다. Off 는 패턴을 보지 않으며 Detect 의 경우 로그만을 기록 하고 트래픽이 전달 되며 block을 선택 하면 해당 패턴의 트래픽은 차단 된다.    
![](/image/waf-register/waf-11.png)
기본값은 전체가 Off로 설정 되어 있다. 
Recommendation 탭을 보면 WAF 가 웹 애플리케이션 및 기존 트래픽을 분석 하여 Protection Rule 을 추천 하여 준다. (등록 후 얼마간의 시간이 지나야 추천 Rule 들이 보이게 된다)

다음 POST에서는 OWASP에 대한 패턴과 WAF가 이를 탐지하는 내용을 보도록 하겠다.