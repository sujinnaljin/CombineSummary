# CombineSummary

[Publisher]

- 값을 방출하는 것. (추후 subscriber 들이 값을 받게 됨)
- complete 되면 값 방출 안함
- 에러 핸들링이 내장 되어 있음

[Operator]

- Publisher protocol 따름
- upstream 과 downstream 으로 되어있어 shared data를 방지함

[Subscriber]

- 모든 subscription은 subscriber로 끝남.
- 방출된 값이나 결과로 "무언가"를 함
- 두가지 내장 subscriber 가 있음
  - sink - output value 와 completion 을 받음
  - assign - output을 property와 bind 함

[Subscription]

- 책에서는 subscription을 두가지를 설명하는데 사용함
  - Subscrption protocol을 따르는 것
  - publisher - operators - subscriber 의 완전한 체인
- subscription의 맨 마지막에 subscriber을 붙이면 publisher를 "활성화" 시킴. 
- 즉 subscriber가 없으면 publisher는 아무 값도 방출하지 않음
- Full-Combine으로 앱이 구성되어있다면 앱의 로직을 subscription들로 설명 가능
- Cancellable 덕분에 메모리 관리에 신경쓸 필요 없음
- subscription (publisher - operators - subscriber 의 완전한 체인) 은 Cancellable을 return 하는데, 해당 객체가 해제되면 subscription도 취소 되고 리소스도 해제됨
- [AnyCancellable] 이라는 collection에 subcription들을 담아 해당 프로퍼티가 취소 될때 안에 담긴 subscription을 취소되게 할 수도 있음

[Chapter 1 - Key points]

- Combine은 시간에 따른 비동기 이벤트를 처리하기 위한 framework

[Hello Publisher]

- Publisher protocol은 한개 혹은 그 이상의 subscriber에 보내기 위한 값의 type을 정의

[Hello Subscriber]

- Subscriber protocol은 publisher로부터 받을 값의 type을 정의
- `sink` 연산자는 두가지 closure 를 제공하는데, 하나는 완료 이벤트 용, 다른 하나는 값 핸들링 용.
- `assign(to:,on:)` 연산자는 받은 값을 object의 프로퍼티에 할당할 수 있게 함

[Hello Cancellable]

- Subscriber 의 동작이 끝나고 더 이상 값을 받지 않기를 원할 때, subscription을 취소하는 것은 리소스를 해제하고 연관된 동작도 중지 시킬 수 있는 좋은 아이디어 (👩🏻‍💻 cancel 하는 것이 completion event 를 보내는 것은 아님)
- AnyCancellable은 `cancel()` 이 구현되어있는 Cancellable을 conform.
- subscription에 명시적으로 cancel 을 호출하지 않으면 publisher가 완료 되거나, 메모리 해제 될때까지 계속 된다.
- subscription을 메모리에 저장하지 않으면 정의된 scope 벗어나는 순간 cancel 됨. 

[Understanding what's going on]

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

[Hello Future]

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

[Hello Subject]

- Passthrough subject는 새로운 값을 원할때마다 publish 할 수 있음
- `.store(in: &subscription)` 은 구독을 `subscription` set에 넣겠다는건데, 이때 `&subscription` 과 같이 inoput 파라미터로 전달됨. 따라서 복사가 일어나는 대신 같은 set이 update 될 수 있음
- Passthrough subject와 다르게 CurrentValue subject는 `subject.value` 처럼 현재 값을 호출 가능
-  CurrentValue subject에서 값을 전달하는 방법은 `send(_:)` 를 호출하는 것과 value property에 새 값을 할당하는 것 두가지

[Type Eraser]

- `.eraseToAnyPublisher()` 을 통해 type erased publisher을 만들 수 있음
- AnyPublisher은 Publisher 을 conform 하는 type erased struct
- AnyCancellable을 Cancellable을 conform 하는 type erased class
- publisher의 type erasure을 사용하고 싶을때는 public - private 쌍으로 된 프로퍼티에서, 안쪽에서는 private publisher에 값을 보내고, 바깥쪽에서는 public publisher에 접근해서 구독하고 싶을때
- AnyPublisher에는 `send(_:)` 연산자가 없기 때문에 새로운 값이 publisher에 더해질 수 없음 


