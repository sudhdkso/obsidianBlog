JPA는 연결된 DB에 데이터를 저장하는 방법으로 `save()`와 `saveAll()`을 제공합니다.

`save()` 개별 객체를 저장할 때 사용하고, `saveAll()`은 여러 개의 객체를 list로 한꺼번에 저장할 때 사용합니다.

# **save() 100개 하기 vs 100개의 데이터 한꺼번에 saveAll()하기**

`save()`를 100개 하는 것과 100개의 데이터를 list로 묶어서 `saveAll()`하는 것은 성능 차이가 발생합니다.

결론적으로 많은 데이터를 저장할 때는 saveAll()을 사용하는 것이 성능 측면에서 유리합니다.

## save()를 활용하여 데이터를 저장

아래에는 list로 들어온 데이터를 개별로 save()를 호출하여 저장하는 방식입니다.

100개가 들어오면 100개 호출되고, 1000개가 들어오면 1000개가 호출되는 방식입니다.

```java
@PostMapping
@Transactional
public ResponseEntity<Void> addProduct(@RequestBody List<AddProductRequest> request) {

    for(AddProductRequest rq : request){
        productPort.save(new Product(rq.name(),rq.price(),rq.discountPolicy()));
    }
    return ResponseEntity.status(HttpStatus.CREATED).build();
}

```

### 100개의 데이터를 save()할 때 소요 시간

![https://velog.velcdn.com/images/sudhdkso/post/67278e8d-aa0e-4c71-9850-b588e7afc8db/image.png](https://velog.velcdn.com/images/sudhdkso/post/67278e8d-aa0e-4c71-9850-b588e7afc8db/image.png)

종료를 하기 까지 약 **5초** 정도 걸리는 것을 확인할 수 있습니다.

## saveAll()을 활용하여 데이터를 저장

```java
@PostMapping("/list")
@Transactional
public ResponseEntity<Void> addListProduct(@RequestBody List<AddProductRequest> request) {
    List<Product> list = request.stream()
            .map( r -> new Product(r.name(),r.price(),r.discountPolicy()))
            .collect(Collectors.toList());

    productPort.saveAll(list);
    return ResponseEntity.status(HttpStatus.CREATED).build();
}

```

### 100개의 데이터를 saveAll()할 때 소요 시간

![https://velog.velcdn.com/images/sudhdkso/post/0b439778-9691-49e6-a9d9-365cefa9f805/image.png](https://velog.velcdn.com/images/sudhdkso/post/0b439778-9691-49e6-a9d9-365cefa9f805/image.png)

약 **1.5초** 정도 시간이 소요되는 것을 확인할 수 있습니다.

여기서는 100개의 데이터만 테스트해보았지만, 실무에서는 더 많은 데이터를 저장하게 될 상황이 발생합니다.

이때 save()보다는 saveAll()을 사용하는 것이 더욱 유리하겠습니다.

save()와 saveAll()은 왜 이렇게 성능 차이가 발생하는 것일까요?

## save()의 코드

```java
@Transactional
	@Override
	public <S extends T> S save(S entity) {

		Assert.notNull(entity, "Entity must not be null.");

		if (entityInformation.isNew(entity)) {
			em.persist(entity);
			return entity;
		} else {
			return em.merge(entity);
		}
	}

```

## saveAll()의 코드

코드를 살펴보면 saveAll()에서는 save()를 호출하는 것을 알 수 있습니다.

```java
@Transactional
@Override
public<SextendsT> List<S> saveAll(Iterable<S> entities) {

   Assert.notNull(entities,"Entities must not be null!");

   List<S> result =newArrayList<>();

	for(S entity : entities) {
	      result.add(save(entity));
	   }

	returnresult;
}

```

그렇다면 **100개의 데이터를 saveAll()하는 것**은 **save()를 100번 호출하는 것**과 동일한게 아닐까요?

## 성능이 차이나는 이유

save()와 saveAll()모두 `@Transactional` 어노테이션이 달린 것을 볼 수 있습니다 .그리고 기본 트랜잭션 propagation 타입은 **REQUIRED**_입니다._ **REQUIRED**는 트랜잭션이 존재하지 않으면 새로운 트랜잭션을 생성하고, 이미 존재하면 그 트랜잭션에 참여하는 유형입니다.

그리고 `@Transactional` 의 경우 AOP 프록시 기반으로, 외부 Bean 객체가 있고, 이 객체의 함수를 호출해야 Intercept가 되어 트랜잭션으로 묶이게 됩니다. 따라서 save()를 여러번 호출하는 경우는 계속 기존의 트랜잭션이 존재하는 지 계속 확인해줘야 하기 때문에 추가로 리소스가 소모됩니다. 하지만 saveAll()의 경우는 내부에서 save()를 호출하기 때문에 saveAll()을 할 때 트랜잭션이 생성되어 하나의 트랜잭션으로 작동하게 됩니다. 그래서 save()를 여러번 하는 것보다 성능이 더 좋은 것입니다.

## save() flow

### 기존트랜잭션이 존재할 경우

- save() 를 호출했는데 기존 트랜잭션이 존재할 경우 save() 는 기존 트랜잭션에 참여하게 됩니다.
- 기존 트랜잭션에 참여 하지만, @Transactional 이 걸려있기에 spring 의 프록시 로직을 타게 됩니다.

### 기존트랜잭션이 없을 경우

- 기존 트랜잭션이 없을 경우 트랜잭션이 생성하고 종료합니다.

### saveAll() flow

### 기존 트랜잭션 존재할 경우

- saveAll()을 호출했을 때 기존 트랜잭션이 존재하면 기존 트랜잭션에 참여하게 됩니다.
- saveAll()에서 save()를 호출할 때는 같은 인스턴스에서 내부 호출하기에 프록시 로직을 타지 않습니다.

### 기존 트랜잭션 없을 경우

- saveAll()을 호출했을 때 트랜잭션을 생성합니다.
- 그리고 saveAll()내부에서 save() 를 여러 번 호출합니다.
    - 같은 인스턴스에서 내부 호출하기에 프록시 로직을 타지 않습니다.

## [참고]

[https://maivve.tistory.com/342https://www.baeldung.com/spring-data-save-saveallhttps://insanelysimple.tistory.com/302](https://maivve.tistory.com/342https://www.baeldung.com/spring-data-save-saveallhttps://insanelysimple.tistory.com/302)