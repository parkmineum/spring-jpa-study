# Batch Size 튜닝 및 Entity 객체 프록시 호출

태그: Study
진행도: 완료

# Batch Size?

`Batch Size`는 **한 번에 로딩하거나 DB에 보내는 SQL의 묶음 크기**를 의미하는데, 지연 로딩(N+1 문제) 최적화나 SQL 쿼리 성능을 최적화를하기 위해 사용되는 옵션이다. 

여러 개의 프록시 객체를 조회할 때 `WHERE` 절(조건)이 같은 여러 개의 `SELECT` 쿼리들을 하나의 `IN` 쿼리로 만들어준다. 

> 간단한 활용 방법을 알아보자.
> 

JPA 에 정의된 Batch Size 에 대한 인터페이스(클래스 파일)를 살펴보면, 

```jsx
@Target({TYPE, METHOD, FIELD})
@Retention(RUNTIME)
public @interface BatchSize {
    int size();
}
```

`size` 를 설정해야 함을 알 수 있습니다. 이는 `IN` 절에 들어갈 요소의 최대 개수를 의미합니다. 
만약 이 `IN` 절에 `size` 보다 더 많은 요소가 들어가야 한다면, 여러 개의 `IN` 쿼리로 나눠 날리게 됩니다. 

위에서 정의한 Entity 에 아래와 같이 적용이 가능하다. 

```jsx
@BatchSize(size = 100)
@Entity
public class Parent {
}

or 

@Entity
public class Parent {

    @BatchSize(size = 100)
    @OneToMany(mappedBy = "parent")
    private List<Child> children = new ArrayList<>();
}
```

이렇게 설정하면, 다른 Entity에서 여러 개의 Parent 객체를 프록시로 호출한다면, `BatchSize` 가 적용되어 `IN` 쿼리로 조회하게 될 것 입니다. 

> **Entity 객체를 프록시로 호출**한다는게 무슨 뜻일까?
> 

⇒ 실제 Entity 객체 대신에 `지연 로딩` 을 지원하는 프록시(대리) 객체를 반환한다는 뜻이다.
 이는 Hibernate가 성능 최적화를 위해 사용하는 기법으로, `@ManyToOne`, `@OneToOne`, `@OneToMany`, `@ManyToMany`와 같은 관계에서 `fetch = FetchType.LAZY`로 지정하면, 연관 엔티티는 즉시 로딩되지 않고 **프록시 객체로 대체된다.** 

```jsx
@Entity
public class Order {
    @ManyToOne(fetch = FetchType.LAZY)
    private Member member;
}
```

여기서 `order.getMember()` 는 실제 Memeber 엔티티가 아니라, Member를 상속한 프록시 객체이다. 
따라서 실제 DB 조회는 해당 프록시의 메서드를 호출하는 시점에 이루어진다.

getMember() 가 호출되면 Hibernate 내부에서 DB에 접근하여 진짜 Member 객체를 생성하고, 
프록시 내부에 진짜 객체를 저장 후, 실제 객체가 정말 필요할때만 로딩하기 위해 프록시에 위임하여 실행한다. 

![스크린샷 2025-04-03 오후 4.50.20.png](%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-04-03_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_4.50.20.png)

### [호출 테스트]

```java
List<Parent> parents = parentRepository.findAll();

// 실제로 사용해야 쿼리가 나가기 때문에 size() 까지 호출
parents.get(0).getChildren().size();
parents.get(1).getChildren().size();
```

Batch Size 를 지정하지 않고, 명시적으로 두 번 호출해보았습니다. 

```java
SELECT * FROM parent

SELECT * FROM child WHERE child.parent_id = 1
SELECT * FROM child WHERE child.parent_id = 2
```

Child 테이블을 조회하기 위해 두 개의 쿼리가 호출되는 것을 알 수 있습니다. 

Batch Size 를 추가한다면, 여러 쿼리를 하나의 `IN` 쿼리로 만들어줍니다. 

```java
SELECT * FROM parent

SELECT * FROM child WHERE child.parent IN (1, 2)
```

만약 `size` 를 100으로 설정 했을 때 데이터가 350개인 경우, 1~100, 101~200, 201~300, 301~350 이렇게
4번에 나누어서 `IN` 쿼리를 날리게 됩니다. 

---

## **Batch Size 설정**

JPA/Hibernate는 기본적으로 INSERT/UPDATE 쿼리를 건당 실행하는데, `hibernate.jdbc.batch_size` 설정을 통해 여러 건의 SQL을 배치(batch) 사이즈로 묶어 전송 가능이 가능하다.

위에서 지연 로딩(N+1 문제) 최적화나 SQL 쿼리 성능을 최적화를하기 위해 사용되는 옵션이라고 한 것을 기억할 것이다. 

> **N+1 문제 최적화**
> 

Order → Member 가 `ManyToOne` 관계라 하자. 

```java
List<Order> orders = orderRepository.findAll();     // Order 10개 조회
for (Order o : orders) {
    System.out.println(o.getMember().getName());   // Member 10명 지연로딩
}
```

기본 설정이라면, `Order` 10개 조회 시 1개의 쿼리, 각 Order에서 `Member` 접근 시 10개의 쿼리를 사용하게 된다. 이는 1+10 = 11개의 쿼리 발생으로 N+1 문제를 초래한다. 

FETCH JOIN 의 해결책도 존재하지만, Bath Size 설정 시 `IN` 절로 묶어서 1~2번의 쿼리로 호출이 가능하게 된다.

> **벌크 저장 최적화**
> 

```java
for (int i = 0; i < 1000; i++) {
    em.persist(new Member(...));
}
```

해당 경우에서도 Bath Size 설정 시, Hibernate가 50개씩 모아 JDBC 배치로 보내게되고, 성능 개선을 할 수 있다.

---

Reference 

https://bcp0109.tistory.com/333

https://velog.io/@joonghyun/SpringBoot-JPA-JPA-Batch-Size%EC%97%90-%EB%8C%80%ED%95%9C-%EA%B3%A0%EC%B0%B0