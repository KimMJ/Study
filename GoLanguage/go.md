# Go language

## 컴파일

```sh
go build
```

## package

* 모든 Go 프로그램은 패키지로 구성되어 있다.
* 프로그램은 main 패키지에서부터 실행을 시작.
* 패키지 이름은 디렉토리 경로의 마지막 이름을 사용하는 것이 규칙.

## import

* 여러개의 package를 소괄호로 감싸서 import를 표현 가능.
* 여러개의 import 문장으로도 표현 가능

```go
import "fmt"
import "math"
```

```go
import (
  "fmt"
  "math"
)
```

## export

* package를 import하면 외부로 export한 메서드, 변수, 상수 등에 접근할 수 있다.
* Go에서는 첫 문자가 대문자로 시작하면 그 패키지를 사용하는 곳에서 접근할 수 있는 exported name이 됨.
* Foo와 FOO는 참조 가능하지만 foo는 불가능하다.

## Function

* 매개변수의 타입을 변수 이름 뒤에 명시함.

```go
package main

import "fmt"

fun add(x int, y int) int {
  return x + y
}

func main() {
  fmt.Println(add(42,13));
}
```

* 두 개 이상의 매개변수가 같은 타입이면 같은 타입을 취하는 마지막 매개변수에만 타입을 명시하고 나머지는 생략 가능.

```
x, y int
```

## Multiple Results

* 하나의 함수는 여러개의 결과를 반환 가능.

```go
package main

import "fmt"

func swap(x, y string) (string, string) {
  return y, x
}

func main() {
  a, b := swap("hello", "world")
  fmt.Println(a, b)
}
```

## Named Results

* Go에서 함수는 여러 개의 결과를 반환 가능.
* 반환 값에 이름을 부여하면 변수처럼 사용할 수도 있음.
* 결과에 이름을 붙히면, 반환 값을 지정하지 않은 return 문장으로 결과의 현재 값을 알아서 반환

```go
package main

import "fmt"

func split(sum int) (x, y int) {
  x = sum * 4 / 9
  y = sum - x
  return
}

func main() {
  fmt.Println(split(17))
}

```

## 변수

* 변수 선언은 var로 한다.
* 함수의 매개변수처럼 타입은 문장 끝에 명시.

```go
package main

import "fmt"

var x, y, z int
var c, python, java bool

func main() {
  fmt.Println(x, y, z, c, python, java)
}
```

* 변수 선언과 함께 변수 각각을 초기화할 수 있음.
* 초기화 하는 경우 타입을 생략 가능.
* 변수는 초기화 하고자 하는 값에 따라 타입이 결정됨.

```go
package main

import "fmt"

var x, y, z int = 1, 2, 3
var c, python, java = true, false, "no!"

func main() {
  fmt.Println(x, y, z, c, python, java)
}
```

* 함수 내에서 `:=` 를 사용하면 var과 명시적인 타입을 생략할 수 있음.
* 하지만 함수 밖에서는 `:=` 를 사용할 수 없음

```go
package main

import "fmt"

func main() {
  var x, y, z int = 1, 2, 3
  c, python, java := true, false, "no!"

  fmt.Println(x, y, z, c, python, java)
}
```

* 상수는 const 키워드와 함께 변수처럼 선언 가능.
* 상수는 character, string, boolean, int 타입 중 하나가 될 수 있음

```go
package main

import "fmt"

const Pi = 3.14

func main() {
  const world = "안녕"
  fmt.Println("Hello", world)
  fmt.Println("Happy", Pi, "Day")

  const Truth = true
  fmt.Println("Go rules?", Truth)
}
```

* 숫자형 상수는 정밀한 값을 표현할 수 있음.
* 타입을 지정하지 않은 상수는 문맥에 따라 타입을 가짐.

## 반복문

* For밖에 없음.
* 기본적인 형태는 C나 Java와 비슷.
* 소괄호는 쓰지 않음.

```go
package main

import "fmt"

func main() {
  sum := 0
  for i := 0; i < 10; i ++ {
    sum += i
  }
  fmt.Println(sum)
}
```

* C와 Java처럼 전.후 처리를 제외하고 조건문만 표현 가능.
* 이 방식으로 while문 사용하는 for를 사용할 수 있음.

```go
package main

import "fmt"

func main() {
  sum := 1
  for sum < 1000 {
    sum += sum
  }
  fmt.Println(sum)
}
```

* 조건문을 생략하면 무한루프 간단하게 표현 가능.

```go
package main

func main() {
  for {

  }
}
```

## 조건문

* if문은 C, Java와 비슷.
* 조건 표현을 위해 ()는 사용하지 않음.
* 하지만 {}는 사용.


```go
package main

import {
  "fmt"
  "math"
}

func sqrt(x float64) string {
  if x < 0 {
    return sqrt(-x) + "i"
  }
  return fmt.Sprint(math.Sqrt(x))
}

func main() {
  fmt.Println(sqrt(2), sqrt(-4))
}
```

* 조건문 앞에 짧은 문장 실행 가능
* 짧은 실행문을 통해 선언된 변수는 if-else scope 내에서만 사용 가능.

```go
package main

import (
  "fmt"
  "math"
)

func pow(x, n, lim float64) float64 {
  if v := math.Pow(x, n); v < lim {
    return v
  }
  return lim
}

func main() {
  fmt.Println(
    pow(3, 2, 10),
    pow(3, 3, 20),
  )
}
```

## 기본 자료형

* Go의 기본 자료형들

```go
bool

string

int   int8  int16   int32   int64
uint  uint8 uint16  uint32  uint64  uintptr

byte // uint8의 다른 이름(alias)

rune // int32의 다른 이름(alias)
     // 유니코드 코드 포인트 값을 표현

float32 float64

complex64 complex128  
```

```go
package main

import (
    "fmt"
    "math/cmplx"
)

var (
    ToBe   bool       = false
    MaxInt uint64     = 1<<64 - 1
    z      complex128 = cmplx.Sqrt(-5 + 12i)
)

func main() {
    const f = "%T(%v)\n"
    fmt.Printf(f, ToBe, ToBe)
    fmt.Printf(f, MaxInt, MaxInt)
    fmt.Printf(f, z, z)
}

>>
bool(false)
uint64(18446744073709551615)
complex128((2+3i))
```

## Structs

* 필드(데이터)들의 조합
* type 선언으로 struct의 이름을 지정할 수 있음.

```go
package main

import "fmt"

type Vertex struct {
  X int
  Y int
}

func main() {
  fmt.Println(Vertex{1, 2})
}
```

* 구조체에 속한 필드(데이터)는 dot으로 접근

```go
package main

import "fmt"

type Vertex struct {
  X int
  Y int
}

func main() {
  v := Vertex{1, 2}
  v.X = 4
  fmt.Println(v.X)
}

>> 4
```
