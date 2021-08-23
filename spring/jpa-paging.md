---
description: Jpa pagination 최적화 고민
---

# JPA Paging

> 사용환경 : Kotlin, Spring Boot 2, Postgresql

### _Paging??_

페이징 처리란 데이터를 조회할 때 항상 모든 데이터를 다 요청한다면  투입되는 리소스가 너무 많아질테니  데이터를 나눠서 조회를 하는 기법을 말한다 \(많은 웹사이트의 게시판이 대표적인 예시\)

> Spring & JpaRepository에는 Pageable객체를 통한 조회를 지원해주어  개발자가 쉽게 페이징을 구현할 수 있다.

#### _Example_

1. 엔티티에 OneToMany 관계가 걸려있을때, paging 처리를 위한 간단한 쿼리  \(0 page, 2 size를 조회해오는 쿼리\)

   ```kotlin
    override fun pagingQuery(): List<Team> {
        return jpaQueryFactory.selectDistinct(team)
            .from(team)
            .leftJoin(team.member, member).fetchJoin()
            .offset(0)
            .limit(2)
            .fetch()
    }
   ```

   > team, member 엔티티 같은 경우는 이전 N+1 쿼리 해결 글에서 썼던 엔티티를 사용하였다.  [https://harrycjy1.github.io/posts/spring/n1query](https://harrycjy1.github.io/posts/spring/n1query)

2. 실행결과 로그

   ```text
   2021-08-18 20:07:44.687  WARN 4376 --- [  restartedMain] o.h.h.internal.ast.QueryTranslatorImpl   : HHH000104: firstResult/maxResults specified with collection fetch; applying in memory!
   Hibernate: 
    select
        distinct team0_.id as id1_1_0_,
        member1_.id as id1_0_1_,
        member1_.team_id as team_id2_0_1_,
        member1_.team_id as team_id2_0_0__,
        member1_.id as id1_0_0__ 
    from
        dev.team team0_ 
    left outer join
        dev.member member1_ 
            on team0_.id=member1_.team_id
   ```

로그를 살펴보면 offset limit 값을 추가했음에도 실행된 쿼리에 적용되지 않았음을 확인할 수 있고,  또한, 쿼리 위에 WARN : HHH000104 firstResult/maxResults specified with collection fetch; applying in memory!  라는 워닝이 뜨는 것도 확인할 수 있다.

#### _이유?_

JPA fetch join과 paging은 같이 사용할 수 없기 때문에 _**\(OneToOne 혹은 ManyToOne의 경우 paging이 적용된다.\)**_ 실행된 쿼리에 offset, limit이 적용되지 않는다.  그럼에도 실행된 결과값을 확인해보면 조회가 제대로 되는 것을 확인할 수 있는데,  이는 저 실행된 쿼리로 가져온 데이터들을 메모리에서 페이징 처리 하기 때문이다.

> HHH000104 워닝은 조회한 데이터를 메모리에서 페이징 처리 하기 때문에 뜨는 warning이다.

#### _해결방법_

운영 중인 서버에서 데이터를 페이징 처리 없이 조회하면 조회된 데이터의 개수가 커질 경우는  상당히 크리티컬한 이슈가 발생할 수 있기 때문에 위와 같은 쿼리는 절대 실행 되어선 안된다.

하지만, 개발 하다보면 OneToMany관계는 흔히 쓰는 관계였고 페이지네이션 또한 필수적인 요구사항이었기 때문에  쓴 방법은 jpa에서 제공하는 **spring.jpa.properties.hibernate.default\_batch\_fetch\_size** 설정을 이용하는 방법이었다.

> application.yml 기준  spring.jpa.properties.hibernate.default\_batch\_fetch\_size :500 적용 default\_batch\_size를 설정하게 되면 설정한 값만큼 in 쿼리를 실행한다.

yml 수정 후 실행 쿼리

```kotlin
    override fun pagingQuery(): List<Team> {
        return jpaQueryFactory.selectDistinct(team)
            .from(team)
//          .leftJoin(team.member, member).fetchJoin()
            .offset(0)
            .limit(10)
            .fetch()
    }
```

결과 로그

```text
Hibernate: 
    select
        distinct team0_.id as id1_1_ 
    from
        dev.team team0_ limit ?
Hibernate: 
    select
        member0_.team_id as team_id2_0_1_,
        member0_.id as id1_0_1_,
        member0_.id as id1_0_0_,
        member0_.team_id as team_id2_0_0_ 
    from
        dev.member member0_ 
    where
        member0_.team_id in (
            ?, ?, ?
        )
```

수정한 쿼리는 yml을 수정한 후 fetch 조인 부분을 제거했다.  실행된 쿼리를 보면 team을 조회하고 member의 수만큼 \(3개\) in 쿼리가 실행된 것을 확인할 수 있다.  만약 조회하는 멤버의 수가\(n\) batch\_size보다 크다면 batch\_size로 나눈 횟수만큼 실행된다.

> 추가
>
> HHH000104 warning 같은 경우 실제 배포 환경에서 심각한 문제가 될 수 있다. 이를 위해 
>
> **fail\_on\_pagination\_over\_collection\_fetch** 라는 ****설정을 제공한다.

```text
spring.jpa.properties.hibernate.query.fail_on_pagination_over_collection_fetch : true
```

해당 설정 후 실행된 쿼리가 HHH000104 warning을 띄우는 쿼리 일 경우 

```text
Caused by: org.springframework.orm.jpa.JpaSystemException: firstResult/maxResults specified with collection fetch. 
In memory pagination was about to be applied. Failing because 'Fail on pagination over collection fetch' is enabled.; 
nested exception is org.hibernate.HibernateException: firstResult/maxResults specified with collection fetch. In memory pagination was about to be applied. Failing because 'Fail on pagination over collection fetch' is enabled.
```

에러를 띄우니 설정을 해둔채 개발하는것을 추천한다.

