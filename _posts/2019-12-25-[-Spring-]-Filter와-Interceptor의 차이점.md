리팩토링 스터디를 진행하면서, 공통 기능을 적용할 때 어떤 방법으로 구현하는게 좋을지에 대한 고민이 생겼다.  
그래서 차이점에 대해 한번 알아봤다(AOP는 약간 결이 다른 것 같아서 제외).  
포스트를 작성하면서 정리해보긴 했지만 여전히 헷갈린다...ㅠㅠㅠ

## Filter vs Interceptor

- Filter와 Interceptor는 실행되는 시점이 다르다.
- Filter는 Web Application에 등록을 하고, Interceptor는 Spring의 Context에 등록을 한다.

### 예외처리

- Filter에서 예외가 발생하면 Web Application에서 처리해야 한다. 
tomcat을 사용한다면 \<error-page>를 잘 선언하던가 아니면 Filter 내에서 예외를 잡아 request.getRequestDispatcher(String)으로 예외 처리를 미뤄야 한다
- 하지만 Interceptor에서 예외가 발생하면? Interceptor의 실행 시점을 보자, Spring의 ServletDispatcher 내에 있다. 
즉 @ControllerAdvice에서 @ExceptionHandler를 사용해서 예외를 처리를 할 수 있다.
- 만약 작성해야할 전후처리 로직에서 예외를 전역적으로 처리하고 싶다면 Interceptor를 사용하자.

### 시점
- Filter는 Servlet에서 처리하기 전후를 다룰 수 있다.
- Interceptor는 Handler를 실행하기전(preHandle), Handler를 실행한 후(postHandle), view를 렌더링한 후(afterCompletion 등, Servlet내에서도 메서드에 따라 실행 시점을 다르게 가져간다.

### Interceptor에서만 할 수 있는 것
- @RequestMapping 선언으로 요청에 대한 HandlerMethod(@Controller의 메서드)가 정해졌다면, handler라는 이름으로 HandlerMethod가 들어온다. HandlerMethod로 메서드 시그니처 등 추가적인 정보를 파악해서 로직 실행 여부를 판단할 수 있다.
- View를 렌더링하기 전에 추가 작업을 할 수 있다. 예를 들어 웹 페이지가 권한에 따라 GNB(Global Navigation Bar)이 항목이 다르게 노출되어야 할 때 등의 처리를 하기 좋다.

### Filter에서만 할 수 있는 것
- ServletRequest 혹은 ServletResponse를 교체할 수 있다

* * *

- 주로 전역적으로 무엇인가를 처리해야하는 로직은 필터로 구현을 하고(인코딩 및 보안관련 처리)
- 디테일한 처리는 인터셉터로 구현 한다.(인증, 권한)
<br>

- Controller 실행 시 Idp 객체 내 memberId에 인코딩을 적용하고자 한다면
    - 1. Filter로 DispatcherSerlvet를 타기 전 인코딩 적용하던가
    - 2. Interceptor preHandle를 이용해 Argument Resolver를 타기 전 인코딩 적용
    - preHandle -> IdpResolver(resolveArgument) -> afterCompletion



* * *

참고
https://jaehun2841.github.io/2018/08/10/2018-08-10-spring-argument-resolver/#argument-resolver-동작-방식
https://supawer0728.github.io/2018/04/04/spring-filter-interceptor/