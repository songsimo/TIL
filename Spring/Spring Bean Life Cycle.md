# Spring Bean Life Cycle

빈 생명 주기는 빈이 인스턴스화되는 시기와 방법, 살아 있을 때 수행하는 작업, 소멸되는 시기와 방법을 나타낸다.

데이터베이스 커넥션 풀로 예를 들면 애플리케이션 시작 시점에 필요한 연결을 미리 해두고 애플리케이션 종료 시점에 연결을 모두 종료하는 작업을 진행해야 하는데, 스프링 컨테이너는 이런 컨텍션 생성과 종료 시점을 제어하게 된다. 

## (1) 스프링 빈 라이프 사이클

> 스프링 컨테이너 시작 -> 스프링 빈 생성 -> 의존성 주입 -> 초기화 콜백 -> 사용 -> 소멸 전 콜백 -> 스프링 종료

* 초기화 콜백: 빈이 생성되고, 빈의 의존관계 주입이 완료된 후 호출
* 소멸전 콜백: 빈이 소멸되기 직전 호출

## (2) Bean 생명주기 콜백의 종류

* 인터페이스 방식(InitializingBean, DisposableBean)
* 메소드 지정 방식
* 어노테이션 방식(@PostConstruct, @PreDestroy)

---
> (2.1) 인터페이스 방식(InitializingBean, DisposableBean)

```
package hello.core.lifecycle;

import org.springframework.beans.factory.DisposableBean;
import org.springframework.beans.factory.InitializingBean;

public class NetworkClient implements InitializingBean, DisposableBean {
    private String url;
    public NetworkClient() {
        System.out.println("생성자 호출, url = " + url);
    }

    public void setUrl(String url) {
        this.url = url;
    }

    public void connect() {
        System.out.println("connect: " + url);
    }

    public void call(String message) {
        System.out.println("call: " + url + " message = " + message);
    }

    public void disConnect() {
        System.out.println("close + " + url);
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        connect();
        call("초기화 연결 메시지");
    }

    @Override
    public void destroy() throws Exception {
        disConnect();
    }
}
```
> 인터페이스 방식의 단점

* 스프링 전용 인터페이스로 코드가 스프링 전용 인터페이스에 의존
* 초기화, 소멸 메서드의 이름을 변경할 수 없음
* 개발자가 직접 코드를 고칠 수 없는 외부 라이브러리에는 적용할 수 없음

---
> (2.2) 메서드 지정 방식: 빈 등록 초기화, 소멸 메서드

설정 정보에 초기화, 소멸 메서드를 지정할 수 있다. @Bean(initMethod = "init", destroyMethod = "close") 과 같이 지정하면 된다.

* NetworkClient

```
package hello.core.lifecycle;

public class NetworkClient {
    private String url;
    
    public NetworkClient() {
        System.out.println("생성자 호출, url = " + url);
    }
    
    public void setUrl(String url) {
        this.url = url;
    }

    public void connect() {
        System.out.println("connect: " + url);
    }

    public void call(String message) {
        System.out.println("call: " + url + " message = " + message);
    }
    
    public void disConnect() {
        System.out.println("close + " + url);
    }

    public void init() {
        System.out.println("NetworkClient.init");
        connect();
        call("초기화 연결 메시지");
    }

    public void close() {
        System.out.println("NetworkClient.close");
        disConnect();
    }
}
```
* LifeCycleConfig

```
@Configuration
static class LifeCycleConfig {
    @Bean(initMethod = "init", destroyMethod = "close")
    public NetworkClient networkClient() {
        NetworkClient networkClient = new NetworkClient();
        networkClient.setUrl("http://hello-spring.dev");
        return networkClient;
    }
}
```
Bean 어노테이션의 속성으로 초기화 메서드명은 'init'으로 종료 메서드명으로는 'close'를 설정했다.


> 메서드 지정 방식의 장점

* 인터페이스 방식과 다르게 메서드 이름을 자유롭게 줄 수 있음
* 스프링 빈이 스프링 코드에 의존하지 않음
* 설정 정보를 이용하기 때문에 외부 라이브러리에도 초기화, 종료 메서드를 적용할 수 있음

> 종료 메서드 추론

* @Bean 어노테이션의 소멸 메서드를 지정하는 속성인 destroyMethod에는 추론 기능이 있다.
* 대부분의 라이브러리에는 관례적으로 'close', 'shutdown' 이라는 이름의 종료 메서드를 사용한다.
* @Bean의 destoryMethod는 default 값으로 '(inferred)'(추론)으로 등록되어 있는데, 해당 값은 'close', 'shutdown' 이라는 이름의 메서드를 자동으로 호출한다.
* 다시 말해, 직접 스프링 빈을 등록하면 관례에 따라 'close'나 'shutdown'으로 지정한다면 따로 속성을 지정하지 않아도 자동으로 찾아서 동작한다.
* 추론 기능을 사용하기 싫을 경우 destoryMethod=""처럼 빈 공백을 지정하면 된다.

---
> (2.3) 어노테이션 지정 방식(@PostConstruct, @PreDestory)

* NetworkClient
```
package hello.core.lifecycle;
import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;

public class NetworkClient {

    ...

    @PostConstruct
    public void init() {
        System.out.println("NetworkClient.init");
        connect();
        call("초기화 연결 메시지");
    }

    @PreDestroy
    public void close() {
        System.out.println("NetworkClient.close");
        disConnect();
    }
}

```

* LifeCycleConfig

```
 // 메서드 지정 방식 참조
```

> 어노테이션 지정 방식의 특징
* 편하고 (최신) 스프링에서 권장하는 방법
* PostConstruct나 PreDestory는 스프링에 종속적인 기술이 아니라 JSR-250 이라는 자바 표준이며, 이는 스프링이 아닌 다른 컨테이너에서도 동작한다.
* 컴포넌트 스캔과 잘 어울린다.
* 하지만, 외부 라이브러리에는 적용하지 못한다. 만약, 외부 라이브러리를 초기화, 종료 해야 하면 @Bean 기능을 이용하면 된다.

---
전체 내용들은 (인프런) 김영한님의 스프링 핵심 원리-기본편을 듣고 정리한 내용입니다.