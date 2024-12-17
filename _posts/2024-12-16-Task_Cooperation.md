---
title: "[iOS] 동시성 프로그래밍 - Task"
date: 2024-12-17
categories:
  - PointFree스터디
  - Concurrency
tags:
  - Concurrency
  - Task
comments: "true"
---
<span style="color:gray">PointFree의 Concurrency 세션 정리 내용입니다!</span>

# [**🔗](https://www.pointfree.co/episodes/ep192-concurrency-s-future-tasks-and-cooperation#task-cooperation) Concurrency's Future: Tasks and Cooperation**

## Task Basic

### Task

**Task**는 Swift 5.5부터 새로 출시된 동시성 도구이다. **Task**는 비동기작업을 실행하는 기본 단위이고, 다음과 같이 생성할 수 있다.  

```swift
Task {
	print(Thread.current)
}
```

```plaintext
<NSThread: 0x1062040f0>{number = 2, name = (null)}
```

내부에서 실행되는 스레드의 정보를 출력해보면 새로운 스레드에서 실행되고 있는 것을 볼 수 있다.     
이 내용은 Thread의 `detachNewThread`메서드와 유사하게 동작하지만 스레드 생성과 작업을 수행하는 방식을 자동으로 처리한다는 점에서 다르다.  

Task의 타입은 다음과 같다.  
![image](https://github.com/user-attachments/assets/e27e3983-5112-4c36-9c06-760113032e37)

`Void`와 `Never` 두 개의 제네릭을 반환하게 된다.  
첫 번째 제네릭은 비동기 작업이 끝난 후 생성되는 값의 타입을 나타내고, 두 번째 제네릭은 클로저 내부에서 던질 수 있는 에러의 타입을 나타낸다.  

다음과 같이 실행 결과의 타입과 에러 타입을 정의할 수 있다.  
```swift
let task: Task<Int, Error> = Task {
  struct SomeError: Error {}
  throw SomeError()
  return 42
}
```

### async/await

Task의 초기화 구문에 있는 클로저에는 **async**키워드가 포함되어 있다.  
![image](https://github.com/user-attachments/assets/622ff9bc-e9f1-4486-b620-780116080ec0)

**async**는 해당 키워드가 붙은 함수에서 **비동기 작업을 수행할 것임을 선언하는 것을 의미**한다.  
**async**키워드는 비동기 컨텍스트를 제공해야만 호출을 할 수 있다.  

예를 들어, 다음과 같은 async로 선언된 함수가 있다.  
```swift
func doSomethingAsync() async {
}
```

이 함수는 비동기 컨텍스트에 있지 않다면 이 함수를 직접 호출할 수 없다.  
```swift
doSomethingAsync() // 컴파일 에러 발생
```

이 함수는 Task 컨텍스트 내부에서 호출할 수 있으며, **await**키워드를 붙여야 한다.  
```swift
Task {
	await doSomethingAsync()
}
```
Task는 새로운 비동기 컨텍스트를 생성하기 때문에 위와 같이 비동기 함수 호출이 가능한 것이다.  

**await**은 비동기 함수를 호출할 때 사용하는 것으로, 비동기 작업을 시작하며 현재 작업을 중단하고 다른 작업이 스레드를 사용할 수 있도록 현재 스레드를 포기하도록 한다. 비동기 작업이 완료되면 Task는 다시 실행을 재개하고, 재개 시에는 이전과 다른 스레드에서 실행될 수도 있다.  

또, 다른 비동기 함수에서 비동기 함수를 다음과 같이 호출할 수 있다.  
```swift
func doSomethingElseAsync() async {
  await doSomethingAsync()
}
```

이러한 방식은 이전에 스레드와 큐를 사용한 방식과 완전히 다르다. 이전에는 비동기 코드와 동기 코드 간에 컴파일러 단계에서 구분을 하지 않았다.  

### 동기-비동기 코드의 구분  
새로운 비동기 처리 도구에서는 동기코드와 비동기 코드를 컴파일 타임에 구분하게 된다.  
이 내용은 Swift의 오류처리와 유사하다.  

예를 들어, 오류를 던질 수 있는 함수는 `throws`로 명시해야하며, 호출 시 실패에 대한 처리를 해야 호출할 수 있다.  
```swift
func doSomethingThatCanFail() throws {} // 오류 가능 함수

doSomethingThatCanFail() // 오류
try doSomethingThatCanFail() // try로 실패에 대한 처리 - 호출 성공

```

또, 새로운 함수 내에서 그냥 호출하면 문제가 발생한다.  
```swift
func doSomething() {
  try doSomethingThatCanFail()
}
```

이를 해결하기 위해서는 함수를 실패할 수 있는 컨텍스트로 만들거나 **do-catch** 구문을 통해 실패 가능 컨텍스트를 생성하여 실패 시 오류를 처리할 수 있는 작업을 정의해야 한다.  

```swift
func doSomethingElseThatCanFail() throws {
  try doSomethingThatCanFail()
}

// 또는 

func doSomething() {
  do {
    try doSomethingThatCanFail()
  } catch {
    // 오류 처리
  }
}
```
Task도 이와 유사하게 동작한다.  

이러한 컴파일러 수준에서의 관리는 부적절한 컨텍스트에서 비동기 작업 또는 실패 가능한 작업을 수행하지 못하도록 방지한다.

### syntactic sugar  
함수에서 사용하는 키워드들은 복잡한 동작을 간단히 표현하는 **syntactic sugar**이다.  

**throws**  
오류를 던지는 throws 키워드를 가진 함수는 `Result<>` 타입을 반환하는 함수로 풀어 설명할 수 있다.  
```swift
(A) throws -> B

(A) -> Result<B, Error>
```

**async**  
비동기 함수에서 async 키워드를 제거하고 `Task<>` 타입을 반환하도록 할 수 있다.  
```swift
(A) async -> B

(A) -> Task<B, Never>
```
Task 타입을 다음과 같이 풀어 설명할 수 있다.  
```swift
(A) -> ((B) -> Void) -> Void
```
이를 언커리 하여 두개의 인수를 받는 형태로 만들 수 있다.  
```swift
(A, (B)->Void) -> Void
```
위와 같은 형태는 콜백 핸들러의 기본 형태이다. 대부분의 Apple 비동기 api는 이러한 형태를 사용하였다.   
```swift
// URLSession의 콜백 핸들러 예시
dataTask: (URL, (Data?, Response?, Error?) -> Void) -> Void
```

위와 같은 형태는 Swift 언어 자체에서 비동기를 지원하게 되며 async 키워드를 통해 복잡한 구문을 간단하게 나타낼 수 있게 되었다.  
```swift
(A) async -> B
```

### Task의 스레드 관리  
Task에서 여러 개의 작업을 실행하면 여러 개의 스레드가 생성된다.  

```swift
Task { print("1", Thread.current) }
Task { print("2", Thread.current) }
Task { print("3", Thread.current) }
Task { print("4", Thread.current) }
Task { print("5", Thread.current) }
```

```plaintext
2 <NSThread: 0x1062040f0>{number = 3, name = (null)}
1 <NSThread: 0x106004bd0>{number = 4, name = (null)}
3 <NSThread: 0x106105c90>{number = 5, name = (null)}
4 <NSThread: 0x106004430>{number = 6, name = (null)}
5 <NSThread: 0x106007f40>{number = 7, name = (null)}
```
그리고 Thread나 Operation Queue에서 처럼 실행 순서는 보장되지 않는다.    
이는 DispatchQueue에서와 달리 기본적으로 **동시성 모델**을 사용하는 것을 의미한다.

그렇다면 1,000개의 작업을 생성할 때 마다 스레드가 생성될까?  
```swift
for n in 0..<1000 {
  Task {
    print(n, Thread.current)
  }
}
```

```plaintext
1 <NSThread: 0x1011caaa0>{number = 2, name = (null)}
0 <NSThread: 0x101304100>{number = 3, name = (null)}
14 <NSThread: 0x101304100>{number = 3, name = (null)}
...
988 <NSThread: 0x1011caaa0>{number = 2, name = (null)}
804 <NSThread: 0x1015040f0>{number = 7, name = (null)}
923 <NSThread: 0x100710190>{number = 11, name = (null)}
989 <NSThread: 0x10070e380>{number = 5, name = (null)}
```
위의 결과로 보아 대략 **10개 정도의 스레드만 생성**된다.  
Task는 스레드 풀을 활용하여 **Thread Explosion문제를 해결**할 수 있다.  
기존 스레드나 큐에 비해 개선된 점은 **스레드 관리를 직접 하지 않아도 된다**는 부분이다.  

또, Task는 일정 시간 대기 후 작업을 스케줄링 할 수 있다. 기존에는 `sleep` 을 사용하여 비효율적이게 처리하였는데, Task에서는 비동기로 동작하고 예외를 던질 수 있는 정적 메서드 **sleep**이 있다.  
```swift
try await Task.sleep(nanoseconds: UInt64)
```
기존 스레드에서 사용하던 sleep과 매우 다른 메서드이다.  
`Task.sleep()` 은 작업을 일정 시간동안 중단하지만, 스레드를 점유하고 있지 않는다.  
예를 들어, 1000개의 작업을 생성하고 1초동안 중단하도록 하면  
```swift
for n in 0..<1000 {
  Task {
    try await Task.sleep(nanoseconds: NSEC_PER_SEC * 1000)
  }
}
```
대기하는 동안 작업은 중단되며 해당 스레드는 다른 작업에서 사용할 수 있도록 해제된다.  
중단점을 통해 스레드를 확인해보면, 적은 수의 스레드가 생성되고, **단일 스택 프레임만 가지고 있음**을 확인할 수 있다.  
```plaintext
Thread 2#0        0x00000001a05ef604 in __workq_kernreturn ()
Thread 3#0        0x00000001a05ef604 in __workq_kernreturn ()
Thread 4#0        0x00000001a05ef604 in __workq_kernreturn ()
...
```
`__workq_kernreturn()` 함수는 GCD가 스레드를 작업이 스케줄링될 때 까지 대기상태로 유지하는 방식이다. 이는 스레드가 아무 작업을 하지 않는 상태에서 **가벼운 방식으로 관리** 되는 것을 의미하며, 다른 작업을 처리하는데 자유롭게 활용될 수 있다.  

즉, `Task.sleep` 은 **협력적 방식으로 동작**하여 다른 작업이 실행될 수 있도록 한다. 시간이 지나 중단된 작업이 다시 실행되지만, 하나의 스레드를 점유하고 있지 않아 효율적이다.  
또, 다시 작업이 재개될 때 이전과 다른 스레드에서 재개될 수 있다.  
```swift
for _ in 1...workCount {
  Task {
    let current = Thread.current
    try await Task.sleep(nanoseconds: 1_000_000)
    if current != Thread.current {
      print("Thread changed from", current, "to", Thread.current)
    }
  }
}
```

```plaintext
944 Thread changed from <NSThread: 0x10648a790>{number = 2, name = (null)} to <NSThread: 0x1063044c0>{number = 3, name = (null)}
945 Thread changed from <NSThread: 0x1063044c0>{number = 3, name = (null)} to <NSThread: 0x10648a790>{number = 2, name = (null)}
...
```
그렇기 때문에 인접한 코드가 동일한 스레드에서 실행될 것이라고 가정해서는 안된다.

## Task 우선순위와 취소

### 우선순위

Task는 다른 동시성 도구들 처럼 우선순위 개념을 지원한다.  
```swift
Task(priority: .low) {
  print("low")
}
Task(priority: .high) {
  print("high")
}
```

```plaintext
high
low
```
우선순위를 통해 Task의 중요도를 전달할 수 있고, 런타임 시 여러 조건에 따라 작업마다 할당 시간을 적절히 부여할 수 있다.  

### 취소  
취소 또한 다른 도구들과 마찬가지로 지원하는 개념이다.  
```swift
let task = Task {
  print(Thread.current)
}
task.cancel()
```

```plaintext
<NSThread: 0x106204290>{number = 2, name = (null)}
```
Task를 변수에 할당하고, 취소를 요청할 수 있다.  
하지만 print구문이 그대로 출력된다.  
Task는 생성 직후 바로 취소하더라도 작업이 시작되는 경우가 많다.  

**✔️ 협력적 취소**  
Task의 작업을 즉시 중단하기 위해서는 **협력적인 방식**으로 확인해야 한다.  
취소가 요청되었다고 바로 종료되지 않기때문에, 개발자가 **취소 상태를 확인하고 작업을 종료하는 방식으로 종료를 구현**해야 한다.   
이러한 이유는 **열려있는 리소스를 닫아야 하는 경우에 안전성을 보장할 수 있기 때문**이다.  

Task의 취소 여부를 확인하기 위해 `Task.isCancelled` 를 통해 boolean 값을 확인할 수 있다.  
```swift
let task = Task {
  guard !Task.isCancelled else {
    print("Cancelled!")
    return
  }
  print(Thread.current)
}
task.cancel()
```

```plaintext
Cancelled!
```

`Task.isCancelled` 는 현재 Task의 취소 상태를 나타낸다. 이는 전역변수가 아니라 Task에만 국한된 **로컬 값**이기 때문에 현재 실행중인 Task의 취소 여부는 Task 컨텍스트 내에서만 확인할 수 있다.

**✔️ 현재 Task**  
Task에는 `Thread.current` 와 같이 `Task.current` 속성은 존재하지 않는다. 스레드와 달리 모든 코드가 Task 컨텍스트 내에서 실행되는 것이 아니기 때문이다.  
현재 Task를 가져오기 위해서는 `withUnsafeCurrentTask` 를 활용해야 한다.  

```swift
Task {
  withUnsafeCurrentTask { task in
    print(task)
  }
}
```

```plaintext
Optional(Swift.UnsafeCurrentTask(_task: (Opaque Value)))
```
위와 같이 사용하여 가져올 수 있다.  

**✔️ Task의 취소 여부: 예외처리**  
Task의 취소는 언어 레벨에서 더 깊이 통합되어 있다.  
비동기 작업에서 작업이 실패할 가능성이 있는 경우, 취소 여부를 확인하는 방식으로 `Task.isCancelled` 를 통한 boolean값 검사 대신 **예외처리**를 사용하여 효과적으로 사용할 수 있다.  
```swift
let task = Task {
  try Task.checkCancellation()
  print(Thread.current)
}
task.cancel()
```
`Task.checkCancelled()` 를 호출할 경우 Task가 취소된 상태라면 예외를 던지게 되고, 그 뒤의 나머지 코드는 실행되지 않는다.  

**✔️ 협력적 취소: 취소의 전달**  
Thread나 DispatchQueue에서는 스레드를 sleep상태로 만들거나 내부 작업이 생성되었을 때 취소가 전달되지 않는 문제가 있었다.  
하지만 Task에서는 **내부의 비동기 작업 또한 취소되는 협력적 취소 개념**을 가지고 있다.  

예를 들어, Task를 1초 동안 sleep 상태로 만든 후 0.1초 뒤에 취소해보자.  
```swift
let task = Task {
  let start = Date()
  defer { print("Task finished in", Date().timeIntervalSince(start)) }
  try await Task.sleep(nanoseconds: NSEC_PER_SEC)
  print(Thread.current) // 스레드 정보 출력
}

Thread.sleep(forTimeInterval: 0.1)
task.cancel()
```

```plaintext
Task finished in 0.10534894466400146
```
위와 같은 결과는 `Task.sleep`이 취소 이벤트를 확인하고, `sleep`을 조기에 종료하였음을 의미한다.  
즉, Task가 취소됨에 따라 **나머지 작업이 즉시 중단**되었다.  

이번에는 비동기 함수 내부에서 `sleep`을 호출한 후 취소를 해보자.  
```swift
func doSomething() async throws {
  try await Task.sleep(nanoseconds: NSEC_PER_SEC)
}

let task = Task {
  let start = Date()
  defer { print("Task finished in", Date().timeIntervalSince(start)) }

  try await doSomething()
  print(Thread.current)
}

Thread.sleep(forTimeInterval: 0.1)
task.cancel()
```

```plaintext
Task finished in 0.10534894466400146
```
결과는 동일하다.   
스레드, OperationQueue, Dispatch Queue에서는 작업 단위의 취소 상태를 관찰하고 직접 취소를 구현해야 했지만 Task에서는 취소가 내부작업까지 전달된다.  

또 다른 예시로, URLSession을 통해 큰 파일을 다운로드하고, 다운로드가 완료되기 전에 취소를 해보자.  
```swift
let task = Task {
  let start = Date()
  defer { print("Task finished in", Date().timeIntervalSince(start)) }

  let (data, _) = try await URLSession.shared.data(
    from: URL(string: "<http://ipv4.download.thinkbroadband.com/1MB.zip>")!
  )
  print(Thread.current, "network request finished", data.count)
}

Thread.sleep(forTimeInterval: 0.5)
task.cancel()
```

```plaintext
Task finished in 0.5094579458236694
```
“network request finished”가 출력되지 않는 것으로 보아 **Task는 네트워크 요청 진행 중일 때에도 취소를 감지하고 작업을 종료할 수 있다**.  

Task의 협력적 취소는 **리소스를 절약**할 수 있도록 동작하는 것을 볼 수 있다.  

## Task Locals  
기존 스레드와 Dispatch Queue에서는 데이터를 저장하기 위해 **스레드 딕셔너리**와 **specifics**를 사용하였다.  
해당 방식에는 몇가지 문제점이 있었다.  
- 스레드 딕셔너리는 [AnyHashable: Any] 타입을 사용했기 때문에 타입 안정성이 부족했고, 강제 캐스팅을 사용해야 했다.
- 새로운 스레드를 생성하였을 때 기존의 스레드 딕셔너리를 상속할 수 없었다.
- DispatchQueue에서 기존 큐를 타겟으로 설정하여 specifics를 전달할 수 있었지만, 이를 명시적으로 값을 전달하는 작업을 구현해야 하고, 모든 큐에서 공통으로 접근할 수 있는 “현재 큐”의 개념이 없다.

Task는 이러한 한계점을 보완하기 위해 **Task Local Values**라는 기능을 제공한다.  
Task Local값을 사용하면 특정 값을 Task에 연결하고, 해당 Task 컨텍스트 내에서 어디든 해당 값을 쉽게 조회할 수 있다.  

**✔️ TaskLocal 정의**  
이전에 예시로 사용한 네트워크 통신 시 효율적인 로그 조회를 위한 요청 ID 할당을 사용해보자.  
Task에 저장할 값을 보관하기 위한 타입을 정의한다.  
```swift
enum MyLocals {
  @TaskLocal static var id: Int!
}
```
**@TaskLocal** property wrapper를 사용하여 정적 변수를 정의한다.  
변수는 반드시 static이어야 하므로 기본 값을 설정해야 한다. 임의의 기본 값을 사용하는 것 보다 사용하기 전에 초기화를 반드시 해야하는 명시적 언래핑 옵셔널을 사용하는 것이 나은 선택이다.

**✔️ TaskLocal 값 조회**  
```swift
print(MyLocals.id) // nil
```
MyLocals 타입에 직접 접근하여 확인할 수 있다.  

**✔️ 값 설정**  
`withValue` 메서드를 사용하여 값을 설정할 수 있다.  
이 메서드는 설정하려는 값과 클로저를 매개변수로 가지고 있다.  
```swift
MyLocals.$id.withValue(
  valueDuringOperation: Int?, operation: () throws -> R
)
```
이러한 방식은 기존 스레드나 DispatchQueue에서 직접 값을 수정할 수 있던 방식과 다르다.    
클로저를 통해 값의 수명을 제어하는 방식으로 설계되어 클로저 실행 동안에만 값이 설정되고, `withValue` 실행이 끝나면 원래 값으로 복원된다.  
매개변수 이름 `valueDuringOperation` 도 이러한 기능을 암시한다.  
```swift
print("before:", MyLocals.id)
MyLocals.$id.withValue(42) {
  print("withValue:", MyLocals.id!)
}
print("after:", MyLocals.id)
```

```plaintext
before: nil
withValue: 42
after: nil
```
값을 설정하고 클로저 내부에서 값을 확인해보면 클로저 내부에서는 값이 변경되지만 클로저 실행이 끝나고 나면 다시 nil로 돌아간다.  
변경된 값이 클로저 내부에서만 유효하지만 이 값을 더 오래 유지할 수 있는 방법이 있다.  
새로운 Task를 시작하게되면 해당 시점의 모든 Task Local 값이 자동으로 상속된다.  

**✔️ 상속**  
```swift
print("before:", MyLocals.id)
MyLocals.$id.withValue(42) {
  print("withValue:", MyLocals.id!)
  Task {
    try await Task.sleep(nanoseconds: NSEC_PER_SEC)
    Task {
      print("Task:", MyLocals.id!)
    }
  }
}
print("after:", MyLocals.id)
```

```plaintext
before: nil
withValue: 42
after: nil
Task: 42
```
내부에 **새롭게 생성된 Task**는 `withValue`가 종료된 이후에도 **변경된 값을 유지**하는 것을 볼 수 있다.  
이러한 것이 가능한 이유는 새로운 Task를 생성하는 순간 **현재 Task의 Task Local 값이 캡쳐**되기 때문이다. 그렇기 때문에 `withValue` 클로저가 끝난 후 nil로 초기화 되어도 **새로운 Task에서는 값이 유지**가 된다.  

다른 비동기 함수를 호출해도 캡쳐된 값이 함수 내에서도 사용 가능하다.  
```swift
func doSomething() async {
  print("doSomething:", MyLocals.id!)
}

print("before:", MyLocals.id)
MyLocals.$id.withValue(42) {
  print("withValue:", MyLocals.id!)
  Task {
    try await Task.sleep(nanoseconds: NSEC_PER_SEC)
    Task {
      print("withValue Task:", MyLocals.id!)
      await doSomething()
    }
  }
}
print("after:", MyLocals.id)
```

```plaintext
before: nil
withValue: 42
after: nil
Task: 42
doSomething: 42
```
Task Local은 **특정 범위의 생명주기를 기준으로 값을 설정**하도록 설계되어있다.    
이를 이용하여 이전의 스레드나 DispatchQueue 방식보다 비동기 작업 흐름 속에서 값을 전달할 수 있다.

## Task Cooperation  
여러 작업 단위를 동시에 실행하려 할 때 스레드는 Thread Explosion 문제가 발생하고, 모든 스레드가 CPU 시간을 놓고 경쟁을 하는 상황이 발생했다. 반면 Diapatch Queue는 스레드 풀을 사용하여 스레드의 수를 낮게 유지하여 과부하를 방지했지만, 풀 내의 스레드 수가 제한되어 있기 때문에 너무 많거나 긴 작업이 할당될 경우 작업 처리가 오래걸리게 된다.

```swift
for _ in 0..<1000 {
  Thread.detachNewThread {
    while true {}
  }
}

Thread.detachNewThread {
  print("Starting prime thread")
  nthPrime(50_000)
}
```
이전에 위와 같이 1000개의 스레드를 생성하여 무한 루프에 빠지게 하여 바쁘게 유지한 뒤 새로운 스레드를 생성하여 50,000번째 소수를 구하도록 하여 테스트를 해보았다.  
작업 처리 시간이 오래 걸리거나 스레드 풀이 막혀 작업을 진행할 수 없는 상황이 발생했었다.  

위와 같은 상황을 Task를 사용하도록 변경하여 살펴보자.
```swift
for _ in 0..<workCount {
  Task {
    while true {}
  }
}

Task {
  print("Starting prime task")
  nthPrime(primeN)
}

```
이 코드를 실행하면 아무 출력도 나타나지 않게 되는데, 중단점을 통해 스레드 상태를 확인해보면 10개의 스레드가 활성 상태에 있지만 모두 무한 루프에 의해 막혀있는 것을 볼 수 있게 된다.  

이러한 제한이 발생하는 이유는 **컴퓨터에는 결국 유한한 수의 코어가 있기 때문**이다.  
코어 수 보다 훨씬 더 많은 스레드를 생성할 수 있다고 하더라도 무분별하게 사용하게 되면 스레드와 리소스의 폭발적인 증가로 이어져 스레드가 실행 시간을 두고 싸우게 될 수 있다.  
Swift는 작업을 처리하기 위해 무한히 많은 스레드를 생성할 수 있는 척하지 않고, **더 작고 합리적인 수의 스레드를 생성**하도록 한다.  
또, 비동기 작업을 처리할 대 모든 작업이 **제한된 스레드를 함께 사용**하도록 한다. 즉, **다른 작업도 실행할 수 있도록 각 작업이 협력**해야 한다.

**✔️ Non-Blocking API**  
Swift는 이 협력을 이루기 위해 Non-Blocking API를 사용한다. 비동기 컨텍스트에서 오랜 시간 동안 블로킹 하지 않는 것이 중요하기 때문이다.  

이전에 살펴본 `await` 을 통해 Non-Blocking 작업을 수행할 수 있다.  
```swift
for n in 0..<1000 {
  Task {
    let (data, _) = try await URLSession.shared.data(
      from: URL(
        string: "<http://ipv4.download.thinkbroadband.com/1MB.zip>"
      )!
    )
    print(n, data.count, Thread.current)
  }
}
```
위 코드는 네트워크 요청을 처리하면서 다른 작업도 동시에 진행할 수 있도록 한다.  

**✔️ Task.yield()**  
비동기 컨텍스트에서 `await` 을 통해 문제없이 Non-Blocking 작업을 수행할 수 있지만 때로는 어느정도 스레드를 점유해야 할 정도로 무거운 계산 작업을 수행해야 할 경우가 있다.  
이러한 경우 **Task의 스레드를 일시적으로 양보하여 다른 작업이 스레드를 사용할 수 있도록 하는 도구**를 제공한다.  
```swift
await Task.yield()
```
`yield` 메서드는 비동기 메서드로 `await`과 함께 사용해야 한다.  
**Task.yield()** 는 **현재 태스크의 스레드를 중단**하고 **다른 태스크에서 사용할 수 있도록** 한다. 이후 제어권이 다시 돌아오면 이어서 작업을 수행할 수 있다.  
스레드를 양보함으로써 동시에 여러 작업을 실행하더라도 전체 스레드를 점유하지 않게 된다.        