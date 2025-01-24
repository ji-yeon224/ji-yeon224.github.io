---
title: "[iOS] 동시성프로그래밍: Sendable, Actor"
date: 2024-12-30 15:00:00 +09:00
categories:
  - PointFree스터디
  - Concurrency
tags:
  - Concurrency
  - Sendable
  - Actor
comments: "true"
---
<span style="color:gray">PointFree의 Concurrency 세션 정리 내용입니다!</span>  

🔗 PointFree  
 [Concurrency's Future: Sendable and Actors ](https://www.pointfree.co/collections/concurrency/threads-queues-and-tasks/ep193-concurrency-s-future-sendable-and-actors)  

✏️ 이전 내용  
[1. 동시성의 과거: 스레드](https://ji-yeon224.github.io/posts/Concurrency_Thread/)  
[2. 동시성프로그래밍: OperationQueue, GCD, Combine](https://ji-yeon224.github.io/posts/Concurrency_OperationQueue_GCD_Combine/)  
[3.동시성프로그래밍: Task](https://ji-yeon224.github.io/posts/Task_Cooperation/)

----
## Sendable and @Sendable  

### 데이터 동기화 및 Data Race 문제  

✔️ **데이터 경쟁 문제**  
이전에 스레드와 DispatchQueue에 관련하여 다루었을 때 데이터 동기화 및 경쟁 문제가 있었다.  
이전에 사용한 해결책을 살펴보자.  
```swift
class Counter {
  let lock = NSLock()
  var count = 0
  func increment() {
    self.lock.lock()
    defer { self.lock.unlock() }
    self.count += 1
  }
}
```
`NSLock` 을 사용하여 count 변수를 증가시킬 때 여러 스레드가 동시에 작업하지 못하도록 막아줬다.  

```swift
let counter = Counter()

for _ in 0..<1000 {
  Task {
    counter.increment()
  }
}

Thread.sleep(forTimeInterval: 2)
print("counter.count", counter.count)

// 출력:
// counter.count 1000
```

그 후 1,000개의 태스크를 만들어 count를 증가시키도록 하였다.  
결과는 항상 1,000이 출력되는 것으로 보아 경쟁 상태를 해결한 것으로 볼 수 있다.  

하지만 락을 사용하는 코드는 100% 안전을 보장할 수 없다. 코드가 복잡해지고 실수하기 쉬워 큰 문제를 발생시킬 가능성이 있다.  

✔️ **Swift의 제한**  
Swift는 이러한 동시성 컨텍스트에서 데이터 경쟁 상황을 방지하기 위해 컴파일러가 오류나 경고를 나타낸다.  
제한된 사항 중 하나는 동시성 컨텍스트에서 가변 변수(var)를 캡쳐하는 것을 금지하고 있다.  

```swift
func doSomething() {
  var count = 0
  Task {
    print(count) // Error
  }
}
```

```plaintext
Error: Reference to captured var ‘count’ in concurrently-executing code
```
가변 변수를 캡쳐할 때 오류가 발생하는 이유는 태스크 외부 또는 내부에서 값의 변경이 발생할 경우 경쟁 상태가 발생할 수 있기 때문이다.  

```swift
count = 0
Task {
  count = 1
}
Thread.sleep(forTimeInterval: 1)
print(count) // 0? 1?
```

가변 캡쳐는 단순 탈출 클로저에서는 허용된다.  
```swift
var count = 0
for _ in 0..<workCount {
  Thread.detachNewThread {
    count += 1
  }
}
Thread.sleep(forTimeInterval: 2)
print("count", count)
```
위와 같이 스레드 기반 코드에서는 컴파일러가 경쟁 상태임을 인지하지 못하기 때문에 오류가 발생하지 않고, 1,000보다 작은 값을 출력하게 된다.  

✔️ **Immutable capture**  
가변 캡쳐는 안되지만 불편 타입의 캡쳐는 문제가 발생하지 않는다.  
```swift
let count = 0
Task {
  print(count)
}
```
`let` 을 사용하거나,  

```swift
var count = 0
Task { [count] in
  print(count)
}
```
`var` 여도 명시적으로 캡쳐리스트로 캡쳐를 하게 된다면 외부 변수와 분리되므로 문제가 발생하지 않는다.  

✔️ **컴파일러의 진단 활성화**  
컴파일러가 경쟁상태와 같은 동시성 문제를 진단할 수 있도록 플래그를 활성화 시킬 수 있다.  
```swift
.executableTarget(
  name: "concurrency",
  dependencies: [],
  swiftSettings: [
    .unsafeFlags([
      "-Xfrontend", "-warn-concurrency",
    ]),
  ]
),
```

해당 설정을 통해 플래그를 활성화 하게 되면  
```swift
for _ in 0..<1000 {
  Task {
    counter.increment() // warning
  }
}
```
위에서 예시로 사용했던 코드에서 경고가 발생하게 된다.  

```plaintext
Capture of ‘counter’ with non-sendable type ‘Counter’ in a @Sendable closure
```
이 경고는 @Sendable 클로저 내에서 non-sendable 타입을 캡쳐할 때 발생한다.  
먼저, Sendable 프로토콜에 대해 알아보자.  

---

## The Sendable protocol  

```swift
/// Sendable 프로토콜은 특정 타입의 값이 
/// 동시성 코드에서 안전하게 사용될 수 있음을 나타냅니다.
public protocol Sendable {
}
```

Sendable 타입은 **Sendable프로토콜을 준수하는 타입**이다. Sendable 프로토콜은 요구사항이 없는 프로토콜이지만 **컴파일러가 내부적으로 Sendable 타입이 프로토콜을 준수하는지 확인하는 작업을 수행**한다. 이를 통해 동시성 코드에서 안전하게 값을 전달할 수 있도록 한다.  

어떠한 타입은 항상 안전하게 전달될 수 있다.  
예를 들어,  
```swift
var count = 0
Task { [count] in
  print(count)
}
```
위와 같은 경우는 문제가 발생하지 않는다. 오류가 없는 이유는 Int 타입이 Sendable 프로토콜을 준수하기 때문이다.  
표준 라이브러리의 대부분의 타입은 값 타입으로 구성되어 있기 때문에 기본적으로 Sendable이다.  
그렇기 때문에 필드가 모두 Sendable인 값 타입은 동시성 코드에서 안전하게 전달될 수 있다.  
```swift
struct User {
  var id: Int
  var name: String
}
let user = User(id: 42, name: "Blob")
Task {
  print(user)
}
```
하지만, 데이터 타입이 더 복잡해지면 자동으로 Sendable을 준수하지 못하는 경우가 생길 수 있다.  
예를 들어, `AttributedString` 타입의 필드를 추가하게 된다면  
```swift
struct User {
  var id: Int
  var name: String
  var bio: AttributedString
}
let user = User(id: 42, name: "Blob", bio: "")
Task {
  print(user) // warning
}
```

```plaintext
Capture of ‘user’ with non-sendable type ‘User’ in a @Sendable closure
```
`@Sendable` 클로저 내에서 User 타입을 캡쳐하지 못한다는 경고가 발생한다. `AttributedString`타입은 Sendable이 아니기 때문에 User타입은 Sendable을 준수하지 않고있다.  

위에서 다루었던 내용들은 값 타입이지만 참조타입도 Sendable로 만들 수 있다.  
```swift
class User: Sendable { // warning 1
  var id: Int // warning 2
  var name: String
  init(id: Int, name: String) {
    self.id = id
    self.name = name
  }
}
```

경고:  
```plaintext
1. Non-final class ‘User’ cannot conform to ‘Sendable’; use ’@unchecked Sendable’
2. Stored property ‘id’ of ‘Sendable’-conforming class ‘User’ is mutable
```
첫 번째 경고는 `final`이 아닌 클래스는 Sendable로 추론할 수 없다는 것을 나타낸다. 이는 서브클래스가 생성되어 가변 상태가 추가될 수 있기 때문이다.   
두 번째 경고는 Sendable이 되려면 가변 필드가 포함될 수 없음을 나타낸다. 따라서 필드를 `let`으로 변경해야 한다.  
```swift
final class User: Sendable {
  let id: Int
  let name: String
  init(id: Int, name: String) {
    self.id = id
    self.name = name
  }
}
```
다음과 같이 수정하면 Sendable을 준수하는 타입이 만들어지지만 User 클래스의 많은 기능이 제한된다. 내부 데이터를 변경할 수 없고, 구초제와 비슷한 동작을 하게 된다.  

어떠한 타입은 Sendable임을 컴파일러에 증명할 수 없을 때가 있다.  
이러한 경우 직접 문제를 해결하고, 컴파일러의 감시 범위에서 벗어나야 한다.  
```swift
class Counter {
  let lock = NSLock()
  var count = 0
  func increment() {
    self.lock.lock()
    defer { self.lock.unlock() }
    self.count += 1
  }
}
let counter = Counter()
Task {
  counter.increment() // warning 
}
```
위의 코드는 Task 구문 내부에서 다음과 같은 경고가 나타난다.  
```plaintext
Capture of ‘counter’ with non-sendable type ‘Counter’ in a @Sendable closure
```
위의 코드는 멀티 스레드 환경에서 안전하게 사용할 수 있도록 되어있는지 알 수 없다.  

만약 Counter를 Sendable로 만들게 된다면  
```swift
class Counter: Sendable { // warning 1
  let lock = NSLock() // warning 2
  var count = 0 // warning 3
  func increment() {
    self.lock.lock()
    defer { self.lock.unlock() }
    self.count += 1
  }
}
```

경고:  
```plaintext
1. Non-final class ‘Counter’ cannot conform to ‘Sendable’; use ‘@unchecked Sendable’
2. Stored property ‘lock’ of ‘Sendable’-conforming class ‘Counter’ has non-sendable type ‘NSLock’
3. Stored property ‘count’ of ‘Sendable’-conforming class ‘Counter’ is mutable
```
`NSLock`은 우리가 제어할 수 있는 타입이 아니기 때문에 Sendable로 만들 수 없다.  

위의 코드가 멀티 스레드 환경에서 안전하게 사용할 수 있다고 확신한다면 컴파일러에게 **@unchecked**로 알려줄 수 있다.  
```swift
class Counter: @unchecked Sendable {
  let lock = NSLock()
  var count = 0
  func increment() {
    self.lock.lock()
    defer { self.lock.unlock() }
    self.count += 1
  }
}
```
이제 경고가 발생하지 않고 컴파일이 되지만 위의 코드는 컴파일러가 안전성을 체크하지 않기 때문에 조심해야 한다.  

---

## @Sendable closures  
**@Sendable 클로저**는 **클로저가 동시성 작업에서 사용되는 것**을 나타낸다.  
이를 이해하기 위해 먼저 **@escaping 클로저**에 대해 알아보자.  

아래와 같은 클로저를 매개변수로 받는 함수는 함수 범위 내에서 해당 클로저를 호출할 수 있다.  
```swift
func perform(work: () -> Void) {
  work()
}
```

그러나 클로저를 함수 범위를 벗어나 사용하려하면 컴파일러가 경고를 나타내게 된다.  
예를들어, 네트워크 통신을 통해 다운로드 작업을 완료한 후 클로저를 호출해보면 경고가 나타난다.  
```swift
func perform(work: () -> Void) {
  print("Begin")
  URLSession.shared.dataTask(
    with: .init(string: "<http://ipv4.download.thinkbroadband.com/1MB.zip>")!
  ) { _, _, _ in
    work()
  }
  print("End")
}

// Warning: Escaping closure captures non-escaping parameter ‘work’
```
**Escaping 클로저**는 **함수 실행 범위가 끝난 후에도 참조되어야 하는 클로저**이다. dataTask의 경우 호출 즉시 반환되기 때문에 **클로저가 함수 실행 범위 이후까지 유효해야 한다**.  
때문에 Non-escaping 클로저를 호출하게 되면 컴파일러가 경고하게 된다.  

Swift에서는 **클로저가 함수 실행 범위를 벗어나는지에 대한 여부를 중요하게 생각**한다.  
이를 뒷받침 하기 위한 예시를 살펴보자면,  
```swift
func incrementAfterOneSecond(value: inout Int) {
  DispatchQueue.global().asyncAfter(deadline: .now() + 1) {
    value += 1
  }
}
```
1초 후에 값을 증가시키는 함수가 있다고 가정해보자.  
```swift
var count = 0
incrementAfterOneSecond(value: &count)
assert(count == 0)
Thread.sleep(forTimeInterval: 2)
assert(count == 1)
```
위의 코드가 유효한 코드라고 가정했을 때, 첫번째 assert에서는 0일 것이다.  
그러나 2초 대기 후 값이 자동으로 1로 업데이트가 된다면 뭔가 어색한 동작으로 보인다. 왜냐하면 **count는 값 타입**인데 **참조타입처럼 동작**하기 때문이다.  

위와 같은 **원격 조작(spooky action at a distance)** 은 **값 타입의 본질을 무너뜨리기 때문**에 Swift에서는 **Escaping 클로저**를 통해 이러한 동작을 방지하고자 한다.  
그렇기 때문에 Swift에서는 `inout` 값을 Escaping 클로저 내부로 전달하는 것은 허용되지 않는다.  

```swift
func perform(work: @escaping () -> Void) {
  print("Begin")
  DispatchQueue(label: "delay").asyncAfter(deadline: .now() + 1) {
    work()
  }
  print("End")
}
perform {
  print("Hello")
}
```
그래서 work클로저를 asyncAfeter 내에서 실행할 수 있도록 하기 위해서는 `@escaping`을 명시적으로 추가해야 한다. `@escaping` 을 통해 클로저가 함수의 생명주기 이후에도 호출될 수 있음을 알려주게 된다.  

그치만 `async` 를 통해 `@escaping`이 필요하지 않은 클로저를 사용할 수 있다.  
```swift
func perform(work: () -> Void) async throws {
  print("Begin")
  _ = try await URLSession.shared.data(from: .init(string: "<http://ipv4.download.thinkbroadband.com/1MB.zip>")!)
  work()
  print("End")
}

Task {
  try await perform {
    print("Hello")
  }
}
```

```plaintext
Begin
Hello
End
```
`async` 를 사용하게 되면서 work 클로저가 **함수 범위 내에서만 호출**된다. `await` 키워드를 통해 **작업이 완료될 때 까지 대기**하게되어 `@escaping` 키워드 없이도 문제가 발생하지 않게 된다.  

그렇기 때문에 Escaping 클로저에서는 사용이 불가능했던 inout값 또한 처리할 수 있게 된다.  

```swift
func perform(value: inout Int, work: () -> Void) async throws {
  print("Begin")
  let (data, _) = try await URLSession.shared.data(from: .init(string: "<http://ipv4.download.thinkbroadband.com/1MB.zip>")!)
  work()
  value += data.count
  print("End")
}

Task {
  var count = 0
  try await perform(value: &count) {
    print("Hello")
  }
  print("count", count)
}
```

```plaintext
Begin
Hello
End
count 1048576
```
다시말해, **모든 작업이 완료된 후 함수가 종료됨을 보장**하기 때문이다.

**✔️ 동시성 작업에서의 Non-escaping**  
하지만, work 클로저를 여러 번 동시에 실행하고싶어 두개의 Task를 생성하게 된다면 오류가 발생하게 된다.  
![image](https://github.com/user-attachments/assets/6ab7673c-5a84-4e87-b66a-4460f6992cb9)
`Task{}`는 클로저를 **생성된 태스크 이후에 호출**해야 하기 때문에 **escaping 클로저**여야 한다.  

![image](https://github.com/user-attachments/assets/bbff450b-2156-46bf-9be2-b095d4c28a58)
하지만 여전히 오류가 발생하는데, 그 이유는 `@Sendable`이 필요한 컨텍스트 내에서 `@Sendable`이 아닌 데이터를 사용하고 있기 때문이다.  
클로저가 외부 데이터를 캡쳐하고 수정하게 된다면 **race condition**문제를 일으킬 수 있기 때문에 동시성 작업을 할 때는 안전한 클로저를 요구하게 된다. `@Sendable`이 필요한 이유이다.  
```swift
func perform(work: @escaping @Sendable () -> Void) {
  ...
}
```

`@Sendable`클로저는 **외부의 가변 상태 데이터를 캡쳐하거나 변경하지 못하도록 한다**.  
![image](https://github.com/user-attachments/assets/80597717-8541-47ba-b929-bf46150506a8)
`@Sendable` 클로저를 추가하게되면서 Swift는 클로저가 동시성 작업에서 사용된다는 것을 알게된다. 그렇기 때문에 count라는 **가변 변수를 캡쳐하지 못하도록 제한**한다.  
**@Sendable** 속성은 함수가 **여러 동시성 작업에서 안전하게 사용될 수 있음을 컴파일러에게 증명**하는 역할을 한다.  

`@Sendable` 은 내부에 클로저를 가지고 있는 타입이 **Sendable**프로토콜을 준수하도록 할 수 있다.  
예를들어, 데이터베이스에 접근하는 작업을 추상화하여 설계한다고 가정해보자.  
```swift
struct DatabaseClient {
  var fetchUsers: () async throws -> [User]
  var createUser: (User) async throws -> Void
}
```
위와 같은 구조체가 있다고 할 때 위의 구조체는 동시성 작업에서 안전하게 사용하지 못한다.    
![image](https://github.com/user-attachments/assets/66252556-7fa1-42b6-a912-e083be23ecb2)
이는 DatabaseClient가 **Sendable** 타입이 아니기 때문이다.  

그렇다면 명시적으로 DatabaseClient를 `Sendable` 을 준수하도록 한다면, 내부 프로퍼티가 `Sendable` 타입이 아니기 때문에 또 문제가 발생하게 된다.  
![image](https://github.com/user-attachments/assets/e38e16cb-54af-4d3e-9c96-1956bdc48a69)
내부 클로저는 `sendable` 타입이 아닌 어떠한 클로저든 가능하기 때문에 가변 데이터를 읽고 쓸 가능성이 있다.  

```swift
struct DatabaseClient {
  var fetchUsers: @Sendable () async throws -> [User]
  var createUser: @Sendable (User) async throws -> Void
}
```
DatabaseClient가 `Sendable`타입을 충족하려면 내부 프로퍼티의 클로저를 `@Sendable`로 명시적으로 지정해야 한다.  
`@Sendable`을 통해 **사용할 수 있는 클로저의 종류를 제한**하여 컴파일러에게 **sendable 타입임을 증명**할 수 있다.  

---
## Actor  
@Sendable을 통해 동시성 작업에서의 안정성을 보장할 수 있게 되었다. Sendable을 만족하지 않을 경우 컴파일러가 문제를 알려주게 된다.    
하지만 `@unchecked Sendable`은 Sendable을 준수하는지 컴파일러가 확인하지 않고 **강제적으로** 준수한다고 컴파일러에게 알려주게 된다.  
```swift
class Counter: @unchecked Sendable {
  let lock = NSLock()
  var count = 0
  func increment() {
    self.lock.lock()
    defer { self.lock.unlock() }
    self.count += 1
  }
}
```
이러한 방식은 문제가 발생하더라도 컴파일러가 감지하지 못하기 때문에 신뢰성이 떨어지는 코드이다.  
코드가 변경되어 **Race Condition 문제**가 발생하게 될 수 있기 때문이다.  

이러한 Data Race 발생 위험성 문제를 해결하기 위해 Swift 5.5에서는 새로운 타입인 **Actor**가 도입되었다.  
Actor는 구조체, 열거형, 클래스와 같이 **새로운 타입**이다.  
```swift
actor counterActor {
}
```
클래스처럼 **참조타입**이지만, 데이터를 보호하고 메서드 호출을 동기화한다.  

```swift
actor CounterActor {
  var count = 0
  var maximum = 0
  func increment() {
    self.count += 1
    self.maximum = max(self.count, self.maximum)
  }
  func decrement() {
    self.count -= 1
  }
}
```
위에서 사용한 코드와 같이 클래스로 구현할 경우 동시성 환경에서 count 변수에 대해 data race 발생 가능성이 존재했지만, actor 타입을 사용할 경우 이러한 문제가 해결된다.  
Actor를 사용하게 되면 **접근을 제한하기 위한 락을 사용할 필요가 없게 된다.**  

**✔️ Actor의 DataRace 방지**  
![image](https://github.com/user-attachments/assets/4e8a33bf-64cf-4879-9ec6-60f1ffa31b9d)
여러 스레드를 생성하여 `increment()`메서드를 호출하려 하면 오류가 발생한다.  
위의 오류는 Swift가 DataRace를 발생시킬 수 있는 작업을 하지 못하도록 막는 것이다.  

**Actor의 핵심 원칙은 내부 데이터를 보호**이기 때문에 Actor 내부의 메서드는 **비동기 컨텍스트에서만 호출**할 수 있다.  
여러 스레드에서 `increment()`메서드에 접근하려고 할 경우 이전 작업이 완료될 때 까지 다른 스레드는 기다려야 한다. 하지만 스레드가 대기 상태에 있으면 리소스가 낭비될 수 있다.  
Actor는 락을 사용하지 않고 **비동기 방식으로 동기화 처리**를 한다. 스레드가 대기해야 할 경우 **스레드를 차단하지 않고 다른 작업을 실행할 수 있도록 넘기게 된다**.  

그렇기 때문에 Actor 내의 메서드를 호출하기 위해서는 **비동기 컨텍스트에서 await 키워드를 통해 호출**해야 한다.  
```swift
for _ in 0..<1000 {
  Task {
    await counter.increment()
  }
}
```
Actor를 통해 **데이터 보호**와 **시스템 효율성을 달성**할 수 있게 된다.  

Actor 내부에서는 `increment()` 메서드를 `async`로 선언하지 않았지만, `await`을 사용하여 호출해야 한다.  
**Actor 내부**에서만 보면 `increment()`는 **동기적**이다. Actor 내부는 **완전히 동기화 된 컨텍스트 내에 있기 때문**에 **가변 데이터를 자유롭게 읽고 쓸 수 있다.**  
```swift
func increment() {
  self.count += 1
  self.maximum = max(self.maximum, self.count)
}
```
그렇기 때문에 Actor 내에 있는 메서드 내에서도 프로퍼티나 또 다른 메서드를 호출할 때 await을 사용하지 않고도 접근할 수 있다.  

하지만 **Actor 외부**에서는 여러 작업이 동시에 `increment()`메서드를 호출할 수 있기 때문에 동기적으로 호출할 수 없다.  
Actor는 순차적으로 처리하기 위한 **동기화 작업이 필요**하고, 효율적으로 처리되기 위해서는 **비동기 컨텍스트에서 작동**해야 한다.  
![image](https://github.com/user-attachments/assets/7f67faf6-d1c8-4fdc-b70b-580e0752bb06)
메서드 뿐만아니라 **프로퍼티 접근** **또한** 비동기 환경에서 이루어져야 한다. 다른 작업에서 프로퍼티 수정을 할 경우 잘못된 값을 얻을 수 있기 때문이다.  
```swift
Thread.sleep(forTimeInterval: 1)
Task {
  await print("counter.count", counter.count)
}
```

Actor를 통해 우리는 동기 코드처럼 코드를 작성할 수 있고, datarace 문제에 신경쓰지 않아도 된다. Actor 외부에서 Actor와 상호작용 할 때에만 비동기 컨텍스트를 신경쓰면 된다.  

만약 1000 번의 증가, 감소 작업을 수행 중에 count와 maximum 값을 출력시키면 어떤 결과가 나올까?  

|                                                                                                              |                                                                                                               |
| ------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------- |
| <img src = "https://github.com/user-attachments/assets/7b283ac9-fe19-45a5-9b5f-533726424501" align="center"> | <img src = "https://github.com/user-attachments/assets/05a74043-0fb8-43c2-9c95-174e5f025694" align="center" > |

같은 코드인데 maximum 값이 다르게 출력된다.    
이는 race condition을 의미하는 것이 아니다. 이는 **비 결정론적인 예시**이다.  
1000 번의 증가와 1000번의 감소 작업이 동시에 실행되고 있는데, **이 작업들의 실행 순서는 결정론적이지 않다**.  
즉, 어떠한 경우에는 increment 작업이 연속적으로 실행되어 maximum값이 높아질 수 있고, 다른 경우에는 increment와 decrement 작업이 번갈아 실행될 수 있다.  
**➡️ 작업의 순서는 예측할 수 없기 때문**에 값이 달라질 수 있는 것이다.

별도로 작업의 **우선순위를 어떻게 지정하느냐**에 달려있기 때문에 **비결정론이 도입**되어있는 것을 의미한다.  
  

----
Swift의 새로운 동시성 도구는 기존의 동시성 도구보다 훨씬 강력한 기능을 가지고 있다.

1. Swift 언어와의 통합  
    비동기와 동시성 개념이 언어에 직접적으로 통합되어있다. **async** 키워드를 통해 비동기 작업을 명확히 표현할 수 있으며, `Sendable` 프로토콜과 `@Sendable` 속성을 통해 동시성 환경에서 사용 가능한 데이터를 표현할 수 있게 되었다.
2. 효율적인 작업 관리  
    명시적으로 스레드 풀이나 작업 큐를 관리하지 않더라도 Swift는 **생성되는 스레드 수를 제한**하며 **수천개의 동시 작업을 수행**할 수 있게 되었다.
3. 향상된 Task  
    Task는 기존 도구들의 기능을 모두 가지고 있을 뿐만아니라 기능들이 더 향상되었다. 취소나 저장 기능은 하위 Task에도 전달이 가능해졌다.
4. 효율적인 스레드 사용  
    Non-blocking 비동기 함수나 `Task.yield`와 같은 도구를 통해 **스레드를 점유하지 않고 다른 작업이 스레드를 사용할 수 있도록 중단할 수 있다.**
5. Actor  
    Swift는 **Actor**라는 새로운 타입을 통해 **컴파일러가 Data race 문제를 감지할 수 있게 되었다**. Actor를 통해 코드를 **간단하고 동기 코드 처럼 작성**할 수 있지만 **내부적으로는 가변 데이터에 대한 접근을 제한**하는 기능을 한다.


----
🔥 다음 내용  
[5. 동시성프로그래밍: 구조적 프로그래밍과 MainActor](https://ji-yeon224.github.io/posts/StructuredProgramming/)