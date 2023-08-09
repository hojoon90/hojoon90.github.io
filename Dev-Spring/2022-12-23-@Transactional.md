---
layout: post
title: "@Transactional 정리"
tags: [Spring]
comments: true
---

## @Transactional 이란
@Transactional은 해당 어노테이션이 명시된 메소드를 하나의 트랜잭션으로 처리함. 해당 어노테이션이 선언된 클래스와 그 하위 클래스에 모두 적용된다. 
그 위 부모 클래스로는 적용되지 않는다.

## ACID
* Atomicity(원자성): 트랜잭션과 관련된 작업들이 중단없이 처리되는 것을 보장하는 것이다. 원자성은 중간단계의 실패 없이 처리되도록 하는 것이다.
* Consistency(일관성): 트랜잭션이 실행을 성공적으로 완료하면 언제나 일관성 있는 데이터베이스 상태로 유지하는 것을 의미한다. 즉 트랜잭션이 일어난 후 DB는
DB의 제약이나 규칙을 만족해야 한다.
* Isolation(독립성): 트랜잭션을 수행 시 다른 트랜잭션이 간섭하지 못하도록 보장하는 것을 의미한다. 트랜잭션 밖에 있는 어떤 연산도 트랜잭션 상의 데이터를 볼 수 없다.
* Durability(지속성): 성공적으로 수행된 트랜잭션은 영원히 반영되어야 한다. 모든 트랜잭션은 로그로 남고 시스템 장애 발생 시, 이전 상태로 되돌릴 수 있다.

## AOP
AOP는 '관점 지향 프로그래밍' 이라고 불림.  
우리가 로직들에 대해 핵심적인 관점과 부가적인 관점을 나누고, 이 관점들을 공통적인 로직이나 기능으로 묶는 것.  
각 클래스에 반복적으로 사용되는 코드들을 모아 모듈화 하여 분리하여 사용하겠다는 것이 AOP 의 취지.

* Aspect: 흩어진 관심사. 중복되는 코드 및 로직들.
* Target: Aspect를 적용하는 곳.
* Advice: 실제 부가기능을 담은 구현체
* JointPoint : Advice가 적용될 위치. 메서드 진입 지점, 생성자 호출 시점, 필드에서 값을 꺼내올 때 등 적용가능
* PointCut : JointPoint의 상세한 스펙을 정의한 것.

## Proxy
Spring은 자체적으로 Proxy 기반의 AOP를 지원한다. 따락서 @Transactional 역시 Proxy 패턴이 이용된다.  
해당 어노테이션이 명시되어있는 클래스나 메소드의 경우 실제 동작 시 Proxy 패턴의 별도 클래스가 동작하여 트랜잭션 작업을 수행.  
Spring에서 Proxy 객체를 생성하는 방식은 대상 객체가 인터페이스가 있냐 없냐에 따라 나뉘게 됨.  
인터페이스가 있을 시엔 Dynamic Proxy, 없을 시엔 CGLIB Proxy를 사용함.

* Dynamic Proxy
  * 대상 객체가 인터페이스를 구현했을 때 사용.
  * 대상 객체와 동일한 인터페이스를 구현.
  * 클라이언트는 이 인터페이스를 통해 필요한 메서드를 호출
  * 인터페이스에 정의되지 않은 메서드에 대해서는 AOP가 적용되지 않음.
* CGLIB Proxy
  * 대상 객체가 인터페이스를 별도로 구현하지 않았을 때 사용.
  * 대상 클래스를 상속 받아서 프록시를 구현.
  * 대상 클래스가 final인 경우엔 프록시 생성이 안되며, 메서드가 final일 경우 AOP가 적용되지 않음.
  * 현재 Spring Boot의 기본 AOP 전략으로 사용중.

## 동작 방식
1. @Transactional 어노테이션이 적용된 곳에 프록시 패턴을 이용해 객체를 하나 만들어줌.
2. 해당 메소드의 앞, 뒤에 트랜잭션 로직이 들어간다. (이 로직으로 인해 commit, rollback이 수행됨.)
3. 실제 동작에따라 commit 혹은 rollback이 수행됨.

```java
public class BoardService{
    @Transactional
    public void postArticle(AllBoard allBoard){
         collectPersister.postArticle(allBoard);
    }
}

```
위 코드로 나타내보면 다음과 같을 것이다.
```java
public class BoardService{
    public void postArticle(AllBoard allBoard){
        Connection conn = dataSource.getConnection();
        try(conn){
            conn.setAutoCommit(false);
            //실제 작성한 로직
            collectPersister.postArticle(allBoard);
            
            conn.commit();
        } catch (SQLException e){
            conn.rollback();
        }
    }
}

```
Proxy의 경우 트랜잭션을 관리하긴 하지만 실제 트랜잭션 상태를 Proxy가 결정할 수는 없음.  
그렇기 때문에 Proxy는 Transaction Manager에게 해당 결정을 위임하여 트랜잭션 처리를 함.

참고 자료
* https://blogshine.tistory.com/291
* https://private-space.tistory.com/98?utm_source=pocket_reader
* https://tecoble.techcourse.co.kr/post/2021-06-25-aop-transaction/?utm_source=pocket_reader
* 웹개발자를 위한 Spring 4.0 프로그래밍