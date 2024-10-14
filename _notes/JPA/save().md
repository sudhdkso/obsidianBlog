---
tags:
---
Jpa에는 DB에 저장하기 위해 save()라는 메서드를 제공합니다. 이 함수는 새로운 데이터를 저장할 때도 사용을 하지만, 원래 있던 데이터의 정보를 업데이트하여 저장할 때도 사용합니다.

save()는 어떤 원리로 저장할 데이터와 업데이트할 데이터를 구분하는걸까요?

먼저 JpaRepository의 save()함수 내부를 살펴보겠습니다.

## save()메서드 구조

```java
@Transactional
@Override
public <S extends T> S save(S entity) {

	if (entityInformation.isNew(entity)) {
		em.persist(entity);
		return entity;
	} else {
		return em.merge(entity);
	}
}
```

`save()`의 경우 매개변수로 받아오는 엔티티가 새로운 엔티티이면 EntityManager의 `persist()` 메서드를 호출하고 그렇지 않다면 `merge()` 메서드를 호출 합니다.

`persist()` 는 엔티티를 영속 상태로 만드는 것이고, `merge()` 는 이미 detach된 엔티티를 다시 영속 상태로 만들기 위해 사용합니다.

## save()메서드 동작 원리

1. 매개변수로 받아오는 엔티티가 영속성 컨텍스트 1차 캐시에 있는지 확인합니다.
2. 1차 캐시에 없다면 DB에 id를 기준으로 select 쿼리를 날려서 조회합니다.
3. DB에 해당 값이 없을 때 insert 쿼리가 발생합니다.
4. DB에서 조회가 되어 1차 캐시에 엔티티가 저장이 되면 트랜잭션 커밋 시점에 파라미터 엔티티의 값과 1차 캐시에 저장되어 있는 엔티티의 값을 비교하여 다른 점이 있을 경우 update 쿼리가 발생합니다.

## 새로운 엔티티인지 확인하는 방법

엔티티가 새로운 엔티티인지 확인하기 위해서 엔티티 식별자(ID)의 상태를 확인합니다.

여기서 식별자란 개발자가 `@Id` 어노테이션을 붙여서 객체를 구별할 수 있게 해주는 고유의 값입니다. 그리고 이 식별자의 값이 객체 타입(Long, String 등...)일 때는 `null`, 원시 타입(int, long 등..)일 때는 0 이면 새로운 엔티티로 인식합니다.

`@Id` 와 함게 `@GenerateValue` 어노테이션을 설정하였다면, 데이터베이스에 식별자 생성을 위임하기 때문에 `save()`를 호출하는 시점에 새로운 엔티티로 인식하여 persist가 호출됩니다. 하지만 단순히 `@Id` 어노테이션만 붙여서 식별자를 직접 할당 하였다면, 이는 `save()` 를 호출 시점에 새로운 엔티티가 아니므로 merge를 호출하게 됩니다.

## [참고]

[https://web-km.tistory.com/46](https://web-km.tistory.com/46)

[https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/domain/Persistable.html](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/domain/Persistable.html)