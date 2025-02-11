# 인덱스
## 인덱스 구조
### MySQL
![image](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FczAehF%2FbtsIde25PaL%2FScU5UikwbkMFCbwUeD9300%2Fimg.webp)

MySQL은 `기본 인덱스`를 가지고 있다. 이를 **Clustered Indexes** 또는 **Index-Organized Table** 이라고 한다.
기본 인덱스는 Key가 PK로 정렬되어 있고, Leaf Node에는 **키가 존재하는 페이지**를 가지고 있기 때문에 추가적인 디스크 I/O가 발생하지 않는다.
이와 다르게 보조 인덱스는 인덱싱을 위한 열이 정렬되어 있고, Leaf Node에는 **기본 키**가 들어 있다.
따라서, MySQL에선 기본 인덱스를 제외한 모든 보조 인덱스는 주 인덱스를 거쳐 데이터를 가져온다. 

### PostgreSQL
![image](https://vladmihalcea.com/wp-content/uploads/2024/03/HeapTable.png)
PostgreSQL은 기본 인덱스가 없고, 모든 인덱스가 보조 인덱스로, 데이터 페이지의 Row ID를 가진다. 따라서 Heap-Organized Table이라고 불린다. 
MySQL의 경우, 기본 인덱스의 Leaf 노드에 페이지를 가지고 있으면서 정렬되어 있기 때문에 Range Query에 매우 빠르게 작동한다. 반변에 PostgreSQL의 경우, 보조 인덱스가 가르키는 힙 테이블은 정렬되어 있지 않아 Random Access가 발생하여 느리게 작동한다.

### Query Cost
```sql
# WHERE
SELECT * FROM T WHERE C2 = 'x2';
```

- MySQL : C2에 대한 보조 인덱스 + 주 인덱스로 전체 행 반환 (2 I/O)
- PostgreSQL : 보조 인덱스 조회해서 힙에서 페이지 가져옴 (단일 I/O)

```sql
# RANGE QUERY
SELECT * FROM T WHERE PK BETWEEN 1 AND 3;
```

- MySQL : 클러스터링 인덱스를 통해 리프 노드에서 바로 가져올 수 있음
- PostgreSQL : 보조 인덱스 조회로 키를 찾지만, 튜플 ID와 페이지만 수집함 → 따라서 힙에서 전체 행을 가져오기 위한 Random Access를 수행함

```sql
UPDATE T SET C1 = ‘XX1’ WHERE PK = 1;
```

- MySQL : 리프 페이지만 새 값으로 업데이트
- PostgreSQL : 새로운 튜플 생성 → 모든 보조 인덱스가 새 튜플 ID로 업데이트
    - 많은 쓰기 I/O 초래 → Uber가 MySQL로 갈아탄 이유

### 결론

단순한 equal 쿼리의 경우 PostgreSQL이 단일 I/O로 성능이 좋음

그러나 Range Query와 Update Query는 MySQL이 리프 노드의 데이터로 인해 빠름

<br/>

# 병렬 처리
## MySQL 8

`멀티 스레드` 사용으로 더 낮은 메모리 사용량, 빠른 컨텍스트 스위칭이 장점이다. 이때문에 단일 스레드 성능이 느릴 경우 병목 현상이 발생할 수 있다.

## PostgreSQL 16

`멀티 프로세스` 사용으로 더 나은 격리성과 더 안정적인 병렬 처리가 가능하다. 프로세스로 인한 높은 메모리 사용량과 프로세스 간 통신 오버헤드가 단점이다.

![image](https://github.com/user-attachments/assets/18ada08e-415a-4a03-aedf-160f0997260f)

MySQL은 단일 프로세스로 인해 장애 복구와 컨텍스트 스위칭 오버에드에서 이점을 가진다.
반면 PostgreSQL은 대규모 시스템에서의 확장성, 대규모 쿼리, 분석에서 이점을 가진다.


## 출처
[MySQL 8 vs. PostgreSQL 16](https://rastalion.dev/mysql-8-0-vs-postgresql-16-%EC%8B%AC%EC%B8%B5-%EB%B9%84%EA%B5%90-%EB%B6%84%EC%84%9D/)
