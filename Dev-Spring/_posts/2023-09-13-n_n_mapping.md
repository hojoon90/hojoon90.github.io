---
layout: post
title: "다대다 매핑 한계와 극복"
tags: [Spring, JPA, 다양한_연관관계]
comments: true
---

## 다대다 매핑
* 실무에서는 연결 테이블에 위와 같은 상품아이디와 멤버아이디만 들어가는 것이 아님
    * 좀 더 여러 컬럼들이 추가된다.
* 이렇게 컬럼이 추가되면 @ManyToMany는 사용할 수 없음.
    * 멤버 엔티티나 상품 엔티티는 추가한 컬럼 매핑이 불가능.
* 그래서 연결 엔티티를 만들어 일대다 다대일 관계로 푼다.

```java
//회원
@Entity
public class Member {
    
    @Id @Column (name = "MEMBER_ID") 
    private String id;
    
    //역방향
    @OneToMany(mappedBy = "member")
    private List<MemberProduct> memberProducts;
    
}

//상품
@Entity
public class Product {
    
    @Id @Column (name = "PRODUCT_ID") 
    private String id;
    
    private String name;
    
}

//회원상품 엔티티
@Entity
@IdClass(MemberProductId.class) 
public class MemberProduct {
  
    @Id
    @ManyToOne
    @JoinColumn(name = "MEMBER_ID")
    private Member member; //MemberProductId.member 와 연결
  
    @Id
    @ManyToOne
    @JoinColumn(name = "PRODUCT_ID")
    private Product product; //MemberProductId.product 와 연결
  
    private int orderAmount;
  
}

//회원상품 식별자 클래스
public class MemberProductId implements Serializable {
  private String member; //MemberProduct.member 와 연결
  private String product; //MemberProduct.product 와 연결
  // hashCode and equals
  @Override
  public boolean equals (Object o) {...}
  @Override
  public int hashCode() {...}
}


```
1. 회원과 회원상품을 양방향 관계로 만듦. 이때 연관관계 주인은 회원상품
2. 상품 엔티티의 경우 회원상품 엔티티로 객체 탐색이 필요하지 않다고 판단하여 연관관계를 만들지 않음.
    1. 상품을 기준으로 회원을 조회할 필요가 없어서..?
3. 회원상품 엔티티의 경우 기본 키(@Id) + 외래 키(@JoinColumn)를 한번에 매핑함.
4. @IdClass 를 사용하여 복합 기본키를 매핑

### 복합 기본키
* 회원 상품 엔티티는 MEMBER_ID와 PRODUCT_ID가 기본 키로 이루어진 복합 기본키(복합 키).
* JPA는 복합키 사용시에 별도의 식별자 클래스 생성.
* 그 후 엔티티에 @IdClass를 사용하여 식별자 클래스 지정.
* 식별자 클래스는 아래 특징들이 있음
    * Serializable 구현
    * equals, hashCode 메소드 구현
    * 기본 생성자
    * 클래스는 public 이어야 함.
    * @IdClass 대신 @EmbeddedId 도 사용 가능
* 부모 테이블의 기본 키(MEMBER_ID, PRODUCT_ID)를 받아와 자신의 기본키 + 외래키로 사용하는 것을 '식별 관계'라고 함.
* 위의 두 기본 키(MEMBER_ID, PRODUCT_ID)는 Member와 Product 관계를 위한 외래키로 사용. 그리고 그 둘을 묶어서 MemberProduct 본인의 복합 기본키로 사용.

### 저장 방식
```java
public class Test4{
    public void save () {
        // 회원저장
        Member member1 = new Member();
        member1.setId("member1");
        member1.setUsername("회원1");
        em.persist(member1);
        
        // 상품저장
        Product product = new Product();
        product.setId("productA");
        productA.setName("상품1");
        em.persist(productA);
        
        // 회원상품 저장
        MemberProduct memberProduct = new MemberProduct();
        memberProduct.setMember(member1); //주문회원 - 연관관계 설정 
        memberProduct.setProduct(productA); //주문상품 - 연관관계 설정
        memberProduct.setOrderAmount(2); //주문수량
        em.persist(memberProduct);
    }
}
```
* 회원상품 엔티티를 만들면서 연관된 회원과 상품을 설정.
* 회원상품 엔티티는 저장 시 연관된 회원의 식별자와 상품의 식별자를 가져와 자신의 기본 키 값으로 사용.

### 조회 방식
```java
class Test5 {
    public void find() {
    // 기본 키값 생성
    MemberProductId memberProductId = new MemberProductId();
    memberProductId.setMember("member1");
    memberProductId.setProduct("productA");

    MemberProduct memberProduct = em.find(MemberProduct.class, memberProductId);
    
    Member member = memberProduct.getMember();
    Product product = memberProduct.getProduct();
    
    System.out.printIn("member = "+member.getUsername()); 
    System.out.printIn("product = "+product.getName()); 
    System.out.println("orderAmount = "+ memberProduct.getOrderAmount());
}
}
```
* 식별자 클래스인 MemberProductId 객체에 조회할 키를 세팅해줌.

### 중간 결론
* 복합키를 사용하는 것은 복잡함.
* ORM에서 처리할게 많아진다. 아래와 같음.
    * 식별자 클래스 생성
    * @IdClass 혹은 @EmbeddedId 사용
    * 식별자 클래스에 equals, hashCode 사용.
* 이를 해결하는 법은 새로운 기본키를 사용하는 것.

## 다대다 매핑: 새로운 방법
* 데이터베이스에서 자동으로 생성해주는 대리키를 Long 값(ID)으로 사용.
* 장점
    * 간편하다
    * 거의 영구적으로 쓸수 있다.
    * 비즈니스에 의존하지 않는다.

기존 MemberProduct 테이블에 새로운 ID 를 하나 생성. (테이블도 MemberProduct -> Order로 변경)
```java
@Entity
public class Order {
    @Id @GeneratedValue
    @Column (name = "ORDER _ID") 
    private Long Id;
    
    @ManyToOne
    @JoinColumn (name = "MEMBER ID")
    private Member member;
    
    @ManyToOne
    @JoinColumn (name = "'PRODUCT ID")
    private Product product; 
    
    private int orderAmount;
}
```
* Order 테이블에 ID 를 하나 새로 생성
* member 필드와 product 필드는 외래키로만 사용.
* 이로 인해 매핑이 단순해지고 이해하기 쉬워짐.
* 회원과 상품 엔티티는 변경사항이 없음.

```java
class Test6{
    public void save() {
        Member member1 = new Member();
        member1.setId("member1");
        member1.setUsername("회원1");
        em.persist(member1);
    
        Product productA = new Product();
        productA.setId("productA");
        productA.setName("상품1");
    
        em.persist(productA);
    
        // 주문저장
        Order order = new Order();
        order.setMember(member1);// 주문회원- 연관관계설정 
        order.setProduct(productA);//주문상품-연관관계설정 
        order.setOrderAmount(2);// 주문 수량
        em.persist(order);
    }
    
    public void find(){
        Long orderId = 1L;
        Order order = em.find(Order.class, orderId);
        Member member = order.getMember(); 
        Product product = order.getProduct();
        System.out.println("member = " + member.getUsername()); 
        System.out.printIn("product = " + product.getName()); 
        System.out.printin("orderAmount = " + order.getOrderAmount());
    }
}
```
* 주문 저장과 조회 로직.

## 정리
* 다대다를 일대다, 다대일로 풀어내기 위해선 연결테이블 생성 시 식별자를 어떻게 구성할 지 선택
    * 식별 관계: 받아온 식별자를 기본키 + 외래키로 사용
    * 비식별 관계: 받아온 식별자는 외래키로만 사용. 기본키는 새로운 키를 추가.
* 객체입장에서는 비식별관계를 사용하는 것이 식별자 클래스를 생성하지 않아도 되므로 단순하게 사용 가능.
* 그렇기에 비식별관계를 추천.