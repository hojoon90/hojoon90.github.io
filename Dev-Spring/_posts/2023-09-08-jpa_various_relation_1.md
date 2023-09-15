---
layout: post
title: "다양한 연관관계 매핑 1"
tags: [Spring, JPA, 다양한_연관관계]
comments: true
---

연관관계 정리 전 아래 내용에 대한 정리 필요

* 단방향/양방향
    * 객체간 관계를 나타내는 것.
    * 객체는 참조용 필드를 갖는 객체만 연관 객체 조회 가능.
    * 한쪽만 참조하면 단방향, 양쪽 다 참조하면 양방향
* 연관관계 주인
    * 양방향 관계일 시 정의 필요 (서로가 참조하므로)
    * 연관관계 주인이 아닐 경우 'mappedBy'속성 사용 후 연관관계 주인의 필드 이름을 값으로 사용.

## 다대일 (N:1)
객체 양방향에서 연관관계 주인은 항상 다(N)쪽임.

### 다대일 단방향
```java
@Entity
public class Member {
    @Id @GeneratedValue
    @Column (name = "MEMBER_ID")
    private Long id;
    
    private String username;
    
    @ManyToOne
    @JoinColumn (name = "TEAM_ID")
    private Team team; 
    
    //Getter, Setter ...
}

@Entity
public class Team {
    @Id @GeneratedValue 
    @Column (name = "TEAM ID") 
    private Long id;
    
    private String name; 
    
    //Getter, Setter ...

}
```
* 회원은 team 으로 팀 엔티티 참조 가능.
* 팀은 회원을 참조할 수 없음.
* @JoinColumn (name = "TEAM_ID")로 외래키 매핑.
* 이로 인해 team 필드로 회원테이블(MEMBER)의 TEAM_ID 외래키 관리.

### 다대일 양방향
```java
@Entity
public class Member {
    
    @Id @GeneratedValue
    @Column(name = "MEMBER_ID")
    private Long id;

    private String username;

    @ManyToOne
    @JoinColumn(name = "TEAM_ID")
    private Team team;

    public void setTeam(Team team) { // 편의 메소드
        this.team = team;
        // 무한 루프에 빠지지 않도록 체크
        if (!team.getMembers().contains(this)) {
            team.getMembers().add(this);
        }
    }
}

@Entity
public class Team {
    
    @Id @GeneratedValue 
    @Column(name = "TEAM_ID ")
    private Long id;
    
    private String name;
    
    @OneToMany(mappedBy = "team")
    private List<Member> members = new ArrayList<Member>();
    
    public void addMember (Member member) {
        this.members.add(member);
        if (member.getTeam() != this){ //무한 루프에 빠지지 않도록 체크
            member.setTeam(this);
        }
    }
}
```
* 양방향은 외래키가 있는쪽이 연관관계 주인
    * 일대다, 다대일은 항상 다(N)에 외래키가 있음.
    * 여기서는 Member 객체의 team 필드가 주인.
* 양방향은 서로를 항상 참조
    * 위의 addMember, setTeam등의 편의 메소드를 양쪽에 모두 만들어 참조되도록 한다.
    * 무한루프 주의. 위 코드의 경우 체크가 없으면 무한으로 팀에 멤버가 추가됨.

## 일대다 (1:N)
다대일 관계의 반대 방향. 엔티티가 한개이상일 수 있으므로 Collection, List, Set, Map 등을 사용.

### 일대다 단방향
하나가 여러 엔티티를 참조. (하나의 팀이 여러 회원을 참조)

```java
@Entity
public class Team {
    @Id @GeneratedValue 
    @Column(name = "TEAM_ID")
    private Long id; 
    
    private String name;
    
    @OneToMany
    @JoinColumn(name = "TEAM_ID") // MEMBER 테이블의 TEAM_ID (FK) 
    private List<Member> members = new ArrayList<Member>();
    
    //Getter, Setter ...
}

@Entity
public class Member {
    
    @Id @GeneratedValue
    @Column(name = "MEMBER_ID")
    private Long id;
    
    private String username;
    
    //Getter, Setter ...
}
```
* 형태가 조금 특이한데, members가 TEAM_ID 외래키를 관리.
* 일대다 관계에서 외래키는 항상 다(N)쪽에 있지만, Member 객체는 참조할 수 있는 필드가 없음.
* 대신 Team 객체엔 members라는 참조 필드가 있으므로, 여기서 외래키를 관리.

### 일대다 단방향의 단점
매핑한 객체가 관리하는 외래키가 다른테이블에 있으므로 별도의 업데이트 쿼리가 필요하다.

```java
class Test{
    public void testSave(){
        Member member1 = new Member ("member1"); 
        Member member2 = new Member ("member2");
        
        Team team1 = new Team("team"); 
        team1.getMembers().add(member1); 
        team1.getMembers().add(member2);
        em.persist(member1); //INSERT-member1
        em.persist(member2); //INSERT-member2
        em.persist(team1); //INSERT-team1, UPDATE-member1. fk, //UPDATE-member2.fk
        
        transaction.commit();
    }
}
```
위 메소드의 실행결과는 아래와 같이 동작하게 됨.
```sql
insert into Member (MEMBER_ID, username) values (null, ?);
insert into Member (MEMBER_ID, username) values (null, ?);
insert into Team (TEAM_ID, name) values (null, ?);
update Member set TEAM_ID = ? where MEMBER_ID = ?;
update Member set TEAM_ID = ? where MEMBER_ID = ?;
```
* Member 객체는 Team 엔티티를 알지 못함.
    * 연관관계는 Team.members에서 관리하기 때문.
* Member 저장 시엔 TEAM_ID를 알지 못하기 때문에 insert 때 TEAM_ID 제외.
    * team 엔티티가 아직 DB에 저장되지 않아 TEAM_ID를 모름.
* team 엔티티를 저장할 때, team.members안에 있는 참조값을 확인하여 해당 member 객체에 TEAM_ID 업데이트.

### 일대다 양방향 관계
일대다 양방향 매핑은 존재하지 않음.  
일대다, 다대일 관계에서 연관관계 주인은 무조건 다(N)쪽에 있다. 즉, @ManyToOne 이 연관관계의 주인이 될 수 밖에 없다.  
아예 못 만드는건 아니고, 일대다 단방향 매핑 반대 엔티티에 다대일 단방향 매핑을 읽기전용으로 만들어주면 된다.  
하지만 이렇게 어렵게 쓰느니 다대일 양방향 매핑으로 쓰는게 좋다. 


