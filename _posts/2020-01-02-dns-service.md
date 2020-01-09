---
layout: post
title: dns-service
date: 2020-01-02 15:02 +0900
categories: edge dns
---
# Edge 서비스
오라클은 2016년 11월 DYN 이란 업체를 인수 하면서 Edge 서비스를 시작 하였다.
현재는 Oracle Cloud 서비스에 통합 되어 Cloud Console 에서 사용 가능하다.

Edge 서비스는 서울을 포함하여 글로벌 하게 분포 되어 있으며 사용자의 네트웍에서 가장 가까운 Edge 서비스를 통해서 Web Application 에 접근 하는 구조 이다. 이러한 구조로 글로벌 하게 일정 수준의 성능을 보여주는 Web Application 을 만들 수 있으며 장애 등에 빠르게 대응이 가능하게 된다.

아래 그림에 OCI DNS 는 Oracle 이 제공 하는 Edge 서비스로 사용자가 app.service.com 에 접근 하게 되면 Recursive Server 는 OCI DNS 에 DNS 질의를 하고 OCI DNS 는 Origin Server 즉 app.service.com 의 주소를 반환 하여 사용자가 app.service.com 에 접근 할 수 있게 한다.
이때 질의를 처리하는 OCI DNS 는 사용자와 가장 가까운 곳에 있는 곳에서 응답을 주며 사용자 위치나 app.service.com 의 상태에 따라 주소를 달리 하여 일정 수준의 성능을 보여 주도록 할 수 있다.
![](/image/dns-service/dns-service-1.png)

그림을 다시 그려 보면 북미와 유럽, 아시아에 각각 Origin 서버를 배치하고 해당 지역의 Edge 에서 app.service.com 를 서비스 한다면 각 지역의 Edge 는 가장 가까운 곳의 Origin 서버에서 서비스가 진행 되도록 주소를 반환 할 수 있다. 각 app.service.com 간의 동기화는 다른 문제이지만 이를 통해 글로벌 하게 Web Application 을 각 지역에 배치 하고 해당 지역 사용자는 가까운 Web Application 을 사용하도록 하여 성능을 확보 할 수 있으며 Asia 혹은 EU 의 서버에 장애가 난 경우 서비스가 가능한 NA 로 서비스가 되도록 경로를 조정 할 수 있다. 보통 DNS 를 이용한 조정이 60 분 가량이 소요 되지만 OCI DNS 는 30 초 가량으로 빠르게 배포 될 수 있으며 REST API 를 제공 하여 서비스 이상을 감지하고 자동으로 DNS 를 조정 할 수 있는 구조를 만들수 있다.
![](/image/dns-service/dns-service-2.png)

# Oracle DNS
실습을 위해서는 우선 도메인이 필요 하며 도메인 관리 업체에 Domain Name Server 를 OCI 에서 진행 할 것이라는 것을 알려 주어야 한다.
OCI 콜솔에 로그인 한 후 Neworking 메뉴에서 DNS Zone Management 를 선택 한다.
DNS Zone 은 도메인 이름을 관리 해주는 솔루션으로 사용할 도메인을 등록 하여 OCI DNS 에서 해당 주소를 서비스 하도록 설정 하여야 한다. 이를 위해 Creae Zone 을 한다.
Zone 생성은 매우 간단하게 Zone Type 을 Primary 로 하고 Zone Name 에 사용할 도메일 이름을 작성 해 준다. (참고로 DNS Zone 은 Oracle Compartment 에 의해 관리 됨으로 생성 하기 전에 꼭 Compartment 를 선택 하여야 한다)
![](/image/dns-service/dns-service-3.png)

생성이 완료 되면 OCI DNS 의 네임서버 정보를 볼 수 있게 된다. 이제 DNS 제공자에게 이 네임서버를 이용해서 서비스 할 것이라는 것을 알리는 과정이 필요하다
![](/image/dns-service/dns-service-4.png)

Godaddy 의 경우 도메인 관리에서 DNS Management 에서 Nameservers 를 변경 하는 것이 가능하다. NameServer 를 Use my own nameservers 로 선택 하고 OCI DNS 에서 제공 하는 Name Server 를 입력 하여 준다
![](/image/dns-service/dns-service-5.png)

# Web Application
도메인 등록이 완료 되었으니 간단히 웹서버를 만들어서 서비스를 하여 DNS 가 정상 동작하는 지 볼것이다. OCI 에 Docker 를 이용하여 Apache 서버를 기동 시킬 것이다. 구매한 도메인이 iamhub.site 임으로 edge.iamhub.site 로 만들 것이다.
우선 클라우드에 Public IP 는 VM 을 구성하고 docker 를 설치 하여 보자.
'# yum install docker-engine
포트는 80 으로 할 것임으로 방화벽과 클라우드 Security List 에 80 이 허용 되도록 변경 하여 준다
'# firewall-cmd --add-service=http

