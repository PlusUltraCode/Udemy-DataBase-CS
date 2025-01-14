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