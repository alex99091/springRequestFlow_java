# Spring의 Request 처리 Flow

Spring MVC 기반의 웹 애플리케이션은 사용자가 보낸 요청(Request)을 처리하고, 결과를 응답(Response)으로 보내기까지 다양한 단계를 거칩니다. 이 과정은 개발자 입장에선 "흐름을 정확히 이해하면 문제 해결이나 확장 개발이 쉬워지는 핵심 개념"입니다. 여기서는 Spring의 요청 처리 구조를 쉽게 풀어보고, 어떤 방식으로 확장되었는지도 함께 설명드리겠습니다.

---

## ✅ Spring의 Request 처리 흐름, 한눈에 보기

Spring에서 하나의 요청이 처리되는 과정은 아래 그림처럼 크게 **선처리 → 컨트롤러 → 후처리** 영역으로 나눌 수 있습니다:

```
[Request]
   ↓
[Filter]         → 요청 정보 전처리 (보안, 인코딩 등)
   ↓
[Interceptor]    → 요청 흐름 제어, 인증 처리, 로깅 등
   ↓
[Argument Resolver] → Controller에 들어갈 파라미터를 세팅
   ↓
[Controller]     → 실제 비즈니스 로직 수행
   ↓
[ReturnValue Handler] → Controller 결과를 Response 형태로 가공
   ↓
[View Resolver]  → 사용자에게 보여줄 화면 결정
   ↓
[Response]
```

이 흐름은 Spring Boot로 웹 개발을 하면서 디버깅하거나 기능을 확장할 때 반드시 이해하고 있어야 하는 구조입니다.

---

## 🔹 선처리 영역 설명

### 1. Filter

* Java 표준 서블릿 기능으로, **가장 먼저 실행되는 컴포넌트**입니다.
* 예: `CharacterEncodingFilter`로 요청의 인코딩을 UTF-8로 강제 설정할 수 있습니다.
* WAS 단계에서 실행되며, 보안 필터나 로깅에도 사용됩니다.

```java
@WebFilter("/*")
public class LogFilter implements Filter {
    public void doFilter(...) {
        System.out.println("요청 발생: " + request.getRequestURI());
        chain.doFilter(request, response);
    }
}
```

### 2. Interceptor

* Spring의 DispatcherServlet 이후 실행됩니다.
* 요청 전/후 처리를 위한 **preHandle, postHandle, afterCompletion** 메서드를 제공합니다.
* 예: 로그인 인증, URL별 권한 검사 등을 이곳에서 처리합니다.

```java
public boolean preHandle(...) {
    if (userNotLoggedIn) {
        response.sendRedirect("/login");
        return false;
    }
    return true;
}
```

### 3. Argument Resolver

* 컨트롤러의 파라미터(예: `@RequestParam`, `@ModelAttribute`)를 분석해 자동으로 값을 바인딩해주는 기능입니다.
* 복잡한 DTO 매핑이나 커스텀 로직이 필요한 경우 직접 구현할 수도 있습니다.

---

## 🟡 Controller

비즈니스 로직이 실제로 실행되는 구간입니다. 예를 들어 상품 조회, 회원가입, 데이터 저장 같은 주요 기능이 여기에 위치합니다.

```java
@GetMapping("/product")
public Product getProduct(@RequestParam Long id) {
    return productService.findById(id);
}
```

---

## 🔻 후처리 영역 설명

### 4. ReturnValue Handler

* 컨트롤러가 리턴한 데이터를 적절한 형식으로 변환합니다.
* 예: `@ResponseBody`로 리턴된 객체를 JSON 문자열로 변환하는 것도 여기서 처리됩니다.

### 5. View Resolver

* 컨트롤러가 리턴한 View 이름(`"mainPage"`)을 실제 화면 파일(`mainPage.jsp`)로 연결해주는 컴포넌트입니다.
* REST API 기반 서비스에서는 거의 사용되지 않고, 전통적인 MVC 웹 프로젝트에서 많이 쓰입니다.

---

## 🏗️ HFW+에서는 어떻게 확장했을까?

사내에서 사용하는 HFW+는 위의 Spring 구조를 바탕으로 다음과 같은 방식으로 확장되었습니다.

### 📌 Filter

* 공통 인코딩 처리, 보안 필터 외에도 업무용 특화 필터가 존재합니다.
* web.xml이나 설정 파일을 통해 업무별 필터를 정의합니다.

### 📌 Interceptor

* 업무 구분에 따라 여러 개의 인터셉터가 등록되어 있습니다.

  * 예: `FixedLengthInterceptor`, `JsonInterceptor`, `XPlatformInterceptor` 등
* 각 인터셉터는 URI 매핑 기준으로 작동하며, 요청 내용을 변환하거나 검증하는 데 사용됩니다.

### 📌 Argument Resolver

* 사내 공통 DTO인 `RequestDTO` 형태를 자동으로 파싱합니다.
* 예: JSON → 객체 바인딩, 인증 정보 자동 주입, LocalContext 연동 등

### 📌 ReturnValue Handler

* 모든 응답을 `{code: 200, msg: '성공', data: ...}` 형태로 통일하기 위한 처리기가 존재합니다.

### 📌 View Resolver

* 요청 URL 패턴에 따라 JSON, TEXT, FIXED-LENGTH 등 다양한 View Resolver가 자동 작동합니다.

---

## ☕ Java 8 이후로 바뀐 점은?

초기에는 Java 1.6 기반이었지만 Java 8로 업그레이드되면서 다음 기능들이 HFW+에 반영되었습니다:

| 항목         | Java 6 | Java 8 이상                  |
| ---------- | ------ | -------------------------- |
| 람다식        | 불가능    | 간결한 람다 기반 콜백 처리 가능         |
| Optional   | 없음     | 널 처리에 안전한 Optional 도입      |
| Stream API | 없음     | 리스트 필터링/매핑 등 간결한 데이터 처리 가능 |
| LocalDate  | 사용 불편  | 시간 처리 편의성 향상               |

---

## ✅ 마무리 요약

* Spring의 요청 흐름은 필터 → 인터셉터 → 컨트롤러 → 응답 핸들러 → 뷰 리졸버 순으로 진행됩니다.
* HFW+는 이를 커스터마이징해 재사용성과 일관성을 높였으며, 업무 특화 기능을 공통 처리 컴포넌트로 정리했습니다.
* Java 8로 업그레이드되면서 HFW+ 내 코드 품질과 유지보수성이 크게 향상되었습니다.

👉 실무에서 이 구조를 이해하면 디버깅, 신규 기능 확장, 장애 대응이 훨씬 쉬워집니다!
