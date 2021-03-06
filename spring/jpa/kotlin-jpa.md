---
description: kotlin과 JPA 사용하기
---

# Kotlin, Jpa

#### JPA를 kotlin & spring boot2에 적용하기 위한 기본 설정을 정리

> JPA 구현체인 Hibernate User guide Link
>
> [https://docs.jboss.org/hibernate/orm/5.4/userguide/html\_single/Hibernate\_User\_Guide.html\#entity](https://docs.jboss.org/hibernate/orm/5.4/userguide/html_single/Hibernate_User_Guide.html#entity)

위 link에서 가져온 requirements 중 kotlin 사용 시 확인해야 하는 두가지

1. The entity class must have a public or protected no-argument constructor. It may define additional constructors as well.
2. The entity class must not be final. No methods or persistent instance variables of the entity class may be final.



> 엔티티 클래스는 반드시 no-arg 생성자가 있어야 한다.
>
> [https://kotlinlang.org/docs/no-arg-plugin.html\#gradle](https://kotlinlang.org/docs/no-arg-plugin.html#gradle)

* 설정

```kotlin
//build.gradle.kts
plugins {
    val kotlinVersion = "1.3.72"
    kotlin("plugin.jpa") version kotlinVersion
}
```

 해당 설정 시 @Entity, @Embeddable, @MappedSuperClass 를 선언한 클래스에 no-arg 생성자를 자동으로 만들어준다.

> 엔티티 클래스는 final일 수 없다. 
>
> [https://kotlinlang.org/docs/all-open-plugin.html](https://kotlinlang.org/docs/all-open-plugin.html)

* 설정

```kotlin
//build.gradle.kts
plugins {
    val kotlinVersion = "1.3.72"
    kotlin("plugin.jpa") version kotlinVersion
}

allOpen {
    annotation("javax.persistence.Entity")
    annotation("javax.persistence.Embeddable")
    annotation("javax.persistence.MappedSuperclass")
}
```

@Entity, @Embeddable, @MappedSuperClass annotation이 붙은 클래스를 open 클래스로 만들어 주는 설정

















