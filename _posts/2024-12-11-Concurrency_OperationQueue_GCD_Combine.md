---
title: "[iOS] 동시성 프로그래밍 - OperationQueue, GCD, Combine"
date: 2024-12-11
categories:
  - PointFree스터디
  - Concurrency
tags:
  - Concurrency
  - GCD
  - OperationQueue
  - Combine
comments: "true"
---
<span style="color:gray">PointFree의 Concurrency 세션 정리 내용입니다!</span>

🔗 이전 내용  
[1. 동시성의 과거: 스레드](https://ji-yeon224.github.io/posts/Concurrency_Thread/)

----
# [🔗](https://www.pointfree.co/episodes/ep191-concurrency-s-present-queues-and-combine) Concurrency’s Present: Queues and Combine

## Operation Queue

Operation Queue는 스레드의 몇가지 문제를 해결하기 위해 도입되었다.  
Operation Queue는 작업을 수행하는 시스템과 실제 작업을 분리한다. 스레드는 직접 스레드를 생성하고 해당 스레드에 수행할 작업을 제공해야 했다.  
Operation Queue를 생성하고 실행 할 작업을 큐에 추가하면 시스템 내부에서 스레드를 생성하여 작업을 할당하게 된다.
```swift
// Thread:
let thread = Thread {
    // 스레드에서 수행할 작업
}
thread.start()
```

```swift
// Operation Queue:
let queue = OperationQueue()
queue.addOperaion{
  print(Thread.current)
}
```

```plaintext
<NSThread: 0x100904c20>{number = 2, name = (null)}
```
비동기로 실행하기 위해 스레드가 생성되었다.

### 스레드와의 공통점

**실행 순서**  
스레드와 마찬가지로 여러 작업을 한 번에 요청 시 실행 순서가 보장되지 않는다.  
![image](https://github.com/user-attachments/assets/65280cb2-b8d9-4b2b-b9d1-3d742b9f2052)
Operation Queue가 추가 스레드를 생성하여 작업을 병렬로 처리한다.  

**우선순위**  
또, 스레드와 마찬가지로 작업의 우선순위를 지정할 수 있다.

```swift
let operation = BlockOperation {  
  print(Thread.current)  
}  
operation.qualityOfService = .background  

queue.addOperation(operation)  
```

우선순위를 지정하기 위해서는 작업을 `BlockOperation` 을 통해 클로저로 정의해야 한다.  
우선순위는 **qualityOfService** 필드를 통해 우선순위를 설정할 수 있다.  

**취소**  
스레드와 동일하게 취소 기능도 지원한다.  

```swift
let operation = BlockOperation {  
  Thread.sleep(forTimeInterval: 1)  
  print(Thread.current)  
}  
Thread.sleep(forTimeInterval: 0.1)  
operation.cancel()  
```

하지만 해당 코드를 실행하면 작업 내 print가 출력된다. 스레드와 마찬가지로 시스템 내부에서 취소 작업을 처리해주지 않는다. 직접 남은 작업이 취소되었는지 정기적으로 확인하여 중단 여부를 직접 정해야 한다.  

### 스레드와의 차이점

✔️ **현재 작업**  
스레드와 달리 Operation Queue에는 취소를 확인할 수 있는 **현재 실행 중인 작업**에 대한 개념이 없다.   `BlockOperation`을 사용하여 작업 객체를 명시적으로 생성한 후에야 취소 여부를 확인할 수 있다.  

```swift
let operation = BlockOperation()
operation.addExecutionBlock { [unowned operation] in
  Thread.sleep(forTimeInterval: 1)
  guard !operation.isCancelled else {
    print("Cancelled!")
    return
  }
  print(Thread.current)
}
```

`BlockOperation`에서 작업을 초기화 한 후 블록 내에서 참조하여 사용할 수 있다.  
하지만 스레드에서 처럼 `sleep` 상태를 유지한 후 종료된다.  

**✔️ 딕셔너리**  
스레드에서 사용한 스레드 딕셔너리와 같이 데이터를 저장하는 기능을 지원하지 않는다.

✔️ **의존성(Dependencies)**  
Operation Queue는 의존성 개념을 지원한다.  
이를 이용하여 한 작업이 완료된 후 다른 작업을 시작하도록 설정할 수 있다.  

```swift
let queue = OperationQueue()

let operationA = BlockOperation {
  print("A")
  Thread.sleep(forTimeInterval: 1)
}
let operationB = BlockOperation {
  print("B")
}
operationB.addDependency(operationA)
queue.addOperation(operationA)
queue.addOperation(operationB)

// 출력:
// A
// B
```

`operationB.addDependency(operationA)` 를 통해 의존 관계를 설정할 수 있다.  
스레드에서 동일한 방식으로 구현하기 위해서는 다른 스레드가 완료되었는지 확인하기 위해 while루프를 사용하여 지속적으로 확인해야 했다.  

### **스레드가 가진 문제 해결: Thread Explosion 및 Starvation 문제**

스레드는 작업을 수행할 때 마다 스레드를 생성하게 되면 수천개의 스레드가 동시에 실행되는 상황이 발생할 수 있다. CPU를 점유하기 위해 서로 경쟁 상태에 들어가게 되며 Thread Starvation이 발생할 수 있다.  

Operation Queue는 현재 실행 중인 작업과 완료된 작업을 전체적으로 파악할 수 있기 때문에 효율적으로 처리할 수 있다.  

```swift
for n in 0..<1000 {
  queue.addOperation { print(n, Thread.current) }
}
```

결과:  
![image](https://github.com/user-attachments/assets/1772f768-688c-40e6-9fd4-073ac315499e){: width="70%"}


1000개의 작업을 요청했지만 60개 정도의 스레드만 생성되었다.  
스레드 풀의 개념과 같이 동작한다. 새로운 작업을 수행할 때 매번 스레드를 생성하는 대신 풀에서 스레드를 요청하여 작업을 수행하도록 한다.  

### Operation Queue의 문제점

**✔️ 비동기 작업에서의 Blocking 문제**  
Operation Queue는 비동기 작업에서 non-blocking방식으로 처리할 수 있는 도구가 없다.  
특정 작업을 일정시간 기다렸다가 실행하려면 `Thread.sleep`을 사용해야 한다.  

```swift
queue.addOperation {
  Thread.sleep(forTimeInterval: 1)
  print(n, Thread.current)
}
```

**✔️ 비동기 작업 간 협력 부족**  
다른 작업과 자원을 효율적으로 공유하거나 협력할 방법을 제공하지 않는다.  

예를 들어, 특정 작업이 대기 중일 경우 자원을 반환하고 다른 작업이 이를 활용하도록 할 방법이 없다. 이로 인해 작업들 간 경쟁이 발생하며 많은 자원을 소모하는 작업이 다른 작업의 수행을 방해할 수 있다.  
```swift
for n in 0..<1000 {
  queue.addOperation {
    print(n, Thread.current)
    while true {} // CPU를 계속 사용
  
}
queue.addOperation {
  print("Starting prime operation")
  nthPrime(50_000) // 50,000번째 소수 계산
}
```
위의 경우 무한루프로 인해 일부 스레드를 계속 점유하고 있는 상태이기 때문에 소수 계산 작업이 실행되지 않는다.  

**✔️ 취소 제한**  
특정 작업을 취소하더라도 해당 작업에 의존성을 가진 다른 작업은 취소되지 않는다.  
```swift
operationB.addDependency(operationA)
operationC.addDependency(operationA)
operationD.addDependency(operationB)
operationD.addDependency(operationC)

operationA.cancel()
```
이전에 사용하였던 의존성 코드를 예시로 사용해볼 때 operationA를 취소하더라도 나머지 B, C, D는 취소되지 않고 그대로 실행된다.  

**✔️ 복잡한 API 설계**  
Operation Queue는 OOP에서 영감을 받아 설계되었기에 비효율적인 모습이 발견된다.  
```swift
let queue = OperationQueue()

let operationA = BlockOperation {
    print("A")
    Thread.sleep(forTimeInterval: 1)
}
let operationB = BlockOperation { print("B") }
let operationC = BlockOperation { print("C") }
let operationD = BlockOperation { print("D") }

operationB.addDependency(operationA)
operationC.addDependency(operationA)
operationD.addDependency(operationB)
operationD.addDependency(operationC)

queue.addOperation(operationA)
queue.addOperation(operationB)
queue.addOperation(operationC)
queue.addOperation(operationD)
```

위의 코드를 살펴보면  
- 모든 작업 선언
- 작업간 의존성 설정
- 모든 작업을 큐에 추가  
와 같은 과정으로 코드를 작성하게 된다. 이와 같은 코드는 직관적이지 못해 작업의 실행 흐름을 이해하기 어렵게 한다.  

**✔️ Race Condition(데이터 경쟁) 문제**  
멀티스레드 환경에서 발생할 수 있는 데이터 경쟁 문제를 해결하지 못한다.  
스레드가 가진 몇가지 문제를 해결하긴 했지만 모든 문제를 해결하지 못해 GCD나 Swift Concurrecny를 사용해야 한다.  

## GCD**(Grand Central Dispatch)**
GCD는 스레드 관점에서의 동시성이 아닌 **큐를 중심**으로 동시성을 생각하도록 설계되었다.  
GCD는 다음과 같이 생성하고 작업을 추가할 수 있다.  
```swift
let queue = DispatchQueue(label: "my.queue")
queue.async {
    print(Thread.current)
}
```

```plaintext
<NSThread: 0x101012230>{number = 2, name = (null)}
```

### GCD의 특징
**✔️ Serial Queue**  
GCD는 **기본적으로 Serial Queue로 설정**되어 있다.  
```swift
queue.async { print("1", Thread.current) }
queue.async { print("2", Thread.current) }
queue.async { print("3", Thread.current) }
queue.async { print("4", Thread.current) }
queue.async { print("5", Thread.current) }
```

```plaintext
1 <NSThread: 0x10601fd70>{number = 2, name = (null)}
2 <NSThread: 0x10601fd70>{number = 2, name = (null)}
3 <NSThread: 0x10601fd70>{number = 2, name = (null)}
4 <NSThread: 0x10601fd70>{number = 2, name = (null)}
5 <NSThread: 0x10601fd70>{number = 2, name = (null)}
```
기본으로 Serial Queue이기 때문에 동일한 스레드에서 작업이 순차적으로 실행된다. 이는 동시성 코드의 복잡성을 줄이기 위해 의도된 것이다.  
Concurrent Queue를 사용하려면 명시적으로 설정해야 한다.  
```swift
let queue = DispatchQueue(label: "my.queue", attributes: .concurrent)
```

**✔️ Thread explosion 방지**  
GCD는 많은 작업을 큐에 추가해도 스레드가 많이 생성되지 않는다.  
```swift
for n in 0..<workCount {
  queue.async {
    print(n, Thread.current)
  }
}
```

```plaintext
0 <NSThread: 0x101108070>{number = 2, name = (null)}
7 <NSThread: 0x101108070>{number = 2, name = (null)}
10 <NSThread: 0x101604330>{number = 10, name = (null)}
5 <NSThread: 0x1014996c0>{number = 7, name = (null)}
…
929 <NSThread: 0x101498d30>{number = 29, name = (null)}
116 <NSThread: 0x10070f940>{number = 57, name = (null)}
999 <NSThread: 0x101204d20>{number = 30, name = (null)}
```

**✔️ Non-Blocking**  
GCD는 특정 시간이 지난 후 작업을 수행하도록 하는 기능을 제공한다.  
GCD는 `Thread.sleep` 과 같이 스레드를 차단하여 리소스 낭비 문제를 해결하였다. `asyncAfter` 를 통해 일정 시간 지난 후 수행하도록 설정할 수 있다.  
```swift
print("before scheduling")
queue.asyncAfter(deadline: .now() + 1) {
    print("1 second passed")
}
print("after scheduling")
```

```plaintext
before scheduling
after scheduling
1 second passed
```

**✔️ 우선순위(QoS)**  
GCD 또한 **QoS(Quality of Service)** 를 사용하여 우선순위를 지정할 수 있다.  
```swift
let queue = DispatchQueue(label: "my.queue", qos: .background)
```

**✔️ DispatchWorkItem**  
GCD에서 작업 단위를 생성하기 위해 `DispatchWorkItem` 을 사용할 수 있다.  
```swift
let item = DispatchWorkItem {
    print(Thread.current)
}
queue.async(execute: item)
```

**✔️ 작업 취소**  
`DispatchWorkItem` 으로 생성한 작업 아이템은 실행 도중 작업 취소를 수행할 수 있다. GCD 또한 협력적으로 동작하기 때문에 실행 중 직접 취소 여부를 확인해야 한다.  
스레드 처럼 현재 실행 중인 디스패치 큐를 확인할 수 있는 개념이 존재하지 않아 취소 여부를 확인하려면 해당 큐 자체에 접근해야 한다.  
```swift
var item: DispatchWorkItem!
item = DispatchWorkItem {
  defer { item = nil }
  let start = Date()
  defer { print("Finished in", Date().timeIntervalSince(start)) }
  Thread.sleep(forTimeInterval: 1)
  guard !item.isCancelled else {
    print("Cancelled!")
    return
  }
  print(Thread.current)
}
queue.async(execute: item)

Thread.sleep(forTimeInterval: 0.5)
item.cancel()
```

```plaintext
Cancelled!  
Finished in 1.0974069833755493
```
0.5초 후에 취소를 요청했지만 완료 시간은 1초가 걸렸다. 시스템적으로 강제로 적용되는 것이 아니기 때문에 1초 동안 `sleep`을 유지하게 되고, 이 `sleep`을 중단할 수 없다.  

**✔️ Specifics**  
스레드의 딕셔너리와 유사하게 **Specifics**가 존재한다.  
스레드 딕셔너리와 같은 방식으로 데이터를 저장하고 검색할 수 있다.  
예시로 네트워크 요청을 처리하는 코드를 사용해보자.  
```swift
func response(for request: URLRequest) -> HTTPURLResponse {
  // 요청을 응답으로 변환하는 작업을 수행
  return .init()
}

let item = DispatchWorkItem {
  response(for: .init(url: URL(string: "<https://www.pointfree.co>")!))
}

let requestId = UUID()
let queue = DispatchQueue(label: "request-\\(requestId)")
queue.async(execute: item)
```

해당 코드에서 고유 요청 ID값을 통해 특정 로그를 쉽게 확인할 수 있도록 해보려 한다.  
```swift
func response(for request: URLRequest) -> HTTPURLResponse {
  let requestId = DispatchQueue.getSpecific(key: requestIdKey)! // 값 검색
  print(requestId, "Making database query")
  Thread.sleep(forTimeInterval: 0.5)
  print(requestId, "Finished database query")
  print(requestId, "Making network request")
  Thread.sleep(forTimeInterval: 0.5)
  print(requestId, "Finished network request")

  return .init()
}

let item = DispatchWorkItem {
  response(for: .init(url: URL(string: "<https://www.pointfree.co>")!))
}

let requestIdKey = DispatchSpecificKey<UUID>() // Specific 키 생성 및 타입 설정
let requestId = UUID()
let queue = DispatchQueue(label: "request-\\(requestId)")
queue.async(execute: item)
queue.setSpecific(key: requestIdKey, value: requestId) // 값 설정
```
스레드 딕셔너리와 달리 데이터의 타입을 명시적으로 지정할 수 있다.  

**Specific의 상속 문제**  
스레드 딕셔너리와 마찬가지로 내부에서 새로 생성된 디스패치 큐에 데이터를 자동으로 상속하지 않는다.  
디스패치 큐에는 이러한 문제를 해결하기 위한 **Targeting**이라는 도구를 제공한다. 이를 통해 새 큐를 부모 큐에 타겟팅하면 Specific 데이터를 상속받을 수 있다.  
```swift
let queue2 = DispatchQueue(label: "queue2", target: queue1) // 타겟팅
queue2.setSpecific(key: idKey, value: 1729)
queue2.async {
  print("queue2", "id", DispatchQueue.getSpecific(key: idKey))
  print("queue2", "date", DispatchQueue.getSpecific(key: dateKey))
}
```

```plaintext
queue1 id Optional(42)  
queue1 date Optional(2022-05-18 17:36:26 +0000)  
queue2 id Optional(1729)  
queue2 date Optional(2022-05-18 17:36:26 +0000)
```

**✔️ 예제**  
데이터베이스 쿼리와 네트워크 요청을 병렬로 실행하도록 해보자.  
```swift
func makeDatabaseQuery() {
  let requestId = DispatchQueue.getSpecific(key: requestIdKey)!
  print(requestId, "Making database query")
  Thread.sleep(forTimeInterval: 0.5)
  print(requestId, "Finished database query")
}

func makeNetworkRequest() {
  let requestId = DispatchQueue.getSpecific(key: requestIdKey)!
  print(requestId, "Making network request")
  Thread.sleep(forTimeInterval: 0.5)
  print(requestId, "Finished network request")
}
```

```swift
func response(for request: URLRequest) -> HTTPURLResponse {
  let databaseQueue = DispatchQueue(label: "database-query")
  databaseQueue.async {
    makeDatabaseQuery()
  }

  let networkQueue = DispatchQueue(label: "network-request")
  networkQueue.async {
    makeNetworkRequest()
  }

  return .init()
}
let item = DispatchWorkItem {
  response(for: .init(url: URL(string: "<https://www.pointfree.co>")!))
}

let requestIdKey = DispatchSpecificKey<UUID>() 
let requestId = UUID()
let queue = DispatchQueue(label: "request-\\(requestId)")
queue.async(execute: item)
queue.setSpecific(key: requestIdKey, value: requestId) 
```
response 메서드에서 각 작업을 실행하기 위한 큐를 생성한다. 비동기적으로 작업하기 위해 `async` 를 사용하는데 `async` 는 non-blocking작업이기 때문에 각 작업이 실행되기 전에 메서드가 종료된다.  
두 작업이 완료될 때 까지 기다리지 않는 문제가 발생하게 된다.  

**DispatchGroup**  
이러한 문제를 해결하기 위해 **DispatchGroup**을 사용할 수 있다.  
DispatchGroup을 통해 여러 작업을 하나의 작업처럼 동작하도록 한다.  
```swift
func response(for request: URLRequest) -> HTTPURLResponse {
  let databaseQueue = DispatchQueue(label: "database-query")
  
  let group = DispatchGroup() // DispatchGroup 생성
    databaseQueue.async(group: group) { // group 부여
      makeDatabaseQuery()
    }
    networkQueue.async(group: group) {
      makeNetworkRequest()
    }
    group.wait()
	
   return .init()
}
```
DispatchGroup을 생성하고 각 비동기 작업을 하나의 그룹 내에서 작업을 처리하게 된다.  
모든 작업이 끝날 때 까지 기다린 후 응답 값을 반환한다.  
각 작업에 specific을 부여해줄 수 있다.  
```swift
func response(
  for request: URLRequest, queue: DispatchQueue
) -> HTTPURLResponse {
  let group = DispatchGroup()

  let databaseQueue = DispatchQueue(
    label: "database-request", target: queue
  )
  databaseQueue.async(group: group) {
    makeDatabaseQuery()
  }

  let networkQueue = DispatchQueue(
    label: "network-request", target: queue
  )
  networkQueue.async(group: group) {
    makeNetworkRequest()
  }

  group.wait()

  return .init()
}

```

```swift
let queue = DispatchQueue(
  label: "request-\\(requestId)", attributes: .concurrent
)
response(
  for: .init(url: URL(string: "<https://www.pointfree.co>")!),
  queue: queue
)
```
메서드의 인수로 상속 받을 큐의 정보를 넘기고, 새로운 큐를 생성할 때 target을 부여하도록 하였다. 해당 작업들을 병렬로 실행해야 하기 때문에 부모 큐의 타입을 concurrent로 설정한다.

```plaintext
48A7C5DD-5F02-4234-923C-F96315A42214 Making database query
48A7C5DD-5F02-4234-923C-F96315A42214 Making network request
48A7C5DD-5F02-4234-923C-F96315A42214 Finished database query
48A7C5DD-5F02-4234-923C-F96315A42214 Finished network request
```
위의 코드를 더 개선해보자면 현재는 각 요청마다 새로운 concurrent 큐를 생성하는데, 이는 비효율적이다.  
서버가 시작될 때 하나의 concurrent 큐를 만들고 요청이 들어올 때 마다 해당 큐를 타겟으로 하는 새로운 큐를 만들도록 한다. 타겟팅을 하도록 하여 독립적인 큐를 생성하는 것이 아니라 연결된 큐를 생성하게 된다.  

이를 통해 sepcific을 공유할 수 있고, 하나의 스레드 풀에서 작업이 분배되어 Thread Explosion 문제를 피할 수 있다.
```swift
let serverQueue = DispatchQueue(
  label: "server", attributes: .concurrent
)
let requestId = UUID()
let requestIdKey = DispatchSpecificKey<UUID>()
let queue = DispatchQueue(
  label: "request-\\(requestId)",
  attributes: .concurrent,
  target: serverQueue
)
queue.setSpecific(key: requestIdKey, value: requestId)
queue.async {
  response(for: .init(url: URL(string: "<https://www.pointfree.co>")!))
}
```

### GCD의 문제점

**✔️ 상속 기능의 한계**  
기본 큐의 specifics를 상속받을 수 있는 기능은 유용하지만 한계가 있다.  
```swift
response(
  for: .init(url: URL(string: "<https://www.pointfree.co>")!),
  queue: requestQueue
)
```
위와 같이 함수에 큐를 전달하여 상속 받도록 할 수 있지만 이는 데이터를 여러 계층에 걸쳐 전달하지 않으려는 목적에는 반하는 것이다.  
새로운 큐를 생성할 때 부모 큐의 specific을 자동으로 상속받는 구조가 더 바람직하다.

**✔️ 작업 취소의 한계**  
Dispatch Queue는 specific을 상속받을 수는 있지만 **취소는 상속받지 못한다**.  
부모 작업이 취소되면 하위 작업도 취소되는 기능은 여전히 불가능하다.  

**✔️ Thread Explosion 문제**  
DispatchQueue는 과도한 스레드 생성 문제를 줄이긴 하지만 제대로 사용하지 않으면 여전히 문제가 발생할 수 있다.  
```swift
for n in 0..<1000 {
  DispatchQueue(label: "queue-\\(n)").async {
    print(Thread.current)
    while true {}
  }
}

Thread.sleep(forTimeInterval: 3)
```
다음과 같은 코드는 매번 DispatchQueue를 생성하기 때문에 과도한 스레드가 생성된다.  
또, Thread Explosion 문제를 해결하더라도 단일 큐에서 너무 많은 작업을 수행할 경우 다른 작업이 실행되지 않는 문제가 발생할 가능성이 존재한다.  

**✔️ 작업 간 협력 문제**  
DispatchQueue는 다른 작업과 협력할 수 있는 도구를 제공하지 않는다.  
네트워크 요청이나 데이터베이스 요청과 같은 대기 시간이 긴 작업을 실행하는 동안 다른 작업이 동일한 스레드를 사용할 수 없다. 대기 중인 작업들은 CPU 시간을 두고 경쟁해야 하며, 리소스를 많이 사용하는 작업은 다른 작업의 실행 시간을 빼앗게 된다.  

**✔️ DataRace 문제**  
GCD는 데이터 접근을 동기화 할 수 있는 도구를 제공하지만 이는 이전에 사용한 `Lock`과 유사하다.  
```swift
class Counter {
  var count = 0
  func increment() {
    self.count += 1
  }
}
let counter = Counter()

let queue = DispatchQueue(label: "concurrent-queue", attributes: .concurrent)

for _ in 0..<1000 {
  queue.async {
    counter.increment()
  }
}

Thread.sleep(forTimeInterval: 1)
print("counter.count", counter.count)
```

```plaintext
출력: counter.count 996
```
위의 코드를 실행하면 1000에 도달하지 못하지만 스레드를 사용할 때보다 **data race 문제가 적게 발생함**을 볼 수 있다.  
여전히 data race 문제가 발생할 수 있고, 이러한 문제를 해결하기 위해 GCD는 “**barrier”** 라는 도구를 제공한다.  

barrier를 사용하여 큐가 현재 작업을 모두 완료한 후에만 다음 작업 단위를 실행하도록 할 수 있다.  
```swift
class Counter {
  let queue = DispatchQueue(label: "counter", attributes: .concurrent)
  var count = 0
  func increment() {
    self.queue.sync(flags: .barrier) {
      self.count += 1
    }
  }
}
```
이제 동일한 1,000이라는 결과를 얻을 수 있지만 이는 시간이 매우 오래 걸린다.  
동시성 환경에서 각각 락 작업을 수행해야 하기 때문에 작업이 느려지고, barrier는 NSLock보다 더 느리기 때문이다.  

## Combine
Combine은 **시간에 따라 값의 스트림을 표현할 수 있는 라이브러리**로 설계되었다. Combine은 동시성을 수행하거나 비동기 작업을 위한 라이브러리가 아니다.  
해당 작업이 가능하지만, DispatchQueue, OperationQueue, Run Loop 등 동시성 도구에 의존하여 수행한다.  

Combine을 사용하여 우리가 동시성을 다룰 때 발생하는 문제들을 해결하는데 도움을 준다.  
먼저, 단일 값을 방출할 수 있는 Future Publisher를 생성해보자.  
```swift
let publisher = Deferred {
  Future<Int, Never> { callback in
    print(Thread.current)
    callback(.success(42))
  }
}
```
Future publisher는 생성 즉시 실행되기 때문에 구독 시점에 실행되도록 Deferred 퍼블리셔로 감싸야 한다.  

```swift
let cancellable = publisher
  .sink {
    print("sink", $0, Thread.current)
  }

_ = cancellable
```
퍼블리셔에서 값을 얻기 위해 `.sink`를 통해 구독해야하고, 구독을 관리하기 위해 반환되는 Cancellable 객체를 저장한다.  
```plaintext
<_NSMainThread: 0x10610a920>{number = 1, name = main}
publisher 42 <_NSMainThread: 0x10610a920>{number = 1, name = main}
```
위의 출력 결과로 보아 비동기 작업이 이루지지 않은 것을 볼 수 있다.  

**✔️ zip**  
Combine은 publisher를 시간에 따라 변하는 값의 시퀀스로 생각하고, 이를 다양한 연산자를 사용해서 결합하여 사용할 수 있는 기능을 제공한다.  
다음은 두 퍼블리셔를 **zip**연산자를 사용하여 결합해 하나의 퍼블리셔를 생성하는 코드이다.
```swift
let publisher1 = Deferred {
  Future<Int, Never> { callback in
    print(Thread.current)
    callback(.success(42))
  }
}
let publisher2 = Deferred {
  Future<String, Never> { callback in
    print(Thread.current)
    callback(.success("Hello world"))
  }
}

let cancellable = publisher1
  .zip(publisher2)
  .sink {
    print("sink", $0, Thread.current)
  }

_ = cancellable
```

```plaintext
<_NSMainThread: 0x10100c890>{number = 1, name = main}
<_NSMainThread: 0x10100c890>{number = 1, name = main}
sink (42, "Hello world") <_NSMainThread: 0x10100c890>{number = 1, name = main}
```
이전에는 많은 코드로 두 작업을 결합했던 작업을 Combine을 사용하여 간단하게 수행할 수 있다.  

**✔️ flatMap**  
만약 첫 번째 publisher가 완료된 후 다음 publisher를 실행하도록 하고싶다면 **flatMap**을 사용할 수 있다.  
```swift
let cancellable = publisher1
  .flatMap { integer in
    Deferred {
      Future<String, Never> { callback in
        print(Thread.current)
        callback(.success("\\(integer)"))
      }
    }
  }
  .zip(publisher2)
  .sink {
    print("sink", $0, Thread.current)
  }

```

```plaintext
<_NSMainThread: 0x10100c890>{number = 1, name = main}
<_NSMainThread: 0x10100c890>{number = 1, name = main}
<_NSMainThread: 0x10100c890>{number = 1, name = main}
sink ("42", "Hello world") <_NSMainThread: 0x10100c890>{number = 1, name = main}
```

**✔️ 비동기 작업**  
publisher에서 비동기 작업을 수행하려면 기존 동시성 도구를 활용할 수 있다.
```swift
let publisher1 = Deferred {
  Future<Int, Never> { callback in
    print(Thread.current)
    callback(.success(42))
  }
}
.subscribe(on: DispatchQueue(label: "publisher1"))

let publisher2 = Deferred {
  Future<String, Never> { callback in
    print(Thread.current)
    callback(.success("Hello"))
  }
}
.subscribe(on: DispatchQueue(label: "publisher2"))

let cancellable = publisher1
  .flatMap { integer in
    Deferred {
      Future<String, Never> { callback in
        print(Thread.current)
        callback(.success("\\(integer)"))
      }
    }
    .subscribe(on: DispatchQueue(label: "publisher3"))
  }
  .zip(publisher2)
  .sink {
    print("sink", $0, Thread.current)
  }

Thread.sleep(forTimeInterval: 1)
_ = cancellable
```

```plaintext
<NSThread: 0x100711ba0>{number = 2, name = (null)}
<NSThread: 0x100711ba0>{number = 3, name = (null)}
<NSThread: 0x106004830>{number = 4, name = (null)}
sink ("42", "Hello") <NSThread: 0x1060052f0>{number = 4, name = (null)}
```
각 publisher가 DispatchQueue에서 작업하도록 설정하였다.  
출력 결과를 보면 이제 메인 스레드가 아닌 다른 스레드에서 동작하고 있음을 볼 수 있다.  

**✔️ 복잡한 의존성 구조**  
이전에 OperationQueue와 DispatchQueue에서 다음과 같은 의존성을 만들었었다.

```plaintext
 A ➡️ B
⬇️    ⬇️
 C ➡️ D
```
이전에는 복잡한 코드를 사용하여야 다음과 같은 구조를 만들 수 있었지만 Combine에서는 간단하게 표현할 수 있다.  

```swift
a
 .flatMap { a in zip(b(a), c(a)) }
 .flatMap { b, c in d(b, c) }
```
a 퍼블리셔를 시작하여 해당 결과 값을 통해 b와 c 작업을 수행하고, 두 작업의 결과를 활용하여 d 작업을 수행한다.  
이러한 스트림을 통해 관계를 간단하게 표현할 수 있지만 이러한 체이닝 방식이 익숙하지 않다면 어려운 방식이다.  

**Swift의 새로운 동시성 도구**인 **async/await**을 사용한다면 더 간단하게 표현할 수 있다.  
```swift
let a = await f()
async let b = g(a)
async let c = h(a)
let d = await i(b, c)
```

----

🔥 다음 내용  
[3. 동시성 프로그래밍: Task](https://ji-yeon224.github.io/posts/Task_Cooperation/)  
[4. 동시성프로그래밍: Sendable, Actor](https://ji-yeon224.github.io/posts/Sendable_Actor/)  
[5. 동시성프로그래밍: 구조적 프로그래밍과 MainActor](https://ji-yeon224.github.io/posts/StructuredProgramming/)  