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
