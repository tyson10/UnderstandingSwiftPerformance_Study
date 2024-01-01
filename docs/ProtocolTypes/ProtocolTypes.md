# Protocol Types

![Untitled](Image/Untitled.png)

- Point, Line이 class 가 아닌 struct로 작성된 경우, 클래스처럼 상속 기반 다형성이 아니므로 vtable이 필요 없다.

그래서 Protocol Witness Table(PWT)을 사용함.

![Untitled](Image/Untitled%201.png)

![Untitled](Image/Untitled%202.png)

- drawables의 element에 접근하여 draw()를 호출하면 해당 프로토콜의 PWT를 타고 실제 구현된 내용까지 도달한다.

### 값 타입 배열

struct는 값 타입이기 때문에 배열에 레퍼런스가 아닌 실제 값이 들어가야 한다. 하지만 특정 프로토콜에 대한 배열은 각 element 별로 크기가 다를 수 있다.
ex) `var drawables: [Drawable]`
배열은 element들을 고정된 크기 안에 저장하기 때문에 문제가 생기는데 이런 경우를 위해서 Extential Container라는 특수한 storage layout을 사용한다.

![Untitled](Image/Untitled%203.png)

- 컨테이너의 처음엔 valueBuffer 3개가 할당된다.

![Untitled](Image/Untitled%204.png)

- `Point`의 경우 buffer를 2만큼만 사용하므로 컨테이너 내부에 값이 저장될 수 있다.

![Untitled](Image/Untitled%205.png)

- `Line`은 buffer를 4개 필요로 하기 때문에 컨테이너 내에 저장되지 못하고 Heap 메모리에 저장된다.
- 해당 Heap 메모리에 대한 포인터를 valueBuffer에 저장한다.

## Value Witness Table(VWT)

### allocate

![Untitled](Image/Untitled%206.png)

- 타입마다 Existential Container를 관리하는 Value Witness Table을 가진다.

![Untitled](Image/Untitled%207.png)

- VWT에는 allocate함수가 있으며, Line의 경우에는 Heap 메모리를 할당하고 valueBuffer에 해당 Heap 메모리의 포인터를 저장하는 동작을 한다.
- Point의 경우 위 작업이 필요하지 않다.(valueBuffer에 들어가므로..)

### copy

![Untitled](Image/Untitled%208.png)

- copy 함수는 실제로 값을 메모리에 저장시킨다.
- Line의 경우 valueBuffer에 저장된 포인터를 따라 이미 할당된 Heap 메모리에 접근하고, 값을 복사한다.
- Point의 경우 valueBuffer에 직접 값을 복사한다.

### destruct

![Untitled](Image/Untitled%209.png)

- 변수의 lifetime이 끝나면 Swift는 VWT의 destruct 함수를 실행시킨다.
- Line의 경우 해당 함수가 호출되면 할당된 Heap 메모리에 대한 레퍼런스 카운트를 감소시킨다.

### deallocate

![Untitled](Image/Untitled%2010.png)

- destruct이후 deallocate를 호출하며 Heap메모리 할당을 해제한다.
- Point의 경우 할당된 Heap메모리가 없으므로 값을 지운다.

## Existential Container

![Untitled](Image/Untitled%2011.png)

![Untitled](Image/Untitled%2012.png)

- Existential Container마다 vwt, pwt를 가지고 있다.
- vwt에서 저장프로퍼티들을 관리하고, pwt에서 프로토콜 메소드를 관리한다.

### 예제

![Untitled](Image/Untitled%2013.png)

- 위 코드를 실행하면 Swift는 Existential Container를 생성한다.

![Untitled](Image/Untitled%2014.png)

- drawACopy함수를 호출하면 Existential Container를 Argument로 전달한다.
- 매개변수에 대한 로컬 변수(local)가 만들어지고 argument가 할당된다.

![Untitled](Image/Untitled%2015.png)

- local의 필드들을 val로 전달된 값으로 초기화 한다.
- vwt와 pwt모두 Heap 메모리에 할당된다.

![Untitled](Image/Untitled%2016.png)

- Point의 경우 buffer크기 미만이라 Heap 할당이 필요하지 않으므로, 바로 값이 복사된다.

![Untitled](Image/Untitled%2017.png)

- Line의 경우 buffer크기보다 커서 Heap 메모리를 할당하고, 값을 복사한다.

![Untitled](Image/Untitled%2018.png)

- draw()를 호출하는 내용은 Swift에서 pwt의 draw를 호출한다.
- projectBuffer는 draw 동작에 사용할 값들의 주소값이다.
    - Point의 경우 container 시작 주소
    - Line의 경우 Heap 메모리에 value가 할당된 주소

![Untitled](Image/Untitled%2019.png)

- draw() 함수의 수행이 완료되고 나면 local이라는 변수는 scope를 벗어나게 되므로 vwt의 destruct를 수행한다.

![Untitled](Image/Untitled%2020.png)

- 레퍼런스 카운트를 감소시키고

![Untitled](Image/Untitled%2021.png)

- buffer에 할당된 값을 deallocate한다.

위와 같은 방식으로 struct가 다형성을 지원할 수 있게된다.

## 정리

1. Point와 Line이 Class 였을때 dynamic dispatch가 일어남.
    1. draw()가 호출되었을때 어떤 draw가 호출되어야 하는지 모르므로 런타임에 vtable을 조회해서 찾아서 호출함
2. 반대로 Point와 Line이 struct일때 protocol을 사용해서 상속구조를 만들 수 있으며, 이 경우에도 draw()함수를 호출할 때 직관적으로 어떤걸 호출해야 할 지 알 수 없다.(static dispatch 불가능)
3. 그래서 existential container를 Stack에 만든다. 해당 컨테이너는 아래와 같이 구성된다.
    1. inline valueBuffer로 3개의 공간
    2. value witness table(vwt)의 주소값 → Heap에 할당
    3. protocol witness table(pwt)의 주소값 → Heap에 할당
4. vwt의 4가지 엔트리중 allocate(메모리 공간 할당)와 copy(값 복사)를 사용해서 값을 할당함.
Line의 경우 valueBuffer에 들어가지 않으므로 Heap 메모리 할당 및 값 복사를 하고, Point 같은 경우에는 valueBuffer에 들어가므로 메모리 할당은 필요 없고 값 복사를 한다.
    1. allocate
    2. copy
    3. destruct
    4. deallocate
5. draw() 함수를 호출할 때 해당 existential container 내부의 pwt를 참조하여 draw의 구현의 위치를 찾는다.
6. draw()가 끝나면 vwt의 destruct를 호출하여 Heap 메모리 할당이 된 경우에는 레퍼런스 카운트를 감소시킨다.
7. deallocate를 호출해서 Heap메모리 할당을 해제하고, stack 메모리(existential container에 대한 메모리)도 해제 시킨다.(스택 포인터 증가시켜서 해제)

## Copy on Write

![Untitled](Protocol%20Types%20d4346a3578b64f78a40a9963381eca01/Untitled%2022.png)

- Pair라는 구조체 내부에 Drawable 변수 두개가 선언되어 있다.
- 하나는 Line, 하나는 Point를 할당한다.
- pair 내부에는 existential container 두개가 저장되고 Line은 Heap 메모리까지 할당된다.

![Untitled](Protocol%20Types%20d4346a3578b64f78a40a9963381eca01/Untitled%2023.png)

- Line의 경우 크기가 커서 Heap 할당이 되므로 비용이 크다.
- copy에 값을 대입하는 경우 Line은 구조체이기 때문에 Heap 메모리에 있는 내용이 복사가 된다. → 새로운 Heap메모리 할당이 계속 이루어짐 → 비용 상승

![Untitled](Protocol%20Types%20d4346a3578b64f78a40a9963381eca01/Untitled%2024.png)

- Line을 클래스로 만들면 이런 복사에 대한 비용을 줄일 수 있음.

![Untitled](Protocol%20Types%20d4346a3578b64f78a40a9963381eca01/Untitled%2025.png)

- 하지만 위와 같이 의도치 않은 상태 공유가 될 수 있다.

![Untitled](Protocol%20Types%20d4346a3578b64f78a40a9963381eca01/Untitled%2026.png)

- 궁극적으론 값이 바뀌었을땐 위와 같이 동작해야 하는데 그것을 Copy on Write로 해결할 수 있음.

![Untitled](Protocol%20Types%20d4346a3578b64f78a40a9963381eca01/Untitled%2027.png)

- Line에 대한 데이터를 가지고 있는 LineStorage 클래스를 선언함
- Line은 LineStorage를 프로퍼티로 가지고 있음

![Untitled](Protocol%20Types%20d4346a3578b64f78a40a9963381eca01/Untitled%2028.png)

- Pair의 두 프로퍼티에 동일한 인스턴스를 대입해도, copy라는 변수에 pair를 대입해도 LineStorage는  클래스이기기 때문에 복사되지 않고 레퍼런스를 전달받는다.
즉, 동일한 인스턴스를 가리킨다.

![Untitled](Protocol%20Types%20d4346a3578b64f78a40a9963381eca01/Untitled%2029.png)

- 다만, Line의 move() 함수를 통해 값을 변경하면 storage가 두 개 이상의 참조 카운트를 가지고 있으면, 값을 복사하도록 한다.
→ 값 변경이 다른 참조된 내용과 공유되지 않게 한다.
→ 값 변경이 있을때만 복사를 하고, 그렇지 않으면 같은 인스턴스를 사용하게 한다.
- 이와 같은 방식으로 복사 비용을 줄일 수 있다.

## 결론

![Untitled](Protocol%20Types%20d4346a3578b64f78a40a9963381eca01/Untitled%2030.png)

- existential container의 valueBuffer에 들어가는 작은 크기의 프로토콜 타입은 Heap메모리 할당이 없으며, 레퍼런스 카운팅도 없기 때문에 아주 빠르다.
- 프로토콜 타입은 PWT를 사용한 Dynamic dispatch를 사용한다.

![Untitled](Protocol%20Types%20d4346a3578b64f78a40a9963381eca01/Untitled%2031.png)

- 반면, valueBuffer에 들어가지 않는 큰 크기의 프로토콜 타입은 Heap 할당이 생기며 비용이 크다.

![Untitled](Protocol%20Types%20d4346a3578b64f78a40a9963381eca01/Untitled%2032.png)

- 이런 큰 크기의 프로토콜 타입은 Indirect storage를 사용한 CoW 기법으로 할당에 대한 비용을 줄일 수 있다.

![Untitled](Protocol%20Types%20d4346a3578b64f78a40a9963381eca01/Untitled%2033.png)
