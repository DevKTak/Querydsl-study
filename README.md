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

## Case 문
**select, 조건절(where), order by에서 사용 가능**
**단순한 조건** 
```java
 List<String> result = queryFactory
         .select(member.age
            .when(10).then("열살") 
            .when(20).then("스무살") 
            .otherwise("기타"))
        .from(member)
        .fetch();
 ```

**복잡한 조건** 
```java

 List<String> result = queryFactory
        .select(new CaseBuilder()
            .when(member.age.between(0, 20)).then("0~20살")
            .when(member.age.between(21, 30)).then("21~30살")
            .otherwise("기타"))
        .from(member)
        .fetch();
```

**orderBy에서 Case 문 함께 사용한 예제**   
예를 들어서 다음과 같은 임의의 순서로 회원을 출력하고 싶다면?
1. 0~30살이아닌회원을가장먼저출력
2. 0~20살회원출력
3. 21~30살회원출력
```java
NumberExpression<Integer> rankPath = new CaseBuilder()
         .when(member.age.between(0, 20)).then(2)
         .when(member.age.between(21, 30)).then(1)
         .otherwise(3);
List<Tuple> result = queryFactory
        .select(member.username, member.age, rankPath)
        .from(member)
        .orderBy(rankPath.desc())
        .fetch();
for (Tuple tuple : result) {
    String username = tuple.get(member.username);
    Integer age = tuple.get(member.age);
    Integer rank = tuple.get(rankPath);
    System.out.println("username = " + username + " age = " + age + " rank = " +
rank); }
```

Querydsl은 자바 코드로 작성하기 때문에 `rankPath` 처럼 복잡한 조건을 변수로 선언해서 `select` 절, `orderBy` 절에서 함께 사용할 수 있다.
```
결과
 username = member4 age = 40 rank = 3
 username = member1 age = 10 rank = 2
 username = member2 age = 20 rank = 2
 username = member3 age = 30 rank = 1
```

> **`DB는 특별한 성능적 유리한 상황이 아니라면 가급적 데이터를 퍼올리는 역할만 하고 비즈니스 로직은 애플리케이션 단에서 처리를 권장합니다.`**
- **✅ `CaseBuilder`**

## 상수, 문자 더하기
### 상수가 필요하면 `Expressions.constant(xxx)` 사용
```java
Tuple result = queryFactory
        .select(member.username, Expressions.constant("A"))
        .from(member)
        .fetchFirst();
```
참고: 위와 같이 최적화가 가능하면 SQL에 constant 값을 넘기지 않는다. 상수를 더하는 것 처럼 최적화가 어려 우면 SQL에 constant 값을 넘긴다.

### 문자 더하기 concat
```java
String result = queryFactory
         .select(member.username.concat("_").concat(member.age.stringValue()))
         .from(member)
         .where(member.username.eq("member1"))
         .fetchOne();
```
- 결과: member1_10  
> 참고: `member.age.stringValue()` 부분이 중요한데, 문자가 아닌 다른 타입들은 `stringValue()` 로 문 자로 변환할 수 있다. 이 방법은 **`ENUM`** 을처리할때도 자주 사용한다.

- **✅ `Expressions.constant()`, `stringValue()`**

## 프로젝션 결과 반환 - 기본
> 프로젝션: select 대상 지정
- 프로젝션 대상이 하나일 땐 구체적인 타입을 지정 (ex. List<String>)
- 프로젝션 타입이 둘 이상일 땐 List<Tuple> 또는 DTO로 조회

## 프로젝션 결과 반환 - DTO 조회
### 순수 JPA에서 DTO 조회
```java 
select new study.querydsl.dto.MemberDto(m.username, m.age) " +
             "from Member m", MemberDto.class)
```


### Querydsl 빈 생성(Bean population)
**3 가지 방법**
- **프로퍼티 접근**
```java
List<MemberDto> result = queryFactory
         .select(Projections.bean(MemberDto.class,
                 member.username,
                 member.age))
         .from(member)
.fetch();
```

- **필드 직접 접근**
```java
 List<MemberDto> result = queryFactory
         .select(Projections.fields(MemberDto.class,
        member.username,
        member.age))
.from(member)
.fetch();
``` 

- **별칭이 다를 때**
```java
package study.querydsl.dto;
 import lombok.Data;
 @Data
 public class UserDto {
     private String name;
     private int age;
 }
```

```java
List<UserDto> fetch = queryFactory
         .select(Projections.fields(UserDto.class,
            member.username.as("name"),
                ExpressionUtils.as(
                JPAExpressions
                    .select(memberSub.age.max())
                    .from(memberSub), "age")
            )
         ).from(member)
         .fetch();
```
- 프로퍼티나, 필드 접근 생성 방식에서 이름이 다를 때 해결 방안
- `ExpressionUtils.as(source,alias)` : 필드나, 서브 쿼리에 별칭 적용
- `username.as("memberName")` : 필드에 별칭 적용

<br>

- **생성자 사용**
```java
 List<MemberDto> result = queryFactory
         .select(Projections.constructor(MemberDto.class,
                 member.username,
                 member.age))
         .from(member)
.fetch(); }
```

## 프로젝션과 결과 반환 - @QueryProjection
> **생성자 + @QueryProjection**

```java
package study.querydsl.dto;
 import com.querydsl.core.annotations.QueryProjection;
 import lombok.Data;
 @Data
 public class MemberDto {
     private String username;
     private int age;
    public MemberDto() {}
    }
    @QueryProjection
    public MemberDto(String username, int age) {
        this.username = username;
        this.age = age;
}   
```
- `./gradlew compileQuerydsl` 
- `QMemberDto` 생성 확인

### @QueryProjection 활용
```java
List<MemberDto> result = queryFactory
         .select(new QMemberDto(member.username, member.age))
         .from(member)
         .fetch();
```
> 이 방법은 `컴파일러로 타입을 체크`할 수 있으므로 가장 안전한 방법이다. 다만 **`DTO에 QueryDSL 어노테이션을 유지 해야 하는 점(외부 라이브러리 의존)과 DTO까지 Q 파일을 생성해야 하는 단점이 있다.
`**

## 동적 쿼리 - BooleanBuilder 사용
### 1. BooleanBuilder

```java
@Test
public void 동적쿼리_BooleanBuilder() throws Exception {
   String usernameParam = "member1";
   Integer ageParam = 10;

    List<Member> result = searchMember1(usernameParam, ageParam);
    Assertions.assertThat(result.size()).isEqualTo(1);
}

private List<Member> searchMember1(String usernameCond, Integer ageCond) {
    BooleanBuilder builder = new BooleanBuilder();
    if (usernameCond != null) {
        builder.and(member.username.eq(usernameCond));
    }
    if (ageCond != null) {
        builder.and(member.age.eq(ageCond));
    }
    return queryFactory
        .selectFrom(member)
        .where(builder)
        .fetch();
}
```

## 동적 쿼리 - Where 다중 파라미터 사용
### 2. Where 다중 파라미터 사용
```java
@Test
public void 동적쿼리_WhereParam() throws Exception {
     String usernameParam = "member1";
     Integer ageParam = 10;
     List<Member> result = searchMember2(usernameParam, ageParam);
     Assertions.assertThat(result.size()).isEqualTo(1);
 }
 private List<Member> searchMember2(String usernameCond, Integer ageCond) {
     return queryFactory
             .selectFrom(member)
             .where(usernameEq(usernameCond), ageEq(ageCond))
             .fetch();
}
 private BooleanExpression usernameEq(String usernameCond) {
     return usernameCond != null ? member.username.eq(usernameCond) : null;
}
private BooleanExpression ageEq(Integer ageCond) {
    return ageCond != null ? member.age.eq(ageCond) : null;
}
```
**Where 다중 파라미터의 장점**
- `where` 조건에 **`null` 값은 무시된다.**
- 메서드를 다른 쿼리에서도 **재활용** 할 수 있다.
- 쿼리 자체의 **가독성**이 높아진다.
- **조합가능**
```java
private BooleanExpression allEq(String usernameCond, Integer ageCond) {
     return usernameEq(usernameCond).and(ageEq(ageCond));
}
```
- **`null` 체크**는 주의해서 처리해야함
- 조합 관련해서 질문 링크: https://www.inflearn.com/questions/94056/%EA%B0%95%EC%82%AC%EB%8B%98-where-%EB%8B%A4%EC%A4%91-%ED%8C%8C%EB%9D%BC%EB%AF%B8%ED%84%B0%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%9C-%EB%8F%99%EC%A0%81-%EC%BF%BC%EB%A6%AC-%EC%82%AC%EC%9A%A9%EC%97%90-%EB%8C%80%ED%95%9C-%EC%A7%88%EB%AC%B8%EC%9E%85%EB%8B%88%EB%8B%A4

## 수정, 삭제 벌크 연산
> **주의:** 벌크 연산은 JPQL 배치와 마찬가지로, 영속성 컨텍스트에 있는 엔티티를 무시하고 실행되기 때문에 배치 쿼리를 실행하고 나면 영속성 컨텍스트를 초기화 하는 것이 안전하다.   
>
> **해결 방법**
> 1. **em.refresh() 사용** 
>       - 벌크 연산 수행 직후 정확한 salary 엔티티를 사용해야 한다면, **`em.refresh(salary)`** 를 사용하여 DB에서 salary를 다시 조회한다.   
> 또는, **`em.flush();` `em.clear();`** 
> 2. **벌크 연산 먼저 실행** 
>       - 벌크 연산 먼저 실행하고 조회하면 된다. 가장 실용적인 해결책이며, JPA와 JDBC를 함께 사용할 때도 유용하다. 
> 3. **벌크 연산 수행 후 영속성 컨텍스트 초기화**
>       - 영속성 컨텍스트에 남아 있는 엔티티를 제거하는 방법이다.

<br>

**쿼리 한번으로 대량 데이터 수정**
```java
 long count = queryFactory
        .update(member)
        .set(member.username, "비회원") 
        .where(member.age.lt(28)) 
        .execute();
```

**기존 숫자에 1 더하기** 
```java
 long count = queryFactory
        .update(member)
        .set(member.age, member.age.add(1))
        .execute();
```
곱하기: `multiply(x)`

**쿼리 한번으로 대량 데이터 삭제** 
```java
 long count = queryFactory
        .delete(member)
        .where(member.age.gt(18))
        .execute();
```

## SQL function 호출하기
SQL function은 JPA와 같이 Dialect에 등록된 내용만 호출할 수 있다.

**member => M으로 변경하는 replace 함수 사용**
```java
 String result = queryFactory
         .select(Expressions.stringTemplate("function('replace', {0}, {1}, {2})",
 member.username, "member", "M"))
         .from(member)
         .fetchFirst();
```

**소문자로 변경해서 비교해라.** 
```java
 .select(member.username)
 .from(member)
 .where(member.username.eq(Expressions.stringTemplate("function('lower', {0})",
 member.username)))
```
**lower 같은 ansi 표준 함수들은 querydsl이 상당부분 내장하고 있다. 따라서 다음과 같이 처리해도 결과는 같다.** 
```java
.where(member.username.eq(member.username.lower()))
 ```

## 동적 쿼리와 성능 최적화 조회 - Where절 파라미터 사용

**Where절에 파라미터를 사용한 예제**
```java
//회원명, 팀명, 나이(ageGoe, ageLoe)
public List<MemberTeamDto> search(MemberSearchCondition condition) {
  return queryFactory
 }
.select(new QMemberTeamDto(
        member.id,
        member.username,
        member.age,
        team.id,
        team.name))
.from(member)
.leftJoin(member.team, team)
.where(usernameEq(condition.getUsername()),
        teamNameEq(condition.getTeamName()),
        ageGoe(condition.getAgeGoe()),
        ageLoe(condition.getAgeLoe()))
.fetch();

 private BooleanExpression usernameEq(String username) {
     // return isEmpty(username) ? null : member.username.eq(username);
     return hasText(username) ? member.username.eq(username) : null;
}
 private BooleanExpression teamNameEq(String teamName) {
     return isEmpty(teamName) ? null : team.name.eq(teamName);
}
 private BooleanExpression ageGoe(Integer ageGoe) {
     return ageGoe == null ? null : member.age.goe(ageGoe);
}
 private BooleanExpression ageLoe(Integer ageLoe) {
     return ageLoe == null ? null : member.age.loe(ageLoe);
}
```

## 사용자 정의 리포지토리
<img width="792" alt="image" src="https://github.com/f-lab-edu/hotel-java/assets/68748397/5db45e94-61f2-48b1-be11-2fc4e493867f">

> \* 네이밍 규칙: MemberRepository + **`impl`** 

강의 부분 링크: https://www.inflearn.com/course/lecture?courseSlug=querydsl-%EC%8B%A4%EC%A0%84&unitId=30150&tab=curriculum

## 스프링 데이터 페이징 활용1 - Querydsl 페이징 연동
## 스프링 데이터 페이징 활용2 - CountQuery 최적화
> **Querydsl** `fetchResults()` , `fetchCount()` Deprecated(향후 미지원)

**1. 사용자 정의 인터페이스 작성**
```java
public interface MemberRepositoryCustom {
    Page<MemberTeamDto> searchPageComplex(MemberSearchCondition condition, Pageable pageable);
}
```

**2. 사용자 정의 인터페이스 구현**
```java
import java.util.List;
import static org.springframework.util.StringUtils.isEmpty;
import static study.querydsl.entity.QMember.member;
import static study.querydsl.entity.QTeam.team;

import org.springframework.data.support.PageableExecutionUtils; //패키지 변경

public class MemberRepositoryImpl implements MemberRepositoryCustom {
    private final JPAQueryFactory queryFactory;

    public MemberRepositoryImpl(EntityManager em) {
        this.queryFactory = new JPAQueryFactory(em);
    
    }

    @Override
    //회원명, 팀명, 나이(ageGoe, ageLoe)
    public <MemberTeamDto> searchPageComplex(MemberSearchCondition condition, Pageable pageable) {
            List<MemberTeamDto> content = queryFactory
                    .select(new QMemberTeamDto(
                            member.id,
                            member.username,
                            member.age,
                            team.id,
                            team.name.as("teamName")))
                    .from(member)
                    .leftJoin(member.team, team)
                    .where(usernameEq(condition.getUsername()),
                            teamNameEq(condition.getTeamName()),
                            ageGoe(condition.getAgeGoe()),
                            ageLoe(condition.getAgeLoe()))
                    .offset(pageable.getOffset())
                    .limit(pageable.getPageSize())
                    .fetch();

            JPAQuery<Long> countQuery = queryFactory
                    .select(member.count())
                    .from(member)
                    .leftJoin(member.team, team)
                    .where(
                            usernameEq(condition.getUsername()),
                            teamNameEq(condition.getTeamName()),
                            ageGoe(condition.getAgeGoe()),
                            ageLoe(condition.getAgeLoe())
                    );

            return PageableExecutionUtils.getPage(content, pageable, countQuery::fetchOne);
    }

    private BooleanExpression usernameEq(String username) {
        return isEmpty(username) ? null : member.username.eq(username);
    }
    private BooleanExpression teamNameEq(String teamName) {
        return isEmpty(teamName) ? null : team.name.eq(teamName);
    }
    private BooleanExpression ageGoe(Integer ageGoe) {
        return ageGoe == null ? null : member.age.goe(ageGoe);
    }
```
> **PageableExecutionUtils.getPage()로 최적화**
- count 쿼리가 생략 가능한 경우 생략해서 처리
    - 페이지 시작이면서 컨텐츠 사이즈가 페이지 사이즈보다 작을 때
    - 마지막 페이지 일 때 (offset + 컨텐츠 사이즈를 더해서 전체 사이즈 구함, 더 정확히는 마지막 페이지이면 서 컨텐츠 사이즈가 페이지 사이즈보다 작을 때)

**3. 스프링 데이터 리포지토리에 사용자 정의 인터페이스 상속**
```java
public interface MemberRepository extends JpaRepository<Member, Long>, MemberRepositoryCustom {}
```

**4. 컨트롤러**
```java
@GetMapping("/v3/members")
    public Page<MemberTeamDto> searchMemberV3(MemberSearchCondition condition, Pageable pageable) {
        return memberRepository.searchPageComplex(condition, pageable);
    }
```

**5. 호출**   
**`http://localhost:8080/v3/members?size=5&page=2`**

