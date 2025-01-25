# 인덱스 확장기능 사용법
Index Range Scan에는 다양한 방법이 존재하고 그 방법들의 특징과 방식을 정리한다.

### 1. Index Range Scan
B*Tree를 이용하여 인덱스를 구성하고 수직적 탐색과 수평적 탐색을 이요한다. 앞서 말했듯이 Range Scan이 정상적으로 수행하려면 선두 컬럼이 가공되지 않은 상태로 있어야한다. 인덱스가 타는 것만을 확인하면안된다. Range Scan이 정상적으로 이루어지는 확인해야한다.

실행계획에는 INDEX (RANGE SCAN) ~ 라고 적혀있다.

### 2. Index Full Scan
수직적탐색없이 리프블록을 수평적탐색으로만 탐색하는 방식이다. 
다만 유의해야하는 것은 Index Full Scan도 결국엔 Index를 이용한다는 것이다. 그말은 Index Full Scan을 사용할 때도 B*Tree구조에서 시작한다고 봐야한다. Table Full Scan과 헷갈리면 안된다. Index Full Scan이 타는 예제는 아래와 같다.

EX)
create index emp_ename_sal_idx on emp (ename,sal);
set autotrace traceonly exp

select * from emp
where sal > 2000
order by ename;

위에 예제에서 emp_ename_sal_idx의 선두컬럼인 ename 을 조건절에 사용하지 않았기에 Index Range Scan은 불가능하다. 하지만 SAL컬럼이 있으므로 Index Full Scan을 통해 SAl이 2000보다 큰 레코드를 찾을 수 있다.
("이 sql에 한정해서 Table Full Scan과의 성능에 큰 차이는 모르겠다..."라고 생각했는데 바로 아래에서 설명해주었다.)

실행계획에는 INDEX (FULL SCAN)~ 라고 적혀있다. 

**Index Full Scan의 효용성**
선두컬럼이 조건절에 없을 때 옵티마이저는 Table Full Scan을 고려한다. 하지만 테이블이 대용량 테이블일 경우 Table Full Scan은 비용적으로 좋지않다. 그래서 옵티마이저는 인덱스 활용을 고려한다. 
데이터 저장공간은 "컬럼길이x레코드 수"에 의해 결정한다. 인덱스는 테이블 보다 차지하는 면적이 적다. (인덱스를 지정할 때 테이블의 일부만 저장하기에) 만약 조건절에서 대부분의 레코드를 필터링하고 극히 일부만 남는 경우 옵티마이저는 Index Full Scan을 하는 쪽이 유리하다고 생각하여 Index Full Scan을 한다.

Index Full Scan의 경우 인덱스에서 조건에 맞는 데이터를 찾는다. 여기서 조건에 맞는 데이터를 찾을 때 인덱스의 리프 블록에서 찾는다. 리프블록의 데이터는 Table 데이터보다 크기가 적기에 적은 IO가 발생하여 성능적 유리함을 가져간다.
하지만 위에서 대부분의 레코드를 필터링해야 유의미한 성능이 나오는 것은 인덱스에서 조건에 맞는 데이터를 찾으면 그 데이터는 우리가 원하는 데이터가 아니기에 생기는 문제이다. 인덱스에 찾은 값은 데이터가 아닌, 데이터를 찾기 위한 Rowid이다. 결국에는 우리가 원하는 데이터를 찾기 위해서는 Rowid로 다시 한번 접근을 해야하기에 추가적인 랜덤액세스가 일어난다. 이러한 이유로 Index Full Scan과 Table Full Scan에서 필터된 레코드의 갯수가 중요한 것이다.

**인덱스를 이용한 소트 연산 생략**
Index Full Scan을 하면 인덱스의 컬럼 순서대로 정렬이 된다. 즉, 상황만 잘 맞으면 정렬연산이 생략된다. 
옵티마이저는 전략적으로 sort order by 연산을 생략할 목적으로 Index Full Scan을 할 수도 있는 것이다. 

EX)
select /\*+ first_rows*/* 
from emp where sal > 1000 order by ename;
에서 sal의 값이 대부분 1000을 넘긴다고 가정하자.

이러한 구조에서는 Index Full Scan은 Table Full Scan보다 불리하다. 심지어 Index Range Scan보다도 불리하다. 하지만 옵티마이저가 Index를 선택한 이유는 first_rows 힌트로 옵티마이저 모드를 바꿨기 때문이다. 
first_rows : 몇개의 행을 빠르게 조회하기 위한 힌트

소트연산을 생략함으로써 전체 집합 중 처음 일부를 빠르게 출력할 목적으로 옵티마이저가 Index Full Scan을 선택했다. 이 선택은 부분처리를 해야할 때 극적인 성능을 보여준다. 다만 fetch를 멈추지 않고 데이터를 끝까지 읽으면 엄청난 비효율을 일으킨다.

