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

  - upstream publisher에서 방출된 현재 값과, 해당 closure에서 반환된 마지막 값을 같이 보냄 (reduce랑 기능이 비슷하다고 보면 됨)

  ```swift
    (0...5).publisher
    .scan(0) { return $0 + $1 }
    .sink { print ("\($0)", terminator: " ") }
   // Prints: "0 1 3 6 10 15 ".
  ```
  
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

## Chapter6. Time Manipulation Operators

### [Shifting time]

- `delay(for:tolerance:scheduler:options)`

  - 모든 값의 시퀀스를 time-shift 함
  - Upstream Publisher 에서 값을 방출할 때마다, 해당 값을 delay 시간 이후에, 지정한 Scheduler 에서 방출

  > Note 
  >
  > - 예제에서 사용하는 `Timer` class는 Foundation 프레임워크의 Timer 클래스의 Combine 확장형.
  >
  > -  `DispatchQueue` 가 아닌 `RunLoop` 와 `RunLoop.Mode` 를 사용.
  > - timer는 connectable한 class. 즉, 값을 방출하기 전에 connect 되어야한다. 예제에서는 첫번째 subscription에서 즉시 connect되는  `.autoconnect()`  를 이용.
  > - 타이머에 대한 자세한 내용은 Chapter 11. Timer에서 확인 가능. 

### [Collecting Values]

- 특정 시간 간격으로 값을 수집하고 싶을 수 있음

- `collect(_:options:)`

  - 첫번째 인자에서 값을 grouping 하기 위한 정책을 accept
  - 아래의  `.byTime` strategy 는 '시간'을 기준으로 값을 모으는 것

  ```swift
  publisher.collect(.byTime(DispatchQueue.main, 
                            .seconds(4)))
           .flatMap { dates in dates.publisher } 
  ```

  - 위의 collect 에서 나오는 값은 이전 4초 동안 받은 값들의 배열
  - collect 가 수집한 값의 배열을 내보낼 때마다 flatMap은 다시 개별 값으로 분해 후, 동시에 모두 방출
  - `.byTimeOrCount` strategy는 수집되는 값의 개수를 제한

  ```swift 
  let collectTimeStride = 4
  let collectMaxCount = 2
  
  publisher.collect(.byTimeOrCount(DispatchQueue.main,
                                  .seconds(collectTimeStride),
                                  collectMaxCount))
           .flatMap { dates in dates.publisher } 
  ```
  - 이렇게 하면 maxCount 가 collect 되었을때랑, 일정 timeStride 지났을때 모두 emit

### [Holding off on events]

- `debounce`

  - 이벤트 사이에 특정 시간이 경과했을때 값을 방출

  ```swift
  subject.debounce(for: .seconds(1.0), scheduler: DispatchQueue.main).share()
  ```

  - debounce 는 upstream에서 값이 방출된 후에 1초 동안 다음 값이 안들어오면 해당 값을 방출 
  - debounce 된 것을 여러 곳에서 구독할 수 있는데, 이때 결과의 일관성을 보장하기 위해 share()을 이용. 이는 debounce에 대한 하나의 subscription point를 제공함으로써 모든 subscriber 에게 같은 값을 보여줌.

  > Note: share()은 다수의 subscriber에게 같은 결과를 제공하기 위해서 publisher에 대한 single subscription이 필요할때 유용. Chapter 13에서 더 자세히 배움

  - 유저가 두 단어 사이에 잠시 멈출 때가 debounce가 값을 방출할 때임

  ```swift 
  0.6s: Subject emitted: Hello
  1.6s: Debounced emitted: Hello
  2.1s: Subject emitted: Hello W
  2.1s: Subject emitted: Hello Wo
  ...
  2.7s: Subject emitted: Hello World
  3.7s: Debounced emitted: Hello World
  ```

  > Note: publisher의 completion 에 대해서는 주의 필요. 만약 publisher가 마지막 값을 방출한 후에 debounce에서 정의한 시간이 경과하기 전에  complete 되었다면, debounced publisher에서는 마지막 값을 볼 수 없음

- `throttle`

  - 특정 시간 간격동안 usptream에서 방출된 최신 혹은 처음의 값을 방출
  - debounce와 유사하지만 근본적인 차이점 존재
    - debounce : 값을 받는 것이 멈추길 기다린 다음, 지정된 시간 간격 후에 최신 값을 방출
    - throttle : 지정된 시간 간격 동안 대기한 다음, 해당 간격 동안 수신한 값의 첫 번째 또는 최신 값 방출. 값의 수신이 멈추는 것과는 상관 없음.

  ```swift 
  subject.throttle(for: .seconds(1.0), scheduler: DispatchQueue.main, latest: false)
  ```

  - throttled subject는 1초 간격으로 subject로부터 받은 첫번째 값을 방출. (latest를 false 로 설정해놨으므로)

   ```swift 
  0.0s: Subject emitted: H
  ...
  0.6s: Subject emitted: Hello
  1.0s: Throttled emitted: H
  2.2s: Subject emitted: Hello W
  ...
  2.7s: Subject emitted: Hello World
  3.0s: Throttled emitted: Hello World
   ```

### [Timing out]

- `timeout`

  - upstream publisher가 element를 방출하지 않고 지정된 시간 간격을 초과할 경우 종료
  - timeout 연산자가 실행되면 publisher 를 complete하거나 사용자가 지정한 error 방출하는데, 두 경우 모두 publisher는 종료됨

  ```swift 
  subject.timeout(.seconds(5), scheduler: DispatchQueue.main)
  ```

  - timedOut publisher는 5초 동안 upstream publisher에서 아무 값을 내보내지 않으면 timeout 됨
  - 위의 형식은 failiure 없이 publisher가 completion 되는 형태인데, `customError` 인자를 통해 커스텀 에러도 제공 가능

  ```swift
  subject.timeout(.seconds(5), scheduler: DispatchQueue.main, customError: { .timeOut })	
  ```


### [Measuring time]

- ` measureInterval(using:)`

  - publisher가 연속해서 내보내는 두 값 사이에 경과한 시간을 확인해야 하는 경우 사용. 시간 측정 용도.

  ```swift 
  subejct.measureInterval(using; DispatchQueue.main)
  ```

  ```swift 
  0.0s: Subject emitted: H
  0.0s: Measure emitted: Stride(magnitude: 16818353)
  0.1s: Subject emitted: He
  0.1s: Measure emitted: Stride(magnitude: 87377323)
  0.2s: Subject emitted: Hel
  0.2s: Measure emitted: Stride(magnitude: 111515697)
  ...
  ```

  - measureInterval이 방출하는 값의 type은 제공된 스케줄러의 `TimeInterval` 
  - DispatchQueue의  `TimeInterval`은 `DispatchTimeInterval`으로, nano seconds로 표현됨
  - DispatchQueue 대신 seconds 로 표현되도록 Runloop 같은 다른 스케쥴러를 사용할 수도 있음. DispatchQueue를 사용하는게 일반적으로 좋은 방법이긴 하지만, 이는 개인적인 선택에 달림.

## Chapter 7. Sequence Operators

- array 나 set 같은 Publisher 값의 collection 들로 작업
- 대부분 sequence 의 개별 값이 아니라 전체 값을 다룸

### [Finding Values]

- `min`

  - publisher가 방출한 가장 작은 값을 찾음
  - greedy 함. 즉, publisher가 `.finished` 완료 이벤트를 방출할때까지 기다림

  ```swift
  publisher.min()
  ```

  - 숫자값들은 Comparable protocol을 준수하기 때문에 combine은 최솟값을 알 수 있음
  - Comparable을 준수하는 값을 방출하는 publisher에 대해서는 인자 없이 min()  사용 가능
  - `min(by:)` 를 사용하여Comparable을 준수하지 않는 값에 대해서 comparator closure를 제공

  ```swift
  publisher.min(by: {$0.count < $1.count})
  ```

- `max`

  - 최댓값을 찾는다는 것만 제외하고는 `min` 과 유사하게 동작

  ```swift
  publisher.max()
  ```

  - greedy 함. 즉, publisher가 `.finished` 완료 이벤트를 방출할때까지 기다림
  - `max(by:)` 를 사용하여Comparable을 준수하지 않는 값에 대해서 comparator closure를 제공

- `first`

  - 첫번째 값을 방출시킨 후 즉시 완료 함
  - lazy 함. 즉, upstream publisher가 종료되길 기다리지 않고, 첫번째 값을 받자마자 subscription을 cancel

  ```swift
  publisher.first()
  ```

  - `first(where:)` 을 사용하여 조건에 맞는 첫번째 값을 방출할 수도 있음

  ```swift
  publisher.first(where: {"Hello world".contains($0)})
  ```

- `last`

  - 마지막 값을 방출
  - greedy 함. 즉 upstream publisher가 finish 될때까지 기다림

  ```swift
  publisher.last()
  ```

  - `last(where:)` 을 사용하여 조건에 맞는 마지막 값을 방출할 수도 있음

- `output(at:)`

  - 특정한 index에 있는 값을 방출

  ```swift
  ["A", "B", "C"].publisher
  .print("publisher")
  .output(at: 1)
  .sink { print("Value at index 1 is \($0)") }
  .store(in: &subscriptions)
  ```

  ```
  // 출력
  publisher: receive subscription: (["A", "B", "C"])
  publisher: request unlimited
  publisher: receive value: (A)
  publisher: request max: (1) (synchronous) //demands one more value
  publisher: receive value: (B)
  Value at index 1 is B
  publisher: receive cancel
  ```

  - 특정한 index의 값만 원하기 때문에, 각 방출마다 one more value를 요구(demand)함
  - 👩🏻‍💻 만약 `.output(at: 2)` 로 요청을 보냈다면 아래와 같이 max request가 조절됨

  ```
  publisher: receive value: (A)
  publisher: request max: (1) (synchronous)
  publisher: receive value: (B)
  publisher: request max: (1) (synchronous)
  ```

  - 👩🏻‍💻 만약 collection size 보다 큰 index를 요구하면 그냥 upstream publisher 가 complete 될때까지 아무 값도 방출하지 않는 것

- `output(in:)`

  - `output(at:)` 이 특정 index에 있는 단일 값을 방출했다면, `output(in:)` 은 특정 범위 (range)에 있는 값들을 방출

  ```swift
  ["A", "B", "C", "D", "E"]
      .output(in: 1...3)
      .sink(receiveCompletion: { print($0) },
            receiveValue: { print("Value in range: \($0)") })
      .store(in: &subscriptions)
  ```

  ```
  // 출력
  Value in range: B
  Value in range: C
  Value in range: D
  ```

  - 특정 범위에 속하는 값들을 개별적으로 방출하는거지, collection으로 방출하는게 아님
  -  👩🏻‍💻 만약 collection size 보다 큰 index가 범위로 들어오면, emit 할 수 있는 범위까지 방출하고 complete 되는 것
  - 원하는 모든 값을 받았으면 즉시 subscription을 취소(cancel)함
  ### [Querying the publisher]

- publisher로 부터 방출되는 값의 entire set 을 다룸. 하지만 publisher에서 방출하는 특정한 값을 내보내진 않고, publisher 전체에 대해 쿼리를 적용한 값을 내보냄

- `count`

  - upstream publisher가 `.finished` 완료 이벤트를 보내고 나면, 지금까지 얼마나 많은 값을 방출했는지를 나타내는 단일 값을 내보냄

  ```swift
  publisher.count()
  ```

  - 👩🏻‍💻 만약 에러로 complete되면 값 못받고 끝나는거

- `contains`

  - 특정 값이 upstream publisher로부터 방출되면 true를 방출 후 바로 subscription을 취소하고, 끝날 때까지 매치되는 값이 없으면 false를 방출

  ```swift
  publisher.contains("C")
  ```

  - lazy 함. 즉 작업을 수행하는 데 필요한 만큼의 업스트림 값만 소비함. 만약 원하는 값을 찾으면 supscription을 취소하고 이후 값들은 생산하지 않음
  - `contains(where:)` 을 사용하여 특정 조건과 일치하는 항목을 찾거나, Comparable protocol 을 준수하지 않는 방출 값들을 확인할 수 있음

  ```swift
  publisher.contains(whre: {$0.id == 800})
  ```

- `allSatisfy`

  - upstream publisher로 부터 방출된 모든 값이 조건에 부합하는지 나타내는 Boolean 값 방출
  - greedy 함. 즉, upstream publisher 가 `.finished` 완료 이벤트를 방출할 때까지 기다림

  ```swift
  publisher.allSatisfy { $0 % 2 == 0 }
  ```

  - 👩🏻‍💻 만약 에러로 complete되면 값 못받고 끝나는거
  - 만약 한개라도 조건을 통과하지 못하면 즉시 false 를 방출하고 subscription을 취소 

- `reduce`

  - upstream publisher 에서 방출된 값들을 기본으로, 새로운 값을 축적
  - seed value와 accumulator closure 를 제공. 
  - 이 closure는 seed value로 시작하는 accumulated value 와, current value 를 받음. 그리고 같은 type의 새로운 accumulated value를 return
  - `.finished` 이벤트를 받으면 최종 accumulated value 를 방출

  ```swift
  publisher.reduce(""){accumulator, value in accumulator + value}
  ```

  > Note: Chapter 3에 소개했던 scan 과 reduce 는 동일한 기능을 수행. 다만 scan은 값이 방출될 때마다 매번 같이 accumulated value를 방출하고, reduce는 upstream publisher가 .finished 완료 이벤트를 보냈을때 한번만 accumulated value를 방출
## Chapter 9. Networking

### [URLSession extensions]

- URLSession을 이용해 다양한 network 작업을 할 수 있음
  - url의 내용을 얻기 위한 데이터 전송 작업
  - 다운로드 & 저장 작업
  - 업로드 작업
  -  두 당사자 간의 데이터를 스트리밍을 위한 Stream 작업
  - 웹소켓 작업
- 그 중 Combine publisher는 첫번째에 해당하는 데이터 전송 작업에 대해서 제공 됨

```swift
URLSession.shared
	.dataTaskPublisher(for: url)
```

### [Codable Support]

- 기존에는 Data를 디코딩하려면 tryMap에서 JSONDecoder 사용

```swift
URLSession.shared
	.dataTaskPublisher(for: url)
	.tryMap { data, _ in
          	try JSONDecoder().decode(MyType.self, from: data)
          }
```

- Combine 은 JSON 디코딩을 편하게 해주는 함수 제공 -> `decode(type:decoder:)`  

```swift
URLSession.shared
	.dataTaskPublisher(for: url)
	.map(\.data)
	.decode(type: MyType.self, decoder: JSONDecoder())
```

- tryMap에서는 매번 JSONDecoder를 초기화해야 하지만, Combine에서 제공하는 decode 함수를 사용하면 publisher를 세팅할 때 한번만 하면 됨

### [Publishing network data to multiple subscribers]

- Publisher는 구독(subscribe)될 때마다 일을 시작. 네트워크 요청의 경우, 다수의 subscriber가 결과를 필요로 할 때 동일한 요청을 여러 번 보내는 것을 의미.
- 하지만 나는 같은것이라면 한번만 요청하고 싶은데..?! -> Combine에서 이 문제를 쉽게 해결할 수 있는 연산자가 딱히 없음. `share()` 을 사용할 수도 있지만, 결과가 return 되기 전에 모든 subscriber가 구독을 마친 상태여야 하기 때문에 까다로움.
- 캐싱을 사용하는 것 외에, 하나의 해결책은 `multicast` 연산자를 사용하는 것.
- `multicast` 연산자는 Subject를 통해 값을 방출하는 `ConnectablePublisher`를 만듦. 얘를 통해 해당 subject를 여러 번 구독할 수 있고, 이후 준비가 되면 publisher의 `connect` 함수를 호출하면 됨

```swift
let publisher = URLSession.shared
  .dataTaskPublisher(for: url)
  .map(\.data)
  .multicast { PassthroughSubject<Data, URLError>() }

let subscription1 = publisher
  .sink {...}

let subscription2 = publisher
  .sink {...}

let subscription = publisher.connect()
```

- publisher는 ConnectablePublisher 이기 때문에 sink로 구독하더라도 바로 시작되지 않음
- 준비가 되었을때 connect를 호출하면, 작업을 시작하고 모든 subscriber에게 값을 전달

## Chapter 10. Debugging

### [Printing events]

- `print(_:to:)`

  - 뭔 상황이 일어나고 있는지 모를때 첫번째로 사용해봐야할 operator
  - 무슨 일이 발생하고 있는지에 대해 많은 정보를 프린트해주는 passthrough publisher 
    - subscription을 받은 시점과 upstream publisher에 대한 설명
    - subscriber의 demand request (요청되는 개수를 알 수 있게 한다)
    - upstream publisher가 방출하는 모든 값
    - 완료 이벤트

  ```swift
  publisher
  	.print("publisher")
  ```

  - `TextOutputStream` 객체를 파라미터로 추가해서 이곳으로 문자열을 redirection 시킬 수 있음. 이렇게 들어온 기존의 정보에 현재 날짜나 시간을 추가하는 등 다양한 방식으로 프린트 가능.

  ```swift
  class TimeLogger: TextOutputStream {
      private var previous = Date()
      private let formatter = NumberFormatter()
      init() {
          formatter.maximumFractionDigits = 5
          formatter.minimumFractionDigits = 5
      }
      // TextOutputStream 프로토콜에서 기본으로 구현해야하는 write 함수. string 파라미터는 redirection 된 기존의 정보
      func write(_ string: String) {
          let trimmed = string.trimmingCharacters(in: .whitespacesAndNewlines)
          guard !trimmed.isEmpty else { return }
          let now = Date()
          print("+\(formatter.string(for: now.timeIntervalSince(previous))!)s: \(string)")
          previous = now
      }
  }
  
  //사용
  publisher
      .print("publisher", to: TimeLogger())
  ```

### [Acting on events - performing side effects]

- `handleEvents(receiveSubscription:receiveOutput:receiveCompletion:receiveCancel:receiveRequest:)`

  - publisher의 라이프사이클에 있는 모든 이벤트를 가로채고, 각 단계에서 조치를 취할 수 있음

  ```swift
  publisher
  .handleEvents(receiveSubscription: { _ in
    print("start")
  }, receiveOutput: { _ in
    print("received")
  }, receiveCancel: {
    print("cancelled")
  })
  .sink {}
  ```

### [Using the debugger as a last resort]

- 위에서 소개된 그 무엇도 문제를 알아내는데 도움이 되지 않을 때 최후의 수단으로 디버거 사용

-  `breakpointOnError()`

   - upstream publisher에서 오류가 발생하면 break

- `breakpoint(receiveSubscription:receiveOutput:receiveCompletion:)`

  - 다양한 이벤트를 가로채고, 디버거를 일시 중지할지 여부를 사례별로 결정 가능

  ```swift
  .breakpoint(receiveOutput: { value in
    return value > 10 && value < 15
  })
  ```

  - subscription와 completion에서 break 할 수 있지만, handleEvents 연산자처럼 cancelation은 가로챌 수 없음

### [Chapter 10- Key points]

- `print` 연산자와 함께 publisher의 lifecycle을 추적 가능
- 고유한 `TextOutputStream`을 생성하여 출력 문자열을 customize 가능
- `handleEvents` 연산자를 사용하여 lifecycle 이벤트를 가로채고 작업 수행 가능
- `breakpointOnError` 및 `breakpoint` 연산자를 사용하여 특정 이벤트를 중단 가능

## Chapter 11. Timers

- `RunLoop` , `Timer`, `DispatchSourceTimer` 각각을 이용해 타이머를 만들 수 있음
- 하지만 Combine에서 이러한 모든 타이머가 동일한 건 아님.

### [Using RunLoop]

- 메인 스레드 및 사용자가 생성하는 모든 스레드는 자체 RunLoop를 가질 수 있음
- 현재 스레드에서 `RunLoop.current` 를 호출하면, 필요할 경우 Foundation에서 생성해줌

> Note: RunLoop 클래스는 thread-safe 하지 않음. 현재 스레드의 실행 루프에 대해서만 RunLoop 메서드를 호출해야 함

- RunLoop는 Scheduler protocol을 구현하는데, 여기에 cancellable 타이머를 만들 수 있는 method도 정의되어 있음

```swift
let runLoop = RunLoop.main

let subscription = runLoop.schedule(
  //언제부터
  after: runLoop.now,
  //얼마나 자주 (간격)
  interval: .seconds(1), 
  //Timer가 이벤트를 발생시키는 것에 여유를 허용하는 것
  //요구한 시간으로부터의 허용 가능한 편차를 지정하여 전원 관리와 반응성을 위한 최적화
  //Timer가 실제 이벤트를 발생하는 시간은 (지정된 시간) ~ (지정된 시간 + tolerance) 
  //Apple 공식 문서에서 권장하는 Tolerance 값은 최소 원래 간격의 10% 이상
  tolerance: .milliseconds(100) 
) {
  print("Timer fired")
}
```

- Cancellable를 반환하기 때문에 아래와 같이 타이머를 중지할 수 있음

```swift
runLoop.schedule(after: .init(Date(timeIntervalSinceNow: 3.0))) {
  cancellable.cancel()
}
```

- 하지만 모든 것을 고려했을 때, RunLoop는 타이머를 만드는 최선의 방법이 아님

### [Using the Timer class]

- Timer는 delegate 패턴과 RunLoop과의 긴밀한 관계 때문에 항상 사용하기 까다로웠음
- Combine은 간단히 publisher로 사용할 수 있는 방법을 제공

```swift
let publisher = Timer
.publish(
  //얼마나 자주
  every: 1.0, 
  //timer가 어느 스레드에 설정될 것인지. (.main, .current 등 사용 가능)
  on: .main, 
  //run loop mode (common은 default mode를 의미)
  in: .common)
.autoconnect()
```

> Note: 이 코드를 DispatchQueue.main이 아닌 다른 Dispatch queue에서 실행하면 예기치 않은 결과가 발생할 수 있음. 왜냐하면 이벤트를 처리하기 위해서는 run loop의 run 메소드를 호출해야 하는데, Dispatch 프레임워크는 run loop를 사용하지 않고 스레드를 관리함. 따라서 main queue 외의 대기열에서는 타이머가 작동하지 않음. 
>
> 👩🏻‍💻 Main Thread는 애플리케이션이 실행될 때 프레임워크 차원에서 자동으로 RunLoop를 설정하고 실행(Main Runloop)하기 때문에 내가 run 호출 안해도 잘 돌아가는 것

- Timer가 반환하는 publisher는 `ConnectablePublisher` 이라서 `connect()` 메소드 호출전까지는 구독을 시작하지 않음. 첫 번째 subscriber가 구독할 때 자동으로 연결되는 `autoconnect()` 연산자를 사용할 수도 있음
- publisher의  output 타입은 `Date` 
- `scan` 을 사용해서 증가하는 값을 방출하게 할 수도 있음

```swift
let subscription = Timer
  .publish(every: 1.0, on: .main, in: .common)
  .autoconnect()
  .scan(0) { counter, _ in counter + 1 }
  .sink { counter in
    print("Counter is \(counter)")
  }

/**
Counter is 1
Counter is 2
Counter is 3
Counter is 4
Counter is 5
*/
```

### [Using DispatchQueue]

- Dispatch queue를 사용하여 타이머 이벤트를 생성할 수 있음 
- Dispatch 프레임워크가 `DispatchTimerSource` 라는 event source를 가지고 있긴 하지만, Combine은 이에 대한 인터페이스를 제공하지 않기 때문에 다른 방법을 사용하여 타이머 이벤트를 생성 가능

```swift
let queue = DispatchQueue.main
let source = PassthroughSubject<Int, Never>()
var counter = 0

let cancellable = queue.schedule(
  after: queue.now,
  interval: .seconds(1)
) {
  source.send(counter)
  counter += 1
}

let subscription = source.sink {
  print("Timer emitted \($0)")
}
```

