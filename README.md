

스프링 애플리케이션에 프록시를 적용하려면 <br>포인트컷과 어드바이스(implements MethodInterceptor)로 구성되어 있는 <br>어드바이저 ( Advisor )를 만들어서 스프링 빈으로 등록하면 된다. <br>그러면 나머지는 앞서 배운 자동 프록시 생성기(AutoProxyCreate)가 모두 자동으로 처리해준다.

# Aspect AOP

## @Aspect AOP 적용

@Aspect를 사용하면 Advice+Pointcut을 함께 적용하여 Advisor를 만들 수 있다.

```java
@Slf4j
@Aspect
public class LogTraceAspect {
  private final LogTrace logTrace;

  public LogTraceAspect(LogTrace logTrace) {
    this.logTrace = logTrace;
  }

  @Around("execution(* hello.proxy.app..*(..))")
  public Object excute(ProceedingJoinPoint joinPoint) throws Throwable{
    // Advice 공통 로직들
    // pointcut + advice => Advisor
    TraceStatus status = null;
    try{
      String message = joinPoint.getSignature().toShortString();
      status = logTrace.begin(message); // "OrderController.request()"

      Object result = joinPoint.proceed();// 로직호출

      logTrace.end(status);
      return result;
    } catch (Exception e){
      logTrace.exception(status, e);
      throw e;
    }
  }
}
```

- **@Aspect** : 애노테이션 기반 프록시를 적용할 때 필요하다.
- **@Around("execution(* hello.proxy.app..*(..))")**
  - @Around 의 값에 포인트컷 표현식을 넣는다. 표현식은 AspectJ 표현식을 사용한다.
  - **@Around 의 메서드**는 어드바이스( **Advice **)가 된다.
- **ProceedingJoinPoint joinPoint** : 어드바이스에서 살펴본 MethodInvocation invocation과 유사한 기능이다. <br>내부에 실제 **호출 대상, 전달 인자, 그리고 어떤 객체와 어떤 메서드**가 호출되었는지 정보가 포함되어 있다.



```java
@Configuration
@Import({AppV1Config.class, AppV2Config.class})
public class AopConfig {
  @Bean
  public LogTraceAspect logTraceAspect(LogTrace logTrace) {
    return new LogTraceAspect(logTrace);
  }
}
```



```java
@Import(AopConfig.class)
@SpringBootApplication(scanBasePackages = "hello.proxy.app")
public class ProxyApplication {
  public static void main(String[] args) {
    SpringApplication.run(ProxyApplication.class, args);
  }
  @Bean
  public LogTrace logTrace() {
    return new ThreadLocalLogTrace();
  }
}
```



--------

# @Aspect 프록시 설명

<img width="645" alt="image-20220711000723665" src="https://user-images.githubusercontent.com/58017318/179382177-686d6bb3-73ac-4256-8d00-81bacbc5fc97.png">

앞서 자동 프록시 생성기를 학습할 때, <br>자동 프록시 생성기( AnnotationAwareAspectJ**AutoProxyCreator** )는<br> **Advisor 를 자동으로 찾아와서** 필요한 곳에 **프록시를 생성**하고 **적용**해준다고 했다. <br>자동 프록시 생성기는 여기에 추가로 하나의 역할을 더 하는데, 바로 **@Aspect** 를 찾아서 이것을 **Advisor 로 만들어준다.**<br>그래서 **이름 앞**에 **AnnotationAware** (애노테이션을 인식하는)가 붙어 있는 것이다.



<img width="647" alt="image-20220711001556530" src="https://user-images.githubusercontent.com/58017318/179382182-a49d9910-a9f2-43e5-a543-f0db8fde8370.png">

자동 프록시 생성기의 작동 과정을 알아보자

1. **생성**: **스프링 빈 대상**이 되는 **객체를 생성**한다. ( **@Bean , 컴포넌트 스캔** 모두 포함)
2. **전달**: 생성된 객체를 **빈 저장소에 등록하기 직전에 빈 후처리기에 전달**한다.
3. 3-1. **Advisor 빈 조회**: 스프링 컨테이너에서 Advisor 빈을 모두 조회한다.<br>3-2. **@Aspect Advisor 조회:** @Aspect 어드바이저 빌더 내부에 저장된 Advisor 를 모두 조회한다. 
4. **프록시 적용 대상 체크**: <br>앞서 3-1, 3-2에서 조회한 **Advisor 에 포함되어 있는 포인트컷**을 사용해서 해당 객체가 **프록시를 적용할 대상**인지 아닌지 **판단**한다. <br>이때 객체의 클래스 정보는 물론이고, 해당 객체의 모든 메서드를 포인트컷에 하나하나 모두 매칭해본다. <br>그래서 조건이 하나라도 만족하면 프록시 적용 대상이 된다. 예를 들어서 메서드 하나만 포인트컷 조건에 만족해도 프록시 적용 대상이 된다.
5. **프록시 생성**: 프록시 적용 대상이면 프록시를 생성하고 프록시를 반환한다. 그래서 프록시를 스프링 빈으로 등록한다. 만약 프록시 적용 대상이 아니라면 원본 객체를 반환해서 원본 객체를 스프링 빈으로 등록한다.
6. **빈 등록**: 반환된 객체는 스프링 빈으로 등록된다.










# 스프링 AOP 개념

## 1.핵심기능과 부가기능

애플리케이션 로직은 **핵심 기능**과 **부가 기능**으로 나눌 수 있다.

- **핵심기능** : 객체가 제공하는 고유의 기능 ex) OrderService의 주문로직
- **부가기능** : **핵심 기능을 보조**하기 위해 제공되는 기능

예시로 

주문 로직을 실행하기 직전에 로그 추적 기능을 사용해야 하면, <br>**핵심 기능**인 주문 로직과 **부가 기능**인 로그 추적 로직이 **하나의 객체 안에 섞여** 들어가게 된다. <br>부가 기능이 필요한 경우 이렇게 둘을 합해서 하나의 로직을 완성한다.



#### 횡단에서 사용되는, 여러 곳에서 공통으로 사용하는 부가 기능

![image-20220711162302242](https://user-images.githubusercontent.com/58017318/178517002-788b6629-472d-48a0-9011-d1f2c40312c9.png)


여러 클래스에서 공통으로 사용되는 부가기능과 핵심기능이 섞여들어가 있는 경우가 발생한다.<br>이는 필터나 인터셉터로는 횡단해서 로직을 적용할 수 없기때문에 Froxy객체를 만들어주는 AOP기능을 활용한다.

----------

## 2. AOP 소개 - 애스펙트

횡단으로 적용되어있는 부가기능과(Advice), 부가기능을 어떤 로직에 적용할 지 선택하는기능(PointCut)을<br>합쳐서 만들어놓았는데 이것이 바로 **애스팩트(@Aspect)**이다.

**@Aspect**는 애스펙트를 사용한 **관점** 지향 **프로그래밍** **AOP(Aspect-Oriented Programming)**이라 한다.

참고로 **AOP는 OOP를 대체하기 위한 것이 아니라 횡단 관심사를 깔끔하게 처리**하기 어려운 **OOP의 부족한 부분을 보조**하는 목적으로 개발되었다.
![image-20220711163557042](https://user-images.githubusercontent.com/58017318/178517010-89f99983-6d3a-4d13-9352-7d1065495175.png)

AOP의 대표적인 구현으로 **AspectJ 프레임워크**가 있다.

AspectJ 프레임워크는 스스로를 다음과 같이 설명한다. <br>- 자바 프로그래밍 언어에 대한 완벽한 **관점** 지향 **확장** <br>- **횡단 관심사**의 깔끔한 모듈화 <br>- 오류 검사 및 처리 <br>- 동기화 <br>- 성능 최적화(캐싱) <br>- 모니터링 및 로깅

------------

## 3. AOP 적용 방식

 AOP를 사용할 때 부가 기능 로직은 어떤 방식으로 실제 로직에 추가될 수 있을까?

1. 컴파일 시점
2. 클래스 로딩 시점
3. 런타임 시점(프록시)



### 1. 컴파일 시점

![image-20220711164555342](https://user-images.githubusercontent.com/58017318/178517015-6be113c9-e3c3-4eec-9542-29ef6a24d764.png)

AspectJ 컴파일러는 Aspect를 확인해서 해당 클래스가 적용 대상인지 먼저 확인하고,<br>적용대상인 경우, java 소스 코드를 컴파일러를 사용해서 .class 를 만드는 시점에 부가 기능 로직을 추가할 수 있다.<br>이렇게 원본 로직에 부가 기능 로직이 추가되는 것을 **위빙**(Weaving)이라 한다.

- 위빙(Weaving): 옷감을 짜다. 직조하다. 애스펙트와 실제 코드를 연결해서 붙이는 것

단점은 **컴파일 시점에 부가 기능을 적용**하려면 **특별한 컴파일러도 필요하고 복잡**하다.



#### 2. 클래스 로딩 시점

![image-20220711165330277](https://user-images.githubusercontent.com/58017318/178517019-125b40cb-4b0f-42d4-b000-619da71811e9.png)
.class 를 JVM에 저장하기 전에 조작할 수 있는 기능을 제공한다. <br>참고로 수 많은 모니터링 툴들이 이 방식을 사용한다. <br>이 시점에 애스펙트를 적용하는 것을 **로드 타임 위빙**이라 한다. 

단점은 로드 타임 위빙은 자바를 실행할 때 **특별한 옵션**`( java -javaagent )`을 통해 **클래스 로더 조작기를 지정**해야 하는데, 이 부분이 번거롭고 운영하기 어렵다.










#### 3. 런타임 시점

<img width="648" alt="image-20220711221245715" src="https://user-images.githubusercontent.com/58017318/178516363-dcb8520d-fc83-4a82-aba0-a49d430520e7.png">

런타임 시점은 컴파일도 다 끝나고, 클래스 로더에 클래스도 다 올라가서 이미 자바가 실행되고 난 다음을 말한다. <br>**자바의 메인( `main` ) 메서드가 이미 실행된 다음**이다. <br>따라서 자바 언어가 제공하는 범위 안에서 부가 기능을 적용해야 한다. **스프링과 같은 컨테이너**의 도움을 받고 **프록시와 DI, 빈 포스트** 프로세서 같은 개념들을 총 동원해야 한다. <br>이렇게 하면 최종적으로 프록시를 통해 스프링 빈에 부가 기능을 적용할 수 있다. 그렇다. 지금까지 우리가 학습한 것이 바로 프록시 방식의 AOP이다.

프록시를 사용하기 때문에 AOP 기능에 일부 제약이 있다. 하지만 특별한 컴파일러나, 자바를 실행할 때 복잡한 옵션과 클래스 로더 조작기를 설정하지 않아도 된다. 스프링만 있으면 얼마든지 AOP를 적용할 수 있다.

**부가 기능이 적용되는 차이를 정리하면 다음과 같다**.

- **컴파일 시점**: 실제 대상 코드에 애스팩트를 통한 부가 기능 호출 코드가 포함된다. AspectJ를 직접 사용해야 한다.
- **클래스 로딩 시점**: 실제 대상 코드에 애스팩트를 통한 부가 기능 호출 코드가 포함된다. AspectJ를 직접 사용해야 한다.
- **런타임 시점**: 위 두가지는 바이트코드를 직접 뜯어 수정하여 적용하는것이지만, <br>런타임 시점은 **실제 대상 코드는 그대로 유지**된다. 대신에 **프록시**를 통해 **부가 기능이 적용**된다. <br>따라서 항상 프록시를 통해야 부가 기능을 사용할 수 있다. 스프링 AOP는 이 방식을 사용한다.



### 결론

**스프링이 제공하는 AOP**는 **프록시를 사용**한다. <br>따라서 **프록시를 통해 메서드를 실행**하는 시점에만 **AOP가 적용**된다. <br>AspectJ를 사용하면 앞서 설명한 것 처럼 더 복잡하고 더 다양한 기능을 사용할 수 있다. <br>그렇다면 스프링 AOP 보다는 더 기능이 많은 AspectJ를 직접 사용해서 AOP를 적용하는 것이 더 좋지 않을까? <br><br>AspectJ를 사용하려면 공부할 내용도 많고, 자바 관련 설정(특별한 컴파일러, AspectJ 전용 문법, 자바 실행 옵션)도 복잡하다.<br> 반면에 **스프링 AOP**는 별도의 **추가 자바 설정 없이 스프링만 있으면 편리하게 AOP를 사용**할 수 있다. <br>실무에서는 스프링이 제공하는 AOP 기능만 사용해도 대부문의 문제를 해결할 수 있다. <br>따라서 스프링 AOP가 제공하는 기능을 학습하는 것에 집중하자.

-----

# AOP 용어 정리

<img width="647" alt="image-20220711230358588" src="https://user-images.githubusercontent.com/58017318/178516487-8245c3d1-ab89-4156-9f98-bf4b4f39641e.png">
<img width="647" alt="image-20220711230427768" src="https://user-images.githubusercontent.com/58017318/178516508-3f613f63-29c0-4ddd-b51f-081b15c70c68.png">

- **조인 포인트(Join point)**
  - 어드바이스가 적용될 수 있는 위치, 메소드 실행, 생성자 호출, 필드 값 접근, static 메서드 접근 같은 프로그램 실행 중 지점
  - 조인 포인트는 추상적인 개념이다. AOP를 적용할 수 있는 모든 지점이라 생각하면 된다.
  - 스프링 AOP는 프록시 방식을 사용하므로 조인 포인트는 항상 메소드 실행 지점으로 제한된다.
- **포인트컷(Pointcut)**
  - 조인 포인트 중에서 어드바이스가 적용될 위치를 선별하는 기능
  - 주로 AspectJ 표현식을 사용해서 지정
  - 프록시를 사용하는 스프링 AOP는 메서드 실행 지점만 포인트컷으로 선별 가능
- **타켓(Target)**
  - 어드바이스를 받는 객체, 포인트컷으로 결정
- **어드바이스(Advice)**
  - 부가 기능
  - 특정 조인 포인트에서 Aspect에 의해 취해지는 조치
  - Around(주변), Before(전), After(후)와 같은 다양한 종류의 어드바이스가 있음
- **애스펙트(Aspect)**
  - 어드바이스 + 포인트컷을 모듈화 한 것 
  - @Aspect 를 생각하면 됨
  - 여러 어드바이스와 포인트 컷이 함께 존재
- **어드바이저(Advisor)**
  - 하나의 어드바이스와 하나의 포인트 컷으로 구성 스프링 AOP에서만 사용되는 특별한 용어
- **위빙(Weaving)**
- 포인트컷으로 결정한 타켓의 조인 포인트에 어드바이스를 적용하는 것 
- 위빙을 통해 핵심 기능 코드에 영향을 주지 않고 부가 기능을 추가 할 수 있음 
- AOP 적용을 위해 애스펙트를 객체에 연결한 상태
  - 컴파일 타임(AspectJ compiler)
  - 로드 타임
  - 런타임, 스프링 AOP는 런타임, 프록시 방식
- **AOP 프록시**
  - AOP 기능을 구현하기 위해 만든 프록시 객체, 스프링에서 AOP 프록시는 JDK 동적 프록시 또는 CGLIB 프록시이다.
