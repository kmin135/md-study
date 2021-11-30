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