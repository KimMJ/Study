# Spring MVC 적용

## `HttpServletRequest`

* 검색 결과는 세션이 아닌 `HttpServletRequest` 객체에 저장해야 한다.
* `HttpServletRequest` 는 클라이언트의 요청으로 생성됐다가 응답 메시지가 클라이언트로 전송되면 자동으로 삭제되는 일회성 객체이다.

## `ModelAndView`

* `ModelAndView` 는 이름에서도 알 수 있듯이 `Model` 과 `View` 정보를 모두 저장하여 리턴할 때 사용된다.
* `DispatcherServlet` 은 `Controller` 가 리턴한 `ModelAndView` 객체에서 `Model` 정보를 추출한 다음 `HttpServletRequest` 객체에 검색 결과에 해당하는 `Model` 정보를 저장하여 JSP로 포워딩한다.
* 따라서 JSP 파일에서는 검색 결과를 세션이 아닌 `HttpServletRequest` 로부터 꺼내 쓸 수 있다.

## `ViewResolver` 활용 하기

* `ViewResolver` 를 사용하면 클라이언트로부터의 직접적인 JSP 호출을 차단할 수 있어서 대부분의 웹 프로젝트에서 `ViewResolver` 사용은 거의 필수적.
* `/WEB-INF` 폴더 내부로 JSP 파일을 이동시키면, 클라이언트는 접근이 불가능하고, `Controller` 는 접근이 가능.
* `web.xml` 에서 `security` 설정을 주는 것이 더 좋을듯.

##### View 이름 앞에 `redirect:` 를 붙이면 `ViewResolver` 가 설정 되어 있더라도 이를 무시하고 Redirect를 한다.
