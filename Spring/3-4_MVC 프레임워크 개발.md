# MVC 프레임워크 개발

## MVC 프레임워크 구조

* 앞서 DispatcherServlet 하나로 Controller의 기능을 구현했다.
* 이러한 방식은 하나의 서블릿으로 Controller를 구현하게 되어 클라이언트의 모든 요청을 하나의 서블릿이 처리하게 된다.
* 즉 수많은 분기 로직을 가질수밖에 없음. 오히려 개발과 유지보수를 어렵게 만듬.
* Controller를 서블릿 클래스 하나로 구현하는 것은 여러 측면에서 문제가 있다.
* 다양한 디자인 패턴을 결합하여 개발과 유지보수의 편의성이 보장되도록 해야 한다.
* 이 때, 프레임워크에서 제공하는 Controller를 사용하면 우리가 직접 구현하지 않아도 된다.
* Spring MVC는 DispatcherServlet을 시작으로 다양한 객체들이 상호작용하면서 클라이언트의 요청을 처리한다.

| 클래스            	| 기능                                                                                  	|
|-------------------	|---------------------------------------------------------------------------------------	|
| DispatcherServlet 	| 유일한 서블릿 클래스로서 모든 클라이언트의 요청을 가장 먼저 처리하는 Front Controller 	|
| Handler Mapping   	| 클라이언트의 요청을 처리할 Controller 매핑                                            	|
| Controller        	| 실질적인 클라이언트의 요청 처리                                                       	|
| ViewResolver      	| Controller가 리턴한 View 이름으로 실행될 JSP 경로 완성                                	|

## MVC 프레임워크 구현

* DispatcherServlet은 클라이언트의 요청을 가장 먼저 받아들이는 Front Controller이다.
* 하지만 실질적인 요청 처리는 각 Container에서 한다.
* 처리 순서
  1. 클라이언트가 요청을 하면 DispatcherServlet이 요청을 받는다.
  2. DispatcherServlet은 HandlerMapping 객체를 통해서 요청을 처리할 Controller를 검색한다.
  3. 검색된 Controller에서 요청을 처리하고 그 다음 View 정보를 리턴한다.
  4. DispatcherServlet은 ViewResolver를 통해 View를 검색한다.
  5. 최종적으로 View에 해당하는 JSP를 실행하고 실행결과가 브라우저로 응답된다.

---

* Controller를 구성하는 클래스를 모두 개발하고 나면, 너무나 복잡한 구조와 수많은 클래스 때문에 오히려 더 혼란스러울 수도 있다.
* 그럼에도 불구하고 복잡한 구조를 사용하는 이유는 DispatcherServlet클래스가 유지보수 과정에서 기존의 기능을 수정하거나 새로운 기능을 추가하더라도 절대 수정되지 않기 때문이다.
