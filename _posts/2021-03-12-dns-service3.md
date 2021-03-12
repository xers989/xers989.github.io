---
layout: post
title: DNS 이전하기
date: 2021-03-12 15:02 +0900
categories: edge dns
---
# Oracle DNS Management
오라클 DNS는 클라우드 기반 도메인 네임서비스로 일반 DNS 네임 서비스와 달리 Origin 시스템의 Health check를 제공 할 수 있어 DR 환경 등에 사용 할 수 있다. 구입한 도메인에 대한 네임서버를 클라우드로 이전 하기 위한 방법으로 Godaddy가 제공하는 네임서버에 등록된 도메인 레코드를 Zone파일로 추출하여 오라클 DNS로 임포트 하는 방법을 알아 본다.

#### GoDaddy Zone File
Godaddy에서 저렴한 비용으로 도메인을 구매 하여 1년간 사용 할 수 있다. DNS관리 페이지에서 레코드를 추가 하거나 혹은 저장된 레코드를 파일로 일괄 추출 하여 오라클 DNS로 이전 하도록 한다. 우선 GoDaddy의 DNS 관리 페이지로 이동 한다.   
![](/image/dns-zone/dnszone01.png){: width="80%" height="80%"}   
페이지 하단에 고급 기능에 영역 파일 내보내기 기능이 있다.   
![](/image/dns-zone/dnszone02.png){: width="80%" height="80%"}   
영역 파일 내보내기(Unix)를 선택 하고 파일로 내용을 저장 한다.
텍스트로 파일은 저장 되며 내용은 다음과 같다.
```bash
; Domain: cloud**.***
; Exported (y-m-d hh:mm:ss): 2021-03-12 05:36:44
;
; This file is intended for use for informational and archival
; purposes ONLY and MUST be edited before use on a production
; DNS server.
;
; In particular, you must update the SOA record with the correct
; authoritative name server and contact e-mail address information,
; and add the correct NS records for the name servers which will
; be authoritative for this domain.
;
; For further information, please consult the BIND documentation
; located on the following website:
;
; http://www.isc.org/
;
; And RFC 1035:
;
; http://www.ietf.org/rfc/rfc1035.txt
;
; Please note that we do NOT offer technical support for any use
; of this zone data, the BIND name server, or any other third-
; party DNS software.
;
; Use at your own risk.


$ORIGIN cloud.**.***

; SOA Record
@	3600	 IN 	SOA	ns09.domaincontrol.com.	dns.jomax.net. (
					2021020600
					28800
					7200
					604800
					3600
					) 

; A Records
www	3600	 IN 	A	193.**.**.**
blog	3600	 IN 	A	193.**.**.**

; CNAME Records
www	3600	 IN 	CNAME	@

; MX Records
; TXT Records
; SRV Records
; AAAA Records
; CAA Records
; NS Records
@	3600	 IN 	NS	ns09.domaincontrol.com.
@	3600	 IN 	NS	ns10.domaincontrol.com.
```




#### Zone 파일 수정
다운로드한 파일을 오라클 DNS Management로 업로드 하기 위해 내용을 수정 하여 준다.
필요한 부분은 $ORIGIN 이후 블록으로 이전 내용은 삭제 하고 준다.
수정한 내용은 다음과 같다.
```bash
$ORIGIN cloud.**.***.
; SOA Record
@	3600	 IN 	SOA	ns09.domaincontrol.com.	dns.jomax.net. (
					2021020600
					28800
					7200
					604800
					3600
					) 

; A Records
www	3600	 IN 	A	193.**.**.**
blog	3600	 IN 	A	193.**.**.**

; CNAME Records
www	3600	 IN 	CNAME	@

; MX Records
; TXT Records
; SRV Records
; AAAA Records
; CAA Records
; NS Records
@	3600	 IN 	NS	ns09.domaincontrol.com.
@	3600	 IN 	NS	ns10.domaincontrol.com.
```   





#### Zone 파일 업로드
오라클 클라우드로 로그인 한 후 DNS Management를 클릭 한후 Create Zone 을 클릭 한다.
주의 할 점은 등록한 도메인은 오라클 클라우드의 모든 엣지에 프로비젼되는 구조로 하나의 도메인이 중복하여 등록이 불가능 하다. 그럼으로 중복 등록되지 않도록 주의하여 관리 한다.   
Create Zone을 클릭 한다.   
![](/image/dns-zone/dnszone03.png)

생성 방법을 Import를 선택 하고 select a file 로 수정한 zone 파일을 선택 하거나 파일 내용을 복사한 후 Create 버튼을 클릭 한다.   
![](/image/dns-zone/dnszone04.png){: width="90%" height="90%"}

생성된 후 도메인에 들어가면 기존에 존재 하던 레코드가 생성 된 것을 볼 수 있다.   
![](/image/dns-zone/dnszone05.png)




#### NS 서버 이전
마지막으로 Godaddy 의 DNS 관리 페이지에서 네임서버의 주소를 Oracle DNS management 의 네임서버로 변경 하면 이전이 완료 된다.
등록할 네임서버는 오라클 DNS management에 등록한 DNS정보에 Zone information에서 찾을 수 있다.   
![](/image/dns-zone/dnszone06.png)

등록할 네임서버 정보   
![](/image/dns-zone/dnszone07.png){: width="80%" height="80%"}
