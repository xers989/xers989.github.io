---
layout: post
title:  WAF SQL Injection
date:   2020-03-12 00:50:00 +0900
categories: Edge WAF SQLInjection
---
# OWASP SQL Injection
첫 번째로 WAF를 이용하여 SQL Injection 공격을 감지하고 이를 막는 방법을 알아보겠다.
SQL 인젝션은 웹 사이트의 보안의 헛점을 이용하여 특정 SQL 쿼리 문을 전송하여 데이터베이스로 부터 데이터를 탈취하는 해킹 기법을 말한다. 클라이언트의 입력을 제대로 필터링 하지 못하여 발생하며 그 피해는 치명적일 수 있어 항상 가장 높은 위협 순위로 꼽히곤 한다. 예를 들어 사용자를 조회 하는 SQL과 웹페이지를 예로 들면 id를 받아서 테이블에서 조회하고 결과를 반환하는 형태일 것이다. 웹페이지는 아마도 http://******/user?id=gildong 형태로 조회 할 것이다.     
![](/image/waf-sqlinjection/waf-01.png)


실행 되는 SQL은 아마도 select id from user where id = 'gildong' 과 같을 것이다. 공격자는 SQL을 예측하여 쿼리를 수정하여 gildong' or '1'='1 형식으로 작성하면 실행되는 SQL은 select id from user where id = 'gildong' or '1' = '1' 로 변경되어 실행 되어 user 테이블로 부터 정보를 탈취할 수 있다.

이러한 공격 방법은 매우 다양하다.

#### Error based SQL Injection
or '1'='1' 과 같은 문장을 추가 하여 항상 참이 되게 하여 공격하는 방법 이다. 로그인 과정에서 사용될 수 있는 Select * from user where id='gildong' and password='abcd' 를 예측하고 공격을 Select * from user where id='gildong' and password ='none' or '1'='1' 이 실행되도록 하면 문장이 항상 참이기 때문에 로그인이 성공 하게 된다.

#### UNION based SQL Injection
SQL 의 Union 문장을 이용하여 추가 SQL이 실행 되도록 하는 공격 방법이다. 테이블의 정보를 탈취하고자 할 때 사용 될 수 있다. Select id from user where id='gildong' and password='none' union select id from user where '1'='1' 과 같이 변경하여 주면 모든 id를 조회 하게 된다.


# SQL Injection Detect
2개가 SQL Injection을 검출하기 위해 해당 공격을 감지 하도록 WAF의 Protect Rule을 설정 하여야 한다. OCI의 WAF 설정으로 들어간다. (Security 탭의 WAF Policies)
WAF를 설정한 Compartment를 선택 하고 등록된 waf policy를 클릭    

![](/image/waf-sqlinjection/waf-02.png)

Protection Rules에서 SQL Injection의 Action을 Detect 로 변경 하여 준다. Classic SQL Injection probings, Common SQL Injects를 선택 하고 Detect를 선택 한다.

![](/image/waf-sqlinjection/waf-03.png)

변경된 설정은 바로 적용되지 않고 unpublished 상태로 있게 된다. 변경이 감지되면 상단에 unpublished list에 나오게 되며 Publish All 버튼을 눌러주면 글로벌하게 변경된 정책이 적용 된다.   
![](/image/waf-sqlinjection/waf-04.png)

변경된 정책의 적용에 수분이 소요 됨으로 잠시 대기 하여 준다. 적용이 완료 되면 Active 상태가 된다.

# Error based SQL Injection Attack
SQL Injection 공격을 위해 웹애플리케이션에 로그인 한다. http://waf.korwaf.site/ 에 로그인 하면 메뉴 중 SQL Injection을 클릭 한다. 공격을 위해 1을 입력 하고 Submit을 누르면 아래 처럼 1번에 대한 사용자 정보가 출력 되는 것을 볼 수 있다.   
![](/image/waf-sqlinjection/waf-05.png)

공격을 위해 다음과 같이 입력 하고 Submit을 클릭 하면

```bash
1' or '1'='1'    
```

아래 처럼 테이블에 정보가 조회 되는 것을 볼 수 있다.

![](/image/waf-sqlinjection/waf-06.png)

이로서 테이블에는 admin, Gordon, Hack, Pablo, Bob 등이 있다는 것을 알 수 있다

#### WAF의 SQL Injection 감지
WAF Policies에서 Metrics 를 보면 Detect 된 요청의 건수를 보여주는 대시보드를 볼 수 있다. 분단위로 조회 하였음으로 3 건 혹은 1건식 Detect 된 내용이 있음을 알 수 있다.   

![](/image/waf-sqlinjection/waf-07.png)

상세 내용을 보기  위해 Logs 를 클릭 하여 조회 조건 중 Action 에서 Detect 를 선택 하고 조회 한다.
Detect 된 리스트를 볼 수 있으며 상세 각 라인별로 간략한 내용을 볼 수 있다. 

![](/image/waf-sqlinjection/waf-08.png)

접근을 시도한 지역과 User Agent 즉, 브라우저 정보를 볼 수 있다. 상세한 내용을 보기 위한 View JSON 을 클릭 한다. 

```json
{
  "action": "DETECT",
  "clientAddress": "203.234.105.56", 
  "countryName": "Republic of Korea",
  "userAgent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:73.0) Gecko/20100101 Firefox/73.0",
  "domain": "waf.korwaf.site",
  "protectionRuleDetections": {
    "959071": {
      "Message": "SQL Injection Attack. Matched Data: ' or '1'= found within ARGS:id: 1' or '1'='1",
      "Message details": "Warning. Pattern match \"(?i:\\\\bor\\\\b ?(?:\\\\d{1,10}|[\\\\'\\\"][^=]{1,10}[\\\\'\\\"]) ?[=<>]+|(?i:'\\\\s+x?or\\\\s+.{1,20}[+\\\\-!<>=])|\\\\b(?i:x?or)\\\\b\\\\s+(\\\\d{1,10}|'[^=]{1,10}')|\\\\b(?i:x?or)\\\\b\\\\s+(\\\\d{1,10}|'[^=]{1,10}')\\\\s*?[=<>])\" at ARGS:id."
    },
    "981242": {
      "Message": "Detects classic SQL injection probings 1/2. Matched attack string: [1' or '1'='1] in [ARGS:id];",
      "Message details": "Warning. Pattern match \"(?i:(?:[\\\"'`\\xc2\\xb4\\xe2\\x80\\x99\\xe2\\x80\\x98]\\\\s*?(x?or|div|like|between|and)\\\\s*?[\\\"'`\\xc2\\xb4\\xe2\\x80\\x99\\xe2\\x80\\x98]?\\\\d)|(?:\\\\\\\\x(?:23|27|3d))|(?:^.?[\\\"'`\\xc2\\xb4\\xe2\\x80\\x99\\xe2\\x80\\x98]$)|(?:(?:^[\\\"'`\\xc2\\xb4\\xe2\\x80\\x99\\xe2\\x80\\x98\\\\\\\\]*?(?:[\\\\ ...\" at ARGS:id."
    },
    "981243": {
      "Message": "Detects classic SQL injection probings 2/2. Matched attack string: [1' or '1'='1] in [ARGS:id];",
      "Message details": "Warning. Pattern match \"(?i:(?:[\\\"'`\\xc2\\xb4\\xe2\\x80\\x99\\xe2\\x80\\x98]\\\\s*?\\\\*.+(?:x?or|div|like|between|and|id)\\\\W*?[\\\"'`\\xc2\\xb4\\xe2\\x80\\x99\\xe2\\x80\\x98]\\\\d)|(?:\\\\^[\\\"'`\\xc2\\xb4\\xe2\\x80\\x99\\xe2\\x80\\x98])|(?:^[\\\\w\\\\s\\\"'`\\xc2\\xb4\\xe2\\x80\\x99\\xe2\\x80\\x98-]+(?<=and\\\\s)(?<=or|xor ...\" at ARGS:id."
    }
  },
  "httpMethod": "GET",
  "requestUrl": "/vulnerabilities/sqli/?id=1%27+or+%271%27%3D%271&Submit=Submit",
  "httpHeaders": {
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8",
    "Accept-Encoding": "gzip, deflate",
    "Accept-Language": "en-US,en;q=0.5",
    "Cache-Control": "max-age=0",
    "Connection": "keep-alive",
    "Cookie": "PHPSESSID=d5ndqi2g5olrddv02qjp8bvdk0; security=low",
    "Host": "waf.korwaf.site",
    "Referer": "http://waf.korwaf.site/vulnerabilities/sqli/?id=admin%27+or+1%3D1+%23&Submit=Submit",
    "Request-Id": "2020-03-12T12:26:13Z|8a3fcdc114|203.234.105.56|DSLYxGAAHQ",
    "Upgrade-Insecure-Requests": "1",
    "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:73.0) Gecko/20100101 Firefox/73.0",
    "X-Client-Ip": "203.234.105.56",
    "X-Country-Code": "KR",
    "X-Forwarded-For": "203.234.105.56"
  },
  "httpVersion": "HTTP/1.1",
  "incidentKey": "2020-03-12T12:26:13Z|8a3fcdc114|203.234.105.56|DSLYxGAAHQ",
  "countryCode": "KR",
  "timestamp": "Thu, 12 Mar 2020 12:26:13 GMT",
  "logType": "PROTECTION_RULES"
}   
```

위 내용을 보고 접속한 클라이 언트의 상세 정보를 볼 수 있으며 protectionRuleDetections 을 보면 http://waf.korwaf.site/vulnerabilities/sqli 페이지에서 SQL Injection Attack 이 감지 된 것을 알 수 있으며 감지된 데이터는 ' or '1'= 이며 전달된 변수는 id 란 것을 알 수 있다.

다시 웹애플리케이션으로 돌아가 View Source를 클릭 하면 다음과 같은 코드로 작성되어 있음을 볼 수 있다.

```php
<?php

if( isset( $_REQUEST[ 'Submit' ] ) ) {
    // Get input
    $id = $_REQUEST[ 'id' ];

    // Check database
    $query  = "SELECT first_name, last_name FROM users WHERE user_id = '$id';";
    $result = mysqli_query($GLOBALS["___mysqli_ston"],  $query ) or die( '<pre>' . ((is_object($GLOBALS["___mysqli_ston"])) ? mysqli_error($GLOBALS["___mysqli_ston"]) : (($___mysqli_res = mysqli_connect_error()) ? $___mysqli_res : false)) . '</pre>' );

    // Get results
    while( $row = mysqli_fetch_assoc( $result ) ) {
        // Get values
        $first = $row["first_name"];
        $last  = $row["last_name"];

        // Feedback for end user
        echo "<pre>ID: {$id}<br />First name: {$first}<br />Surname: {$last}</pre>";
    }

    mysqli_close($GLOBALS["___mysqli_ston"]);
}

?> 
```
SQL 실행이 $id 로 직접 웹페이지의 Query 파라미터를 받아 sql 문자열을 만든 후에 실행 되는 형태로 작성하여 SQL injection 공격에 취약해 진것을 볼 수 있다. 공격이 감지 되었음으로 코드를 수정하여 대비하거나 WAF 의 Protection Rule 의 Action을 block 으로 하여 해당 접근을 차단 할 수 있다.

# UNION based SQL Injection Detect
Union based Injection 공격을 위해 웹애플리케이션의 SQL Injection을 클릭 한다. 공격을 위해 다음과 같이 union 을 포함하여 작성 한 후 submit 을 클릭 한다.

```bash
' union all select user_id as first_name, first_name from users #
```

결과는 다음과 같이 출력 된다.   

![](/image/waf-sqlinjection/waf-09.png)
union all 로 원하는 SQL 을 실행 시켰으며 \# 을 붙여 주어 뒤에 올 수 있는 SQL 문장을 주석으로 처리 하도록 한 것이다.
이로서 user_id는 1,2,3,4,5 가 있다는 것을 알 수 있다.

#### WAF의 SQL Injection 감지
WAF Policies 의 logs 를 조회 하여 보면 Detect 가 된 요청을 볼 수 있다.

![](/image/waf-sqlinjection/waf-10.png)

View JSON 을 클릭 하면 아래와 같은 메시지를 볼 수 있으며 id 변수에 union all select user_id as first_name, first_name from users # 이란 변수가 전달 되어 SQL Injection 으로 의심되는 패턴이 입력 되어 감지 된 것을 알 수 있다. 

```json
"protectionRuleDetections": {
    "950001": {
      "Message": "SQL Injection Attack. Matched Data: union all select found within ARGS:id: ' union all select user_id as first_name, first_name from users #",
      "Message details": "Warning. Pattern match \"(?i:\\\\b(?:(?:s(?:t(?:d(?:dev(_pop|_samp)?)?|r(?:_to_date|cmp))|u(?:b(?:str(?:ing(_index)?)?|(?:dat|tim)e)|m)|e(?:c(?:_to_time|ond)|ssion_user)|ys(?:tem_user|date)|ha(1|2)?|oundex|chema|ig?n|pace|qrt)|i(?:s(null|_(free_lock|ipv4_compat|ipv4_mapped|ipv4| ...\" at ARGS:id."
    },
    "959073": {
      "Message": "SQL Injection Attack. Matched Data: union all select found within ARGS:id: ' union all select user_id as first_name, first_name from users #",
      "Message details": "Warning. Pattern match \"(?i:(?:(?:s(?:t(?:d(?:dev(_pop|_samp)?)?|r(?:_to_date|cmp))|u(?:b(?:str(?:ing(_index)?)?|(?:dat|tim)e)|m)|e(?:c(?:_to_time|ond)|ssion_user)|ys(?:tem_user|date)|ha(1|2)?|oundex|chema|ig?n|pace|qrt)|i(?:s(null|_(free_lock|ipv4_compat|ipv4_mapped|ipv4|ipv ...\" at ARGS:id."
    }
  },
```