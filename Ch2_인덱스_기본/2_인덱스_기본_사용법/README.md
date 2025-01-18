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
- 정상적으로  Range를 지정할 수 있다. 20070101'을 통해 시작점을 정하고 수평적탐색을 통해 끝지점을 찾는다.

2. where substr(생년월일,5,2) ='05'
-  가공된 컬럼이 있어 시작점을 찾을 수 없다. 끝지점도 찾을 수 없다.

3. where nvl(주문수량,0)<100
- 주문수량으로 인덱스를 만들었을 때 이러한 조건절은 정상적으로 인덱스가 사용되지않는다.

4. where 업체명 like '%대한%'
- 중값값으로 대한을 갖는 데이터를 수직적으로 찾을 수 없다 .

5. where (전화번호 =:tel_no OR 고객명 = :cust_nm )
- OR 조건의 경우 수직적 탐색을 통해 전화번호가 '01012345678'이거나 고객명이 '홍길동'인 어느 한 시작지점을 찾을 수 없다. 

####  OR Expansion
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