# CombineSummary

## Chapter 1. Hello Combine

### [Publisher]

- 값을 방출하는 것. (추후 subscriber 들이 값을 받게 됨)
- complete 되면 값 방출 안함
- 에러 핸들링이 내장 되어 있음

### [Operator]

- Publisher protocol 따름
- upstream 과 downstream 으로 되어있어 shared data를 방지함

### [Subscriber]

- 모든 subscription은 subscriber로 끝남.
- 방출된 값이나 결과로 "무언가"를 함
- 두가지 내장 subscriber 가 있음
  - sink - output value 와 completion 을 받음
  - assign - output을 property와 bind 함

### [Subscription]

- 책에서는 subscription을 두가지를 설명하는데 사용함
  - Subscrption protocol을 따르는 것
  - publisher - operators - subscriber 의 완전한 체인
- subscription의 맨 마지막에 subscriber을 붙이면 publisher를 "활성화" 시킴. 
- 즉 subscriber가 없으면 publisher는 아무 값도 방출하지 않음
- Full-Combine으로 앱이 구성되어있다면 앱의 로직을 subscription들로 설명 가능
- Cancellable 덕분에 메모리 관리에 신경쓸 필요 없음
- subscription (publisher - operators - subscriber 의 완전한 체인) 은 Cancellable을 return 하는데, 해당 객체가 해제되면 subscription도 취소 되고 리소스도 해제됨
- [AnyCancellable] 이라는 collection에 subcription들을 담아 해당 프로퍼티가 취소 될때 안에 담긴 subscription을 취소되게 할 수도 있음

### [Chapter 1 - Key points]

- Combine은 시간에 따른 비동기 이벤트를 처리하기 위한 framework

## Chapter 2. Publishers & Subscribers

### [Hello Publisher]

- Publisher protocol은 한개 혹은 그 이상의 subscriber에 보내기 위한 값의 type을 정의

### [Hello Subscriber]

- Subscriber protocol은 publisher로부터 받을 값의 type을 정의
- `sink` 연산자는 두가지 closure 를 제공하는데, 하나는 완료 이벤트 용, 다른 하나는 값 핸들링 용.
- `assign(to:,on:)` 연산자는 받은 값을 object의 프로퍼티에 할당할 수 있게 함

### [Hello Cancellable]

- Subscriber 의 동작이 끝나고 더 이상 값을 받지 않기를 원할 때, subscription을 취소하는 것은 리소스를 해제하고 연관된 동작도 중지 시킬 수 있는 좋은 아이디어 (👩🏻‍💻 cancel 하는 것이 completion event 를 보내는 것은 아님)
- AnyCancellable은 `cancel()` 이 구현되어있는 Cancellable을 conform.
- subscription에 명시적으로 cancel 을 호출하지 않으면 publisher가 완료 되거나, 메모리 해제 될때까지 계속 된다.
- subscription을 메모리에 저장하지 않으면 정의된 scope 벗어나는 순간 cancel 됨. 

### [Understanding what's going on]

```swift
public protocol Publisher {
  associatedtype Output
  associatedtype Failure : Error

  // 4
  func receive<S>(subscriber: S)
    where S: Subscriber,
    Self.Failure == S.Failure,
    Self.Output == S.Input
}

extension Publisher {
  // 3
  public func subscribe<S>(_ subscriber: S)
    where S : Subscriber,
    Self.Failure == S.Failure,
    Self.Output == S.Input
}
```

- 3. subscriber가 publisher에 attach 하기 위해  `subscribe(_:)` 호출
- 4. `subscribe(_:)` 구현부는 `receive(subsribe:)` 를 호출하는데 이곳에서는 subcription을 만듦

``` swift
public protocol Subscriber: CustomCombineIdentifierConvertible {
  associatedtype Input
  associatedtype Failure: Error

  // 3
  func receive(subscription: Subscription)
  func receive(_ input: Self.Input) -> Subscribers.Demand
  func receive(completion: Subscribers.Completion<Self.Failure>)
}
```

- 3. publisher는 subscsriber에게 만든 subscription을 주기 위해  `receive(subscription:)` 을 호출
- 즉 publisher 와 subscriber의 연결 고리는 subscription

```swift
public protocol Subscription: Cancellable, CustomCombineIdentifierConvertible {
  func request(_ demand: Subscribers.Demand)
}
```

- Subscriber가 얼마나 많은 값을 받을지 결정하기 위해 `request(_:)` 를 호출.
- 이렇게 받을 값의 수를 결정하는 것을 backpressure management 라고 함
- Subscriber의 `receive(_)` 에서 받을 값을 조정하는 것은 더하기로 동작함.

👩🏻‍💻 구독 동작 순서

Publisher protocol: `publisher.subscribe(subscriber)` 

Publisher protocol: `receive(subscriber:)`  => 역할: subscription 생성

Subscriber protocol: `receive(subscription:)`

Subscription protocol: `request(_:) ` => 역할 : max 조절 (backpressure management)

### [Hello Future]

- Future 은 비동기적으로 단일 결과 값을 만들고 종료하는데 이용할 수 있음
- Future은 단일 값 방출 후 종료되거나, fail 되는 publisher 인데, 이는 값이 준비 되었을 때 promise라고 부르는 closure을 호출함으로써 동작.

```swift
final public class Future<Output, Failure> : Publisher
  where Failure: Error {
  public typealias Promise = (Result<Output, Failure>) -> Void
  ...
}
```

- Promise는 type alias로서, Result를 받는 closure를 지칭
- Future 은 promise 를 재실행하지 않고 대신 결과를 공유하거나 재생(replay) 함
- Future은 만들어지자마자 실행됨. 즉, 다른 publisher와 다르게 subsucriber 가 필요 없음

### [Hello Subject]

- Passthrough subject는 새로운 값을 원할때마다 publish 할 수 있음
- `.store(in: &subscription)` 은 구독을 `subscription` set에 넣겠다는건데, 이때 `&subscription` 과 같이 inoput 파라미터로 전달됨. 따라서 복사가 일어나는 대신 같은 set이 update 될 수 있음
- Passthrough subject와 다르게 CurrentValue subject는 `subject.value` 처럼 현재 값을 호출 가능
-  CurrentValue subject에서 값을 전달하는 방법은 `send(_:)` 를 호출하는 것과 value property에 새 값을 할당하는 것 두가지

### [Type Eraser]

- `.eraseToAnyPublisher()` 을 통해 type erased publisher을 만들 수 있음
- AnyPublisher은 Publisher 을 conform 하는 type erased struct
- AnyCancellable을 Cancellable을 conform 하는 type erased class
- publisher의 type erasure을 사용하고 싶을때는 public - private 쌍으로 된 프로퍼티에서, 안쪽에서는 private publisher에 값을 보내고, 바깥쪽에서는 public publisher에 접근해서 구독하고 싶을때
- AnyPublisher에는 `send(_:)` 연산자가 없기 때문에 새로운 값이 publisher에 더해질 수 없음 

## Chapter 3. Transforming Operators

- operator는 publisher
- publisher로부터 들어오는 값들에 동작을 수행하는 것을 operator 라고 함
- operator는 publisher를 반환

### [Collecting values]

- `collect()`
  - publisher에서 방출되는 각각의 값을 하나의 array로 모음
  - upstream publisher가 complete 될때 동작
  - 아니면 `collect(2)` 같이 몇개의 값을 받아 묶을 건지 정할 수 있음
  - `collect()`  나 다른 buffering operator 와 같이 개수 제한이 없는 연산자를 사용하면 수신된 값을 저장하기 위해 메모리가 무한히 늘어날 수 있기 때문에 주의 필요. 

### [Mapping values]

- `map(_:)`
  - Swift의 standard map 처럼 동작
  - map 계열의 연산자는 key path를 이용해서 한가지 / 두가지 / 세가지 값을 매핑할 수 있는 세 버전이 존재.
    - `map<T>(_:)`
    - `map<T0, T1>(_:,_:)`
    - `map<T0, T1, T2>(_:,_:,_:)`
- `tryMap(_:)`
  - `map` 을 포함한 여러 operator 들은 상응하는 try operator를 가지는데, 여기서는 error 를 throw 할 수 있음 
  - 에러가 throw 되면 해당 에러를 downstream 으로 보냄
  - throw를 호출할 때는 try 키워드 사용 필요

### [Flattening Publishers]

- `flatMap(maxPublishers:_:)`

  - 여러개의 upstream publisher 들을 하나의 downstream publisher로 flatten (정확히는 이러한 publisher 들의 방출 값들을 flatten 하는 것)

    애플 문서: upstream publisher의 모든 요소를 지정한 최대 publishers 수까지 새로운 publisher로 변환

  - 주로 Publisher 들을 값으로 방출하는 Publisher 를 구독할 때 사용

  - 들어오는 모든 publisher 들을 버퍼에 저장하기 때문에 메모리 이슈가 있을 수 있음  

  - maxPublishers 파라미터를 통해 받을 publisher의 개수를 정할 수도 있음. 디폴트는 `.unlimited`

    `flatMap { $0.message }`

    `flatMap(maxPublishers: .max(2)) { $0.message }`

### [Replacing upstream output]

- `replaceNil(with:)`

  - optional 값을 받아서 그 중 nil을 지정한 값으로 대체

  - nil을 non-nil 로 바꿀 뿐, optional을 non-optional로 만드는 것은 아님. 

    `["A", nil] -> [Optional("A"), Optional("specifiedValue")]`

  - `??` 연산자는 또 다른 optional을 반환 가능하지만 replaceNil 은 불가

    `.replaceNil(with: "-" as String?)` 하면 optional must be unwrapped 라는 에러 받을 것 

- `replaceEmpty(with:)`

  - 만약 publisher가 아무 값도 방출하지 않고 끝난다면 값을 대체 (정확히는 insert)
  - 참고로 Empty publisher은 demo나 테스팅 목적으로 좋음

### [Incrementally transforming outout]

- `scan(_:_:)`
  - upstream publisher에서 방출된 현재 값과, 해당 closure에서 반환된 마지막 값을 같이 보냄
  - 비슷하게 동작하는 `tryscan` 이라는 error throwing 연산자도 있음. 만약 클로저에서 에러 방출하면 `tryscan`은 해당 에러와 함께 fail 함

## Chapter 4. Filtering Operators

### [Filtering basics]

- `filter`
  - 조건에 매칭되는 값만 통과시킴
- `removeDuplicates`
  - argument 패스 안해도 됨
  - `Equatable` conform 하는 값들에 대해 자동으로 적용 됨
  - `Equatable` 을 conform 하지 않는 값들에 대해서는 다른 overload 함수 이용해서 동일 여부 나타내는 bool 값 리턴하면 됨 

### [Compacting and ignoring]

- `compactMap`
  - nil 값은 제거하고 non-nil 값으로 republish 
- `ignoreOutput`
  - 오직 completion 이벤트만을 방출 
  - 어떤 값이, 얼마나 많이 방출되었는지 신경쓰지 않음

### [Finding values]

- `first(where:)`
  - 조건에 맞는 첫번째 값 방출
  - lazy 함. 즉, 조건에 맞는 첫번째 값이 나올때까지 모든 값들을 처리하다가, 만약 첫번째 값이 나오면 subscription을 취소하고 완료함. 이로써 upstream은 더 이상 값을 방출하지 않음.
- `last(where:)`
  - 조건에 맞는 마지막 값 방출
  - `first(where:)` 과는 반대로 greedy 함. 조건에 매칭되는 값이 있나 판단하기 위해 모든 값들을 기다림. 따라서 upstream publisher는 언젠가 완료되어야 함.

### [Dropping values]

- `dropFirst`
  - default가 1인 `count` parameter 받음
  - 처음 `count` 만큼 값을 무시 
- `drop(while:)`
  - 처음으로 조건이 false 가 될때까지의 값 무시
- `drop(untilOutputFrom)`
  - 두번째 publisher (인자로 받는 것) 에서 최소 한개의 값을 방출하기 전까지는 upstream에서 방출되는 값 무시
  - 만약 isReady 라는 publisher에서 값을 방출하기 전까지, 유저의 버튼 클릭을 무시하고 싶을때 사용하기 알맞은 연산자

### [Limiting values]

- 조건이 맞을때까지 값을 받은 후 publisher를 완료함

- `prefix(_:)`
  - `dropFirst` 와는 반대로 특정 개수만큼만 받고 complete
  - `first(where:)` 처럼 lazy 함. 즉, 필요한만큼만 값을 받고 완료함
- `prefix(while:)`
  - 조건이 true 일때까지 값을 받음
  - 조건이 false가 되면 publisher는 완료
- `prefix(untilOutputFrom:)`
  - `drop(untilOutputFrom)` 과는 반대로 second publisher에서 값을 방출할때까지 값을 받음

### [Chapter 3- Key points]

- 값은 신경쓰지 않고 완료만 파악하고 싶을때 `ignoreOutput` 이 적당
- First-style 연산자는 lazy 함. 필요한만큼만 값을 받고 완료함. 
- Last-style 연산자는 greedy 함. 전체 범위만큼의 값을 알아야함.
- `drop` family 연산자를 이용해서 필요한 만큼 값을 무시할 수 있음
- `prefix` family 연산자를 이용해서 필요한 만큼만 값을 받을 수 있음.

## Chapter 5. Combining Operators

- 사용자의 이름, 비밀번호, 체크 박스 등의 데이터를 결합해서 하나의 publisher로 합치고 싶을때 유용

### [Prepending]

- original publisher 이전에 값을 추가

- `prepend(Output...)`

  - 인자로 값의 variadic list 를 받음

  - original publisher와 같은 Output 타입이면 얼마든지 값을 받을 수 있음

  - 마지막 prepend 가 첫번째로 upstream에 적용된다

    ```swift
    publisher
    	.prepend(1, 2)
    	.prepend(-1, 0)
    	.sink { print($0) }
    
    //-1, 0, 1, 2 ....
    ```

- `prepend(Sequence)`

  - 인자로 Sequnce를 conform하는 객체를 받음
  - ex) Array, Set, Stridable

- `prepend(Publisher)`

  - 인자로 Publisher로 받음
  - 인자로 받은 publisher에서 방출되는 값을 original publisher 이전에 추가
  - 앞에 추가된 publisher가 반드시 완료 되고난 후에야 original publisher에서 마저 값을 방출할 수 있음

### [Appending]

- prepend 와 비슷하게 동작

- `append(Output...)`
  - 인자로 값의 variadic list 를 받음
  - original publisher가 완료되고 난 후에 받은 값들을 뒤에 붙임
  - upstream 은 반드시 완료되어야함. 그렇지 않으면 이전 publisher가 값을 모두 방출했는지 알지 못하기 때문에 appending 이 동작하지 않음. 
  - 이 속성은 모든 append operator family에게 적용됨. 즉 original publisher가 completion 을 보내지 않는한 appending은 동작하지 않음
- `append(Sequence)`
  - 인자로 Sequnce를 conform하는 객체를 받아서 original publisher에서 모든 값을 방출하고 나면 값을 append 함
- `append(Publisher)`
  - 인자로 Publisher로 받아서 original publisher에서 모든 값을 방출하고 나면 값을 append 함

### [Advanced Combining]

- `switchToLatest`

  - 기존 publisher의 구독을 취소하고, 최신으로 들어온 publisher로 모든 구독 전환

  - publisher을 방출하는 publisher에만 사용할 수 있음 (`let myPublisher = PassthroughSubject<PassthroughSubject<Int, Never>, Never>()`)

  - 만약 `myPublisher` 에 새로운 publisher를 방출하면 기존의 구독은 취소하고 새로운 것으로 전환됨

  - `myPublisher` 에 complete를 방출하면 활성화 되어있는 모든 구독도 complete

  - 만약 사용자가 네트워크 요청을 트리거하는 버튼을 탭했다고 가정. 동일 버튼을 다시 탭해서 두번째 네트워크 요청을 트리거. 만약 첫번째 요청은 무시하고 마지막 요청만 사용하고 싶을때 switchToLatest는 좋은 선택지

    ```swift
    let taps = PassthroughSubject<Void, Never>()
    
    taps
      .map { _ in getImage() }
      .switchToLatest()
      .sink(receiveValue: { _ in })
      .store(in: &subscriptions)
    
    taps.send()
    taps.send()
    ```

- `merge(with:)`

  - 다른 publisher에서 방출하는 값(동일 type)들을 교차로 배치(interleave)
  - Combine은 publisher를 8개까지 merge할 수 있는 overload 제공

- `combineLatest`

  - 다른 type 값을 방출하는 publisher들을 결합
  - 아무 publisher에서 값을 방출할때마다, 모든 publisher의 마지막 값을 tuple로 묶어서 방출
  - 따라서 모든 publisher는 적어도 하나의 값을 방출해야 combineLatest가 동작함
  - publisher를 4개까지 결합할 수 있는 overload 제공

- `zip`

  - Swift 표준 라이브러리에 있는 zip 과 동작이 유사
  - publisher들에서 같은 index에 있는 값들을 tuple로 방출
  - 각 publisher가 값을 방출하기를 기다리다가 현재 index에 해당하는 값을 모든 publisher에서 방출하면 하나의 tuple로 묶어서 방출
  - 만약 다른 publisher에서 짝지어 사용할 값이 없다면 무시됨 