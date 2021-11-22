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
