# Callback Function

* JavaScript에서 function은 일급 객체(Object)임.
* 즉, 다른 일급 객체(String, Array, Number...)와 똑같이 사용가능.
* 다시 말해 Object이므로 변수 안에 담을수도 있고 인수로서 다른 함수에 전달할수도 있고, 함수에서 만들어질수도 있고 반환될수도 있다.
* 여기서 Callback Function은 특정 함수에 매개변수로서 전달된 함수.
* 전달받은 함수는 그 Callback function을 함수 안에서 호출함.
* 가장 아래에 있는 함수를 실행했다고 해서 프로그램이 끝난 것이 아니라 모든 Callback function들이 다 끝나고 나서 프로그램이 종료된다.
* 따라서 blocking 코드를 사용하는 서버보다 더 빠르게 요청 처리 가능.
