---
layout: post
title: "Projection 정리"
tags: [Spring, JPA, QueryDSL]
comments: true
---
Projection은 QueryDSL을 통해 데이터를 조회해올 때, 엔티티와 다른 타입을 반환해야할 때 사용한다.  
Projection 사용법은 아래 4가지 방법이 존재함.

* Projection.bean
    * setter 기반.
    * setter가 안열려있으면 사용할 수 없다.
* Projection.fields
    * field에 값을 직접 주입.
    * type 이 다르면 매칭되지 않고, 런타임 시점에 에러 확인 가능.
* Projection.constructor
    * 해당 클래스와 클래스 안에 있는 필드값들을 넘겨주면 매핑하여 해당 객체로 반환?
* @QueryProjection
    * 해당 클래스 안에 직접 생성자를 만들면 QType 으로 만들어 주고, new 생성을 통해 객체를 만든다

이중 제일 추천하는 방법은 @QueryProjection 이다.

## @QueryProjection

@QueryProjection은 생성자를 통해 DTO로 데이터를 조회해오는 방식이다. 정확히는 QType의 DTO를 만들어서 사용하는 것이다.  
이 방식의 좋은점은 new QDTO 로 클래스를 생성하여 사용하기 떄문에 컴파일 시점에서 에러를 잡을 수 있다는 것이다.

하지만 DTO는 @QueryProjection 애노테이션이 달리는 순간 Querydsl에 의존성을 가지게 되기 때문에 구조적으로는 조금 고민해 볼 필요가 있다.  
사용법은 우선 DTO 클래스의 생성자에 @QueryProjection 애노테이션을 달아준다.
```java
public class TestDTO {
    private String name;
    private int age;
    
    @QueryProjection
    public TestDTO(String name, int age){
        this.name = name;
        this.age = age;
    }
}
```
실제 사용법은 아래와 같다
```java
List<TestDTO> result = queryFactory
        .select(new QTestDTO(test.name, test.age))
        .from(member)
        .fetch();
```

## 그 외
그 외의 나머지 방법들은 간단하게 사용법만 작성 하도록 하겠다.

### Projection.bean
setter 메소드를 이용하여 조회하는 방법이다. setter를 이용하기 때문에 setter 접근이 되어야 한다.  
단순 조회 DTO 라면 setter 로 값을 변경하는데 크게 문제는 없지만, 영속화 데이터를 변경하는 객체라면 setter 오픈이 문제 될 수 있다.
```java
public void testProjectionBean(){
    List<TestDTO> result = queryFactory
        .select(Projection.bean(
                TestDTO.class,
                test.name, 
                test.age
        ))
        .from(member)
        .fetch();
}
```

### Projection.field
fielder 에 값을 직접 입력하는 방식이다. 사용법은 Projection.Bean과 거의 유사하며, 반환 타입이 다를 경우 매칭 되지 않는다.  
또한 컴파일 시점에서 에러를 발견할 수 없고 런타임 시점에서 에러 확인이 가능하다.
```java
public void testProjectionBean(){
    List<TestDTO> result = queryFactory
        .select(Projection.field(
                TestDTO.class,
                test.name.as("username"), 
                test.age
        ))
        .from(member)
        .fetch();
}
```
필드이름이 매칭되지 않을 경우 ```.as()``` 를 통해 alias를 줄 수 있다.

### Projection.constructor
생성자를 이용하여 조회 결과를 반환하는 방식. 생성자 기반이기 때문에 객체 불변으로 사용 가능하지만, 매핑 시 문제가 발생할 수 있다.  
값을 넘길 때, 생성자와 필드의 순서를 맞춰줘야 한다. 필드가 많은 DTO 일 수록 이런 실수가 일어날 가능성이 높다.
```java
public void testProjectionBean(){
    List<TestDTO> result = queryFactory
        .select(Projection.constructor(
                TestDTO.class,
                test.name.as("username"), 
                test.age
        ))
        .from(member)
        .fetch();
}
```
또한 위 두개 경우 처럼 런타임 시점에 에러확인이 가능하다. 