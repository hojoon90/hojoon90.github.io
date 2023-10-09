---
layout: post
title: "다양한 연관관계 매핑 2"
tags: [Spring, JPA, 다양한_연관관계]
comments: true
---

## 일대일 (1:1)
일대일 관계는 양쪽이 서로 하나의 관계만 가지는 것. 사용자와 사물함과 같은 관계.  
(사용자는 하나의 사물함만 사용. 사물함도 한명의 사용자만 사용.)
* 일대일은 그 반대도 일대일임.
* 주테이블, 대상테이블 어느곳에서든 외래키를 가질 수 있음.
* 그렇기에 주테이블에 외래키를 두는법, 대상테이블에 외래키를 두는 법 두가지로 나뉨.

### 주 테이블에 외래키
* 주 객체가 대상 객체를 참조하는 것처럼 주 테이블에서 대상 테이블을 참조함.
* 주 테이블만 확인해도 연관관계 파악이 쉽다.
* JPA에서도 주 테이블에 외래키가 있으면 편리

### 주테이블 단방향
주 테이블은 Member, 대상 테이블은 Locker 이다.
```java
@Entity
public class Member {
    @Id @GeneratedValue
    @Column(name = "MEMBER_ID")
    private Long id;

    private String username;

    @OneToOne
    @JoinColumn(name = "LOCKER_ID")
    private Locker locker;
}

@Entity
public class Locker {
    @Id @GeneratedValue
    @Column (name = "LOCKER_ID")
    private Long id;
    
    private String name;

}
```
* 일대일 관계에선 @OneToOne 어노테이션 사용
* 주 테이블에 Locker 객체 필드가 있고, @JoinColumn을 이용해 외래키가 LOCKER_ID 라는 것을 명시
* 즉, Member(주테이블)에서 Locker(대상테이블)를 관리하고 있음.
* 다대일 단방향과 거의 비슷

### 주테이블 양방향
```java
@Entity
public class Member {
    @Id
    @GeneratedValue
    @Column(name = "MEMBER_ID")
    private Long id;
    private String username;
    
    @OneToOne
    @JoinColumn(name = "LOCKER_ID")
    private Locker locker;
}

@Entity
public class Locker {
    @Id
    @GeneratedValue
    @Column(name = "LOCKER_ID")
    private Long id;
    private String name;
    
    @OneToOne(mappedBy = "locker")
    private Member member;
}
```
* 양방향이기 때문에 연관관계 주인이 필요
* MEMBER 테이블이 외래키를 갖고 있으므로 Member.locker 가 연관관계 주인.
* Locker 에서는 member 필드에 mappedBy를 통해 연관관계 주인이 아니라고 설정.

### 대상 테이블에 외래키
* 대상 테이블에 외래키를 두는 방법. 전통적인 방법.
* 테이블 관계를 일대일에서 일대다로 변경할 때 구조 유지 가능.

### 대상 테이블 단방향
* 일대일 관계 중 대상 테이블에 외래키가 있는 단방향은 JPA 에서 지원하지 않음.
* 단방향 관계를 Locker에서 Member 방향으로 수정(Locker에서 @OneToOne 사용).
    * 이는 주 테이블을 Locker로 변경한다는 이야기와 동일하다. 현재 주 테이블은 Member이다.
* 또는 양방향 관계를 만든 후 연관관계 주인을 Locker 로 설정.

### 대상 테이블 양방향
```java
@Entity
public class Member {
    @Id @GeneratedValue
    @Column(name ="MEMBER_ID")
    private Long id;
    
    private String username;
    
    @OneToOne(mappedBy = "member")
    private Locker locker;
}
@Entity
public class Locker {
    @Id @GeneratedValue
    @Column(name ="LOCKER_ID")
    private Long id;
    
    private String name;
    
    @OneToOne
    @JoinColumn(name = "MEMBER_ID")
    private Member member;
}
```
* 주 엔티티인 Member 대신에 대상 엔티티인 Locker 를 연관관계의 주인으로 만듦
* Member 엔티티엔 mappedBy가 설정 된 것을 볼 수 있다.

## 다대다 (N:N)
* 연관관계DB 에서는 테이블 2개로 다대다 관계 표현이 안된다.
* 이 둘을 연결해주는 연결 테이블이 있어야 한다.
    * 다대다를 일대다, 다대일 관계로 풀어낸다.
* 반면 객체는 객체 2개로 다대다를 만들 수 있다.
* 두 필드를 모두 컬렉션 객체를 사용해 만들면 된다.

### 단방향
다대다 단방향 관계인 회원과 상품 엔티티를 만들어보자.
```java
@Entity
public class Member {
    @Id
    @Column(name = "MEMBER ID")
    private String id;
    
    private String username;
    
    @ManyToMany
    @JoinTable(
            name = "MEMBER_PRODUCT", 
            joincolumns = @JoinColumn(name = "MEMBER_ID"), 
            inverseJoinColumns = @JoinColumn(name = "PRODUCT_ID")
    )
    private List<Product> products = new ArrayList<Product>();
}

@Entity
public class Product {
    @Id
    @Column(name = "PRODUCT_ID")
    private String id;
 
    private String name;
}

```
* 여기서 MEMBER_PRODUCT는 MEMBER 테이블과 PRODUCT 테이블을 연결해주는 연결 테이블.
    * 즉 MEMBER -> MEMBER_PRODUCT -> PRODUCT 방향
* @ManyToMany 와 @JoinTable을 이용하여 연결테이블을 바로 매핑.
* 그렇기에 MEMBER_PRODUCT 엔티티가 필요 없음.
* @JoinTable 속성은 아래와 같음.
    * name: 연결 테이블을 지정.
    * joinColumns: 연결할 방향인 회원과 매핑할 조인 컬럼 정보 지정.
    * inverseJoinColumn: 반대 방향인 상품과 매핑할 조인 컬럼 정보 지정.
* MEMBER_PRODUCT 는 단순 연결 테이블일 뿐이므로 @ManyToMany 로 매핑할 경우 이 테이블은 신경쓰지 않아도 됨.

```java
public class test1{
    public void save() {
        Product productA = new Product();
        productA.setId("productA");
        productA.setName("상품A");
        em.persist(productA);
        
        Member member1 = new Member();
        member1.setId("member1");
        member1.setUsername("회원1");
        member1.getProducts().add(productA); //연관관계 설정
        em.persist(member1);
    }
}
```
* 연관관계 주인인 회원 테이블과 상품 테이블의 연관관계를 설정하여 저장하는 코드.
* 이로 인해 회원 저장 시 상품도 저장됨.
```sql
INSERT INTO PRODUCT...
INSERT INTO MEMBER...
INSERT INTO MEMBER_PRODUCT
```

아래는 조회 시 로직이다.
```java
class Test2 {
    public void find() {
        Member member = em.find(Member.class, "member1");
        List<Product> products = member.getProducts(); //객체그래프탐색 
        for (Product product : products) {
            System.out.printin("product.name = " + product.getName());
        }
    }
}
```
* 조회 시 저장된 상품 1이 조회 됨.
* member.getProduct() 메소드 실행 시 아래 쿼리가 실행 됨.
```sql
SELECT * FROM MEMBER_PRODUCT MP
INNER JOIN PRODUCT P ON MP.PRODUCTID=P.PRODUCT_ID
WHERE MP.MEMBER ID=?
```
* MEMBER_PRODUCT와 상품 테이블을 조인하여 연관 상품을 조회함.

### 양방향

* 다대다 매핑에선 역방향에도 @ManyToMany 를 사용한다.
* 원하는 곳에 mappedBy를 사용하여 연관관계 주인을 지정.
* 양방향 연관관계는 편의 메소드를 추가하여 관리하는 것이 좋음.

아래와 같이 회원 엔티티에 메소드를 추가
```java
public void addProduct (Product product) {
    products.add(product); //Member 엔티티의 상품 필드에 상품을 추가
    product.getMembers().add(this); //상품 엔티티의 멤버 필드에도 이 객체를 추가.
}
```
* 이렇게 메소드를 만들면 아래와 같이 직관적인 코드작성 가능
    * member.addProduct(product);

양방향 연관관계가 되었으므로 역방향으로 객체 조회 가능
```java
class Test3 {
    public void findInverse() {
        Product product = em.find(Product.class, "productA");
        List<Member> members = product.getMembers();
        for(Member member : members){
            System.out.printIn("member = " + member.getUsername());
        }
    }
}
```
* 상품 객체에서 멤버 리스트를 조회.