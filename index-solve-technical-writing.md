# 토미의 Coupon 인덱스 코드로 학습하는 인덱스

```java
public class IndexPerformanceTest {

    private static final String BASE_URI = "http://localhost:8080";  
    private static final Long MIN_COUPON_ID = 1L;  
    private static final Long MAX_COUPON_ID = 351160L;  
    private static final Long MIN_MEMBER_ID = 1L;  
    private static final Long MAX_MEMBER_ID = 250000L;  
    private static final int THREAD_COUNT = 10;  
    private static final int TEST_DURATION_SECONDS = 10;  
    private static final long MILLISECONDS_IN_SECOND = 1000L;  
  
    @BeforeEach  
    void setUp() {  
        RestAssured.baseURI = BASE_URI;  
    }

	// ... 테스트 메서드 5문제

	private void executeMultipleRequests(AtomicBoolean running,  
	                                     AtomicInteger requestCount,  
	                                     AtomicLong totalElapsedTime,  
	                                     Runnable runnable) throws InterruptedException {  
	    ExecutorService executorService = Executors.newFixedThreadPool(THREAD_COUNT);  
	    for (int i = 0; i < THREAD_COUNT; i++) {  
	        executorService.execute(() -> executeRequest(running, requestCount, totalElapsedTime, runnable));  
	    }  
	  
	    Thread.sleep(MILLISECONDS_IN_SECOND);    // 스레드에 실행 요청 후 1초간 대기한 후 요청을 시작하도록 변경한다.  
	    running.set(true);  
	    Thread.sleep(TEST_DURATION_SECONDS * MILLISECONDS_IN_SECOND);  
	    running.set(false);  
	  
	    executorService.shutdown();  
	    executorService.awaitTermination(10, TimeUnit.SECONDS);  
	}  
  
	private void executeRequest(AtomicBoolean running,  
	                            AtomicInteger requestCount,  
	                            AtomicLong totalElapsedTime,  
	                            Runnable runnable) {  
	    while (!running.get()) {  
	        // 요청을 시작할 때까지 대기한다.  
	    }  
	  
	    long elapsedTime = 0;  
	    while (running.get()) {  
	        long startTime = System.currentTimeMillis();  
	        runnable.run();  
	        long endTime = System.currentTimeMillis();  
	  
	        elapsedTime += endTime - startTime;  
	        requestCount.incrementAndGet();  
	    }  
	  
	    totalElapsedTime.addAndGet(elapsedTime);  
	}
}
```

각 테스트 메서드는 먼저 해당 API의 상태 코드를 확인하고, 이후 executeMultipleRequests 메서드를 호출하여 executeRequest 메서드를 통해 여러 스레드로 동시에 요청을 보냅니다.

1. executeMultipleRequests 메서드 
   - 주어진 Runnable을 여러 스레드에서 실행하여, 지정된 시간 동안 요청을 보냅니다. 
   - AtomicBoolean, AtomicInteger, AtomicLong을 사용하여 멀티스레드 환경에서 안전하게 상태를 관리합니다.
2. executeRequest 메서드 
   - 주어진 Runnable을 실행하여 요청을 보내고, 소요된 시간을 측정합니다.
    - 요청 수와 총 소요 시간을 각각 증가시켜 테스트 결과를 집계합니다.

이 코드는 각 테스트 메서드에서 RestAssured.get 메서드를 호출하여 랜덤한 쿠폰 ID에 대한 요청을 보냅니다. 구체적으로 ThreadLocalRandom.current().nextLong(MIN_COUPON_ID, MAX_COUPON_ID + 1)를 사용해 최소 쿠폰 ID와 최대 쿠폰 ID 사이의 랜덤한 값을 생성하며, executeMultipleRequests와 executeRequest 메서드가 이 과정을 10개의 스레드 (THREAD_COUNT)에서 동시에 실행하며, 10초 (TEST_DURATION_SECONDS) 동안 계속됩니다.

결과적으로, 각 스레드는 서로 다른 랜덤 ID로 조회 API 요청을 보내며, 각 테스트 메서드에서는 요청 수와 평균 소요 시간을 출력하여 성능 기준을 검증합니다. 평균 소요 시간이 100ms 이하이어야 한다는 조건이 있습니다.

1번 문제

```java
@Test  
void 쿠폰의_발급_수량_조회() throws InterruptedException {  
AtomicBoolean running = new AtomicBoolean(false);  
AtomicInteger requestCount = new AtomicInteger(0);  
AtomicLong totalElapsedTime = new AtomicLong(0);

    int statusCode = RestAssured.get("/coupons/" + ThreadLocalRandom.current()  
            .nextLong(MIN_COUPON_ID, MAX_COUPON_ID + 1) + "/issued-count").statusCode();  
    assertThat(statusCode).withFailMessage("쿠폰의 발급 수량 조회 API 호출에 실패했습니다. 테스트 대상 서버가 실행중인지 확인해 주세요.").isEqualTo(200);  
  
    executeMultipleRequests(running, requestCount, totalElapsedTime,  
            () -> RestAssured.get("/coupons/" + ThreadLocalRandom.current()  
                    .nextLong(MIN_COUPON_ID, MAX_COUPON_ID + 1) + "/issued-count"));  
  
    System.out.println("Total request count: " + requestCount.get());  
    System.out.println("Total elapsed time: " + totalElapsedTime.get() + "ms");  
  
    long averageElapsedTime = totalElapsedTime.get() / requestCount.get();  
    System.out.println("Average elapsed time: " + averageElapsedTime + "ms");  
  
    assertThat(averageElapsedTime).isLessThanOrEqualTo(100L);
}

// 실행 결과 출력은 이런 식으로 나온다.
Total request count: 12261  
Total elapsed time: 100029ms  
Average elapsed time: 8ms
```

/coupons/{coupon_id}/issued-count API를 호출하는 컨트롤러 메서드는 다음과 같습니다.

```java
// CouponController.class
@GetMapping("{couponId:^\\d+$}/issued-count")  
public Long getCouponIssuedCount(@PathVariable("couponId") Long couponId) {  
    Coupon coupon = couponService.getCoupon(couponId);  
    return memberCouponService.findIssuedCouponCount(coupon.getId());  
}
```

couponService.getCoupon(couponId) 메서드는 일반적으로 couponId로 Coupon을 조회하는 메서드입니다. 문제 해결을 위해 봐야할 부분은 memberCouponService.findIssuedCouponCount(coupon.getId())에서 데이터를 조회하는 부분입니다.

```java
// MemberCouponService.class
@Transactional(readOnly = true)  
public Long findIssuedCouponCount(Long couponId) {  
    return memberCouponRepository.countByCoupon_Id(couponId);  
}

// MemberCouponRepository.class
@Repository  
@Transactional  
public interface MemberCouponRepository extends JpaRepository<MemberCoupon, Long> {

    List<MemberCoupon> findByMemberIdAndUsedAndUseEndedAtAfter(Long memberId, boolean used, LocalDateTime now);  
  
    Long countByCoupon_Id(Long couponId);  
  
    Long countByCoupon_IdAndUsed(Long couponId, boolean used);  
  
  
    @Modifying  
    @Query("update MemberCoupon mc set mc.used = :used, mc.usedAt = :usedAt where mc.id in :memberCouponIds")  
    void updateUsedAndUsedAt(@Param("memberCouponIds") List<Long> memberCouponIds,  
                             @Param("used") boolean used,  
                             @Param("usedAt") LocalDateTime usedAt);
}
```

MemberCouponService에서 MemberCouponRepository의 countByCoupon_id(couponId) 메서드를 호출하는데, 이는 다음과 같은 쿼리를 발생시킵니다

```sql
-- MemberCouponRepository.countByCoupon_id(couponId)
select count(*)  
from member_coupon mc  
where mc.coupon_id = couponId;
```

### Coupon.class
```java
// Coupon.class
@Entity  
@Table(name = "coupon")  
@Getter  
@NoArgsConstructor(access = AccessLevel.PROTECTED)  
public class Coupon {

    @Id  
    @GeneratedValue(strategy = GenerationType.IDENTITY)  
    @Column(name = "id")  
    private Long id;  
  
    @Column(name = "discount_amount")  
    private int discountAmount;  
  
    @Column(name = "minimum_order_price")  
    private int minimumOrderPrice;  
  
    @Column(name = "coupon_status", columnDefinition = "VARCHAR(30)")  
    @Enumerated(value = EnumType.STRING)  
    private CouponStatus couponStatus;  
  
    @Column(name = "issuable", columnDefinition = "BOOLEAN")  
    private boolean issuable;  
  
    @Column(name = "usable", columnDefinition = "BOOLEAN")  
    private boolean usable;  
  
    @Column(name = "issue_started_at", columnDefinition = "DATETIME(6)")  
    private LocalDateTime issueStartedAt;  
  
    @Column(name = "issue_ended_at", columnDefinition = "DATETIME(6)")  
    private LocalDateTime issueEndedAt;  
  
    @Column(name = "limit_type", columnDefinition = "VARCHAR(20)")  
    @Enumerated(value = EnumType.STRING)  
    private CouponLimitType limitType;  
  
    @Column(name = "issue_limit")  
    private Long issueLimit;  
  
    @Column(name = "issue_count")  
    private Long issueCount;  
  
    @Column(name = "use_limit")  
    private Long useLimit;  
  
    @Column(name = "use_count")  
    private Long useCount;  
  
    @Column(name = "created_at", columnDefinition = "DATETIME(6)")  
    private LocalDateTime createdAt;  
  
    @Column(name = "modified_at", columnDefinition = "DATETIME(6)")  
    private LocalDateTime modifiedAt;  
  
    public void issue() {  
        if (this.issueStartedAt.isAfter(LocalDateTime.now()) || this.issueEndedAt.isBefore(LocalDateTime.now())) {  
            throw new IllegalArgumentException("쿠폰을 발급할 수 없는 시간입니다.");  
        }  
        if (couponStatus.isNotIssuable() || !this.issuable) {  
            throw new IllegalArgumentException("쿠폰을 발급할 수 없는 상태입니다.");  
        }  
        if (this.limitType.isNotIssueCountLimit()) {  
            return;  
        }  
        if (this.issueLimit <= this.issueCount) {  
            throw new IllegalArgumentException("쿠폰을 더 이상 발급할 수 없습니다.");  
        }  
        this.issueCount++;  
    }  
  
    public boolean isIssuableCoupon(LocalDateTime localDateTime) {  
        if (this.issueStartedAt.isAfter(localDateTime) || this.issueEndedAt.isBefore(localDateTime)) {  
            return false;  
        }  
        if (couponStatus.isNotIssuable() || !this.issuable) {  
            return false;  
        }  
        if (this.limitType.isNotIssueCountLimit()) {  
            return true;  
        }  
        return this.issueLimit > this.issueCount;  
    }  
  
    public void use() {  
        if (couponStatus.isNotUsable() || !this.usable) {  
            throw new IllegalArgumentException("쿠폰 사용이 불가능합니다.");  
        }  
        if (this.limitType.isNotUseCountLimit()) {  
            return;  
        }  
        if (this.useLimit <= this.useCount) {  
            throw new IllegalArgumentException("쿠폰을 더 이상 사용할 수 없습니다.");  
        }  
        this.useCount++;  
    }  
  
    public boolean isUsableCoupon() {  
        if (couponStatus.isNotUsable() || !this.usable) {  
            return false;  
        }  
        if (this.limitType.isNotUseCountLimit()) {  
            return true;  
        }  
        return this.useLimit > this.useCount;  
    }  
  
    public void setStatus(CouponStatus couponStatus) {  
        this.couponStatus = couponStatus;  
    }  
  
    public void setIssuable(boolean issuable) {  
        this.issuable = issuable;  
    }  
  
    public void setUsable(boolean usable) {  
        this.usable = usable;  
    }  
}
```

### MemberCoupon.class
```java
// MemberCoupon.class
@Entity  
@Table(name = "member_coupon")  
@Getter  
@NoArgsConstructor(access = AccessLevel.PROTECTED)  
public class MemberCoupon {

    /**  
     * 모든 쿠폰은 발급 일시부터 7일간 사용할 수 있다.     */    
     private static final int COUPON_USABLE_DAYS = 7;  
  
    @Id  
    @GeneratedValue(strategy = GenerationType.IDENTITY)  
    private Long id;  
  
    @Column(name = "member_id")  
    private Long memberId;  
  
    @ManyToOne(fetch = FetchType.EAGER)  
    @JoinColumn(name = "coupon_id")  
    private Coupon coupon;  
  
    @Column(name = "issued_at", columnDefinition = "DATETIME(6)")  
    private LocalDateTime issuedAt;  
  
    @Column(name = "use_ended_at", columnDefinition = "DATETIME(6)")  
    private LocalDateTime useEndedAt;  
  
    @Column(name = "used", columnDefinition = "BOOLEAN")  
    private boolean used;  
  
    @Column(name = "used_at", columnDefinition = "DATETIME(6)")  
    private LocalDateTime usedAt;  
  
    @Column(name = "modified_at", columnDefinition = "DATETIME(6)")  
    private LocalDateTime modifiedAt;  
  
    public static MemberCoupon issue(Long memberId, Coupon coupon) {  
        coupon.issue();  
  
        MemberCoupon memberCoupon = new MemberCoupon();  
        memberCoupon.memberId = memberId;  
        memberCoupon.coupon = coupon;  
        memberCoupon.issuedAt = LocalDateTime.now();  
        memberCoupon.useEndedAt = memberCoupon.issuedAt.plusDays(COUPON_USABLE_DAYS);  
        memberCoupon.modifiedAt = LocalDateTime.now();  
        memberCoupon.used = false;  
  
        return memberCoupon;  
    }  
  
    public void use() {  
        if (this.used) {  
            throw new IllegalStateException("이미 사용한 쿠폰입니다.");  
        }  
        this.coupon.use();  
  
        this.used = true;  
        this.usedAt = LocalDateTime.now();  
        this.modifiedAt = LocalDateTime.now();  
    }  
}
```

Coupon과 MemberCoupon의 관계는 1:N @ManyToOne 관계입니다. MemberCoupon은 Coupon의 id를 FK로 가지고 있습니다. 외래 키 제약 조건(FK Constraints)을 설정할 때 외래 키 컬럼에 대한 인덱스가 자동으로 생성됩니다. show index from member_coupon; 조회를 통해 member_coupon 테이블에 생성된 인덱스를 볼 수 있습니다. coupon_id 컬럼에 대한 외래 키 인덱스는 FKkxw7ja7v55gk4a368w3gs6s0j라는 Key_name으로 지정되어 있습니다.
![[Pasted image 20240829195911.png]](image/Pasted image 20240829195911.png)

| #  | Table        | Non_unique | Key_name                     | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment | Visible |
|----|--------------|------------|-------------------------------|---------------|-------------|-----------|-------------|----------|--------|------|-------------|---------|----------------|---------|
| 1  | member_coupon | 0          | PRIMARY                       | 1             | id          | A         | 468036      |          |        |      | BTREE       |         |                | YES     |
| 2  | member_coupon | 1          | FKkxw7ja7v55gk4a368w3gs6s0j | 1             | coupon_id   | A         | 1207        |          |        | YES  | BTREE       |         |                | YES     |

```sql
explain select count(*)
from member_coupon mc  
where mc.coupon_id = 1;
```

조회 쿼리의 맨 앞에 explain을 붙여서 실행 계획을 볼 수 있습니다.

![[Pasted image 20240829200549.png]](image/Pasted image 20240829200549.png)

| #  | id | select_type | table | partitions | type | possible_keys                  | key                           | key_len | ref  | rows | filtered | Extra         |
|----|----|-------------|-------|------------|------|--------------------------------|-------------------------------|---------|------|------|----------|---------------|
| 1  | 1  | SIMPLE      | mc    |            | ref  | FKkxw7ja7v55gk4a368w3gs6s0j | FKkxw7ja7v55gk4a368w3gs6s0j | 9       | const | 183  | 100      | Using index   |

type은 ref, key는 FKkxw7ja7v55gk4a368w3gs6s0j, Extra는 Using index입니다. filtered는 쿼리에서 얼마나 많은 행이 필터링되는지를 백분율로 나타냅니다. 값은 100으로, 모든 행이 조건을 만족하는 것을 의미합니다. filtered가 100%면 모든 행이 조건을 만족하지만, 이는 반드시 커버링 인덱스라는 것을 의미하지는 않습니다.

>커버링 인덱스란, 쿼리에 필요한 모든 컬럼이 인덱스에 포함되어 있어 데이터 페이지에 접근하지 않고도 쿼리 결과를 반환할 수 있는 경우를 말합니다. 따라서 filtered가 100%일 경우 인덱스를 사용해 모든 관련 데이터를 찾은 것이며, 인덱스가 쿼리에 필요한 모든 컬럼을 포함해야 커버링 인덱스로 간주될 수 있습니다.

## EXPLAIN의 주요 요소

- id: 쿼리의 고유 식별자. 
- select_type: 쿼리의 유형(예: SIMPLE, PRIMARY, SUBQUERY 등). 
- table: 쿼리에서 참조하는 테이블. 
- type: 조인의 유형(ref, eq_ref, ALL 등). 
- possible_keys: 쿼리에 사용할 수 있는 인덱스. 
- key: 실제로 사용된 인덱스. 
- key_len: 사용된 인덱스의 길이. 
- ref: 인덱스가 참조하는 열. 
- rows: 데이터베이스가 쿼리를 실행하기 위해 읽어야 할 예상 행 수. 
- filtered: 쿼리에서 필터링된 행 비율. 
- extra: 추가적인 정보(예: Using index, Using where 등).

```sql
explain analyze select count(*)
from member_coupon mc  
where mc.coupon_id = 1;

-> Aggregate: count(0)  (cost=60.8 rows=1) (actual time=0.0957..0.0957 rows=1 loops=1)
-> Covering index lookup on mc using FKkxw7ja7v55gk4a368w3gs6s0j (coupon_id=1)  (cost=18.6 rows=183) (actual time=0.0389..0.0832 rows=183 loops=1)
```

## EXPLAIN ANALYZE

EXPLAIN ANALYZE는 EXPLAIN의 확장으로, 쿼리를 실제로 실행한 후의 실행 계획과 함께 실행 시간 등의 실제 성능 정보를 제공합니다.
- 실제 실행 시간: 쿼리가 실제로 실행되는 데 걸린 시간.
- 실제 행 수: 쿼리 실행 중에 처리된 실제 행 수.
- 메모리 사용량: 쿼리 실행 시 사용된 메모리 양.

[Mysql Explain](https://cheese10yun.github.io/mysql-explian/) 블로그에서 발췌한 내용이며 자세한 설명을 볼 수 있습니다.
### type

>type은 접근 방식을 표시하는 필드다. 접근 방식은 테이블에서 어떻게 행데이터를 가져올것인가를 가리킨다. 위 EXPLAIN에서는 ALL, eq_ref, ref가 있는데 ALL, eq_ref는 조인시 기본 키나 고유키를 사용하여 하나의 값으로 접근(최대 1행만을 정확하게 패치), ref는 여러 개의 행을 패치할 가능성이 있는 접근을 의미한다.**접근 방식은 대상 테이블로의 접근이 효율적일지 여부를 판단하는 데 아주 중요한 항목이다.**
>
>이들 접근 방식 가운데도 주의가 필요한 것은 ALL, index, ref_or_null이다.**ALL, index 두 가지는 테이블 또는 특정 인덱스가 전체 행에 접근하기 때문에 테이블 크기가 크면 효율이 떨어진다. ref_or_null의 경우 NUL이 들어있는 행은 인덱스의 맨 앞에 모아서 저장하지만 그 건수가 많으면 MySQL 서버의 작업량이 방대해진다. 다시 말해서 ALL 이외의 접근 방식은 모두 인덱스를 사용한다.**
>
>접근 방식이 ALL 또는 index인 경우는 그 쿼리로 사용할 수 있는 적절한 인덱스가 없다는 의미일 수도 있다. 위 쿼리에서 Country 테이블에 대한 접근은 ALL이지만 이는 WHERE 구의 조건을 지정하지 않았기 때문이다. 그러한 쿼리에서 드라이빙 테이블에 접근한다면 전체 행을 스캔 할수 밖에 없다.

| 접근 방식           | 설명                                                                                                                             |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| const           | 기본 키 또는 고유키에 의한 loockup(등가비교), 조인이 아닌 가장 외부의 테이블에 접근 하는 방식, 결과는 항상 1행이다. 단 기본 키, 고유 키를 사용하고 있으므로 범위 검색으로 지정하는 경우 const가 되지 않는다 |
| system          | 테이블에 1행밖에 없는 경우의 특수한 접근 방식                                                                                                     |
| ALL             | 전체 행 스캔, 테이블의 데이터 전체에 접근한다.                                                                                                    |
| index           | 인덱스 스캔, 테이블의 특정 인덱스의 전체 엔트리에 접근한다.                                                                                             |
| eq_ref          | 조인이 내부 테이블로 접근할 때 기본키 또는 공유 키에 의한 lookup이 일어난다. const와 비슷하지만 조인의 내부 테이블에 접근한다는 점이 다르다                                          |
| ref             | 고유 키가아닌 인덱스에 대한 등가비교, 여러 개 행에 접근할 가능성이 있다.                                                                                     |
| ref_or_null     | ref와 마찬가지로 인덱스 접근 시 맨 앞에 저장되어 있는 NULL의 엔트리를 검색한다.                                                                              |
| range           | 인덱스 특정 범위의 행에 접근한다                                                                                                             |
| fulltext        | fulltext 인덱스를 사용한 검색                                                                                                           |
| index_merge     | 여러 개인스턴스를 사용해 행을 가져오고 그 결과를 통합한다.                                                                                              |
| unique_subquery | IN 서브쿼리 접근에서 기본 키 또는 고유 키를 사용한다. 이 방식은 쓸데 없는 오버헤드를 줄여 상당히 빠르다.                                                                 |
| index_subquery  | unique_sunquery와 거의 비슷하지만 고유한 인덱스를 사용하지 않는 점이 다르다. 이 접근 방식도 상당히 빠르다                                                            |

## 다시 돌아가서

expain 결과로 나온 데이터를 보면 type은 ref, key는 FKkxw7ja7v55gk4a368w3gs6s0j인 것을 볼 수 있습니다. 즉, 해당 쿼리는 인덱스를 사용해서 조회가 되었다는 것을 의미하고, 사용된 인덱스는 coupon_id 외래 키 인덱스인 것을 알 수 있습니다. where 조건으로 필요한 것은 coupon_id 한 개 뿐이기 때문에 이러한 조회가 가능했습니다. 그래서 따로 인덱스를 설정해주지 않아도 100ms 제한에 통과할 수 있었습니다.

```java
Total request count: 12261  
Total elapsed time: 100029ms  
Average elapsed time: 8ms
```

### 외래 키 인덱스를 사용하지 않으면 어떻게 될까?

이대로 넘어가기엔 coupon_id 외래 키 인덱스를 사용하지 않고 조회하면 얼마나 걸릴지 궁금해져 외래 키 제약 조건과 인덱스를 제거하고 다시 수행해 보았습니다.

```sql
ALTER TABLE member_coupon DROP FOREIGN KEY FKkxw7ja7v55gk4a368w3gs6s0j;  
ALTER TABLE member_coupon DROP INDEX FKkxw7ja7v55gk4a368w3gs6s0j;
```

![[Pasted image 20240829204204.png]](image/Pasted image 20240829204204.png)
![[Pasted image 20240829204245.png]](image/Pasted image 20240829204245.png)
보다시피 인덱스를 제거하고 explain으로 실행 계획을 살펴보니 type이 ALL, key는 null로 표시됩니다. type으로 ALL은 테이블의 전체 데이터에 접근했다는 것을 의미합니다. 결과적으로 평균 경과 시간이 100ms를 초과했기 때문에 검증에 실패합니다.

```java
Total request count: 913 
Total elapsed time: 100258ms
Average elapsed time: 109ms
```

다시 member_coupon 테이블에 coupon 테이블의 id를 외래 키 제약 조건으로 등록했습니다.

```sql
ALTER TABLE member_coupon  
ADD CONSTRAINT FKkxw7ja7v55gk4a368w3gs6s0j FOREIGN KEY (coupon_id) REFERENCES coupon (id);
```

## 2번 문제

```java
@Test  
void 쿠폰의_사용_수량_조회() throws InterruptedException {  
    AtomicBoolean running = new AtomicBoolean(false);  
    AtomicInteger requestCount = new AtomicInteger(0);  
    AtomicLong totalElapsedTime = new AtomicLong(0);

    int statusCode = RestAssured.get("/coupons/" + ThreadLocalRandom.current()  
            .nextLong(MIN_COUPON_ID, MAX_COUPON_ID + 1) + "/used-count").statusCode();  
    assertThat(statusCode).withFailMessage("쿠폰의 사용 수량 조회 API 호출에 실패했습니다. 테스트 대상 서버가 실행중인지 확인해 주세요.").isEqualTo(200);  
  
    executeMultipleRequests(running, requestCount, totalElapsedTime,  
            () -> RestAssured.get("/coupons/" + ThreadLocalRandom.current()  
                    .nextLong(MIN_COUPON_ID, MAX_COUPON_ID + 1) + "/used-count"));  
  
    System.out.println("Total request count: " + requestCount.get());  
    System.out.println("Total elapsed time: " + totalElapsedTime.get() + "ms");  
  
    long averageElapsedTime = totalElapsedTime.get() / requestCount.get();  
    System.out.println("Average elapsed time: " + totalElapsedTime.get() / requestCount.get() + "ms");  
  
    assertThat(averageElapsedTime).isLessThanOrEqualTo(100L);
}
```

2번 문제는 쿠폰의 id로 해당 쿠폰의 사용 수량을 조회하는 API를 호출하는 문제입니다.

```java
// CouponController.class
@GetMapping("{couponId:^\\d+$}/used-count")  
public Long getCouponUsedCount(@PathVariable("couponId") Long couponId) {  
    Coupon coupon = couponService.getCoupon(couponId);  
    return memberCouponService.findUsedCouponCount(coupon.getId());  
}
```

```java
// MemberCouponService.class
@Transactional(readOnly = true)  
public Long findUsedCouponCount(Long couponId) {  
    return memberCouponRepository.countByCoupon_IdAndUsed(couponId, true);  
}
```

```java
// MemberCouponRepository.class
Long countByCoupon_IdAndUsed(Long couponId, boolean used);
```
```sql
select count(*)
from member_coupon mc
where mc.coupon_id = couponId
and mc.used = used;
```

countByCoupon_IdAndUsed메서드를 호출하면 이러한 쿼리가 발생합니다.

조건에 사용되는 컬럼은 member_coupon 테이블의 외래 키 컬럼인 coupon_id와 boolean 값인 used 컬럼 두 개입니다. 먼저 member_coupon 테이블에 생성된 인덱스를 조회해야 합니다.

```sql
show index from member_coupon;
```

![[Pasted image 20240830010640.png]](image/Pasted image 20240830010640.png)

| # | Table        | Non_unique | Key_name                        | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment | Visible |
|---|--------------|------------|---------------------------------|---------------|-------------|-----------|-------------|----------|--------|------|-------------|---------|----------------|---------|
| 1 | member_coupon| 0          | PRIMARY                         | 1             | id          | A         | 468036      |          |        |      | BTREE       |         |                | YES     |
| 2 | member_coupon| 1          | FKkxw7ja7v55gk4a368w3gs6s0j    | 1             | coupon_id   | A         | 1207        |          |        | YES  | BTREE       |         |                | YES     |

현재 member_coupon 테이블에 member_coupon_id 인덱스를 제외하면
coupon 테이블의 외래 키 컬럼 인덱스만 존재합니다. 그리고 coupon_id의 Cardinality는 1207 입니다.

## 커버링 인덱스가 아닌 조회 조건의 일부만 만족하는 조회 쿼리를 날리면 과연 인덱스를 잘 탈까?

```sql
explain select count(*)  
from member_coupon mc  
where mc.coupon_id = 1  
and mc.used = true;
```

![[Pasted image 20240830011011.png]](image/Pasted image 20240830011011.png)

| # | id | select_type | table | partitions | type | possible_keys                     | key                             | key_len | ref   | rows | filtered | Extra       |
|---|----|--------------|-------|------------|------|-----------------------------------|---------------------------------|---------|-------|------|----------|-------------|
| 1 | 1  | SIMPLE       | mc    |            | ref  | FKkxw7ja7v55gk4a368w3gs6s0j      | FKkxw7ja7v55gk4a368w3gs6s0j    | 9       | const | 183  | 10       | Using where |

explain으로 실행 계획을 보니, type은 ref, key는 coupon_id 외래 키 인덱스를 사용하여 조회됩니다. 그런데 Extra 컬럼을 보면 Using where라고 나와있습니다. 인덱스를 사용했다면 Using index가 나와야 하는데 이게 무슨 의미일까요?

결론부터 말하면 인덱스를 사용한 것입니다. Using where는 인덱스를 활용하여 데이터를 찾았지만, 추가적인 조건을 만족하기 위해 where 절로 필터링을 했다는 것을 의미합니다. 즉, 인덱스가 사용되어 조회 성능이 향상되었지만, used 컬럼에 대한 인덱스가 걸려있지 않기 때문에, 모든 조건을 인덱스만으로 처리할 수는 없어 where 절로 추가 필터링이 필요했던 것입니다.

```java
Total request count: 9804
Total elapsed time: 100019ms
Average elapsed time: 10ms
```

테스트 수행 결과 인덱스만을 사용해서 조회하고 있지는 않지만, 인덱스와 함께 where절을 사용하여 조회하고 있기 때문에 비교적 조회 성능이 좋아 테스트에 통과합니다.

## 커버링 인덱스를 구성해서 비교

그렇다면 쿼리에 필요한 모든 컬럼이 인덱스에 포함되어 데이터 페이지에 접근하지 않고도 쿼리 결과를 반환할 수 있는 커버링 인덱스를 구성하면 얼마나 성능이 개선될까요? coupon_id와 함께 used 컬럼을 복합 인덱스로 생성해 보겠습니다.

```sql
create index idx_coupon_used on member_coupon (coupon_id, used);
```

![[Pasted image 20240830014647.png]](image/Pasted image 20240830014647.png)
coupon_id, used 컬럼을 묶어서 idx_coupon_used라는 이름의 복합 인덱스를 생성했습니다.
복합 인덱스를 구성하고 다시 한 번 실행 계획을 살펴보면 다음과 같습니다.

```markdown
| #  | id | select_type | table | partitions | type | possible_keys  | key           | key_len | ref         | rows | filtered | Extra       |
|----|----|-------------|-------|------------|------|----------------|---------------|---------|-------------|------|----------|-------------|
| 1  | 1  | SIMPLE      | mc    |            | ref  | idx_coupon_used | idx_coupon_used | 11      | const,const | 6    | 100      | Using index |
```

type는 ref, key로 방금 만든 idx_coupon_used 인덱스를 사용하고 있고, filtered는 100, Extra 또한 Using index로 나옵니다.

```java
Total request count: 12920  
Total elapsed time: 100051ms  
Average elapsed time: 7ms
```

테스트 실행 결과 커버링 인덱스를 구성해도 생각보다 큰 차이가 나지 않았습니다.

```sql
show index from member_coupon;
```

![[Pasted image 20240830021433.png]](image/Pasted image 20240830021433.png)
idx_coupon_used 인덱스의 Cardinality를 보면 coupon_id는 1344, used는 2505입니다.
인덱스를 활용해서 최대한 중복도가 낮은 컬럼 순으로 데이터를 필터링하기 위해 Cardinality가 높은 컬럼인 used, coupon_id, idx_used_coupon 순서로 인덱스를 생성해 실행해 보았습니다.

```java
Total request count: 11494  
Total elapsed time: 99948ms  
Average elapsed time: 8ms
```

앞선 두 개의 인덱스 구성과 결과가 크게 다르지 않았습니다.

```sql
select count(distinct used) from member_coupon; -- 결과 2
```

used 컬럼은 boolean 값이기 때문에 true, false 두 개 뿐 이므로, 이론상 used 컬럼의 Cardinality가 coupon_id 컬럼보다 낮게 나와야 합니다. 왜 이런걸까요?

## Cardinality

Cardinality는 인덱스에서 고유 값의 수를 나타내며, 동일한 값이 여러 번 존재할 경우 그 수가 반영됩니다. 즉, 중복도가 높으면 Cardinality는 낮고, 중복도가 낮으면 Cardinality가 높습니다. used 컬럼은 true와 false 두 가지 값만 가지므로, 이론적으로는 2가 되어야 하지만, 데이터가 특정 값에 집중되어 있다면 Cardinality 수치가 낮게 나올 수 있습니다.

### Cardinality가 낮음에도 coupon_id의 인덱스의 순서를 먼저 두어야 하는 이유?

일반적으로 외래 키는 데이터베이스에서 자주 사용되는 필터링 조건입니다. coupon_id가 쿼리에서 직접적으로 사용되는 경우가 많기 때문에, 인덱스의 첫 번째 컬럼으로 두는 것이 효율적입니다. used가 boolean 필드로서 두 가지 값만 가지므로, 인덱스에서의 선택성이 떨어질 수 있습니다.

반면, coupon_id는 더 다양한 값을 가질 수 있어 데이터 선택성이 높습니다. 선택성이 높은 컬럼을 인덱스의 앞부분에 두는 것이 쿼리 성능에 좋습니다. 현재 데이터로는 coupon_id가 Cardinality가 낮을 수 있어도, 시간이 지남에 따라 데이터 타입이 boolean인 used 컬럼보다 Cardinality가 높아질 가능성이 많습니다.

또한 외래 키는 일반적으로 다른 테이블과의 조인에서 사용됩니다. coupon_id가 조인 조건으로 사용될 경우, 인덱스의 첫 번째 컬럼으로 위치하는 것이 조인 성능을 향상시킬 수 있습니다.
