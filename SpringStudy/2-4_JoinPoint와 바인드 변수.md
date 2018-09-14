# JoinPoint와 바인드 변수

* 예외가 발생한 비즈니스 메소드 이름이 무엇인지, 그 메소드가 속한 클래스와 패키지 정보는 무엇인지 알아야 정확한 예외 처리 로직을 구현할 수 있음.
* 스프링에서는 이런 다양한 정보들을 이용할 수 있도록 JoinPoint 인터페이스 제공.

## JoinPoint 메소드

| 메소드                   	| 설명                                                                                              	|
|--------------------------	|---------------------------------------------------------------------------------------------------	|
| Signature getSignature() 	| 클라이언트가 호출한 메소드의 시그니처(리턴타입, 이름, 매개변수) 정보가 저장된 Signature 객체 리턴 	|
| Object getTarget()       	| 클라이언트가 호출한 비즈니스 메소드를 포함하는 비즈니스 객체 리턴                                 	|
| Object[] getArgs()       	| 클라이언트가 메소드를 호출할 때 넘겨준 인자 목록을 Object 배열로 리턴                             	|

* 주의해야할 점은 Before, After Returning, After Throwing, After 어드바이스에서는 JoinPoint를 사용해야 하고, 유일하게 Around 어드바이스에서만 ProceedingJoinPoint를 매개변수로 사용해야 한다.
* 이는 Around 어드바이스에서만 proceed() 메소드가 필요하기 때문.
* JoinPoint 객체를 사용하려면 단지 JoinPoint를 어드바이스 메소드 매개변수로 선언만 하면 된다.

## Before

* Before 어드바이스는 비즈니스 메소드가 실행되기 전에 동작할 로직 구현.
* 호출된 메소드 시그니처만 알 수 있으면 다양한 사전 처리 로직 구현 가능.
* 매개변수로 JoinPoint 선언

## After Returning

* After Returning은 비즈니스 메소드가 수행되고 나서, 결과 데이터를 리턴할 때 동작하는 어드바이스.
* 어떤 메소드가 어떤 값을 리턴했는지 알아야 사후처리 가능.
* 매개변수로 JoinPoint, Object 선언.
* 이 Object 타입의 변수를 바인드 변수라고 한다.
* 바인드 변수는 비즈니스 메소드가 리턴한 결괏값을 바인딩할 목적으로 사용.
* 어떤 값이 리턴될지 모르기 때문에 Object 타입으로 선언.
* xml에서 `<aop:after-returning>`의 returning 속성을 매개변수로 선언된 바인드 변수 이름으로 설정.

## After Throwing

* After Throwing은 비즈니스 메소드가 수행되다가 예외가 발생할 때 동작하는 어드바이스.
* 어떤 메소드에서 어떤 예외가 발생했는지 알아야 함.
* 매개변수로 JointPoint, Exception 선언.
* xml에서 `<aop:after-throwing>`의 throwing 속성을 매개변수로 선언한 바인드 변수 이름으로 설정.

## Around 어드바이스

* 다른 어드바이스들과는 다르게 ProceedingJoinPoint 객체를 매개변수로 받아야 한다.
* ProceedingJoinPoint 객체는 비즈니스 메소드를 호출하는 proceed()메소드를 가지고 있고, JoinPoint를 상속했다.
