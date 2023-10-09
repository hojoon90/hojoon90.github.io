---
layout: post
title: "성능 이슈와 코드 개선"
tags: [Spring, JPA]
comments: true
---

## 이슈 사항
현재 개발 후 오픈을 앞두고 있는 앱의 API 속도가 빠르게 나오지 않는 이슈가 생겼다.   
API 호출 시 최대 5초까지 응답 시간이 걸리는데,
정작 응답으로 오는 데이터는 그렇게 많은 양의 데이터는 아니었다.   
게다가 이 API 는 앱의 메인 화면에서 호출하는 API 이기 때문에 속도가 빠르게 나와줘야해서 빠르게 병목 구간을 찾아 내서 코드 개선을 진행하였다.

## 이슈의 원인
먼저 로그 레벨을 디버깅으로 돌리고, 메소드 구간 별 소요 시간을 체크해 보았다.  
문제가 된 부분은 모두 쿼리에서 데이터를 가져오는 부분이었다.  
이번 프로젝트를 진행하면서 처음으로 JPA 를 사용해 보았는데, JPA에 대해 확실하게 알지 못하고 적용하다 보니, 조회와 관련된 성능 이슈들이 발생한 것이다.

### 첫번째 이슈

먼저 첫번째 문제가 된 쿼리는 아래 쿼리이다.
```java
@Query("select new com.domain.primary.store.model.dto.EventStoreDto(e, s) "
        + "from Event e inner join Store s on e.storeId = s.id "
        + "where s.deleted = false "
        + "and e.period.endDate >= :selectDate ")
List<EventStoreDto> findWithStoreBySelectDate(@Param("selectDate") LocalDate selectDate);
```
JPQL 을 통해서 이벤트와 스토어가 조인된 데이터를 조회해오는 것인데, 조회 방식에 문제가 있었다. 해당 쿼리는 아래와 같이 동작했다.
```sql
select event.id, store.id from event inner join store on event.store_id = store.id;

select * from event where event.id = ?
select * from store where event.id = ?
select * from event where event.id = ?
select * from store where event.id = ?
select * from event where event.id = ?
select * from store where event.id = ?
select * from event where event.id = ?
select * from store where event.id = ?
select * from event where event.id = ?
select * from store where event.id = ?
......
```
이벤트 아이디와 스토어 아이디를 먼저 전체 조회 한 다음 해당 아이디들을 조회하는 쿼리를 하나씩 호출하고 있었다.(...)  
당연한 이야기이지만 이렇게 조회되면 성능에 영향이 안 갈 수 없다.  
원인은 JPQL에 생성자로 객체를 바로 넣으면서 생기는 문제 같은데, 일단 급한대로 QueryDSL을 이용해 쿼리 한번으로 조회할 수 있도록 변경해두었다.
아래 DTO에 데이터가 들어가면서 발생하는 문제였다.
```java
@EqualsAndHashCode
@Getter
@AllArgsConstructor
@NoArgsConstructor
public class EventStoreDto {
    private Event event;
    private Store store;

}
```
해당 원인은 N+1 문제가 발생한 것으로 보이는데, 해당 현상에 대해 확인하고 적절한 방법을 찾아볼 예정이다. (업데이트 예정)

### 두번째 이슈

두번째 문제는 아래 쿼리에서 발생하였다.
```java
    @Override
    @RedisCacheable(value = "EVENT:findEventRatio", key = "#selectDay")
    public List<EventStoreDto> findWithStoreBySelectDate(LocalDate selectDate) {
        return queryFactory.select(
            Projections.constructor(
                EventStoreDto.class,
                event,
                store
            )
        )
        .from(event)
        .innerJoin(store)
        .on(event.storeId.eq(store.id))
        .where(
            store.deleted.eq(false)
            , store.usageStatusType.ne(UsageStatusType.FREEZE)
            , store.usageStatusType.ne(UsageStatusType.SIGN_OUT)
            , event.period.endDate.goe(selectDate)
        ).fetch();
    }
```
단순 조건에 맞춰 이벤트와 상점을 조인하여 가져오는 쿼리인데, 해당 메서드 실행 시 약 500ms 정도 소요되는 것을 확인했다.  
실행 쿼리를 가져와 DB에서 다이렉트로 조회하였을때는 약 10ms 정도밖에 걸리지 않았기 때문에 메소드에서 소요시간이 많이 걸린다고 판단하였다.  
확인 결과 Projections.constructor 로 생성자를 만들어줄 때 시간이 오래 걸리는 것을 확인했다.  
그래서 일단은 아래와 같이 Tuple 을 사용하여 객체를 새로 만들어주는 방식으로 임시로 변경해주었다.
```java
    @Override
    @RedisCacheable(value = "EVENT:findEventPeriodData")
    public List<EventPeriodDto> findWithStartEndDate() {
        List<Tuple> fetch = queryFactory.select(
                        event,
                        store
                )
                .from(event)
                .innerJoin(store)
                .on(event.storeId.eq(store.id))
                .where(
                        store.deleted.eq(false)
                        , store.usageStatusType.ne(UsageStatusType.FREEZE)
                        , store.usageStatusType.ne(UsageStatusType.SIGN_OUT)
                        , event.period.endDate.goe(LocalDate.now())
                )
                .fetch();

        List<EventPeriodDto> dto = new ArrayList<>();
        for (Tuple tuple: fetch){
            Event event1 = tuple.get(event);
            dto.add(new EventPeriodDto(event1.getId(), event1.getPeriod(), event1.getCreatedAt()));
        }
        return dto;
    }
```

방법이 일단 속도가 나오게 하기 위한 임시 방편이므로, 추후 수정은 필요해 보인다.  
일단 다행히 API 호출 속도는 30~50 ms 정도로 감소하여 빠르게 API 호출이 가능해졌다.  
여기서 궁금한 점은 QueryProjection을 이용하여 객체 자체에 데이터를 넣어줬는데, 왜 이 방식이 속도가 오래 걸렸는지에 관한 것이다.  
그래서 QueryProjection이 무엇이며, 어떻게 사용하는지에 대해 한번 또 정리를 할 예정이다.

## 정리
결국 문제는 JPA 를 사용하면서 확실하게 공부하지 못하고 사용하다 보니 성능상에 많은 문제가 발생하였다.  
특히 간단한 API 인데도 불구하고 DB 에서 데이터를 조회하는 부분에 많은 시간이 소요된 것을 보고 확실하게 알지 못하고 쓰면 안쓰는게 더 낫다는 생각이 들었다.  
이번 이슈를 해결하면서 정리해야 할 내용을 아래 두가지이다.
* N+1 이슈
* QueryProjection 에 대한 공부

위 내용들에 대해 공부하여 조만간 정리한 글을 올려볼 생각이다. 