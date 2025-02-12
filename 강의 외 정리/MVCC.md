# 동시성 제어

DBMS가 동시에 작동하는 여러 트랜잭션의 상호 간섭으로부터 일관성을 보장하는 것이다. 일반적으로 동시성과 일관성의 반비례를 띈다.

### 낙관적 동시성 제어

사용자 간 동시에 데이터 수정이 이루어지지 않는다고 가정하여, 데이터 읽기 과정에서 Lock을 걸지 않고 수정 시점에 값이 변경됐는지를 검사한다.

### 비관적 동시성 제어

사용자 간 동시에 데이터 수정이 이루어진다고 가정한다. 데이터 읽는 시점에 Lock을 건다. 이는 시스템 동시성 측면에서 성능이 떨어질 수 있기 때문에 주의가 필요하다.

## 공유 락과 베타 락
- 공유 락 = 읽기 락
- 베타 락 = 쓰기 락

베타락을 얻기 위해선 어떠한 락도 걸려있지 않아야 한다.
베타락이 걸렸을 경우에는 어떤 공유 락도 진입이 불가능하다.

<br/>

# MVCC
## Locking의 문제점

읽기 작업과 쓰기 작업을 동시에 수행하는데에 문제가 있다. 또한 일관성 유지를 위해 오래 Lock을 유지하거나 테이블 수준의 Lock을 사용해야한다.
이럴 경우, 동시성은 더욱 떨어지는 점도 문제점으로 꼽힌다.

## MVCC
동시 접근이 가능한 DB에서 동시성 제어를 위해 사용하는 기술로, Snapshot을 이용하여 이루어진다.
- 데이터 변경 시, Snapshot을 읽고 개발자가 수정을 한다.
- 만약 데이터 변경이 취소 되면, 최근 Snapshot을 토대로 복구가 이루어진다.
- 데이터 조회 시, 가장 최근 버전의 데이터를 읽게 된다.

### 장점
1. 일반적인 RDBMS보다 매우 빠름
2. 여러 버전의 Snapshot 관리를 위한 시스템 필요
3. 데이터 버전 충돌 시, 애플리케이션 영역에서의 문제 해결 필요

## MySQL

REPEATABLE-READ(Default)

```sql
# 초기 셋팅
insert into member(id, name, area) values(1, 'sungwon', 'incheon');
```

![image](https://github.com/user-attachments/assets/8c254c73-ffb3-43a5-a15c-ce3525c31e29)

위 사진에서 두 SELECㅅT 쿼리 사이에 다른 트랜잭션이 커밋을 한 상태임에도 일관된 조회가 수행되는 것을 알 수 있다.
두 쿼리 모두 {1, 'sungwon', 'incheon'} 을 조회가능하다. 이를 타임라인으로 표현하면 아래와 같다.

![image](https://github.com/user-attachments/assets/8a6964a0-fb20-4707-8d71-95081569bca3)


다른 트랜잭션에서 변경 점이 Commit되었음에도 MySQL에선 트랜잭션 이전의 데이터를 가져오는 것일까? 이는 `InnoDB 버터 풀`과 `Undo 로그`가 구분되어 있기 때문이다.

![image](https://github.com/user-attachments/assets/6f7828c3-283a-48a1-94d0-695ceb18e6ec)

업데이트된 변경 점은 InnoDB 버퍼 풀에 바로 적용되게 되고, 수정 이전 로그는 Undo 로그에 남게 된다.
이 상태에서 REPEATABLE READ나 SERIALIZABLE 수준의 트랜잭션에서 변경 점에 대한 SELECT가 이루어진다고 가정하자.
DB는 InnoDB 버퍼 풀이나 디스크가 아닌 Undo 로그에선 데이터를 가져와 이를 출력한다.
만약 변경 점들이 Commit 되지 않고, Rollback을 호출하게 되면 Undo 로그를 바탕으로 InnoDB 버퍼 풀은 복구된다.

## PostgreSQL
PostgreSQL의 MVCC는 Record 대신 Tuple이라는 용어를 사용하고, 각 튜플마다 xmin, xmax라는 값을 가진다. 이 값들은 Transaction ID의 범위를 말한다. 예를 들어 xmin = 100, xmax = 103 이라는 의미는 100번 트랜잭션부터 103번 트랜잭션까지의 트랜잭션에서 읽힐 수 있다는 것을 의미한다.

다음은 UPDATE 시, Tuple의 값들이 변경되는 과정이다. 이것을 통해 어떻게 MVCC가 관리되는지 확인할 수 있다.

<img width="662" alt="image" src="https://github.com/user-attachments/assets/2fbd8f23-7a4c-45f8-9400-880f076dca90" />

트랜잭션이 끝나고 새로운 튜플이 추가되면서 더이상 참조되지 않는 튜플들이 생기게 된다. 이를 `Dead Tuple`이라고 한다. 이런 Dead Tuple들은 다른 Tuple 조회 시 페이지에 포함될 수 있기 때문에 Disk 사용량과 쿼리 속도에 영향을 끼치게 된다. 이를 해결하기 위해 PostgreSQL은 VACUUM을 제공한다.

### VACCUM
우아한 기술 블로그의 내용을 참조했다.
[PostgreSQL VACCUM](https://techblog.woowahan.com/9478/)

### VACCUM 미사용
```sql
### autovacuum off
testdb=> alter table tb_test set (autovacuum_enabled = off);
ALTER TABLE

### delete 전 count 조회
testdb=> select count(*) from tb_test;
  count
----------
 10000000
(1 row)
Time: 548.779 ms

### delete 후 count 조회
testdb=> delete from tb_test where ai != 1;
DELETE 9999999
Time: 30142.916 ms (00:30.143)

testdb=> select count(*) from tb_test;
 count
-------
     1
(1 row)
Time: 497.835 ms
=> 1건을 count하는데도 오래걸림

### Dead Tuple 확인
testdb=> SELECT relname, n_live_tup, n_Dead_tup, n_Dead_tup / (n_live_tup::float) as ratio
FROM pg_stat_user_tables;
 relname  | n_live_tup | n_Dead_tup |  ratio
----------+------------+------------+----------
 tb_test  |          1 |    9999999 |  9999999
=> Dead Tuple이 9999999개로 증가함, 반면에 live Tuple은 1건

### 테이블 사이즈 조회
testdb=> dt+ tb_test
                            List of relations
 Schema |  Name   | Type  | Owner  | Persistence |  Size   | Description
--------+---------+-------+--------+-------------+---------+-------------
 public | tb_test | table | master | permanent   | 1912 MB |
=> 1건짜리 테이블임에도 거의 2GB인 상황
```

tb_test 테이블은 데이터를 1건 가지고 있지만 조회 속도와 테이블 크기 면에서 무언가 1건 답지 않은 이질감이 듭니다(1건 있는데 0.5초라니).
그 이유는 tb_test는 delete 수행으로 인해 Dead Tuple들이 대량 발생했고 실제 데이터는 1건이지만 1912MB의 큰 테이블로 잔뜩 bloating되어 남아있기 때문입니다(실행 계획도 갱신이 되지 않았고요).

### VACUUM 사용

```sql
### autovacuum off
testdb=> alter table tb_test set (autovacuum_enabled = off);
ALTER TABLE

### vacuum 수행 후 Dead Tuple 확인
testdb=> vacuum tb_test;
VACUUM
Time: 11273.959 ms (00:11.274)

testdb=> SELECT relname, n_live_tup, n_Dead_tup, n_Dead_tup / (n_live_tup::float) as ratio
FROM pg_stat_user_tables;
 relname  | n_live_tup | n_Dead_tup | ratio
----------+------------+------------+-------
 tb_test  |          1 |          0 |     0
=> Dead Tuple이 0개로 모두 정리됨

### 데이터 조회 속도 및 테이블 사이즈 비교
testdb=> select count(*) from tb_test;
 count
-------
     1
(1 row)
Time: 9.971 ms
=> 앞에서와 달리 1건 counting이 바로 완료됨

testdb=> dt+ tb_test;
                           List of relations
 Schema |  Name   | Type  | Owner  | Persistence | Size  | Description
--------+---------+-------+--------+-------------+-------+-------------
 public | tb_test | table | master | permanent   | 1912 MB |
=> vacuum 수행 후 dead tuple은 정리되었지만 테이블 사이즈는 그대로임

### vacuum full 수행 후 테이블 사이즈 확인
testdb=> vacuum full tb_test;
VACUUM

testdb=> dt+ tb_test;
                           List of relations
 Schema |  Name   | Type  | Owner  | Persistence | Size  | Description
--------+---------+-------+--------+-------------+-------+-------------
 public | tb_test | table | master | permanent   | 40 kB |
=> vacuum full 수행 후에는 테이블 사이즈도 줄어들었음
```

Vacuum 후 Dead Tuple이 모두 정리되었고 테이블 조회 속도 또한 같은 1건을 count 하지만 훨씬 빨라졌음을 확인했습니다 (497ms → 10ms)

그러나 이때 알아두어야 할 것은
Vacuum full이 아닌 단순 Vacuum으로는 물리 디스크로 이미 할당된 크기는 회수 되지 않았다는 점입니다.(경우에 따라 vacuum으로도 회수되는 경우가 있지만 본 글에서는 다루지 않음) 

<br/>
# 출처
[MVCC(다중 버전 동시성 제어)란?](https://mangkyu.tistory.com/53)
