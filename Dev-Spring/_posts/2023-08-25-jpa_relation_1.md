---
layout: post
title: "단방향과 양방향, 연관관계의 주인"
tags: [Spring]
comments: true
---

## 단방향과 양방향?

기본적인 쿼리 사용시엔 고민하지 않을 내용이다. JPA를 사용하게 되면 객체 중심 모델링으로 되므로 객체 간 관계에 대해 고민해야 한다.

**테이블과 객체의 차이**

- 테이블은 외래키(FK)를 사용하여 양방향 쿼리가 가능. 조인을 통해 쿼리 조회가 가능하다.
- 객체의 경우 참조용 필드를 가진 객체만 연관된 객체를 조회할 수 있게 된다.

연관관계 매핑을 이해하기 위해선 아래 단어들에 대해 정리할 필요가 있다.

- 방향: 양방향과 단방향이 있음. 한쪽만 참조용 필드만 있다면 단방향, 서로의 참조용 필드가 있다면 양방향. 이는 객체관계에서만 해당되며, 테이블 관계는 항상 양방향.
- 다중성: 1:1(일대일), 1:N(일대다), N:1(다대일), N:M(다대다) 이 있음. 여러 이벤트는 하나의 상점에 있으므로 이는 다대일이다.
- 연관관계 주인: 객체를 양방향으로 만들면 연관관계의 주인이 필요.

## 연관관계 차이

객체에서는 기본적으로 단방향 관계가 맺어짐.

```java
class Event {
    Store store;
}

class Store{

}

```

Event는 Store 를 알 수 있지만, Store는 Event를 알 수 없음.

테이블에서의 연관관계

```sql
select *
from event e
left join store s on e.store_id = s.id

select *
from store s
left join event e on s.id = e.store_id

```

store_id 외래 키 하나로 양방향 조인.  
객체는 언제나 단방향 연관관계임.  
서로 다른 단방향 2개를 이용해 양방향처럼 걸어줄 수 있음.

**객체는 언제나 단방향 관계만 맺을 수 있음.**

```java
class Event {
    Store store;
}

class Store{
    Event event;
}

```

- 객체
    - 참조로 연관관계를 맺음.
    - 참조를 사용하므로 단방향 연관관계.
    - 단방향 연관관계 2개로 양방향 연관관계를 만듦.
- 테이블
    - 외래키로 연관관계를 맺음.
    - 외래키를 사용하므로 양방향 연관관계.

## JPA 에서의 객체 관계 매핑

### @ManyToOne

- 다대일(n:1)의 관계.
- 이벤트가 N, 상점이 1의 관계.
- 이벤트가 N이기 때문에 외래키 관리. (DB를 생각해보자.)

단방향 처리.

```java
@Entity
public class Event{
    @Id
    private Long Id;

    private String eventName;

    @ManyToOne
    @JoinColumn(name = "store_id")
    private Store store;
}

@Entity
public class Store{
    @Id
    private Long Id;

    private String storeName;
    ...
}

```

- 단방향 처리이며, Event에 @ManyToOne으로 Store 객체를 참조하도록 함.
- Event 객체의 store 와 event 테이블의 store_id를 매핑 -> '연관관계 매핑'이라고 함.
- 하지만 Store엔 별다른 어노테이션을 걸지 않음.

### @OneToMany

- 일대다(1:N)의 관계.
- 양방향 처리 때 사용.

양방향 처리

```java
@Entity
public class Event{
    @Id
    private Long Id;

    private String eventName;

    @ManyToOne
    @JoinColumn(name = "store_id")
    private Store store;
}

@Entity
public class Store{
    @Id
    private Long Id;

    private String storeName;

    @OneToMany(mappedBy = "store") //연관 관계 주인 지정 (속성값은 Event.store)
    List<Event> events = new ArrayList<>();
    ...
}

```

- 양방향 처리시엔 1쪽에 @OneToMany 어노테이션을 걸어줌.
- 양방향이므로 연관관계 주인을 mappedBy 로 지정.
- mappedBy 값은 대상 변수명을 따라 지정. 위에서는 Event 객체의 store 라는 이름의 변수이므로 store로 지정.

### 연관관계의 주인

테이블(DB)의 경우 외래키 하나로 양방향 연관관계가 가능.  
객체는 하나의 참조(주소)를 통해 단방향 연관관계가 가능함. 단방향 일때는 이 참조로 외래키를 관리함.  
하지만 양방향 연관관계를 맺으면 두개의 참조가 생기게 됨. (한 객체당 하나의 참조가 생기므로.)  
참조는 두개인데 외래키는 하나이므로 이 외래키를 관리할 객체가 있어야 함.  
이를 연관관계의 주인이라고 함.  
연관관계의 주인을 정하는것 = 외래키 관리자를 선택하는 것.


연관관계의 주인  

- 데이터베이스 연관관계와 매핑
- 외래키 관리 (등록, 수정, 삭제)
- mappedBy 속성 사용하지 않음.

주인이 아닌 쪽

- 읽기만 가능
- mappedBy를 통해 연관관계 주인 지정.

연관관계의 주인은 테이블에 외래키가 있는 곳의 객체로 지정해주어야 한다.  
DB에서 일대다, 다대일 관계에서 외래키는 항상 다 쪽에 있음. 그렇기에 @ManyToOne(다대일)은 mappedBy 속성을 가지지 않는다.  

### 연관관계 저장

연관관계 저장시에 외래키 주인이 아닌 곳에서 입력된 값은 코드가 무시됨

```java
// getEvents 만 가능할 뿐, add 는 무시된다.
store1.getEvents().add(event1);  //무시됨
store1.getEvents().add(event2);  //무시됨

```

반면 연관관계 주인쪽에서 입력된 값을 엔티티 매니저가 사용하여 외래키 관리.

```java
event1.setStore(store1);
event2.setStore(store2);

```

### 연관관계 주의점

그렇다면 외래키의 주인이 아닌 곳은 값을 따로 세팅하지 않아도 되는가?  
순수 객체까지도 고려한다면 양방향에 모두 값을 넣어주는게 안전하다.

```java
public class Test1{
  public void test1() {
      Store store1 = new Store("store1", "스토어1");
      Event event1 = new Event("event1", "이벤트1");
      Event event2 = new Event("event2", "이벤트2");
      event1.setStore(store1); // 연관관계설정 event1 -> store1
      event2.setStore(store1); // 연관관계설정 event2 -> store1
      List<Event> events = store1.getEvents();
      System.out.println("events.size = " + events.size());
  }
}
//결과: events.size = 0

public class Test2{
  public void test2() {
    Store store1 = new Store("store1", "스토어1");
    Event event1 = new Event("event1", "이벤트1");
    Event event2 = new Event("event2", "이벤트2");

    event1.setStore(store1); //객체 + DB에 넣어주는 값
    store1.getEvents().add(event1); //객체에 넣어주는 값

    event2.setStore(store1);
    store1.getEvents().add(event2);

    List<Event> events = store1.getEvents();
    System.out.println("events.size = " + events.size());
  }
}
//결과: events.size = 2

```

set 메서드를 활용하면 좀 더 안전하게 코드 작성 가능.

```java
public class Event{
  @Id
  private Long Id;

  private String eventName;

  @ManyToOne
  @JoinColumn(name = "store_id")
  private Store store;

  public void setStore(Store store){
      this.store = store;
      store.getEvents().add(this);
  }
}

```

이렇게 해주면 setter 하나로 동시에 처리 가능.  
이 때, 연관관계 변경 시 관계 삭제 메소드도 넣어줘야 한다.  
만약 Store1에서 Store2로 변경 시 아래와 같이 작성할 수 있음.

```java
event1.setStore(store1);
event1.setStore(store2);    //1에서 2로 변경
Event findEvent = store1.getEvents(); // event1이 조회 됨.

```

event1 은 store 변경이 되었지만, store1에서 event1과의 관계는 끊어지지 않았음. (List에서 삭제되지 않음.)  
그래서 setter에 기존 관계를 삭제하는 로직까지 넣어주어야 함.

```java
  public void setStore(Store store){
      if(this.store != null){
        this.store.getEvents().remove(this);
      }
      this.store = store;
      store.getEvents().add(this);
  }

```