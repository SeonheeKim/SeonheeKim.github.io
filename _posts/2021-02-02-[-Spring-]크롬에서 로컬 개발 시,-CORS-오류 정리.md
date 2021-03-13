### 크롬에서 로컬 개발 시, CORS 오류 정리  

- 신규 환경의 어드민 프로젝트에서 로컬 개발할 때, 오류로 헤맸던 부분에 대해 정리합니다.  
<br>

### 오류 발생!!  
Spring Boot + Gradle로 구성된 프로젝트이므로 초기 설정들을 intelliJ가 간편하게 해결해줬습니다.  
그런데 각 front-end, back-end 서버를 정상 구동한 후, <span style="color:#0052cc">**API 요청을 보내는 순간**</span> 오류가 발생했습니다.  

![preflight_request_error](https://raw.githubusercontent.com/SeonheeKim/SeonheeKim.github.io/master/content/images/2021-02-02/preflight_request_error.JPG)  

해석해보자면 프론트 서버(http://local-test.admin.com:3000 )에서 백엔드 서버( http://local-test.admin.com/api/~ )로  
보낸 요청이 `CORS policy`에 의해 차단되었다는 내용이었습니다.  
<br>

#### CORS 란?  
- Cross-Origin Resource Sharing(교차 출처 리소스 공유)의 약자로,   
출처가 다른 도메인에서의 요청이라도 서버 단에서 권한을 허용하면 리소스 접근 가능하도록 하는 정책  
<br>
- 기능적 개요 : 서버 데이터에 부수 효과(side effect)를 일으킬 수 있는 HTTP 요청에 대해, CORS 명세는 브라우저가 요청을 OPTIONS 메서드로 "프리플라이트"(preflight, 사전 전달)하여 지원하는 메서드를 요청하고, 서버의 "허가"가 떨어지면 실제 요청을 전송  
<br>

### CORS 허용 방법  
아 신규 어드민은 <span style="color:#0052cc">**프론트와 백엔드 서버의 포트가 달라서, same origin이 아니므로**</span>  
CORS 설정해서 서버단에서 접근을 허용해야 CORS policy에 위반되지 않는구나 하고 생각했습니다.  
<br>
그래서 CORS 설정 방법들을 시도해봤습니다.  
<br>

**방법 1. CorsRegistry 등록**  
- 웹 어플리케이션의 WebConfiguration 내 CorsRegistry를 등록하는 방법입니다.  
그런데 이미 기존 소스에도 WebConfiguration 내 CorsRegistry가 등록되어 있었습니다.  

``` java
@Configuration
public class WebConfiguration implements WebMvcConfigurer {
    ...
    
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
            .allowedOrigins("http://localhost:3000", 
                            "http://local-test.admin.com:3000",
                            "http://dev-test.admin.com",
                            "http://alpha-test.admin.com",
                            "http://beta- test.admin.com",
                            "http://test.admin.com")
            .allowedMethods("*")
            .allowedHeaders("*")
            .allowCredentials(true)
            .maxAge(3000);
    }
    
    ...
    
}
```

<br>

**방법 2. Controller에 @CrossOrigin 어노테이션 적용**  
- 해당 어노테이션은 CORS를 스프링을 통해 설정할 수 있는 기능입니다.  

``` java
@RestController
@RequestMapping
@CrossOrigin("http://local-test.admin.com:3000")
public class MileageApiController {
    ...
}
```

<br>

**방법 3. 프론트에서 API 호출할 때, proxy 기능 사용하기**

``` xml
module.exports = {
    devServer: {
        proxy: {
            '/api': {
                target: 'http://local-test.admin.com',
                changeOrigin: true,
                pathRewrite: { '^/api': '' },
            },
        }
    }
}
```

<br><br>
그.런.데 아무리 서버단에서 CORS를 설정해줘도 같은 오류가 발생했습니다.  
어째서인지 고민을 하다, 오류 메시지를 더 정확하게 확인해봤습니다.  
<br>
`Response to preflight request doesn't pass access control check: Redirect is not allowed for a preflight request.`
<br>

> Preflight Request : 실제 요청 전에 인증 헤더를 전송하여 서버의 허용 여부를 미리 체크하는 테스트 요청  

결국 해당 오류는 실제 요청 서버단 CORS 설정과 별개로,  
preflight 방식의 request 과정에서 200 응답이 아닌 <span style="color:#0052cc">**302 redirect 응답을 받게되는 점이 문제**</span>라고 생각했습니다.  

### 어째서 302 응답이...?  
그렇다면 어째서 302 응답이 온 것일까요? response header를 확인하고야 원인을 알게 됐습니다.  
<br>
![error_response](https://raw.githubusercontent.com/SeonheeKim/SeonheeKim.github.io/master/content/images/2021-02-02/error_response.JPG)  

URL을 가려서 확인이 어렵겠지만... location 부분을 확인해보니 <span style="color:#0052cc">**preflight request일 때도 어드민 인증 필터**</span>를 거치는데,  
인증 통과를 할 수 없는 요청이므로 error 페이지로 리다이렉트되고 있었던 것입니다.  
<br>

### 프론트 서버의 요청일 경우, Preflight Request 허용 설정  
원인이 파악되니 이제 해결 방법을 구체화할 수 있었습니다.  
어드민 인증 모듈에 프론트 서버에서의 요청은 CORS 설정 해주면 되는 것입니다!  
<br>
CORS 설정하는 방법은 위에서 많이 시도했는데,  
어드민 인증 모듈의 인증 filter가 타기 전에 설정되어야하므로 CORS filter를 Bean으로 등록하는 방법으로 구현했습니다.  

``` java
@Configuration
public class CertificationFilterConfiguration implements WebMvcConfigurer {
    ...
    
    @Bean
    public FilterRegistrationBean<CorsFilter> corsFilter() 
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowCredentials(true);
        config.addAllowedOrigin("http://local-test.admin.com:3000");
        config.addAllowedHeader("*");
        config.addAllowedMethod("*");
        config.setMaxAge(3000L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);

        FilterRegistrationBean<CorsFilter> filterRegistrationBean = new FilterRegistrationBean<>();
        filterRegistrationBean.setFilter(new CorsFilter(source));
        filterRegistrationBean.setOrder(0);
        return filterRegistrationBean;
    }
    
    ...

```
<br>
CORS filter를 등록한 후, 실제 동작을 확인해보니 아래와 같이 정상 응답을 받을 수 있었습니다.  

![success_response](https://raw.githubusercontent.com/SeonheeKim/SeonheeKim.github.io/master/content/images/2021-02-02/success_response.png)  
<br>

### 정리하자면...  
결국 원인은 Cross Origin일 경우 리소스 간의 접근을 허용해줘야하는데  
인증 필터로 인해 preflight request가 실패하고 있었던 것입니다.  
<br>
제 경우에는 자체 어드민 인증 모듈로 인해 발생했지만,  
검색해보니 Spring Security를 적용한 프로젝트에서도 발생하는 경우가 있었습니다.  
<br>

#### **추가적 내용**  
로컬 테스트를 성공적으로 마친 후, 한 가지 의문이 생겼습니다.  
왜 dev, alpha에선 CORS filter가 없어도 정상 동작이 되고 있는가였습니다.  
이미 리얼까지 반영되어 동작 중인 프로젝트인데도 말이죠...!   
<br>
확인해본 바로는 IE에서는 다른 브라우저와 다르게 동일 출처 정책 예외사항이 2가지가 존재했습니다.  
- 신뢰할 수 있는 사이트 : 양쪽 도메인 모두가 높음 단계의 보안 수준을 가지고 있는 경우  
동일 출처 제약을 적용하지 않습니다.  
- <span style="color:#0052cc">**포트 : Internet Explorer는 동일 출처 정책 검사에 포트를 포함하지 않습니다.**</span>

<br>
현재 어드민 환경 IE여서 포트가 다르더라도 동일 출처로 판단되어 CORS 이슈가 발생하지 않는 것이 아닐까 추측됩니다.  
실제로 CORS 필터를 이제거하고 IE에서 로컬 테스트하면 정상 동작되고 있었습니다.  
<br>

### 참고 내용
Same Origin 판단 예시  
- 기준 URL : http://local-test.admin.com  

| URL | 동일 여부 | 이유 |
| --- | --- | --- |
| http://local-test.admin.com/mileage | true | path만 다름 |
| http://local-test.admin.com/mileage/userHistory | true | path만 다름 |
| https://local-test.admin.com | false | 프로토콜이 다름 |
| http://local-test.admin.com:3000 | false | 포트가 다름 |
| http://admin.com | false | 호스트가 다름 |

> 참고 : SOP(Same-Origin policy 동일 출처 정책)은 다른 출처에서 가져온 리소스와의 상호 작용을 제한

<br><br>

### 참고 출처  
- https://velog.io/@jmkim87/지긋지긋한-CORS-파헤쳐보자  
- https://velog.io/@yejinh/CORS-4tk536f0db  
- https://vvshinevv.tistory.com/62  
- https://vvshinevv.tistory.com/61?category=861457  
- http://wiki.gurubee.net/display/SWDEV/CORS+%28Cross-Origin+Resource+Sharing%29  
- https://charsyam.wordpress.com/2020/06/23/tip-spring-boot-2-1-0-에서의-cors-설정  
- https://developer.mozilla.org/ko/docs/Web/Security/Same-origin_policy  
- https://evan-moon.github.io/2020/05/21/about-cors  
- https://developer.mozilla.org/ko/docs/Web/HTTP/CORS  
- https://www.popit.kr/cors-preflight-인증-처리-관련-삽질  
