---
layout: post
title:  WAF SQL Blind Injection
date:   2020-03-13 13:00:00 +0900
categories: Edge WAF SQLInjection
---
# OWASP SQL Blind Injection
이전의 SQL Injection 공격은 SQL을 변경하여 원하는 데이터를 가져오게 하여 정보를 탈취하는 방법이다. 기존 공격 방법이 웹을 통하여 데이터베이스 메시지를 확인 할 수 있었다. 반면 메시지를 확인 할 수 없는 경우에는 Blind Injection 이라는 공격 방법을 통해 데이터를 탈취할 수 있다. 공격 방법은 SQL을 변경 하여 실행 하도록 하고 데이터 베이스 메시지 대신 서버로 부터 응답 결과 만을 보고 데이터를 탈취하는 방법 이다. 상자속에 든 물건을 알아보기 위해 손을 넣어서 만져보는 것처럼 보이지 않는 데이터를 알아보기 위해 다양한 공격을 하여 데이터를 탈취하는 것이다. 원하는 데이터를 탈취하기 위해 계속적으로 공격을 시도하여 결과를 비교하여 데이터를 탈취함으로 자동화 된 툴을 통해 공격이 이루어 진다.

#### Blind SQL Injection - Boolean based SQL
SQL 의 질의 결과가 참/거짓으로 하였을 때 서버의 반응으로 부터 데이터를 탈취하는 기술이다. 정상 요청의 경우 서버는 정상 반응을 하는 것을 확인 하고 변경된 질의를 하여 서버의 반응이 정상인지 아닌지를 보는 방법 이다. SQL injection과 유사하나 사용되는 SQL 의 패턴이 다르다. 원하는 데이터를 얻기 위해 하나씩 진행 하기 때문에 문자열 관련된 함수 (substr, limt 등)가 많이 사용 된다.

#### Blind SQL Injection - Time based SQL
SQL 의 결과를 볼 수 없을 때 사용 하는 공격 방법인 Blind SQL 의 패턴으로 시간을 이용하여 참/거짓을 구분 하는 공격 방법 이다. sleep 함수 등을 이용하여 데이터베이스의 종류 (mySQL의 경우 응답시간을 지연 시킬 수 있다)를 감지할 수 있고 해당 데이터베이스에 맞는 함수 IF, sleep 등을 이용하여 그 결과를 참인 경우 일정 시간 지연 되도록 하여 데이터를 탈취할 수 있다.


# SQL Blind Injection Detect
SQL Bind injection 공격을 감지 하기 위해서는 WAF Policies 의 protection rules 에서 Blind SQL injections, SQL injection using sleep() or benchmark() 를 선택 하여 Action 으로 detect 를 하여 줍니다. (기존에 SQL injection 을 감지하기 위해 Classic SQL Injection probings, Common SQL Injects를 선택 하였다 만약 선택하지 않았다면 2개도 detect 하여 준다)

![](/image/waf-sqlblind/waf-01.png)

변경된 내용은 아직 적용되어 있지 않음으로 publish All 버튼을 클릭 하여 적용 하여 줍니다. 적용이 시작되면 Updating 상태가 되고 전체 WAF 서버에 변경 된 정책을 적용하게 되어 약간의 시간이 소요 된다.

![](/image/waf-sqlblind/waf-02.png)

# SQL Blind Injection - boolean based Attack
SQL Injection 공격을 위해 웹애플리케이션에 로그인 한다. http://waf.korwaf.site/ 에 로그인 하면 메뉴 중 SQL Injection (blind)을 클릭 한다. 공격을 위해서는 정상적인 응답이 오는 경우를 알고 있어야 한다. 알고 있는 정보로는 1이라는 사용자가 있는 것을 알고 있음으로 1을 입력 후 Submit 을 눌러 보자. 아래 그림 처럼 1 이라는 사용자가 존재 한다는 것을 알 수 있다.

![](/image/waf-sqlblind/waf-03.png)

거짓인 데이터에 대해서 웹페이지의 반응을 보기 위해 아래와 같이 입력 하고 실행 해 준다.

```bash
' or '1'='2'    
```

![](/image/waf-sqlblind/waf-04.png)

위 그림처럼 거짓일 때는 ***User ID is MISSING from the database*** 라는 메시지가 출력 되는 것을 알 수 있다.

아래 문장을 입력 후 실행 하여 서버의 반응 보면
```bash
' union select 1 #
' union select 1,2  #
```

' union select 1,2 # 를 입력 하였을 때 참인 것을 볼 수 있다 이를 통해 실행 되는 SQL 의 Select 항목이 2개임을 알 수가 있다.

다음으로 데이터베이스 이름을 알기 위해 다음을 순서대로 입력 하고 실행 해 보자.
```bash
1' and length(database()) =1 
1' and length(database()) =2
1' and length(database()) =3 
1' and length(database()) =4 
```
***length(database())=4*** 일때 결과가 참인 것을 볼 수 있을 것이다. 이로서 데이터 베이스 이름의 길이가 4 라는 것을 알았고 다음을 입력 하여 데이터 베이스 이름을 유추 할 수 있다.
```bash
1' and substr(database(),1,1) = 'a' 
1' and substr(database(),1,1) = 'b' 
1' and substr(database(),1,1) = 'c' 
1' and substr(database(),1,1) = 'd' 
```

![](/image/waf-sqlblind/waf-05.png)

d 에서 참으로 나오는 것을 볼 수 있으며 이로서 데이터 베이스 이름의 길이는 4 이고 첫 글자라 d 라는 것을 알수 있다. 이와 같은 방법으로 알고자 하는 정보를 입력 하여 맞는지 아닌지를 계속적으로 체크하면 원하는 데이터를 탈취 할 수 있다.
이후에 테이블 명, 갯수를 확인 하고 이름, 컬럼 정보를 알아보는 SQL 을 수행하여 전체 정보를 탈취 하게 된다.

#### WAF의 SQL Blind Injection - boolean based 감지
WAF Policies에서 Metrics 를 보면 Detect 된 요청의 건수를 보여주는 대시보드를 볼 수 있다. 분단위로 WAF가 감지한 내용을 볼 수 있다.    

![](/image/waf-sqlblind/waf-06.png)

상세 내용을 보기  위해 Logs 를 클릭 하여 조회 조건 중 Action 에서 Detect 를 선택 하고 조회 한다.
Detect 된 리스트를 볼 수 있으며 상세 각 라인별로 간략한 내용을 볼 수 있다. 

![](/image/waf-sqlblind/waf-07.png)

접근을 시도한 지역과 User Agent 즉, 브라우저 정보를 볼 수 있다. 상세한 내용을 보기 위한 View JSON 을 클릭 한다. 

```json
"protectionRuleDetections": {
    "950001": {
      "Message": "SQL Injection Attack. Matched Data: substr( found within ARGS:id: 1' and substr(databse(),1,1) = 'd' ",
      "Message details": "Warning. Pattern match \"(?i:\\\\b(?:(?:s(?:t(?:d(?:dev(_pop|_samp)?)?|r(?:_to_date|cmp))|u(?:b(?:str(?:ing(_index)?)?|(?:dat|tim)e)|m)|e(?:c(?:_to_time|ond)|ssion_user)|ys(?:tem_user|date)|ha(1|2)?|oundex|chema|ig?n|pace|qrt)|i(?:s(null|_(free_lock|ipv4_compat|ipv4_mapped|ipv4| ...\" at ARGS:id."
    },
    "950007": {
      "Message": "Blind SQL Injection Attack. Matched Data: substr found within ARGS:id: 1' and substr(databse(),1,1) = 'd' ",
      "Message details": "Warning. Pattern match \"(?i:(?:\\\\b(?:(?:s(?:ys\\\\.(?:user_(?:(?:t(?:ab(?:_column|le)|rigger)|object|view)s|c(?:onstraints|atalog))|all_tables|tab)|elect\\\\b.{0,40}\\\\b(?:substring|users?|ascii))|m(?:sys(?:(?:queri|ac)e|relationship|column|object)s|ysql\\\\.(db|user))|c(?:onstraint ...\" at ARGS:id."
    },
    "959073": {
      "Message": "SQL Injection Attack. Matched Data: substr( found within ARGS:id: 1' and substr(databse(),1,1) = 'd' ",
      "Message details": "Warning. Pattern match \"(?i:(?:(?:s(?:t(?:d(?:dev(_pop|_samp)?)?|r(?:_to_date|cmp))|u(?:b(?:str(?:ing(_index)?)?|(?:dat|tim)e)|m)|e(?:c(?:_to_time|ond)|ssion_user)|ys(?:tem_user|date)|ha(1|2)?|oundex|chema|ig?n|pace|qrt)|i(?:s(null|_(free_lock|ipv4_compat|ipv4_mapped|ipv4|ipv ...\" at ARGS:id."
    },
    "981242": {
      "Message": "Detects classic SQL injection probings 1/2. Matched attack string: [1' and substr(databse(),1,1) = 'd' ] in [ARGS:id];",
      "Message details": "Warning. Pattern match \"(?i:(?:[\\\"'`\\xc2\\xb4\\xe2\\x80\\x99\\xe2\\x80\\x98]\\\\s*?(x?or|div|like|between|and)\\\\s*?[\\\"'`\\xc2\\xb4\\xe2\\x80\\x99\\xe2\\x80\\x98]?\\\\d)|(?:\\\\\\\\x(?:23|27|3d))|(?:^.?[\\\"'`\\xc2\\xb4\\xe2\\x80\\x99\\xe2\\x80\\x98]$)|(?:(?:^[\\\"'`\\xc2\\xb4\\xe2\\x80\\x99\\xe2\\x80\\x98\\\\\\\\]*?(?:[\\\\ ...\" at ARGS:id."
    },
    "981243": {
      "Message": "Detects classic SQL injection probings 2/2. Matched attack string: [1' and substr(databse(),1,1) = 'd' ] in [ARGS:id];",
      "Message details": "Warning. Pattern match \"(?i:(?:[\\\"'`\\xc2\\xb4\\xe2\\x80\\x99\\xe2\\x80\\x98]\\\\s*?\\\\*.+(?:x?or|div|like|between|and|id)\\\\W*?[\\\"'`\\xc2\\xb4\\xe2\\x80\\x99\\xe2\\x80\\x98]\\\\d)|(?:\\\\^[\\\"'`\\xc2\\xb4\\xe2\\x80\\x99\\xe2\\x80\\x98])|(?:^[\\\\w\\\\s\\\"'`\\xc2\\xb4\\xe2\\x80\\x99\\xe2\\x80\\x98-]+(?<=and\\\\s)(?<=or|xor ...\" at ARGS:id."
    }
  },  
```

메시지를 살펴 보면 SQL Injection Attack 이 발생 하였으며 감지된 패턴이 id 파라미터에 ***1' and substr(databse(),1,1) = 'd'*** 이 들어간 것이 감지 된 것을 알 수 있다.
또한 Blind SQL Injection Attack 으로 substr 이 감지 된 것을 볼 수 있다. 공격이 발생 하였을 때 한개의 패턴 만이 발생 하는 것보다 다수의 패턴이 동시에 사용 될 수 있으며 이 중 감지 되는 내용이 보여 지고 메시지 번호 등을 통해 정확한 내용을 검색 해 볼 수 있다.

다시 웹애플리케이션으로 돌아가 View Source를 클릭 하면 다음과 같은 코드로 작성되어 있음을 볼 수 있다.

```php
<?php

if( isset( $_GET[ 'Submit' ] ) ) {
    // Get input
    $id = $_GET[ 'id' ];

    // Check database
    $getid  = "SELECT first_name, last_name FROM users WHERE user_id = '$id';";
    $result = mysqli_query($GLOBALS["___mysqli_ston"],  $getid ); // Removed 'or die' to suppress mysql errors

    // Get results
    $num = @mysqli_num_rows( $result ); // The '@' character suppresses errors
    if( $num > 0 ) {
        // Feedback for end user
        echo '<pre>User ID exists in the database.</pre>';
    }
    else {
        // User wasn't found, so the page wasn't!
        header( $_SERVER[ 'SERVER_PROTOCOL' ] . ' 404 Not Found' );

        // Feedback for end user
        echo '<pre>User ID is MISSING from the database.</pre>';
    }

    ((is_null($___mysqli_res = mysqli_close($GLOBALS["___mysqli_ston"]))) ? false : $___mysqli_res);
}

?> 
```
SQL 실행이 $id 로 직접 웹페이지의 Query 파라미터를 받아 sql 문자열을 만든 후에 실행되며 실행 결과는 선택되는 레코드의 존재에 여부에 따라 메시지가 보여지는 것을 알 수 있다. 레코드가 있다 없다라는 정보를 통해 원하는 정보를 탈 취 할 수 있는 취약한 코드가 되는 것이다. 취약성점을 방지 하기 위해 코드를 수정 하거나 WAF 에서 감지 하여 공격자 정보를 파악 하고 Block 하는 조취를 하는 것이 필요하다

# Blind SQL Injection - Time based SQL Attack
SQL Injection 공격을 위해 웹애플리케이션에 로그인 한다. http://waf.korwaf.site/ 에 로그인 하면 메뉴 중 SQL Injection (blind)을 클릭 한다. 해당 페이지는 조회 결과의 유무에 따라 보여지는 메시지가 달라 boolean 을 이용한 공격이 가능하지만 시간을 이용한 공격을 위해 메시지를 볼 수 없는 것으로 가정 한다.    

우선 다음과 같이 입력하여 ***1' and sleep(15) #*** 한 후 Submit 을 클릭 하여 보면 응답 시간이 느리게 오는 것을 알 수 있다. 이를 통해 웹 애플리케이션이 사용하는 데이터베이스가 mySQL 인것을 알 수 있다. 데이터 베이스 명을 알아 보기 위해 다음을 입력 하여 실행 하여 보자.

![](/image/waf-sqlblind/waf-04.png)

위 그림처럼 거짓일 때는 ***User ID is MISSING from the database*** 라는 메시지가 출력 되는 것을 알 수 있다.

```bash
1' and IF (length(database()) =1, sleep(15),0) # 
1' and IF (length(database()) =2, sleep(15),0) # 
1' and IF (length(database()) =3, sleep(15),0) # 
1' and IF (length(database()) =4, sleep(15),0) # 
```

응답 시간이 ***1' and IF (length(database()) =4, sleep(15),0) #*** 일 때 느려지는 것을 확인 할 수 있다. 이를 통해 데이터 베이스 이름의 길이가 4라는 것을 알 수 있다.

![](/image/waf-sqlblind/waf-08.png)

#### WAF의 SQL Blind Injection - Time based 감지
WAF 의 Metric 에서 detect 항목이 추가된 것을 확인 할 수 있으며 Logs 에서 메시지를 확인 하여 보면 다른 Detect 항목과 동일 하게 클라이언트 정보 (지역, 브라우저)를 볼 수 있다.
자세한 정보를 보기 위해 View JSON 을 클릭 하여 보면 다음 메시지를 확인 할 수 있다.

```json
"protectionRuleDetections": {
    "950001": {
      "Message": "SQL Injection Attack. Matched Data: IF ( found within ARGS:id: 1' and IF (length(database()) =4, sleep(15),0) #",
      "Message details": "Warning. Pattern match \"(?i:\\\\b(?:(?:s(?:t(?:d(?:dev(_pop|_samp)?)?|r(?:_to_date|cmp))|u(?:b(?:str(?:ing(_index)?)?|(?:dat|tim)e)|m)|e(?:c(?:_to_time|ond)|ssion_user)|ys(?:tem_user|date)|ha(1|2)?|oundex|chema|ig?n|pace|qrt)|i(?:s(null|_(free_lock|ipv4_compat|ipv4_mapped|ipv4| ...\" at ARGS:id."
    },
    "959073": {
      "Message": "SQL Injection Attack. Matched Data: IF ( found within ARGS:id: 1' and IF (length(database()) =4, sleep(15),0) #",
      "Message details": "Warning. Pattern match \"(?i:(?:(?:s(?:t(?:d(?:dev(_pop|_samp)?)?|r(?:_to_date|cmp))|u(?:b(?:str(?:ing(_index)?)?|(?:dat|tim)e)|m)|e(?:c(?:_to_time|ond)|ssion_user)|ys(?:tem_user|date)|ha(1|2)?|oundex|chema|ig?n|pace|qrt)|i(?:s(null|_(free_lock|ipv4_compat|ipv4_mapped|ipv4|ipv ...\" at ARGS:id."
    },
    "981243": {
      "Message": "Detects classic SQL injection probings 2/2. Matched attack string: [1' and IF (length(database()) =4, sleep(15),0) #] in [ARGS:id];",
      "Message details": "Warning. Pattern match \"(?i:(?:[\\\"'`\\xc2\\xb4\\xe2\\x80\\x99\\xe2\\x80\\x98]\\\\s*?\\\\*.+(?:x?or|div|like|between|and|id)\\\\W*?[\\\"'`\\xc2\\xb4\\xe2\\x80\\x99\\xe2\\x80\\x98]\\\\d)|(?:\\\\^[\\\"'`\\xc2\\xb4\\xe2\\x80\\x99\\xe2\\x80\\x98])|(?:^[\\\\w\\\\s\\\"'`\\xc2\\xb4\\xe2\\x80\\x99\\xe2\\x80\\x98-]+(?<=and\\\\s)(?<=or|xor ...\" at ARGS:id."
    },
    "981272": {
      "Message": "Detects blind sqli tests using sleep() or benchmark().",
      "Message details": "Warning. Pattern match \"(?i:(sleep\\\\((\\\\s*?)(\\\\d*?)(\\\\s*?)\\\\)|benchmark\\\\((.*?)\\\\,(.*?)\\\\)))\" at ARGS:id."
    }
  }
```

일반적인 SQL Injection 공격으로 들어온 것이 감지 되었으며 추가로 blind sql 로 sleep() 혹은 benchmark() 를 사용한 공격이 감지 되었다는 것을 확인 할 수 있다.