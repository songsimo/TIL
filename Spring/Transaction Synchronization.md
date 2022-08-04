# 트랜잭션

Service Layer에서 트랜잭션 디폴트로 Propagation.REQUIRED 인 메서드가 있는 상황에서 Repository에 readonly=true가 있다면, 트랜잭션이 전파될까? 아니면 새로 생성이 될까?

> 트랜잭션을 다시 찾아보게 된 질문이었다. 명확하게 이유를 설명할 수 없었는데 Spring과 연관되어 있기에 정리하고자 한다.

1. JdbcTemplate를 사용할 수 없다.
2. Service Layer에서 DB 커넥션을 관리해야 한다.
3. DB Connection 정보를 DAO의 메서드에 파라미터로 전달해야 한다.

현재 트랜잭션을 유지하기 위해 서비스 계층에서 DA 계층으로 Connection을 전달하는 구조이다. 서비스 계층의 메서드 안에 비즈니스 로직 + JDBC 기술이 섞이게 된다. 이는 비즈니스 로직과 기술 로직의 결합 및 유지보수의 어려움을 발생시킨다.

<br>

---
## 1. 트랜잭션 동기화

Connection을 DAO에 파라미터를 넘겨야 하기에 같은 기능이라도 트랜잭션 유지/유지X에 따라 메서드가 분리되게 된다. 파라미터에 특정 DA 기술이 들어가서 DAO가 더이상 기술에 독립적인 존재일 수 없다.

만약, JDBC에서 JPA나 하이버네이트로 바꾼다면? 서비스 계층은 물론 DAO 파라미터와 로직도 바꿔야만 한다.

```text
스프링은 이를 어떻게 해결하였을가?
```

* 트랜잭션 동기화(Transaction Synchronization)

Spring은 Connection을 특별한 저장소에 보관해두고, DAO에서는 저장된 Connection을 가져다가 사용하는 방법을 제시한다.

이를 통해, Connection을 파라미터로 전달할 필요가 없어지고 저장소에서 가져오기 때문에 필요에 따라 트랜잭션 단위 형성이 가능해진다.

또한, 저장소는 각 (작업) 스레드마다 독립적으로 Connection을 관리하기 때문에 멀티스레드 환경에서도 충돌 위험이 없다.

---
## 2. 트랜잭션 추상화

트랜잭션 동기화를 이용하여 더 이상 DAO가 파라미터로 Connection를 받지는 않게 되었지만 아직 문제가 남아있다. 

트랜잭션의 경계 설정을 하는 Service Layer가 아직 특정 DA 기술에 종속되어 있는 구조이다.

트랜잭션은 로컬 트랜잭션(Local Transaction)과 전역 트랜잭션(Global Transaction)이 있다. 로컬 트랜잭션은 하나의 자원 관리자(데이터베이스)가 참여하는 트랜잭션이다. 전역 트랜잭션은 하나 이상의 자원 관리자가 참여하는 트랜잭션이다.

```text
전역 트랜잭션일 경우 트랜잭션 처리 구현을 어떻게 해야할까?

스프링에서는 어떻게 이 문제를 해결하였을까?
```

* 트랜잭션 추상화

스프링은 PlatformTransactionManager 추상화와 DataSourceTransactionManager, JpaTransactionManger, HibernateTransactionManager 등의 구현체를 제공한다.

의존성 주입을 통해 필요한 기술을 사용하게 된다.

---
## 3. AOP

트랜잭션 동기화와 트랜잭션 추상화는 기술적인 부분을 분리시켰다. 이제 특정 기술에 대한 종속적인 관계는 없어졌다.

그래도 아직은 아쉬운 부분이 남아있다. 무엇이 아쉬울까?

트랜잭션 경계를 설정하는 기술적인 부분, 트랜잭션을 얻고 커밋하거나 롤백하는 부분은 반복적이고 어느 비즈니스 로직이나 똑같은 코드다.

```text
이를 비즈니스 로직에서 따로 분리할 수 있는 방법이 없을까?
```
* AOP

관점지향 프로그래밍으로 이런 반복적으로 사용되는 부가 기능을 핵심 기능으로부터 분리하는 방법이다. Service Layer는 이제 핵심 기능에만 집중할 수 있게 된다.

Spring에서는 AOP를 프록시 기반으로 만들었는데, 본 주제에서 벗어나기에 설명을 생략한다.

참고링크
https://github.com/cheese10yun/blog-sample/blob/master/spring-transaction/READEMD.md
https://ojt90902.tistory.com/866?category=967948
https://jongmin92.github.io/2018/04/08/Spring/toby-5/