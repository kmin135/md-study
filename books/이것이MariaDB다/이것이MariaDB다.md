# 개요

* 이것이MariaDB다 / 우재남 / 한빛미디어 / 2019년
* MariaDB 10.3.x 기준으로 실습
* https://cafe.naver.com/thisismysql 에서 실습코드 제공

# 기본소감

* DBMS를 처음접하는 초보자용으로 작성된 책임
* 필요한 부분만 편집해서 정리함

# 실습환경구성

* 실습파일 디렉터리에 작성한 docker-compose.yml 로 시놀로지에 mariadb container 올려서 실습함
* `additional.cnf` 가 잘 반영되야 기본 인코딩이 `utf8mb4` 가 되니 주의. 또한 해당 파일은 리눅스에서는 `644` 권한 줘야함.

```bash
sudo docker-compose up -d
# 잘 안 되면 로그 확인
sudo docker-compose logs -f
```

# 클라이언트 프로그램

## 모델링툴

* 교재에서 소개한 모델링 툴은 `dbForge Studio`, `SQLyog` 정도. 둘 다 제대로된 사용을 하려면 유료임.
  * 유명한 건 `ERwin` 인데 매우 고가
* 검색 좀 해봤으나 역시 무료로 쓸만한건 `Mysql Workbench` 정도인듯.
  * mysql 용이라 완벽호환은 아니겠으나 기본적인 모델링에는 문제가 없음
* `Database Workbench` 도 무료버전이 존재하는데 개인개발 및 비상업용도로만 가능

## 일반 클라이언트

* 모델링도구 검색하다가 `dbeaver` 무료버전 깔아봤는데 모델링 도구는 없지만 일반 클라이언트용으로 좋음
  * 여러 유료 에디션도 존재함
  * quest 사의  `Toad For Mysql` 을 대체할만해보임. 특히 이 툴은 EOS 되고 `Toad Edge` 라는 열화판으로 대체되서 더욱 더 그럼.
* 다만 Intellij 결제하면 datagrip 을 쓸 수 있으므로 이 책의 학습에는 datagrip을 사용함

# Chap06 SQL 기본

## quote(') 와 backtick(`)

* 쿼리에서 quote 는 문자열 리터럴을 의미하고
* backtick 은 칼럼을 의미한다
* double quote(") 는 sql_mode에 따라서 quote 또는 backtick 처럼 동작한다.
  * 따라서 혼란을 막기 위해 double quote 는 안 쓰는게 좋다.

## ANY, SOME, ALL

```sql
-- 서브쿼리의 결과 중에서 하나라도 만족하는 행을 얻음
-- 아래에서는 서브쿼리의 결과가 170, 173 두 행이 나오므로 170이상 이거나 173이상이라는 의미가 됨
-- 결국 170이상인 모든 유저라는 의미가 됨
-- SOME 은 ANY 와 동일함
select * from usertbl
where height >= ANY (select height from usertbl where addr = '경남');

-- 반면에 ALL은 서브쿼리의 결과를 모두 만족해야하므로 173 이상이라는 의미가 됨
select * from usertbl
where height >= ALL (select height from usertbl where addr = '경남');

-- '= ANY (서브쿼리)' 는 `IN (서브쿼리)` 와 동일
select * from usertbl
where height = ANY (select height from usertbl where addr = '경남');
```

## ROLLUP

* group by 와 함께 사용하여 총합, 중간 합계를 얻을 수 있음
* https://mariadb.com/kb/en/select-with-rollup/

```sql
SELECT year, SUM(sales) FROM booksales GROUP BY year;
+------+------------+
| year | SUM(sales) |
+------+------------+
| 2014 |     173944 |
| 2015 |     180518 |
+------+------------+
2 rows in set (0.08 sec)

SELECT year, SUM(sales) FROM booksales GROUP BY year WITH ROLLUP;
+------+------------+
| year | SUM(sales) |
+------+------------+
| 2014 |     173944 |
| 2015 |     180518 |
| NULL |     354462 |
+------+------------+
```

## WITH 절과 CTE

* WITH 절은 CTE (Common Table Expression) 를 표현하기 위한 구문
* 기존의 뷰, 파생 테이블, 임시 테이블 등의 역할을 대신할 수 있음
* CTE는 ANSI-SQL99 표준에서 도입됨
  * MariaDB는 10.2.1 부터 도입됨. 추가로 Recursive 방식은 10.2.2 부터 가능하고, Recursive 방식일 때 CYCLE 디텍션은 10.5.2 부터 가능함.
* 비재귀적과 재귀적 2가지 방식으로 나뉨
* https://mariadb.com/kb/en/common-table-expressions/

```
WITH CTE_테이블이름(열이름)
AS
(
    쿼리문
)
SELECT 열이름 FROM CTE_테이블이름;
```
```sql
WITH temp(userid, total)
AS ( SELECT userid AS '사용자', SUM(price*amount) AS '총구매액'
     FROM buytbl GROUP BY userid )
SELECT * FROM temp ORDER BY total;
```

* 위와 같이 WITH 구문으로 일시적으로 참조할수 있는 테이블을 정의하고 FROM 절에 사용할 수 있음
* 이를 통해 복잡한 쿼리를 단순화시킬 수 있음
* result set에 임시로 이름을 지어준 것이라 볼 수 있음
> Common Table Expressions (CTEs) are a standard SQL feature, and are essentially temporary named result sets.

```
WITH AAA (칼럼들)
AS ( AAA의 쿼리문 ),
    BBB (칼럼들)
        AS ( BBB의 쿼리문 )
SELECT * FROM [AAA 또는 BBB]
```

* 위 처럼 중복 CTE도 가능함
* 단, BBB에서 AAA를 참조할 수 있지만 그 반대는 안 됨. (아직 정의되지 않은 CTE를 미리 참조할 수 없다.)

```sql
-- https://stackoverflow.com/questions/50799157/use-a-cte-to-update-or-delete-in-mysql
WITH ToDelete AS 
(
   SELECT ID,
          ROW_NUMBER() OVER (PARTITION BY lastName, firstName ORDER BY ID) AS rn
   FROM mytable
)   
DELETE FROM mytable USING mytable JOIN ToDelete ON mytable.ID = ToDelete.ID
WHERE ToDelete.rn > 1; 
```

* 위와 같이 delete나 update에도 응용할 수 있음
* cte 자체를 수정하거나 삭제하는건 안 됨

# Chap07 SQL 고급

## Data Types

* 대략적인 내용만 정리, 크기 등의 구체적인 스펙은 아래 공식 문서를 참조
* https://mariadb.com/kb/en/data-type-storage-requirements/
### 숫자 데이터

* 정수형 : 기본은 INT, 큰수는 BIGINT
* 실수형 : 기본은 FLOAT, 큰수는 DOUBLE
  * 프로그래밍 언어와 마찬가지로 매우 큰 수를 저장할 수 있지만 소수점 이하 n자리(float 7, double 15)까지만 정확도가 보존되므로 정확한 값이 필요한 경우 주의
* 정확한 실수형 : `DECIMAL(m,[d])` 은 명시적으로 전체 자리수와 소수점 이하 자리수를 지정할 수 있어 정확한 실수를 저장하기 용이함, 가변 바이트임.
  * DECIMAL(5,2) : 전체 자리수 5자리, 소수점 이하 2자리  `ex. 123.45`
  * 21-11-28기준 10.2.1 버전 이후로는 m은 최대 65, d는 최대 38
  * m (기본10), d (기본0) 모두 생략가능하나 DECIMAL을 쓸거라면 목적에 맞게 명확히 정의하는게 좋겠다.
* 그 외에 `BIT(N)`, TINYINT 등이 존재하니 용량에 민감한 테이블을 설계한다면 고려
* `UNSIGNED` 설정이 가능하므로 음수가 필요없고 더 큰 값을 저장하기를 원할 경우 고려

### 문자 데이터

* 고정길이 : CHAR (최대 255)
  * 실제값의 크기 관계 없이 설정한 크기를 모두 사용
* 가변길이 : VARCHAR(M) (최대 65532)
  * 지정가능한 최대 M은 최대 row size와 인코딩에 따라 다름.
  * 예를 들어 utf8 일 경우 1자에 3bytes로 잡아서 21,844 까지 지정가능함. 그러나 다른 char 타입들이 있을 경우 최대 row size에 걸려서 그 미만으로 잡아야할 수 있음.
  * INSERT/UPDATE 시의 속도는 CHAR 보다 상대적으로 느림.
* TEXT : TEXT (최대 64KB), LONGTEXT (최대 약 4GB)
  * 실제 저장할 수 있는 최대 문자 길이는 인코딩에 따라 다름. 
  * 또한 client, server의 최대 패킷 크기 설정이나 가용 메모리에 따라 최대값만큼 사용하지 못 할 수 있으니 주의 필요.
* ENUM
* SET

> 생각거리 : CHAR vs VARCHAR
>
> > 이 책도 그렇고 성능적인 면에서 CHAR 가 유리하다는게 일반적인 설명이고 성능에 대한 비교는 인터넷에서도 많은 토론이 있음. 내 개인적인 결론은 성능이 좋아지는게 맞긴 한데 Trailing spaces 의 처리가 sql_mode에 따라 달라지는 문제가 있고 오늘날의 컴퓨팅 성능에서 둘의 성능차이는 큰 의미가 없으므로 개발 생산성 면에서 VARCHAR 로 기준을 통일하는게 좋다고 봄. 

 ```sql
 -- char, varchar 의 차이점
CREATE OR REPLACE TABLE strtest (c CHAR(10), vc VARCHAR(10));
INSERT INTO strtest VALUES('Maria   ', 'Maria   ');

-- 둘 다 true, true
SELECT c='Maria',c='Maria   ' FROM strtest;
SELECT vc='Maria',vc='Maria   ' FROM strtest;

-- true, false
SELECT c LIKE 'Maria',c LIKE 'Maria   ' FROM strtest;
-- false, true
SELECT vc LIKE 'Maria',vc LIKE 'Maria   ' FROM strtest;

-- (Maria),(Maria   )
SELECT CONCAT('(', c, ')'), CONCAT('(', vc, ')') FROM strtest;
 ```

> 생각거리 : ENUM, SET
>
> > 프로그래밍 언어의 enum과 set 역할을 할 수 있는 타입들임. 내 생각은 이런것들은 프로그래밍 언어에게 맡기고 DBMS 상에서는 VARCHAR 로 정의하는게 용이하다고 봄.

 ### 이진 데이터

* 공식 문서에서는 CHAR 들과 같이 String Data Type으로 구분하나 이진 데이터 저장에 특화된 타입들이므로 구분해서 작성
* 고정길이 : BINARY(n)
* 가변길이 : VARBINARY(n)
* BLOB : BLOB (최대 64KB), LONGBLOB (최대 약 4GB)

> 생각거리 : 이진 데이터의 DB 저장
>
> > 안 좋은 생각임. 일단 비용면에서 DBMS 스토리지가 통상의 NAS 보다 비쌈. 성능 면에서도 일반적인 쿼리하기도 바쁜 DB에 이미지나 영상같은 blob을 저장하고 읽는것은 낭비임.

### 날짜와 시간 데이터 형식

* YEAR : 년도만 `YYYY`
* DATE : 날짜만 `YYYY-MM-DD`
* TIME : 시간만 `HH:MM:SS`, microsecond (0~6) 지정 가능
* DATETIME : 날짜+시간, `YYYY-MM-DD HH:MM:SS`, microsecond (0~6) 지정 가능
  * 자동 타임존 변환을 제공하지 않음
  * `1000-01-01 00:00:00.000000` ~ `9999-12-31 23:59:59.999999`
  * 8 bytes
* TIMESTMAP : DATETIME과 값 형식은 동일함
  * 현재 세션의 `time_zone` 값에 따라 자동 타임존 변환을 제공함
    * unix time 을 저장하는거라 가능한 것
  * UTC 기준 `1970-01-01 00:00:01` ~ `2038-01-19 03:14:07` 까지만 저장가능
  * 4 bytes 로 datetime 의 절반

```sql
CREATE OR REPLACE TABLE timetest (dt DATETIME, ts TIMESTAMP);
SET TIME_ZONE = "europe/london";
insert into timetest values(NOW(), NOW());
-- 런던으로 했으므로 utc+0 기준으로 저장되었고 값도 당연히 동일함
select dt, ts from timetest;

SET TIME_ZONE = "asia/seoul";
-- 서울(utc+9) 로 세션 time_zone을 변경하면 datetime은 값은 그대로고 timestamp 는 utc+9 이 적용됨
select dt, ts from timetest;
```

> 생각거리 : DATETIME vs TIMESTAMP
>
> > 용량이나 타임존 세팅을 해주는 점에서 timestamp 도 장점이 있으나 아무래도 지원 가능 범위의 이슈로 datetime이 무난하다. timezone 의 경우 app단에서 처리하면 된다.

### 기타

* GEOMETRY : 공간 데이터 형식으로 선, 점 및 다각형 같은 공간 데이터 개체를 저장하고 조작
* JSON : JSON 문서 저장에 특화된 필드



## 내장 함수

* 종종 필요하다고 생각되는 함수들 중에 정리할만한 것들만 작성함
* 필요하면 공식문서를 보자
  * https://mariadb.com/kb/en/built-in-functions/
* 형변환 : CAST, CONVERT

```sql
SELECT CAST('1' AS INTEGER);
```

* 길이 체크 함수

```sql
-- 순서대로 bits 크기, 문자수, bytes 크기
-- 72, 3, 9, 27, 3, 3
select bit_length('가나다'), char_length('가나다'), length('가나다'),
       bit_length('abc'), char_length('abc'), length('abc');
```

* 시스템 정보 함수

  > datagrip 에서는 결과가 잘 안나왔다. 해당 프로그램에서 결과 출력을 위해 먼저 호출해서 결과가 안 나오는 걸지도 모르겠다. 쉘로 접근해서 해보면 잘 된다.

```sql
-- 직전의 SELECT 문에서 조회된 행의 개수
select found_rows();
-- 직전의 update, insert, delete의 영향을 받은 행의 수
select row_count();
```

* 기타

```sql
-- 지정한 초만큼 일시정지 후 0을 리턴
-- 도중에 인터럽트되면 1을 리턴
select sleep(5);
-- 소수점 이하도 가능 (공식문서에는 마이크로초를 지정 가능하다고 함)
select sleep(1.5);
```

### REPEAT 를 이용한 용량 실험 (max_allowed_packet)

```sql
-- longtext는 약 4GB 까지 저장 가능
create table maxTbl (col1 longtext, col2 longtext);
-- repeat는 입력한 문자열을 지정횟수만큼 반복해주는 함수로 테스트에 용이함
insert into maxTbl values (repeat('a', 1000000), repeat('가', 1000000));
-- 1000000,3000000
-- 즉, 약 1MB, 3MB 임을 알 수 있음
select length(col1), length(col2) from maxTbl;

-- 0을 한 개씩 붙여 총 40MB 정도를 저장하려고 하니 max_allowed_packet 크기에 걸림
-- 버전에 따라 다른데 테스트 한 버전의 기본값은 16777216임 (16MB)
insert into maxTbl values (repeat('a', 10000000), repeat('가', 10000000));

-- 서버의 max_allowed_packet 값을 최대값인 1024MB 로 설정 후 재시작 한 뒤 다시하면 정상적으로 저장됨
show variables like '%max_allowed_packet%';

-- 다음과 같이 3GB를 살짝 넘는 데이터 저장을 시도하니 역시 max_allowed_packet 에 걸려서 에러가 남
insert into maxTbl values (repeat('a', 10000000), repeat('가', 1073741824));

-- 그래서 서버에서 직접 쉘로 붙어서 하니 1GB 이상도 packet 에러나는 안 남.
-- 근데 서버 스펙의 문제로 OOM 발생 
-- 좋은 서버라도 이런짓은 하지 말자
insert into maxTbl values (repeat('a', 50000000), repeat('a', 1073741824));
-- ERROR 5 (HY000): Out of memory (Needed 0 bytes)

/*
- 즉, longtext가 스펙상 4GB 까지 저장이 가능하지만 대부분의 케이스에서 dbms는 원격접속하므로 max_allowed_packet 의 최대값에 걸려 1GB 이상의 데이터를 저장할 수 없음을 알 수 있음.
- 하지만, 애초에 칼럼 하나에 1GB 이상의 데이터를 저장하려는 설계가 문제임. 비용면에서나 성능면에서도 잘못된 선택.
- 로컬 접속은 영향을 받지 않음도 알 수 있음. 소켓을 열지 않고 직접 접속이기 때문으로 보임.
- 참고로 max_allowed_packet 값은 서버와 클라이언트 각각 지정이 필요한데 클라이언트는 기본값이 1GB라 딱히 건드릴 일은 없음
- replication 할 때는 slave_max_allowed_packet 값이 따로 있으니 참고
https://mariadb.com/kb/en/server-system-variables/#max_allowed_packet
*/
```



## 윈도우 함수

* 윈도우 함수는 10.2.0 부터 도입됨
* https://mariadb.com/kb/en/window-functions/

---

* 현재 행과 관련된 일련의 행에서 계산을 수행할 수 있음
* 집계 함수와 비집계 함수로 구분됨



### 순위 함수

* 비집계 함수 중에서 순위를 표현하는 함수들
* RANK(), NTILE(), DENSE_RANK(), ROW_NUMBER() 등

```sql
/*
function (expression) OVER (
  [ PARTITION BY expression_list ]
  [ ORDER BY order_list [ frame_clause ] ] ) 
*/

-- 각 행의 관계속에서 순위를 쉽게 추출할 수 있음
select row_number() over (order by height desc, name asc) '키순서', 
  name, addr, height
from usertbl
order by `키순서`;

-- PARITION BY 를 활용해 전체 순위가 아니라 그룹별 순위를 추출할 수도 있음
select row_number() over (partition by addr order by height desc, name asc) '키순서',
  name, addr, height
from usertbl
order by addr, `키순서`;

-- DENSE_RANK(), RANK() 는 동일 순위가 있을 경우의 처리가 다르니 필요에 따라 사용

-- NTILE 은 순위를 매긴뒤 지정한 값만큼 그룹번호를 매김
select ntile(4) over (order by height desc, name asc) '반',
  name, addr, height
from usertbl
order by `반`;
```



### 분석 함수

* 비집계 함수 중에서 이동 평균, 백분율, 누계 등을 계산할 때 활용
* CUME_DIST(), LEAD(), LAG(), FIRST_VALUE(), PERCENT_RANK() 등

```sql
-- 키 내림차순 정렬한 뒤 각 행의 다음행의 키와의 차이를 얻는 쿼리
-- 마지막행은 비교할 다음행이 없으므로 null
-- LAG()는 LEAD와 방향이 반대
select name, addr, height '키',
       height - (LEAD(height, 1) OVER (order by height desc)) '다음 사람과 키 차이'
from usertbl;
```



## 피벗

* group by, sum, if 를 조합한 일반적인 피벗을 소개함

> 내의견 : 보통 mysql pivot 으로 찾아보면 group by 절 쓴 뒤 필드마다 sum 같은 집계함수 + if 문으로 노가다방식으로 짜는게 많다. 다만 이 방식은 열이 동적으로 늘어나는 케이스를 커버할 수 없다. 딱히 대안은 못 봤다.
>
> 또한 개발용이 아니라 데이터 추출용이라면 이런 노가다 방식 말고 보통 클라이언트 프로그램들이 결과를 토대로 pivot, tranpose 기능을 제공하기 때문에 그걸 이용하는게 나을 수도 있다.



## JSON

* 10.2 부터 JSON 데이터 처리를 지원하는 기능이 추가됨

```sql
-- DB의 데이터를 JSON 값으로 변환할 때는 JSON_OBJECT, JSON_ARRAY 를 사용
select JSON_OBJECT('name', name, 'height', height) AS 'JSON 값'
from usertbl;

/*
{"name": "바비킴", "height": 176}
{"name": "은지원", "height": 174}
*/

-- 반면 JSON 값을 받아 유효성체크, 검색, 변환 등을 담당하는 함수들도 있음
-- JSON_VALID, JSON_SEARCH, JSON_EXTRACT, JSON_INSERT, JSON_REPLACE, JSON_REMOVE
```



## 조인

* INNER JOIN
* OUTER JOIN (LEFT, RIGHT, FULL)

---

* CROSS JOIN
  * 한쪽 테이블의 모든 행들과 다른 쪽 테이블의 모든 행을 조인하는 것
  * 결과 행수는 두 테이블의 행수를 곱한 것이 됨
  * 즉, 카티션곱
  * 테스트 데이터 만드는 정도 말고 딱히 쓸 일은 없음

```sql
-- cross join 자체가 전체곱을 의미하므로 ON 절은 사용 불가
select * 
from buytbl
  cross join usertbl;
  
-- 아래와 동일함
select *
from buytbl, usertbl;
```

---

* SELF JOIN
  * 별도의 구문이 있는 건 아니고 자기 자신과 조인함을 의미
  * 예를 들어 회사 부서 테이블의 계층 구조를 표현할 때 사용할 수 있음

```sql
select l1.dept_name, l2.dept_name '상위 부서'
from dept l1
  join dept l2
    on l1.up_dept_cd = l2.dept_cd
```



## UNION

* 여러 SELECT 문의 결과를 합침
* 합쳐야하므로 두 SELECT 문의 결과 열의 개수 같아야하고 데이터 형식이 호환되야함
* 그냥 `UNION`은 중복을 제거하고 `UNION ALL` 은 중복을 포함함
  * `UNION` 은 실제로는 `UNION DISTINCT` 의 축약 표현임



## SQL 프로그래밍

* 프로시져 등에 활용하는 SQL 프로그래밍을 다루는 장.
* 가볍게 읽고 넘어감. 레거시로 개발된 프로시져를 고쳐야할 때나 공부하면 된다고 봄.

> 생각거리 : 프로시져를 써야할까?
>
>   여전히 쓰는 경우가 있겠으나 지양하는게 맞다고 봄. 이유는 DB는 WAS 대비 더 비싼 자원이고 스케일아웃이 힘듬. 따라서 스케일아웃이 용이한 WAS 단에 비즈니스 로직을 두고 DB에서는 단순 데이터관리만 하는게 좋다고 봄. 이를 통해 DBMS에 대한 종속성을 낮추는 효과도 있음. 또한 프로시져는 상대적으로 로깅이나 디버깅이 힘들다는 문제도 존재함.
>
>   또 프로시져를 쓰면 앱단에서는 뜬금없이 call any_complex_procedure(); 이런식의 호출이 등장하게 되는데 이러면 비즈니스 로직이 앱과 dbms 단으로 분산되서 전체 파악이 힘들어지는 문제도 존재함. mybatis 를 써도 그렇지만 orm 을 활용하는경우라면 더욱 더.

### 동적 SQL

* 자바의 PreparedStatement 처럼 SQL 을 동적으로 세팅하여 실행할 수 있음

```sql
SET @current = now();
-- 이름대로 동적쿼리를 준비만 함
PREPARE myQuery FROM 'SELECT ?';
-- 준비한 동적쿼리 실행
EXECUTE myQuery USING @current;
-- 생성한 동적쿼리를 해제
DEALLOCATE PREPARE myQuery;

-- LIMIT 절에는 유저정의변수가 올 수 없는데 동적 쿼리를 이용하면 가능함
SET @limit = 5;

-- ERROR!
-- SELECT * FROM usertbl LIMIT @limit;

PREPARE limitQuery FROM 'SELECT * FROM usertbl LIMIT ?';
EXECUTE limitQuery USING @limit;
DEALLOCATE PREPARE limitQuery;
```

# Chap08 테이블과 뷰

## 제약조건

```sql
-- 테이블에 정의된 KEY 보기
SHOW KEYS FROM userTbl;
```

### 기본키 제약조건

> 생각거리 : 기본키로 복합키 vs 인공키
>
>   흔한 주제인데 무엇보다 변경에 강하다는 점 때문에 인공키를 쓰는게 맞다고 생각한다. 앱단에서 봐도 JPA같은 ORM 을 쓸 때 복합키를 쓰면 PK 용으로 별도 클래스를 만드는 등의 복잡함이 생기므로 더욱 더 그렇다.
>
>   단점이라면 키만으로는 데이터를 유추할 수 없고 쿼리문이 상대적으로 복잡해지는 것인데 장점을 생각하면 감내할만하다.



### 외래키 제약조건

* FK 업데이트, 삭제시의 동작을 정의하는 reference_option 에는 RESTRICT (= NO ACTION), CASCADE, SET NULL, SET DEFAULT 가 있음
  * 생략하면 기본 옵션은 NO ACTION 임

```
[CONSTRAINT [symbol]] FOREIGN KEY
    [index_name] (index_col_name, ...)
    REFERENCES tbl_name (index_col_name,...)
    [ON DELETE reference_option]
    [ON UPDATE reference_option]
```



> 생각거리 : 외래키는 필수인가
>
>   이것 역시 논란의 주제. 찬성하는 측은 데이터 무결성을 보장하기 위해서는 원칙적으로 외래키를 쓰는게 맞으며 외래키를 사용함에 따른 약간의 성능 저하(외래키 제약조건을 지키기 위해 DB가 해야할 일들이 있으니)는 무시할만하다는 입장임. 약간의 성능저하도 오히려 앱단에서 무결성을 맞추기 위해 하는 작업들이 외래키가 발생시키는 성능 저하보다 더 클 수 있다는 의견임. 
>
>   안 쓰는 측은 외래키 때문에 테스트 데이터를 만들기가 어렵고 요구사항 변경에 따른 설계변경 때 외래키로 인한 번거로운 점이 있어 편의성 면에서 안 쓰는 입장. 
>
>   내 생각은 외래키를 안 쓰는 것은 SI 현장등에서 일정에 쫓기며 빠르게 작업하다보니 발생하는 안 좋은 관습이라고 본다. 나의 경우에는 전문 DBA 없이 직접 설계하다보니 그런 점도 있었다. 돌이켜보면 확실히 외래키 때문에 데이터 컨트롤이 귀찮아지지만 그만큼 DBMS 단에서 무결성을 보장한다는 큰 장점을 얻을 수 있기에 사용하는 것이 맞다고 본다. 설계변경시의 편리하다는 말도 단기적인 편리를 위해 장기적인 안정성을 버리는 선택일 것이다. 

### UNIQUE 제약조건

* 중복되지 않는 유일 값이란 점에서 PK와 유사하나 NULL을 허용함 (NULL은 여러 개 입력되도 무관)

### CHECK 제약조건 

* 필드에 입력되는 데이터를 점검하는 기능을 함. 예를 들면 생년월일을 나타내는 필드에 1900 이상의 값이 오도록 하는 것.
* ALTER로 적용할 경우 기존 입력값들을 무시하고 설정하는 것도 가능함

```sql
-- 위반하면 에러 발생. 
-- check_constraint_checks 시스템 변수를 0으로 하여 무시할 수도 있음
-- ...
birthYear INT CHECK (birthYear >= 1900),
-- ...
```

### 그 외의 제약조건

* DEFAULT : 기본 값 설정
* NOT NULL



## 테이블 압축

* 기본적으로 테이블 단위로 압축기능 제공
  * https://mariadb.com/kb/en/innodb-compressed-row-format/
* 10.3.2부터는 테이블 전체가 아니라 특정 칼럼만 압축하는 기능이 도입됨
  * https://mariadb.com/kb/en/storage-engine-independent-column-compression/
* 압축설정만 하면 이후에는 일반 테이블 쓰듯이 사용하면 됨.

> 내 생각 : 테이블 압축
>
>   대용량 데이터를 입력해야하지만 읽을일은 가끔식만 있는 테이블이라면 스토리지 비용 절감을 위해 고려할만하다. 예를 들면 액세스 로그를 DB에 저장하고 가끔 감사를 받을 때만 조회하는 케이스 정도?
>
>   하지만 압축을 하고(insert), 푸는(select) 과정에서 더 많은 CPU를 사용하니 사용목적을 고려하고 실험을 통해 트레이드 오프를 잘 따져서 도입을 결정해야한다. 근데 돈이 많으면 안 쓰지 않을까?



## 임시 테이블

* `TEMPORARY` 키워드가 붙는 것 말고는 일반 테이블 생성과 동일함
* 이름대로 임시이며 현재 세션이 종료되면 삭제됨

```sql
CREATE TEMPORARY TABLE [IF NOT EXISTS] 테이블이름 (열 정의 ...)
```



## 뷰

* 뷰는 가상 테이블처럼 동작하는 저장된 쿼리.

```sql
CREATE VIEW v_usertbl
AS
    SELECT userID, name, addr FROM usertbl;
select * from v_usertbl;
```

* 뷰를 통해 DML도 가능함.
  * 단, 뷰에 집계함수가 쓰이거나 `UNION, JOIN, DISTINCT, GROUP BY` 등이 쓰인 경우는 안 됨. 원본소스가 불명확하거나 존재하지 않는 케이스이므로 당연한 내용.
  * DML 을 상정하고 뷰를 만드는 경우 `WITH CHECK OPTION` 을 주면 뷰의 조건을 만족하는 DML만 하도록 제한할 수 있음.
    * 예를 들면 키가 170이상인 경우만 조회하는 VIEW를 만들 때 `WITH CHECK OPTION`을 주면 170이상인 데이터만 넣을 수 있는 식임. 해당 옵션이 없으면 170미만의 데이터도 입력이 가능한데 입력만 가능하고 VIEW 로 조회는 안 됨. 이런 현상을 막기 위해 있는 옵션임.
* 뷰가 참조하던 테이블이 제거되면 뷰 조회도 불가능함.

```sql
-- 뷰에 에러가 나는 경우 CHECK TABLE 명령어로 상태를 확인할 수 있음
-- CHECK TABLE 자체는 table 의 에러를 체크하는 명령임
CHECK TABLE v_usertbl;

-- 아래와 같이 view의 세부 정의를 확인하는 것도 가능
-- IS_UPDATABLE 이라는 열은 이 뷰를 통해서 DML 이 가능한지를 나타냄
SELECT * FROM information_schema.VIEWS WHERE TABLE_SCHEMA = 'sqldb' AND TABLE_NAME = 'v_usertbl';
```



### 장점

* 권한 세분화 : DB 사용자에 따라 테이블의 전체 권한을 줄 수 없을 경우 제공가능한 필드만으로 제한하고 view 권한만 줄 수 있음. 이를 통해 사용자의 권한을 세분화할 수 있음.
  * 특히 뷰를 통해 제한적으로 DML도 가능하므로 특정 필드만 편집가능한 권한을 만드는 것도 가능함.
* 복잡한 쿼리의 단순화 : 자주 사용되는 쿼리를 view로 정의해두면 쿼리가 편리해짐.



> 내생각 : 딱히 뷰는 사용한적이 없는데 자주 쓰이는 쿼리에 적용하면 타이핑도 줄일 수 있고 쿼리의 가독성도 높일 수 있겠다. 다만 앱단에서 사용할 경우에는 테이블 스키마 변경이 있을 경우 뷰에 미치는 영향도까지 확인해야하니 부작용에 주의해야할 듯. 운영중 데이터 추출용으로만 쓰는 건 부담없이 해볼만할 것 같다.



# Chap09 인덱스

* 장점 : (잘 쓰면) 검색 속도가 빨라짐
* 단점 : 
  * 인덱스 생성을 위한 추가 스토리지 확보 필요
  * 빈번한 DML 이 발생할 경우 인덱스 재구성으로 인한 성능저하 발생



## 인덱스 종류

* 클러스터형 인덱스 : 기본적으로 PK를 말한다고 보면 됨
  * 클러스터형 인덱스는 테이블당 1개만 존재
  * PK로 지정된 필드(들)이 자동으로 클러스터형 인덱스로 지정됨
  * PK가 없는 경우 Unique이고 NOT NULL인 필드가 있으면 그 필드가 클러스터형 인덱스로 지정됨
  * 둘 다 있으면 PK가 우선임
  * 행 데이터를 클러스터형 인덱스(보통 PK)에 따라 자동 정렬함
    * 이 때문에 order by 없이 select 하면 기본으로 PK로 정렬되는 것
* 보조 인덱스 : 그 외의 모든 인덱스
  * unique로 지정된 열
  * 직접 생성한 인덱스들

```sql
-- 테이블의 인덱스 확인
SHOW INDEX FROM TBL1;
```



## 인덱스 생성

* CREATE TABLE 에서 같이 명시하여 생성
* ALTER TABLE 로 생성
* CREATE INDEX 로 생성
* 이미 운영중인 테이블에 인덱스를 생성하는 경우 데이터양에 따라 많은 시간이 소요될 수 있으므로 운영시간에 작업에는 주의가 필요함. 개발환경에서 충분히 테스트를 수행하고 생성계획을 수립해야함.



## 인덱스의 내부 구조 B-Tree

* 인덱스는 내부구조로 B-Tree (Balanced Tree) 를 사용함

  * 다른 데이터 구조도 있으나 기본은 B-Tree

* mariadb의 최소 저장 단위인 page (기본값 16KB) 를 node로 하여  tree구조를 생성하게 되며, 이를 통해 전체 검색대비 훨씬 빠르게 데이터를 검색할 수 있는 것.

* 반면에 변경 작업(DML)이 발생할 경우 리스트처럼 단순히 append하고 끝이아니라 tree 구조에 변경이 발생하므로 (페이지 분할) DML, 특히 insert 때 성능 저하가 발생함.

* 검색이 빠르고 변경이 느린 이유는 개념적으로는 List와 이진탐색트리(BST)를 생각해봐도 됨.

  * 검색은 BST가 List보다 빠르지만
  * 추가는 단순 append 하는 List대비 BST 구조를 유지하기 위해 여러 연산이 필요한 BST가 불리한 것과 마찬가지임

  

### 클러스터형 인덱스 정리

* 클러스터형은 인덱스 자체의 리프 페이지가 곧 데이터 페이지임. 인덱스 자체에 데이터가 포함된 것으로 볼 수 있음.
* 이런 특성으로 인해 검색은 클러스터형이 더 빠름.
  * 클러스터형은 리프 페이지를 바로 읽으면 되는 반면, 보조형은 리프 페이지까지 간 뒤 주소를 읽어 데이터 페이지를 읽어야함. 
  * 1,2개 검색할 때는 큰 차이가 없겠으나 범위 검색을 할 경우에는 이미 정렬되어 있는 클러스터형은 접근해야할 페이지를 최소화할 수 있지만 보조 인덱스는 데이터 페이지가 정렬되어 있지 않으므로 여러 페이지를 접근해야해서 성능이 느려짐.
* 반면에 DML (특히 insert) 은 클러스터형이 더 느림. 인덱스의 관리 비용이 더 크기 때문임.
* 1개만 생성할 수 있음 (보통 PK)



### 보조 인덱스 정리 

* 데이터 페이지는 그대로 둔 상태에서 별도의 공간에 인덱스를 구성함
* 인덱스의 리프 페이지는 클러스터형과 달리 데이터가 아니라 데이터가 위치하는 주소 값(RID) 임.
* 보조 인덱스는 여러개 생성할 수 있음. 그러나 불필요한 인덱스는 공간 낭비 및 DML의 성능 저하를 발생시키므로 신중히 생각해서 생성해야함.



### 혼합형 인덱스

* 클러스터형 인덱스 + 보조 인덱스가 있는 경우
  * 검색 성능을 위해 PK외에 하나라도 인덱스를 추가 생성하면 혼합형 인덱스를 사용하는 셈.



TBD...



> 인덱스를 어느 열에 지정할까?
>
>   인덱스를 위한 B-Tree 구성을 생각하면 우선 클러스터형 인덱스를 담당할 PK는 정수형이 성능면에서 적절함. 트리 구조를 유지하기 위한 저장공간, 순서를 얻기 위한 연산을 고려한 것임. 흔히 쓰이는 UUID 같은 string은 아무래도 용량이 더 클 수 밖에 없고 비교 연산에서도 정수형보다는 느리기 때문임. 물론 실제로는 성능만 보고 결정하는 것은 아니므로 목적에 맞는 결정이 필요함.
>
>   보조 인덱스의 경우는 카디널리티가 충분히 큰 열이 좋음. 예를 들면 값이 Y 또는 N인 열에 인덱스를 지정해봤자 쓸데없이 공간만 차지할 뿐 성능상 이득을 볼 수 없기 때문임. 이런 경우는 실행 계획을 봐도 mariadb가 그냥 index를 안 써버림.



```sql
-- page size
show variables like 'innodb_page_size';
```





# Chap10 스토어드 프로그램

* 스토어드 프로시저
* 스토어드 함수
* 트리거



> 내 생각 : 스토어드 프로그램에 대해
>
>   레거시 프로젝트들은 비즈니스 로직을 스토어드 프로시저 (이하 SP) 로 구현하는 경우가 많았다. 최근에는 SP는 잘 쓰이지 않는데 아래와 같이 장단점을 정리해봤다. 정리하면 최근의 많은 웹시스템들은 SP를 안 쓰는게 더 좋다고 본다. 반면 대용량 데이터 배치 작업등이 필요한 경우 등은 SP가 더 유리할 수 있다.



## SP 의 장점

* 운영 중 무중단으로 즉각 반영이 가능함
  * 다만 이점은 세션 클러스터링, CI&CD 의 발전으로 앱에서도 무중단으로 로직 변경이 가능하므로 다소 퇴색된 장점.
* 통신트래픽을 최소화할 수 있음.
* 대용량의 처리에 더 유리함. (성능이 더 좋음)
  * 예를 들어 GB 이상의 데이터를 처리해야하는 비즈니스 로직이라면 was로 처리할 경우 시간도 오래걸리고 네트워크 비용도 많이 발생할 것임.
  * 반면에 DB는 바로 스토리지에 접근하여 처리가 가능하므로 성능에서 유리함.



## SP 의 단점

* 유지보수가 힘듬
  * 형상관리가 되지 않음
    * 실수로 개발계에만 반영하고 운영계에는 미반영하는 등 휴먼에러가 발생할 여지가 있음
  * 디버깅이 힘듬
* 비즈니스 로직이 특정 DBMS 벤더에 종속적이게 될 수 있음. (DBMS 의 변경 부담이 커짐)
* 성능 상승에 제한이 있음
  * 대개의 경우 RDBMS는 스케일업이 용이하고 스케일아웃은 안 되거나 제한적으로만 가능.
  * 때문에 스케일업으로 한계에 달한 경우 더 이상의 트래픽을 처리할 수 없을 수 있음. 
  * 반면에 WAS는 스케일아웃이 용이함. 때문에 DB는 단순 CRUD만 수행하도록하고 WAS를 스케일아웃 함으로써 더 많은 요청을 처리할 수 있음
  * 또한 DB가 상대적으로 WAS보다 비싸다는 경제적인 문제도 있음.



> 내 생각 : 위의 장단점은 스토어드 함수(이하 SF)도 어느정도 공유한다고 본다. 그래도 SF는 쿼리의 편의성을 높이는 간단한 용도라면 적용할만하다. 
>
>   트리거도 애플리케이션에서 하나의 트랜잭션으로 묶어서 작업하는 것으로 대체가 가능하므로 굳이 DBMS 쪽의 관리 포인트를 늘릴 필요는 없다고 생각한다.



# Chap11 전체 텍스트 검색과 파티션

* TBD

* 정리 참고
  * https://nesoy.github.io/articles/2018-05/Database-Shard
  * https://nesoy.github.io/articles/2018-02/Database-Partitioning



# 기타 : 향후의 웹시스템

* 이건 공부하면서 이런저런 토론글들을 보면서 느낀 생각을 정리한 것이다.
* MSA 로 아키텍쳐의 유행이 바뀌고 JPA도 대중화되면서 DBMS도 가볍게 유지하고 비즈니스는 앱단에서 처리하는게 보편적인 상황이라고 본다.
* 반면 mybatis 는 레거시 시스템의 유지나 SI 현장등에서 빈번한 요구사항 변경, JPA 개발자 구인의 어려움, 복잡한 통계성 Select 쿼리 등의 상황을 고려하면 앞으로도 쓰일거라 본다.
* 신규 프로젝트라면 앱단에서는 JPA와 같은 ORM을 베이스로 하되 필요에 따라 mybatis 같은 SQL Mapper 를 제한적으로 이용하고, DBMS 는 스토어드 프로그램 사용을 지양하고 ACID를 보장하는 데이터저장소의 의미로 쓰는 것이 좋다고 생각한다.
  * 다만 이는 일반적인 웹시스템을 상정하고 생각한 것이고 시스템의 목적에 따라 구성은 달라질 수 있을 것이다.



# 기타 : OLTP, OLAP

* On-Line Transcation Processing : 트랜잭션 처리를 위한 정보 시스템
  * 빠른 처리, 트랜잭션 처리
* On-Line Analytical Processing : 데이터 분석에 특화된 시스템
  * 분석, 히스토리, 의사결정
* OLTP 를 통해 데이터가 누적되고 이를 OLAP 으로 분석하여 의사결정을 내리는 식
  * OLTP -> ETL (Extract Transform Load) -> OLAP
* 자세한 건 구글링
