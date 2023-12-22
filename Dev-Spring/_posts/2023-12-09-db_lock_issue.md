---
layout: post
title: "DB Lock 발생과 이슈 해결"
tags: [Spring, JPA, Mysql]
comments: true
---

## 문제점
프로젝트 내에서 통계 데이터 갱신 중 DB Lock Timeout 이 발생하는 현상을 확인했다.  
이로 인해 해당 Exception 때문에 정상적인 통계 갱신이 이루어지지 않았다.

## 문제 확인

```java
@Transactional
// 해당 데이터를 모두 새로 갱신.
public Object methodA(){

    ...

      //통계 데이터 적재
      for(Integer date: dateList) {
          makeStatsData(args);
      }

    ...    
}
```

```java
private void makeStatsData(args) {
    ...
    while(!updateDate.equals(latestDate)){
        Integer updateDateInt=Integer.parseInt(updateDate.format(DateTimeFormatter.ofPattern(CommonConstants.YYYYMMDD)));
        //해당 날짜 통계 데이터 삭제
        statsDataService.deleteStatsDatas(updateDateInt);
        for(Keyword k:keywordData){
            statsDataService.setStatsData(k,updateDateInt);
        }
        ...
    }
}

```

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void deleteStatsDatas(Integer yyyymmdd) {
	aStatRepository.deleteByYyyymmdd(yyyymmdd);
    bStatRepository.deleteByYyyymmdd(yyyymmdd);
    cStatRepository.deleteByYyyymmdd(yyyymmdd);
}

```

일자별로 삭제 메서드가 실행된 후, 데이터 적재 메서드가 키워드 별로 반복하면서 적재되는 구조.
첫번째 Delete 후 Insert 까지는 정상 처리 되지만, 그 이후 부터 처리가 되지 않음.
확인 결과 트랜잭션 간 락으로 인해 갱신되지 않았다.

## 발생 원인

1. @Transactional(propagation = Propagation.REQUIRES_NEW) 로 새로운 삭제 트랜잭션을 만듦.
2. 기존 트랜잭션 처리 동안 락이 발생. 이후 락이 풀리지 않음.
    1. @Transactional 이 걸려있는 메서드가 정상 종료 되어야 commit이 일어나고 락이 풀림.
3. 삭제 트랜잭션이 락 대기 상태에 들어가면서 통계 처리가 안됨.

## Lock

- DB가 처리하는 가장 작은 단위
- 트랜잭션 별 순서 보장을 위해 사용
- 하나의 트랜잭션이 끝날때까지 다른 트랜잭션을 막는다.

Lock 은 요소와 상황에 따라 알아야 하지만, 여기서는 요소에 대해서만 간단하게 확인하고 넘어간다.

- Shared-Lock (S-Lock)
- Exclusive-Lock(X-Lock)

### Shared-Lock (공유 락, S-Lock)

- row-level lock
- 읽기에 대한 lock
- S-Lock을 사용하는 쿼리들은 서로 같은 row에 접근할 수 있다.
    - 읽는 용도이기 때문.
    - 즉, 여러 트랜잭션이 하나의 row에 각자 S-Lock 을 걸고 데이터를 읽을 수 있다.
- S-Lock 이 걸려있는 row에는 X-Lock 을 걸 수 없다
    - S-Lock을 건 트랜잭션이 읽고 있으므로 수정, 삭제등이 안된다.

### Exclusive-Lock(배타적 락, X-Lock)

- row-level lock
- 쓰기에 대한 lock
- X-Lock 이 걸려있으면 해당 Lock이 풀리기 전까진 다른 트랜잭션이 접근 못함.
    - 쓰기 도중에 데이터가 바뀌면 안되기 때문.
- X-Lock이 걸려있는 row에는 S-Lock, X-Lock 모두 걸 수 없다.
    - 락이 걸려있는 상태에서는 데이터 수정, 삭제, 조회등을 못하게 막는다.

즉, 발생 원인을 정리하면 아래와 같다.

1. methodA(이하 Tx_1) 실행
2. Tx_1 안에 makeStatsData 메서드로 첫번째 날짜 실행
3. 메소드 안의 while문에서 삭제 메서드(deleteStatsDatas) 실행
4. deleteStatsDatas (이하 Tx_2) 가 별도 트랜잭션으로 동작 하면서 삭제 실행.
    1. 별도 트랜잭션이므로 삭제 후 commit 까지 완료됨.
    2. 삭제 시 X-Lock이 걸리고 삭제가 정상적으로 처리되고 commit 되면서 X-Lock이 풀림
5. Tx_1 안의 통계 처리 로직이 동작함.
    1. 삭제 후 X-Lock이 풀렸으므로 정상적으로 Insert 실행.
    2. 이 때 X-Lock 이 로우에 걸림.
6. 통계 처리 후 다음 날짜의 makeStatsData 실행
    1. Tx_1이 아직 끝나지 않았기 때문에 X-Lock 은 아직 풀리지 않은 상태. (commit이 안된 상태)
7. makeStatsData 메서드 안의 Tx_2 실행
8. Tx_2는 X-Lock이 풀리지 않은 상태라 락 대기에 들어감.
9. Tx_2가 락대기에 들어가면서 다른 메소드들도 실행되지 않고 대기하다가 Exception 발생

아래는 실제 lock 상태일 때 트랜잭션을 조회한 쿼리이다.

### Lock 조회 쿼리

select * from information_schema.INNODB_LOCKS;

| lock_id | lock_trx_id | lock_mode | lock_type | lock_table                     | lock_index | lock_space | lock_page | lock_rec | lock_data |
| --- | --- | --- | --- |--------------------------------| --- | --- | --- | --- | --- |
| 1431401:1917:19:85 | 1431401 | Х | RECORD | 'PROJECT': STATS_BRAND_SA...   | PRIMARY | 1917 | 19 | 85 | 2830 |
| 1431393:1917:19:85 | 1431393 | X | RECORD | 'PROJECT': STATS_BRAND_SA... | PRIMARY | 1917 | 19 | 85 | 2830 |
- 두 트랜잭션 모두 서로 X-Lock 이 걸려있는 상황 (lock_mode 값 X)

### Lock 대기 조회 쿼리

select * from information_schema.INNODB_LOCK_WAITS;

| requesting_trx_id | requested_lock_id | blocking_trx_id | blocking_lock_id |
| --- | --- | --- | --- |
| 1431401 | 1431401:1917:1985 | 1431393 | 1431393:1917:19:85 |
- 1431401 트랜잭션이 작업을 요청했지만 1431393 트랜잭션이 블로킹하고 있어 대기하는 상황

### 트랜잭션 조회 쿼리

select * from information_schema.INNODB_TRX;

| trx_id | trx_state | trx_started | trx_requested_lock_id | trx_wait_started | trx_weight | trx_mysql_thread_id | trx_query | trx_operation_state | trx_tables_in_use | trx_tables_locked | trx_lock_structs | trx_lock_memory_bytes | trx_rows_locked | trx_rows_modified | trx_concurrency_tickets | trx_isolation_level | trx_unique_checks | trx_foreign_key_checks | trx_last_foreign_key_error | trx_is_read_only | trx_autocommit_non_locking |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1431401 | LOCK WAIT | 2023-12-04 09:02:09 | 1431401:1917:19:85 | 2023-12-04 09:02:09 | 41 | 2727 | delete from STATS_BRAND_SALE where YYYYMMDD = 20231122 | fetching rows | 1 | 1 | 15 | 1128 | 27 | 26 | 0 | READ UNCOMMITTED | 1 | 1 | NULL | 0 | 0 |
| 1431393 | RUNNING | 2023-12-04 09:01:56 | NULL | NULL | 4192 | 2726 | NULL |  | 0 | 8 | 192 | 24696 | 3692 | 4000 | 0 | READ UNCOMMITTED | 1 | 1 | NULL | 0 | 0 |
- 실제 상태도 1431401은 LOCK WAIT 상태로 락이 풀리길 기다리고 있고, 1431393은 락이 걸려있는 채로 있는 상황.

## 해결 방안

두가지 정도를 생각해볼 수 있음.

- 삭제 메서드를 한 트랜잭션 안에서 처리하되, 삭제가 바로 일어날 수 있도록 flush 처리.
- 삭제 메서드와 통계 처리 메서드를 별도 트랜잭션으로 분리하여 락 대기에 빠지지 않도록 처리

두가지 방법 중 첫번째 방법으로 선택하여 처리하였음.

현재 통계 로직의 경우 통계 데이터 처리 시 한 곳이라도 문제가 생기면 모두 롤백 해주어야 한다.
그렇기에 별도 트랜잭션으로 동작 할 필요성이 없어서 하나의 트랜잭션으로 묶어줌.
또한, 삭제 쿼리를 바로 DB로 전달하기 위해 flush도 추가해주었다.

```java
@Transactional
public void deleteStatsDatas(Integer yyyymmdd) {
    aStatRepository.deleteByYyyymmdd(yyyymmdd);
    aStatRepository.flush();    // 즉시 삭제를 위해 쿼리를 flush 해준다.

    bStatRepository.deleteByYyyymmdd(yyyymmdd);
    bStatRepository.flush();

    cStatRepository.deleteByYyyymmdd(yyyymmdd);
    cStatRepository.flush();
}
```

이후 통계 처리시 정상적으로 삭제와 등록 처리가 되는것을 확인했다.
트랜잭션에 대해 제대로 알지 못하고 개발하면 위와 같은 이슈가 발생할 수 있다는 것을 이번에 깨달음…