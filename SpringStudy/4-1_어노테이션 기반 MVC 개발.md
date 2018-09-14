# 어노테이션 기반 MVC 개발

* 어노테이션을 통해 과도한 XML 설정으로 인한 문제를 해결.
* `HandlerMapping`, `Controller`, `ViewResolver` 같은 여러 클래스를 등록해야 함.

## 어노테이션 관련 설정

* Spring MVC 에서 어노테이션을 사용하려면, 먼저 `<bean>` 루트 엘리먼트에 context 네임스페이스를 추가.
* 그리고 `HandlerMapping`, `Controller`, `ViewResolver` 클래스에 대한 `<bean>` 등록을 모두 삭제하고, `<context:component-scan>` 엘리먼트로 대체.

## `@Controller` 사용하기

* `Controller` 클래스를 모두 일일이 `<bean>` 등록할 필요 없이, 클래스 선언부 위에 `@Controller` 를 붙이면 된다.
* `@Component`를 상송한 `@Controller` 는 `@Controller` 가 붙은 클래스의 객체를 메모리에 생성하는 기능 제공.
* 단순히 객체를 생성만 하는 것이 아니라, `DispatcherServlet` 이 인식하는 `Controller` 객체로 만들어줌.

## `@RequestMapping` 사용하기

* `@RequestMapping` 을 이용하여 `HandlerMapping` 설정을 대체한다.
* `@RequestMapping` 을 메소드 위에 설정. (어떤 요청에 대해 실행될지.)

## 클라이언트 요청 처리

* 대부분 `Controller` 는 사용자의 입력 정보를 추출하여 VO(Value Object) 객체에 저장.
* 비즈니스 컴포넌트의 메소드를 호출할 때 VO 객체를 인자로 전달.

## `Command` 객체

* 사용자의 입력 정보는 `HttpServletRequest` 의 `getParameter()` 메소드로 추출할 수 있음.
* 그러나 사용자 입력 정보가 많으면 그만큼의 자바 코드가 필요하고, 입력 정보가 변경될 때마다 `Controller` 클래스는 수정이 되어야 한다.
* 이 때 `Command` 객체를 이용하면 해결할 수 있다.
* `Command` 객체는 `Controller` 메소드 매개변수로 받은 VO(Value Object) 객체라고 볼 수 있다.
* `HandlerMapping` 에 등록한 메소드의 매개변수로 사용자가 입력한 값을 매핑할 클래스를 선언하면, 스프링 컨테이너가 해당 메소드를 실행할 때 `Command` 객체를 생성하여 넘겨줌.
* 이 때, `Command` 객체에 세팅까지 해서 넘겨줌.
* 결과적으로 사용자 입력 정보 추출, VO 객체 생성, 값 설정을 모두 컨테이너에서 자동으로 처리.

### 구동 방식

1. 서블릿 컨테이너는 클라이언트의 HTTP 요청이 서버에 전달되는 순간
2. `HttpServletRequest` 객체를 생성하고 HTTP 프로토콜에 설정된 모든 정보를 추출하여 `HttpServletRequest` 객체에 저장.
3. 그리고 이 `HttpServletRequest` 객체를 `service()` 메소드를 호출할 때, 인자로 전달해줌.

### 클라이언트가 글 등록을 요청했다면

1. 매개변수에 해당하는 객체를 생성
2. 사용자가 입력한 파라미터의 값들을 추출하여 객체에 저장. 이 때, 클래스의 setter 메소드들이 호출됨.
3. 메소드를 호출할 때, 사용자 입력값들이 설정된 객체가 인자로 전달됨.

이 때 주의해야할 점은 Form 태그 안의 파라미터 이름과 `Command` 객체의 setter 메소드 이름이 반드시 일치해야 한다.
