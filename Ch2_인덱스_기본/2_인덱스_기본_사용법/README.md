
### 1. 인덱스를 사용한다는 것

색인을 예로들면 우리가 찾고자하는 단어를 ㄱ,ㄴ,ㄷ..으로 찾을 것이다. 이것이 인덱스와 비슷한 구조이다. 이것이 가능한 이유는

1. 정렬이 되어있기때문

2. 찾고자 하는 단어들이 모여있기 때문

  

이것이 보장되어있지않으면 색인,인덱스 모두 쓸모없는 것이 된다.

이번에는 다른 조건으로 찾아보자 기존에는 시작 단어를 기준으로 찾았다.

1. 단어를 포함하는 것

2. n1~n2번째에 단어를 포함하는 것

  

라는 기준이라면 단어를 기존에 쓰던 방식이 유효할까? 기존에는 시작점을 기준으로 찾았기에 이러한 기준으로 찾는다면 기존의 방식은 사용할 수 없다. 즉, 가공한 값이나 중간값을 찾을 때에는 색인 전체를 스캔해야한다.

색인을 사용하지 않는 것은 아니다. 다만 비효율적으로 색인 전체를 스캔해야 유의미한 결과가 나올 것 이다.

  

DB에서도 마찬가지이다. 인덱스 컬럼(선두컬럼)을 가공하지 않아야 인덱스가 정상적으로 작동한다.

정상적으로 작동한다는 것은 리프블록의 시작점을 찾아 스캔을 하다가 중간에 멈추는 것을 의미한다. 즉, 리프블록에 일부만 스캔하는 **Index Range Scan**이다. 컬럼을 가공해도 인덱스는 사용할 수 있지만, 시작점을 찾을 수 없고 결국 블록 전체를 스캔하는 Index Full Scan 방식으로 작동한다.

  
  

### 2. 인덱스를 Range Scan할 수 없는 이유

앞서 정리했듯이 컬럼을 가공을하면 정상적으로 Range Scan이 불가능하다. Range Scan에서 방식은 시작점을 찾고 범위를 지정하기 위해 끝지점을 찾았다. 하지만 아래는 이러한 방식이 불가능한 SQL들이다.

  

1. where 생년월일 between '20070101' and '20070131'

- 정상적으로 Range를 지정할 수 있다. 20070101'을 통해 시작점을 정하고 수평적탐색을 통해 끝지점을 찾는다.

  

2. where substr(생년월일,5,2) ='05'

- 가공된 컬럼이 있어 시작점을 찾을 수 없다. 끝지점도 찾을 수 없다.

  

3. where nvl(주문수량,0)<100

- 주문수량으로 인덱스를 만들었을 때 이러한 조건절은 정상적으로 인덱스가 사용되지않는다.

  

4. where 업체명 like '%대한%'

- 중값값으로 대한을 갖는 데이터를 수직적으로 찾을 수 없다 .

  

5. where (전화번호 =:tel_no OR 고객명 = :cust_nm )

- OR 조건의 경우 수직적 탐색을 통해 전화번호가 '01012345678'이거나 고객명이 '홍길동'인 어느 한 시작지점을 찾을 수 없다.

  

#### OR Expansion

5번 항목을 보면 OR조건을 이용한 데이터조회는 Range Scan이 불가능하였다. 하지만 아래와 같이 처리하면 같은결과지만 Index Range Scan이 가능해진다.

  

select * from 고객

where 고객명 = :cust_nm --고객명이 선두 컬럼인 인덱스가 Range Scan된다.

union all

select * from 고객

where 전화번호 = :tell_no -- 전화번호가 선두 컬럼인 인덱스가 Range Scan된다

and 고객명<> :cust_nm or 고객명 is null)

  
  

우선 Table에는 당연히 고객명이 :cust_nm인경우, :cust_nm이 아닌경우, Null인 경우가 있다. 위에 sql은 :cust_nm인 경우 위에 sql로 조회가 된다. Range Scan을 이용해서 아주 빠르게된다. 그리고 :cust_nm이 아니고 null인 경우 아래 sql로 조회가 된다. 전화번호가 Index Range Scan 으로 빠른속도로 전화번호에 해당하는 데이터만 조회해올 것이다. 그리고 두 SQL을 Union all을 하다보니 빠른 속도로 5번의 SQL의 결과를 보여줄 것이다.

이를 OR Expansion이라 한다. 아래 SQL은 use_concat 힌트를 이용해서 OR Expansion을 유도한 것이다.

  

select /\*+ use_concat\*/ * from 고객

where ( 전화번호= :tel_no OR 고객명 = :cust_nm )

  

Excution Plan

----------

생략

  

이런식으로 힌트를 사용하여 실행계획을 보면 고객명과 전화번호를 이용한 INDEX를 사용한 것을 볼 수 있다. 이런식의 힌트와 Union all 을 이용하지 않으면 Index Range Scan은 불가능하다.

  

다음과 같은 경우도 OR과 비슷하다.

~

where 전화번호 in (:tel_no1,:tel_no2) --Index Range Scan 불가능하다.

  

하지만 아래와 같이 수정하면 가능해진다.

~

where 전화번호 =:tel_no1

union all

~

where 전화번호=:tel_no2

  

그래서 이러한 in에 대해서 SQL 옵티마이저는 IN-List Iteratior 방식을 사용한다 in-List의 갯수만큼 Index Range Scan을 이용한다. 그리고 나온 결과만큼 Union ALL을 하여 기존의 IN과 같은 결과를 얻을 수 있다.

ex)

SELECT /*+ INLIST_ITERATOR */ employee_id, first_name, last_name

FROM employees

WHERE department_id IN (10, 20, 30, 40, 50);

  

**인덱스를 정상적으로 사용한다**

라는 것은 리프 블록에서 스캔시작점을 찾아서 스캔을 하다가 중간에 멈추는 것을 의미한다. 위에서 본 예제들은 **정상적**으로 사용할 수 없는 예제들이었다. 다만 OR과 IN에 대해서는 HInt와 sql변환을 통해 정상적으로 Index Range Scan을 사용하는 SQL 로 처리할 수 있었다.

### 3. 더 중요한 인덱스 사용 조건
만약 인덱스를 "소속팀+사원명+연령"으로 설정하고 where 사원명="홍길동"; 으로 하면 정상적으로 Index Range Scan이 될까?
실제 그림을 그려서 보면 정상적으로 처리가 안되는 것을 볼 수 있다. 
즉, **인덱스를 정상적으로 Range Scan하기 위한 첫번째 조건은 인덱스 선두 컬럼이 조건절에 있어야한다. **

아래와 같은 예제에서는 어떻게 Range Scan하는지 보자
 TXA1234_IX02 Index: 기준연도+과세구분코드+보고회차+실명확인번호
 
 select * from TXA1234
 where 기준연도 = :stdr_year
 and substr(과세구분코드,1,4) = :txtn_dcd
 and 보고회차 =:rpt_tmrd
 and 실명확인번호 =:rnm_cnfm_no

여기서 과세구분코드를 가공하고있다. 하지만 이러한 경우에는 Index Range Scan을 사용한다. 인덱스의 선두컬럼인 기준연도가 가공되지않아서 Range Scan이 가능해진다. 그러나 좋은 성능을 보장하지는 않는다.

**인덱스 잘 타니까 튜닝 끝?**
"인덱스를 탄다"라는 말은 Index Range Scan 한다와 같은 의미이다. 그래서 실행계획을 보고 Index가 타는 것만을 보는 개발자가 많다. 
EX)

주문상품_N1 index : 주문일자+ 상품번호
하루 데이터 쌓이는 양 100만건

sql1
select * from 주문상품 
where 주문일자= :ord_dt
and 상품번호 like '%PING%';

sql2 
select * from 주문상품 
where 주문일자= :ord_dt
and substr(상품번호,1,4) = 'PING';

두개의 sql 모두 선두컬럼이 가공되지 않나 Range Scan을 할 것이다. 하지만 인덱스가 타는 것은 맞지만 잘 타고있는것일까는 다른 문제이다. 인덱스가 잘 타는지는 인덱스 리프 블록에서 스캔하는 양을 따져봐야한다. 위에 sql1,2에 상품번호 모두 스캔 범위를 줄이는데는 크게 영향을 줄 수가없다. sql1은 중간값검색, sql2는 컬럼을 가공했기 때문이다. 그렇다면 어떠한 주문일자 기준으로 스캔하는 데이터량은 100만건이 될 것이다. 이 문제는 "인덱스 스캔 효율화(3.3절)"에서 자세히 본다.

### 4. 인덱스를 이용한 소트 연산 생략
인덱스를 Range Scan할 수 있는 이유는 **정렬**이 되어있기에 가능한 것이다. 즉 테이블은 정렬이 안되어있는 상태이고, 인덱스는 정렬이 되어있는 상태이다. 정렬이 되어있기에 인덱스를 이용하면 소트 연산에서도 부수적인 효과를 얻는다. 

EX)
Table : 장비번호+ 변경일자 + 변경순번 (모든 컬럼이 pk)
index : 장비번호+ 변경일자 + 변경순번

이러한 가정에선 index가 장비번호 변경일자 변경순번으로 정렬되어있다. 그리고 아래와 같이 sql을 조회하면

select * from 상태변경이력
where 장비번호 = 'C'
and 변경일자 = '20160316';

index가 정상적으로 탈 것이다. 그리고 변경순번으로 정렬이 되어있을 것이다. 위에 sql은 order by절이 없으므로 정렬을 하지않았지만 만약 order by가 있다면?
있어도 결국엔 index가 정상적으로 타기만 한다면 정렬된 결과를 보장하기에 옵티마이저는 따로 정렬연산을 수행하지 않는다.

select * from 상태변경이력
where 장비번호 = 'C'
and 변경일자 = '20160316'
order by 변경순번;

위에 sql은 order by가 있든 없든 같은 실행계획을 수행한다.

내림차순의 경우에도 인덱스를 이용해 정렬을 한다. 우리가 앞서서 봐왔던 방식에는 리프 블록을 찾으러갈 때 가장 작은 값을 찾았다. 하지만 내림차순의 경우 가장 큰값을 찾고, 리프블록을 왼쪽으로 탐색한다.(여태까지 말한 대부분의 방식은 오른쪽으로 탐색, 이러한 것이 가능한 이유는 리프블록이 양방향 연결 리스트 구조이기에 가능함)

내림차순으로 sql을 실행시키면 실행계획에 아래와 같이 나타난다.
2 1 INDEX (RANGE SCAN **DESCENDING**) OF '상태변경이력_PK' (INDEX (UNIQUE))


### 5. ORDER BY 절에서 컬럼가공
여태까지 봐왔던 예시에서 인덱스 컬럼을 가공해서 인덱스를 제대로 활용하지 못한 경우는 대부분 조건절에서 가공을 했다. 하지만 조건절이 아닌 ORDER BY, SELECT-LIST에서 컬럼을 가공해도 정상적으로 사용하지 못하는 경우도 있다. 

위에서 사용한 상태변경이력 테이블을 기준으로 아래 sql은 order by 연산을 생략할 수 있다. 자동으로 변경일자, 변경순번으로 정렬을 해준다. 
select * from 상태변경이력
where 장비번호 ='C'
order by 변경일자,변경순번;


하지만 아래 sql인 경우에는 정렬연산을 생략할 수 없다. 
select * from 상태변경이력
where 장비번호='C'
order by 변경일자||변경순번;

인덱스에서는 변경일자와 변경순번을 가공하지 않은 상태로 정렬이 되어있는데 sql내에서 가공을 하여 정렬연산을 요구했기에 정렬연산을 수행하게된다.


다음 예시는 ORDER BY 연산이 일어난 연산이다.
SELECT * --1
FROM--2
(
	SELECT TO_CHAR(A.주문번호, 'FM000000')AS '주문번호',A.업체번호,A.주문금액--3
	FROM 주문 A--4
	WHERE A.주문일자=:dt--5
	AND A.주문번호 < NVL(:next_ord_no,0)--6
	ORDER BY 주문번호--7
)
WHERE ROWNUM<=30;--8

해당 SQL이 ORDER BY 연산이 일어나는 이유는 7번째 줄때문이다. 주문번호는 테이블 A의 가공되지않은 컬럼을 나타내는지 아니면 TO_CHAR(A.주문번호, 'FM000000')AS '주문번호' 로 가공한 컬럼을 가르키는지 생각해보면 답은 알 수 있다.
7번째 줄의 주문번호는 TO_CHAR로 가공된 컬럼을 가르키고 있는 ORDER BY이다. 그래서 해당 줄의 주문번호를 A.주문번호로 수정을 하면 정상적으로 ORDER BY 연산을 수행하지 않고 정렬된 결과를 볼 수 있을 것이다. 


### 6. SELECT -LIST에서 컬럼 가공
아래 SQL은 옵티마이저가 따로 정렬연산을 하지않는 SQL이다.
MIN)
SELECT MIN(변경순번)
FROM 상태변경이력
WHERE 장비번호='C'
AND 변경일자 = '20180316'

MAX)
SELECT MAX(변경순번)
FROM 상태변경이력
WHERE 장비번호='C'
AND 변경일자 = '20180316'

MIN의 경우 수직탐색을 할 때 왼쪽지점으로 내려가서 찾는 첫번째 레코드가 MIN값이 된다. 
MAX의 경우 수직탐색을 할 때 오른쪽지점으로 내려가서 찾는 첫번째 레코드가 MAX값이 된다.
두개의 SQL모두 정렬연산을 사용하지 않고 최댓값과 최솟값을 찾을 수 있다. 두개의 SQL의 실행계획을 보면
**FIRST ROW**라는 단어와 INDEX RANGE SCAN(**MAX / MIN**)을 볼 수 있다. 
다만 아래와 같이 컬럼을 가공을 할 경우는 해당하지 않는다. 
SELECT NVL(MAX(TO_NUMBER(변경순번)),0)
~이하생략
이 SQL에서는 변경순번을 숫자값을 기준으로 바꾼 후 MAX를 요구하였으나 인덱스는 문자를 기준으로 하였기에 정렬연산을 사용하게된다. 만약 정렬연산없이 찾고싶을 경우에는 이렇게하면된다.
SELECT NVL(TO_NUMBER(MAX(변경순번)),0)
~이해생략

다음예제
SELECT 장비번호, 장비명 상태코드,
&nbsp;(
&nbsp;&nbsp;SELECT MAX(변경일자)
&nbsp;&nbsp;FROM 상태변경이력
&nbsp;&nbsp;WHERE 장비번호  = P.장비번호
&nbsp;)AS 최종변경일자
FROM 장비 P
WHERE 장비구분코드 ='A001'

해당 SQL에서 실행계획을보면 FRIST ROW와 MIN MAX는 보이고있다. 만약여기에 최종변경순번까지 보고싶다면 어떤식으로 SQL을 짤 수 있을까

SELECT 장비번호, 장비명 상태코드,
&nbsp;(
&nbsp;&nbsp;SELECT MAX(변경일자)
&nbsp;&nbsp;FROM 상태변경이력
&nbsp;&nbsp;WHERE 장비번호  = P.장비번호
&nbsp;)AS 최종변경일자,
&nbsp;(
&nbsp;&nbsp;SELECT MAX(변경순번)
&nbsp;&nbsp;FROM 상태변경이력
&nbsp;&nbsp;WHERE 장비번호  = P.장비번호
&nbsp;&nbsp;AND 변경일자 =	
&nbsp;&nbsp;&nbsp;(
&nbsp;&nbsp;&nbsp;&nbsp;SELECT MAX(변경일자) FROM 상태변경이력
&nbsp;&nbsp;&nbsp;&nbsp;WHERE 장비번호 = P.장비번호
&nbsp;&nbsp;&nbsp;)
&nbsp;)AS 최종변경일자,
FROM 장비 P
WHERE 장비구분코드 ='A001'

SQL자체도 복잡한데 우선 성능도 안좋은 SQL이다. 상태변경이력 테이블을 많이 읽는 SQL이다. 그러면 SQL자체를 조금 간단하게 나타내보면 아래와 같이 나타낼 수 있다.

SELECT 장비번호, 장비명, 상태코드
&nbsp;,SUBSTR(최종이력,1,8) AS 최종변경일자
&nbsp;,SUBSTR(최종이력,9) AS 최종변경순번
FROM(
&nbsp;SELECT 장비번호, 장비명, 상태코드
&nbsp;&nbsp;,(SELECT MAX(변경일자||변경순번)
&nbsp;FROM 상태변경이력
&nbsp;WHERE 장비번호 = P.장비번호)최종이력
&nbsp;FROM 장비 P
&nbsp;WHERE 장비구분코드 = 'A001'
)
우선 간단하게 원래 SQL보다는 간단해지기는 했다. 하지만 성능으로써는 장비당 이력이 많지않은 경우에는 크게 상관없지만, 이력이 많다면 문제가 생길 수 있는 SQL이다. 각 장비에 속한 과거 데이터를 모두 읽어야 하므로 이력이 많을 경우 성능이 그전 SQL보다 안 좋을 수도있다.
이런 SQL은 Top N 알고리즘을 알아야 하므로 5장 3절 4항에서 설명한다.

### 7. 자동 형변환
SELECT * FROM 고객 WHERE 생년월일 = 19821225
해당 SQL은 우리가 앞에서 계속 설명했던 선두컬럼이 가공되지않아야 INDEX가 정상적으로 탄다.의 좋은 예시인 거처럼 보인다. 하지만 실제로 옵티마이저는 해당 SQL을 FULL SCAN을 한다.
그리고 실행계획 아래쪽에 조건절 정보를 보면
1 - filter(TO_NUMBER("생년월일"))=19821225)라고 적힌것을 볼 수있다. 해당 sql은 옵티마이저가 아래와 같이 변환했다는 뜻이다.
SELECT * FROM 고객 WHERE TO_NUMBER(생년월일) = 19821225
위와 같은 SQL을 보면 왜 인덱스가 안 탔는지 알 수있다. 가공이 되었기에 인덱스가 타지않은 것이다.
그러면 왜 형변환을 한것일까? 우선 생년월일은 문자컬럼이다. 하지만 우리가 조건에 넣은 값은 숫자이다. 값이 다르기에 DBMS가 자동으로 형변환을 한 것이다. (DBMS마다 어떻게 처리하는지는 다르다.)

날짜 포맷 또한 자동변환이 일어난다. 그래서 날짜포맷을 쓸 경우는 아래와 같이 지정해주는 습관을 가지자.
SELECT * FROM 고객
WHERE 가입일자 = TO_DATE('01-JAN-2018','DD-MON-YYYY')
심지어 NLS_DATE_FORMAT 파라미터가 다르게 설정된 환경에선 잘못된 결과 또는 오휴가 날 수도 있으니 꼭 습관화하자.

앞서 숫자와 문자의 경우 우선순위가 숫자가 높지만, LIKE의 경우는 다르다. LIKE의 경우 문자열 비교 연산자이다. 그러다보니 숫자가 문자로 변환이 된다. 실행계획을 보면 조건절에 FILTER로 TO_CHAR가 쓰여진 것을 볼 수있다.

**자동 형변환의 주의할점!(기능적)**
3장 3절을 보고나야 이것이 이해될 것이다. LIKE조건을 옵션 조건처리 할 때가 있다. 
아래 SQL 두개의 예시를 보면
1. 사용자가 계좌번호를 입력했을 때
SELECT * FROM 거래
WHERE 계좌번호 = :ACNT_NO
AND 거래일자 BETWEEN :TRD_DT1 AND :TRD_DT2

2. 사용자가 계좌번호를 입력하지 않았을 때
SELECT * FROM 거래
WHERE 거래일자 BETWEEN :TRD_DT1 AND :TRD_DT2

다만 많은 개발자가 아래와 같은 방법을 쓴다.
SELECT * FROM 거래
WHERE 계좌번호 = :ACNT_NO || '%'
AND 거래일자 BETWEEN :TRD_DT1 AND :TRD_DT2
(실제 이 내용을 정리하는 본인도 회사를 다니던 시절 이렇게 사용했다. )

이 방식에서는 LIKE 와 BETWEEN을 같이 사용해서 인덱스 효율이 좋지못하다. 그리고 계좌번호 컬럼이 숫자형일 때 주의가 필요하다. 

**품질적인 측면에서의 자동 형변환 주의!!**
아래와 같은 예시를 보면 쉽게 알 수 있다.
WHERE N_COL = V_COL

에러가 날 가능성이 있는 코드이다. 앞서 말했든시 문자와 숫자를 조건에 넣으면 문자가 숫자로 바뀐다. 사용자가 '123'을 넣었다면 문제가 없겠지만 '일이삼'을 넣은 순간 에러가 발생할 것이다. 

아래 sql은 에러가 아닌 결과 오류가 나는 SQL이다.
SELECT ROUND(AVG(SAL)) AVG_SAL
&nbsp;,MIN(SAL) MIN_SAL
&nbsp;,MAX(SAL) MAX_SAL
&nbsp;,MAX(DECODE(JOB,'PRESIDENT',NULL,SAL)) MAX_SAL2
FROM EMP;

RESULT
AVG_SAL:2073
MIN_SAL:800
MAX_SAL:5000
MAX_SAL2:950

MAX_SAL2는 가장 많은 급여를 받을 거 같은 직업을 넣고 그 직업은 제외한 직업 중 가장 높은 급여를 조회를 하는 SQL이다. 하지만 결과 MAX_SAL2는 평균 급여보다도 낮다. 거의 최솟값이다. 실제 레코드 단위로 조회해보면 가장 높은 급여를 가진 직업은 'ANALYST'였고, 2개의 3000인 데이터가 있었다.

우선 이러한 값 오류가 일어난 이유는 DECODE(a,b,c,d)에서 a=b이면 c반환 아니면 d반환을 한다.  DECODE에서는 c를 기준으로 d의 데이터타입을 형변환 시킨다. 그리고 c의 값이 null일경우 varchar2로 관리를 한다. 그러면 네번째 인자 sal의 데이터타입은 varchar2로 가져올거고, 그 값중 가장 큰 값은 "950"이 되는거다.("950">"3000" , 950<3000)

아래와 같이 쓰면 피할 수 있다.
&nbsp;,MAX(DECODE(JOB,'PRESIDENT',0,SAL)) MAX_SAL2
&nbsp;,MAX(DECODE(JOB,'PRESIDENT',TO_NUMBER(NULL),SAL)) MAX_SAL2

이러한 형변환 함수들을 쓰면 성능에 영향은 별로 없다. 이것보다 블록IO 줄이는게 더 중요하다. 형변환 함수를 생략하면 옵티마이저가 자동으로 형변환 함수를 생성하기에 의도치않는 형변환을 방지하기위해 SQL의도에 맞게 형변환 함수를 써서 옵티마이저에게 넘기자자