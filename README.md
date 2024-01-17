# Querydsl
## JPQL이 제공하는 모든 검색 조건 제공 
```java
member.username.eq("member1") // username = 'member1'
member.username.ne("member1") //username != 'member1'
member.username.eq("member1").not() // username != 'member1'
member.username.isNotNull() //이름이 is not null
member.age.in(10, 20) // age in (10,20)
member.age.notIn(10, 20) // age not in (10, 20)
member.age.between(10,30) //between 10, 30
member.age.goe(30) // age >= 30
member.age.gt(30) // age > 30
member.age.loe(30) // age <= 30
member.age.lt(30) // age < 30
member.username.like("member%") //like 검색 
member.username.contains("member") // like ‘%member%’ 검색
member.username.startsWith("member") //like ‘member%’ 검색
```

## 결과 조회
- `fetch()` : 리스트 조회, 데이터 없으면 빈 리스트 반환 
- `fetchOne()` : 단 건 조회
    - 결과가 없으면 : `null`
    - 결과가 둘 이상이면 : `com.querydsl.core.NonUniqueResultException` 
- `fetchFirst()` : `limit(1).fetchOne()`
- `fetchResults()` : 페이징 정보 포함, total count 쿼리 추가 실행
- `fetchCount()` : count 쿼리로 변경해서 count 수 조회

## 정렬
```java
List<Member> result = queryFactory
             .selectFrom(member)
             .where(member.age.eq(100))
             .orderBy(member.age.desc(), member.username.asc().nullsLast())
             .fetch();
```

- `desc()` , `asc()` : 일반 정렬
- `nullsLast()` , `nullsFirst()` : null 데이터 순서 부여

## 페이징
### **전체 조회 수가 필요하면?**
```java
@Test
public void paging2() {
    QueryResults<Member> queryResults = queryFactory
            .selectFrom(member)
            .orderBy(member.username.desc())
            .offset(1)
            .limit(2)
            .fetchResults();
    assertThat(queryResults.getTotal()).isEqualTo(4);
    assertThat(queryResults.getLimit()).isEqualTo(2);
    assertThat(queryResults.getOffset()).isEqualTo(1);
    assertThat(queryResults.getResults().size()).isEqualTo(2);
}
```
> **주의: count 쿼리가 실행되니 성능상 주의!**   
> 참고: 실무에서 페이징 쿼리를 작성할 때, 데이터를 조회하는 쿼리는 여러 테이블을 조인해야 하지만, count 쿼리 는 조인이 필요 없는 경우도 있다. 그런데 이렇게 자동화된 count 쿼리는 원본 쿼리와 같이 모두 조인을 해버리기 때문에 성능이 안나올 수 있다. count 쿼리에 조인이 필요없는 성능 최적화가 필요하다면, count 전용 쿼리를 별 도로 작성해야 한다.

## 집합
### 집합 함수
```java
/**
  * JPQL
  * select
* COUNT(m),
  *    SUM(m.age),
  *    AVG(m.age),
  *    MAX(m.age),
  *    MIN(m.age)
  * from Member m
  */
//회원수 //나이 합 //평균 나이 //최대 나이 //최소 나이
@Test
public void aggregation() throws Exception {
    List<Tuple> result = queryFactory
            .select(member.count(),
                    member.age.sum(),
                    member.age.avg(),
                    member.age.max(),
                    member.age.min())
            .from(member)
            .fetch();
    Tuple tuple = result.get(0);
    assertThat(tuple.get(member.count())).isEqualTo(4);
    assertThat(tuple.get(member.age.sum())).isEqualTo(100);
    assertThat(tuple.get(member.age.avg())).isEqualTo(25);
    assertThat(tuple.get(member.age.max())).isEqualTo(40);
    assertThat(tuple.get(member.age.min())).isEqualTo(10);
}
```

### GroupBy 사용
```java
/**
* 팀의 이름과 각 팀의 평균 연령을 구해라. */
@Test
 
  public void group() throws Exception {
     List<Tuple> result = queryFactory
             .select(team.name, member.age.avg())
             .from(member)
             .join(member.team, team)
             .groupBy(team.name)
             .fetch();
     Tuple teamA = result.get(0);
     Tuple teamB = result.get(1);
     assertThat(teamA.get(team.name)).isEqualTo("teamA");
     assertThat(teamA.get(member.age.avg())).isEqualTo(15);
     assertThat(teamB.get(team.name)).isEqualTo("teamB");
     assertThat(teamB.get(member.age.avg())).isEqualTo(35);
 }
```

**groupBy(), having() 예시**
```java
.groupBy(item.price)
.having(item.price.gt(1000))
```

## 조인
> **join(조인 대상, 별칭으로 사용할 Q타입)**

### 기본 조인
```java
/**
* 팀 A에 소속된 모든 회원 */
@Test
public void join() throws Exception {
    QMember member = QMember.member;
    QTeam team = QTeam.team;
    List<Member> result = queryFactory
            .selectFrom(member)
            .join(member.team, team)
            .where(team.name.eq("teamA"))
            .fetch();
    assertThat(result)
            .extracting("username")
}
```
- `join()` , `innerJoin()` : 내부 조인(inner join) 
- `leftJoin()` : left 외부 조인(left outer join) 
- `rightJoin()` : rigth 외부 조인(rigth outer join)
- JPQL의 `on` 과 성능 최적화를 위한 `fetch` 조인 제공 다음 on 절에서 설명

### 세타 조인
> 연관관계가 없는 필드로 조인

```java
/**
* 세타 조인(연관관계가 없는 필드로 조인) * 회원의 이름이 팀 이름과 같은 회원 조회 */
 @Test
 public void theta_join() throws Exception {
     em.persist(new Member("teamA"));
     em.persist(new Member("teamB"));

     List<Member> result = queryFactory
             .select(member)
             .from(member, team)
 }
 ```
 - from 절에 여러 엔티티를 선택해서 세타 조인
- 외부조인불가능 다음에설명할조인on을사용하면외부조인가능

### 조인 - on절
- ON절을 활용한 조인(JPA 2.1부터 지원)
    1. 조인 대상 필터링
    2. 연관관계 없는 엔티티 외부 조인


### 1. 조인 대상 필터링
예) 회원과 팀을 조인하면서, 팀 이름이 teamA인 팀만 조인, 회원은 모두 조회
```java
/**
* 예) 회원과 팀을 조인하면서, 팀 이름이 teamA인 팀만 조인, 회원은 모두 조회
* JPQL: SELECT m, t FROM Member m LEFT JOIN m.team t on t.name = 'teamA'
* SQL: SELECT m.*, t.* FROM Member m LEFT JOIN Team t ON m.TEAM_ID=t.id and
 t.name='teamA'
  */
@Test
public void join_on_filtering() throws Exception {
    List<Tuple> result = queryFactory
            .select(member, team)
            .from(member)
            .leftJoin(member.team, team).on(team.name.eq("teamA"))
            .fetch();
    for (Tuple tuple : result) {
        System.out.println("tuple = " + tuple);
    }
}
```
 
**결과**
```
t=[Member(id=3, username=member1, age=10), Team(id=1, name=teamA)]
t=[Member(id=4, username=member2, age=20), Team(id=1, name=teamA)]
t=[Member(id=5, username=member3, age=30), null]
t=[Member(id=6, username=member4, age=40), null]
```

> **참고:** on 절을 활용해 조인 대상을 필터링 할 때, 외부조인이 아니라 내부조인(inner join)을 사용하면,   
> where 절 에서 필터링 하는 것과 기능이 동일하다. 따라서 on 절을 활용한 조인 대상 필터링을 사용할 때,    
> **내부조인 이면 익 숙한 where 절로 해결하고**, 정말 외부조인이 필요한 경우에만 이 기능을 사용하자.

### 2. 연관관계 없는 엔티티 외부 조인
예) 회원의 이름과 팀의 이름이 같은 대상 **외부 조인**
```java
/**
* 2. 연관관계 없는 엔티티 외부 조인
* 예) 회원의 이름과 팀의 이름이 같은 대상 외부 조인
* JPQL: SELECT m, t FROM Member m LEFT JOIN Team t on m.username = t.name
* SQL: SELECT m.*, t.* FROM Member m LEFT JOIN Team t ON m.username = t.name */
 @Test
 public void join_on_no_relation() throws Exception {
     em.persist(new Member("teamA"));
     em.persist(new Member("teamB"));
     List<Tuple> result = queryFactory
             .select(member, team)
             .from(member)
             .leftJoin(team).on(member.username.eq(team.name))
             .fetch();
     for (Tuple tuple : result) {
         System.out.println("t=" + tuple);
}
```
- 하이버네이트 5.1부터 `on` 을 사용해서 서로 관계가 없는 필드로 외부 조인하는 기능이 추가되었다. 물론 내부 조 인도 가능하다.
- 주의! 문법을 잘 봐야 한다. **leftJoin()** 부분에 일반 조인과 다르게 엔티티 하나만 들어간다.
    - 일반조인: `leftJoin(member.team, team)` **`// 연관관계가 있다면 알아서 ID 값으로 맞춰줌`**
    - on조인: `from(member).leftJoin(team).on(xxx)`
 
 **결과**
 ```
 t=[Member(id=3, username=member1, age=10), null]
 t=[Member(id=4, username=member2, age=20), null]
 t=[Member(id=5, username=member3, age=30), null]
 t=[Member(id=6, username=member4, age=40), null]
 t=[Member(id=7, username=teamA, age=0), Team(id=1, name=teamA)]
 t=[Member(id=8, username=teamB, age=0), Team(id=2, name=teamB)]
 ```

 ## 서브 쿼리
 > `com.querydsl.jpa.JPAExpressions` 사용

 ### 서브 쿼리 eq 사용
```java
/**
 * 나이가 가장 많은 회원 조회
 */
@Test
public void subQuery() throws Exception {
    QMember memberSub = new QMember("memberSub");
    List<Member> result = queryFactory
            .selectFrom(member)
            .where(member.age.eq(
                    JPAExpressions
                            .select(memberSub.age.max())
                            .from(memberSub)
    )) .fetch();
    assertThat(result).extracting("age")
            .containsExactly(40);
}
```

### select 절에 subquery
```java
List<Tuple> fetch = queryFactory
        .select(member.username,
                JPAExpressions
                        .select(memberSub.age.avg())
                        .from(memberSub)
        ).from(member)
        .fetch();
for (Tuple tuple : fetch) {
    System.out.println("username = " + tuple.get(member.username));
    System.out.println("age = " +
tuple.get(JPAExpressions.select(memberSub.age.avg())
            .from(memberSub)));
}
```
**from 절의 서브쿼리 한계**
JPA JPQL 서브쿼리의 한계점으로 from 절의 서브쿼리(인라인 뷰)는 지원하지 않는다. 당연히 Querydsl도 지원하지 않는다. 하이버네이트 구현체를 사용하면 select 절의 서브쿼리는 지원한다. Querydsl도 하이버네이트 구현체를 사용 하면 select 절의 서브쿼리를 지원한다.

**from 절의 서브쿼리 해결방안**
1. 서브쿼리를 join으로 변경한다. (가능한 상황도 있고, 불가능한 상황도 있다.)
2. 애플리케이션에서 쿼리를 2번 분리해서 실행한다.
3. nativeSQL을 사용한다.

> **DB는 데이터를 필터링, 그룹핑하여 가져오는 용도로만 사용하고 비즈니스 로직은 애플리케이션에서 풀어내자**
