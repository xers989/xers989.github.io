---
layout: post
title: 동일한 리전에 생성된 VNC 간 네트워크 연결 하기 (Local Peering)   
date: 2021-03-13 22:00 +0900
categories: network peering
---
# Oracle VCN Local Peering
오라클 클라우드는 VCN을 이용하여 네트워크를 구성하여 사용 할 수 있다. VCN은 CIDR를 이용하여 네트워크를 정의 하고 내부에 Subnet을 구성 할 수 있다. VCN은 Region개념으로 생성한 VCN은 해당 region 내에서만 유효 하다. 클라우드 활용이 증가 하면서 하나의 Region에 복수개의 클라우드 구독(Tenancy)으로 중요 프로젝트를 구분하기도 하며 하나의 클라우드 내에 복수개의 VCN을 구성하여 운영/테스트/개발 등 환경을 구성하기도 한다. 이러한 환경 구성으로 인해 별도 구성된 VCN 간에 네트워크 연결이 필요할 수 있다. 이번 글에서는 하나의 Region에 생성된 2개의 Tenancy간의 VCN 연결을 하도록 하겠다. 오라클 클라우드는 하나의 Region내에 생성된 VCN 간의 전용 연결을 위해 Local Peer Gateway 를 제공하여 인터넷 라인을 통하지 않고 네트워크를 연결 할 수 있다.


#### VCN 구성
테스트는 2개의 Tenancy에 각각 VCN을 구성 하여 테스트를 진행 할 것이며 편의상 Tenancy를 Tenancy1, Tenancy2로 하여 테스트 한다.
Tenancy1에는 VCN 을 10.0.0.0/16 으로 네트워크를 구성하고 Tenancy2는 VCN을 10.1.0.0/16 으로 구성하여 IP가 중복되지 않도록 하여 
원할한 네트워크 통신이 되도록 한다. (CID가 중복되면 네트워 연결이 불가능하다)
VCN 생성은 OCI Console의 Networking 에서 Create VCN으로 생성 하면 된다. 테스트를 위해 Public Subnet을 하나씩 구성 한다.
SSH로 접속 하여 테스트를 진행 해야 함으로 Internet Gateway도 VCN생성시 추가해 준다. Wizard를 이용하면 간편하게 구성 할 수 있다.
Tenancy1의 Subnet은 다음과 같다.   
![](/image/localpeer/localpeer02.png){: width="100%" height="100%"}   
Tenancy2의 Subnet도 다음과 같이 구성한다.   
![](/image/localpeer/localpeer01.png){: width="100%" height="100%"}   


#### Local Peer 추가
Tenancy1 의 VCN에 네트워크 연결을 위하여 게이트웨이를 추가 한다. VCN을 클릭 하여 들어가면 Local Peering Gateways메뉴를 볼 수 있다. Create Local Peering Gateway를 클릭 하여 게이트웨이를 추가 한다. 이름은 알아볼 수 있도록 Tenancy2와 연결을 위한 것임으로 LGtoTenancy2로 한다.  
![](/image/localpeer/localpeer03.png){: width="80%" height="80%"}   
Tenancy2에도 동일한 방법으로 Gateway를 추가 하여 준다.  
![](/image/localpeer/localpeer04.png){: width="80%" height="80%"}   



#### Local Peer 연결 작업
이제 구성한 게이트웨이간의 연결이 필요하다. 연결을 위해서는 한쪽 게이트웨이의 OCID를 다른 쪽 게이트웨이에 복사하여 주는 것으로 연결이 완료 된다. 연결은 Tenancy2에서 Tenancy1로 연결을 생성 하도록 하겠다. 필요한 데이터는 Tenancy1에 생성한 Local Peering Gateway 의 OCID값으로 생성한 게이트웨이에서 값을 복사 할 수 있다. Tenancy1에 생성한 LGtoTenancy2의 OCID값을 복사한다.   
![](/image/localpeer/localpeer05.png){: width="90%" height="90%"}   

Tenancy2의 OCI console로 이동 하여 생성한 게이트웨이 LGtoTenancy1에서 Establish Peering Connection을 선택 하여 준다.   
![](/image/localpeer/localpeer06.png){: width="100%" height="100%"}   

커넥션 생성을 위한 데이터 입력 화면에서 Enter Local Peering Gatway OCID를 선택 하고 Tenany1의 Local Peering Gateway OCID값을 복사해 줍니다.   
![](/image/localpeer/localpeer07.png){: width="80%" height="80%"}   

연결이 완료 되면 Peering Status 가 Peered로 변경 되면 Peer 대상 네트워크의 CIDR 정보를 볼 수 있다.   
![](/image/localpeer/localpeer08.png){: width="100%" height="100%"}   


#### Route 규칙 생성
상호 네트워크 생성이 완료 되었음으로 이후 작업은 라우팅 설정이 필요 합니다. Tenancy1의 VCN에서 라우팅 테이블을 선택 합니다. VCN을 Wizard로 생성한 경우 2개의 Route Table이 생성 된다 (기본 라우팅 테이블, Private Subnet용 라우드 테이블)
사용하고 있는 라우팅 테이블내에서 Add Route Rules를 클릭 하여 줍니다. (2개 모두에 생성)   
![](/image/localpeer/localpeer09.png){: width="80%" height="80%"}   
VCN내부에서 10.1.0.0/16에 대한 요청은 Local Peering Gateway로 향하게 됩니다.
동일하게 Tenancy2에 생성한 VCN의 Route Table에도 규칙을 생성 하여 줍니다. VCN내부에서 10.0.0.0/16에 대한 요청이 Local Peering Gateway로 항하도록 합니다.   
![](/image/localpeer/localpeer16.png){: width="80%" height="80%"}   



#### Tenacy1에 네트워크 접근 권한 설정
연결된 VCN이 다른 Tenancy간에 생성 된 것이기 때문에 상호 접근에 대한 권한 설정이 필요 하다.
Tenancy1의 OCI Console에 로그인 하여 Identity의 Groups에서 Group을 우선 생성 합니다.   
![](/image/localpeer/localpeer11.png){: width="80%" height="80%"}   
Identity>Policies를 선택 하여 새로운 Policy를 생성 합니다. Policy는 Root Compartment에 생성 하여 줍니다. (Define과 Endorse는 Root에 생성하여야 한다)   
![](/image/localpeer/localpeer12.png){: width="90%" height="90%"}   
Policy Builder에 다음 내용을 입력 합니다.   
```bash
Define tenancy Tenancy2 as %Tenancy2_tenancy_OCID%
Allow group lpg-manage-group to manage local-peering-from in tenancy
Endorse group lpg-manage-group to manage local-peering-to in tenancy Tenancy2
Endorse group lpg-manage-group to associate local-peering-gateways in compartment chatbot with local-peering-gateways in tenancy Tenancy2
```
Tenancy2_tenancy_OCID는 Tenancy2 의 OCID 값을 복사하여 주면 된다.   
내용은 Tenancy2 라는 이름의 Alias를 생성 하여 주고 Tenancy2의 local-peering-to 및 local-peering-gateways 의 상호 연경에 대한 권한을 부여를 확인 하는 것이다.


#### Tenacy2에 네트워크 접근 권한 설정
Tenancy2에 권한 설정을 위해 Group을 하나 생성 하여 줍니다. 이 그룹은 Tenancy1에게 부여해줄 그룹이 됩니다.   
![](/image/localpeer/localpeer13.png){: width="80%" height="80%"}  
Identity>Policies에 Policy를 생성 하고 다음 문장을 작성 하여 준다.(Policy는 root에 생성 한다)   
```bash
Define tenancy Tenancy1 as %Tenancy1_tenancy_OCID%
Define group Tenancy1Group as %Tenancy1_Group_OCID%
Admit group Tenancy1Group of tenancy Tenancy1 to manage local-peering-to in tenancy
Admit group Tenancy1Group of tenancy Tenancy1 to associate local-peering-gateways in tenancy Tenancy1
    with local-peering-gateways in tenancy
```
Tenancy1_tenancy_OCID 는 Tenancy1의 OCID 이며 Tenancy1_Group_OCID는 Tenancy1에 생성한 그룹으로 Tenancy1 네트워크 접근 권한에 생성에 설정한 Group의 OCID 이다. (lpg-manage-group)   
Alias 로  쌩성한 Tenancy1 의 Tenancy1Group에 tenancy 의 local-peering-to 관리 권한 및 연결에 대한 권한을 허용 한 것이다.   

#### Security List
이제 연결 테스트를 위해 Subnet의 Security List를 수정하여 준다. 테스트를 수행할 Tenancy1 의 Subnet에 Tenancy2의 CIDR 접근을 허용 하여 준다. Private IP 를 이용한 접근임으로 아래와 같이 Ingress Rule에 모든 Protocol 에 대해 접근을 허용 하도록 한다. (운영 환경에서는 가능하면 Port를 특정하여 관리 하는 것을 추천 한다)   
![](/image/localpeer/localpeer14.png){: width="80%" height="80%"}  
동일 한 방법으로 Tenancy2의 Subnet에 Tenancy1의 CIDR 접근을 허용 하여 준다.   
![](/image/localpeer/localpeer15.png){: width="80%" height="80%"}  


#### SSH 접근 테스트
테스트를 위해 Tenancy2에 생성한 Bastion서버에 Public IP를 이용하여 접근 하고 SSH로 Tenancy1의 Linux서버에 접근 하도록 하겟다.
다음은 Bastion 서버에 접근하는 내용이다.
```bash
[jakekim@Jakes-Mac]$ ssh -i ./key/*** opc@bastion.***
Last login: Sun Mar 14 12:18:44 2021 from 175.196.74.118
[opc@bastion ~]$ ifconfig
ens3: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 9000
        inet 10.1.0.2  netmask 255.255.255.0  broadcast 10.1.0.255
        ether 02:00:17:00:71:a5  txqueuelen 1000  (Ethernet)
        RX packets 7181836  bytes 2001911554 (1.8 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 6237884  bytes 5711297531 (5.3 GiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```
IP 가 10.1.0.2 로 Tenancy2의 CIDR (10.1.0.0/16)에 생성된 인스턴스라는 것을 알 수 있다. 이제 Tenacy1에 생성한 인스턴스 10.0.0.4 에 SSH로 접근 하도록 하겠다.
```bash
[opc@bastion ~]$ ssh -i ./*** opc@10.0.0.4
Last login: Sun Mar 14 12:31:55 2021 from 10.1.0.2
[opc@linux ~]$ ifconfig
ens3: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 9000
        inet 10.0.0.4  netmask 255.255.255.0  broadcast 10.0.0.255
        ether 02:00:17:00:c9:44  txqueuelen 1000  (Ethernet)
        RX packets 7358519  bytes 2038554374 (1.8 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 6286176  bytes 5200370202 (4.8 GiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

[opc@linux ~]$ 
```
연결이 성공된 것을 볼 수 있다.
