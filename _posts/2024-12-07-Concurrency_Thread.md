---
title: "[iOS] 동시성의 과거: 스레드"
date: YYYY-MM-DD HH:MM:SS +09:00
categories:
  - PointFree스터디
  - Concurrency
tags:
  - Thread
  - Concurrency
comments: "true"
---
*PointFree의 Concurrency 세션 정리 내용입니다!*

# [🔗](https://www.pointfree.co/episodes/ep190-concurrency-s-past-threads) Concurrency's Past: Threads

## Thread Basics

Operation Queue나 GCD 등 스레드를 고려하지 않고 비동기 코드를 작성하도록 발전해왔다.  
동시성 코드를 잘 이해하기 위해서는 동시성 코드의 시작인 Thread에 대해 공부해야 한다. 운영체제는 수십, 수백개의 논리적 스레드를 관리한다.

Thread 는 클래스타입으로 정의되어있다. POSIX 스레드를 기반으로 만들어져서 사용자가 해당 내용에 대해 알지 못해도 스레드를 쉽게 다룰 수 있도록 애플에서 Thread 클래스를 제공한다.

```swift
Thread.detachNewThread{
   Thread.sleep(forTimeInterval: 1)
   print(Thread.current)
}
```

위의 내용을 실행시키면 출력되지 않고 프로그램이 종료된다.  
`detachNewThread` 를 통해 메인 스레드가 아닌 새로운 스레드를 생성하게 되었고, 그 내부에서 동작하기 때문에 `sleep` 을 기다리지 않고 프로그램이 종료되는 것.

그렇기 때문에
```swift
Thread.detachNewThread{
   Thread.sleep(forTimeInterval: 1)
   print(Thread.current) // 1
}
Thread.sleep(forTimeIntervar: 1.1)
print(Thread.current) // 2
```

위의 코드를 실행시킨 후 현재 실행되고있는 범위의 스레드 정보를 출력시키면 서로 다른 정보를 출력하게 된다.

```
<NSThread: 0x106404d00>{number = 2, name = (null)} // 1
<_NSMainThread: 0x10600aa90>{number = 1, name = main} // 2
```
  
**스레드의 순서는 보장되지 않는다.** 
```swift
Thread.detachNewThread{
   print("1", Thread.current)
}
Thread.detachNewThread{
   print("2", Thread.current)
}
Thread.detachNewThread{
   print("3", Thread.current)
}
```

여러 개의 스레드를 생성할 때, 스레드의 생성 순서대로 실행되지 않을 수 있다.

운영체제는 스레드에 실행시간을 할당하는 복잡한 로직을 처리하기 때문에 실행 순서를 예측할 수 없다.

`detachNewThread`를 통해 다른 스레드를 방해하지 않고 작업을 수행할 수 있다. 이론적으로는 컴퓨터에 유한한 수의 코어가 존재한다. 하지만 운영체제 내부적으로 복잡한 작업을 처리하여 여러 스레드가 각자 작업을 수행하고 다른 스레드로 넘어가도록 하여 여러 작업이 교대로 이루어질 수 있도록 한다.

## Priority and Cancellation

### **우선순위**

Thread의 초기화 함수로 스레드를 생성하여 스레드의 우선순위를 설정할 수 있다.

```swift
let thread = Thread {
  Thread.sleep(forTimeInterval: 1)
  print(Thread.current)
}
thread.start()
thread.threadPriority = 0.75
```

우선순위는 0에서 1 사이의 double 값으로, 운영체제가 해당 스레드를 더 빨리 실행하도록 권장할 수 있지만 보장 되진 않는다.

### **취소**
```swift
thread.cancel()
```

`cancel()` 을 호출하여 **실행 중인 스레드를 중지 요청**을 보낼 수 있다.
`cancel()`로 스레드의 실행을 멈출 수 있는 것이 아니라 스레드 종료 요청을 보내 내부의 플래그를 변경하도록 한다.

취소를 통해 더 우선순위가 높은 작업을 수행할 때 현재 작업을 취소하고 다음 작업을 수행하도록 할 수 있다.

```swift
let thread = Thread {
  Thread.sleep(forTimeInterval: 1)
  guard !Thread.current.isCancelled else {
    print("Cancelled")
    return
  }
  print(Thread.current)
}
```

`Thread.current.isCancelled` 플래그를 사용하여 스레드 내부에 플래그를 확인하는 코드를 넣어주는 것이 좋다.
플래그를 자주 확인하여 불필요한 작업을 빠르게 종료할 수 있도록 하는 것이 좋다.
다만, 취소 매커니즘이 다른 스레드 작업들과 완전히 결합되어있다.
`Thread.sleep()` 이 동작하고 있을 동안 취소 플래그를 확인하지 않는다.
```swift
let thread = Thread {
  Thread.sleep(forTimeInterval: 1)
  print(Thread.current)
}
thread.start()
Thread.sleep(forTimeInterval: 0.01)
thread.cancel()
```

`Thread.sleep` 에서 취소된 스레드를 인지하지 못하고 취소가 되더라도 지정된 시간이 지날 때 까지 기다리게 된다.
이러한 이유는 아마도 CPU에서 오버헤드를 피하기 위한 의도로 보여진다.
우선순위와 취소는 스레드에서 **Cooperative(협조)** 의 특성을 보여주는 기능이다. 스레드에서 낮은 우선순위를 설정하여 다른 스레드들이 CPU에서 더 많은 작업을 할 수 있도록 하고, 스레드 취소를 감지하게 되면 스레드의 작업을 중단하여 불필요한 작업을 빠르게 끝낼 수 있도록 할 수 있다.

## Problems: Coordination

**✔️ 딕셔너리 사용의 안정성**  
Thread Dictionary는 말 그대로 딕셔너리 타입이기 때문에 안정성 부분에서 떨어지게 된다. 키 이름을 잘못 입력했을 경우 컴파일러가 해당 문제를 잡을 수 없기 때문에 **런타임 시 문제**가 발생할 가능성이 있다.


**✔️ 스레드 딕셔너리 전달 문제**  
스레드 딕셔너리를 통해서 데이터를 쉽게 전달할 수 있지만 문제는 **단일 스레드에 한해서만 동작**한다는 것이다. 기존 스레드 내부에서 새로 스레드를 생성하더라도 기존 딕셔너리 데이터를 전달할 수 없다.

예시) 병렬 처리
```swift
func makeDatabaseQuery() {
  let requestId = Thread.current.threadDictionary["requestId"] as! UUID
  print(requestId, "Making database query")
  Thread.sleep(forTimeInterval: 0.5)
  print(requestId, "Finished database query")
}

func makeNetworkRequest() {
  let requestId = Thread.current.threadDictionary["requestId"] as! UUID
  print(requestId, "Making network request")
  Thread.sleep(forTimeInterval: 0.5)
  print(requestId, "Finished network request")
}

func response(for request: URLRequest) -> HTTPURLResponse {
  let start = Date()
  let requestId = Thread.current.threadDictionary["requestId"] as! UUID

  let databaseQueryThread = Thread { makeDatabaseQuery() }
  databaseQueryThread.start()

  let networkRequestThread = Thread { makeNetworkRequest() }
  networkRequestThread.start()

  // TODO: join threads somehow

  print(requestId, "Completed in", Date().timeIntervalSince(start))
  return .init()
}

// 새 스레드에서 네트워크 요청
let thread = Thread {
  response(for: .init(url: .init(string: "<https://www.pointfree.co>")!))
}
thread.threadDictionary["requestId"] = UUID()
thread.start()
```

`makeDatabaseQuery()` 와 `makeNetworkRequest()` 가 내부에서 생성된 스레드를 통해서 병렬적으로 실행된다.  
위의 코드는 오류가 발생하는데 그 이유는 외부에서 생성된 스레드 딕셔너리의 데이터가 내부에서 생성된 스레드에 전달이 되지 않아 requestId값을 찾는데에 오류가 발생한다.  
→ 내부에서 스레드를 생성하였지만 **스레드는 자식 스레드의 개념이 없기 때문에** 별도의 스레드가 생성된다.

위의 코드가 원하는대로 동작하려면 **새로 생성한 딕셔너리에 데이터를 복사**해야 한다.
```swift
let databaseQueryThread = Thread { makeDatabaseQuery() }
databaseQueryThread.threadDictionary.addEntries(
  from: Thread.current.threadDictionary as! [AnyHashable: Any]
)
databaseQueryThread.start()
```


**✔️ 스레드 조인**  
위의 예시 코드에서 `databaseQueryThread()` 와 `networkRequestThread()` 는 서로 다른 비동기 작업을 수행하게 된다. 그렇기 때문에 두 작업이 끝나기 전에 함수가 종료된다. 두 작업의 완료를 기다리는 매커니즘이 스레드에는 존재하지 않는다.
```swift
while !databaseQueryThread.isFinished || !networkRequestThread.isFinished {
  Thread.sleep(forTimeInterval: 0.1)
}
```

해당 코드처럼 임시로 while루프를 통해 스레드 종료 여부를 판단하는 방법 뿐이다.  
이 방식은 **비효율적**인 방식이다.


**✔️ 스레드 취소 전달**  
위의 스레드 딕셔너리 문제와 유사하게 스레드 취소 작업 또한 내부 스레드에 전달되지 않는다.
```swift
let thread = Thread {
    print("Outer thread started")
    Thread.sleep(forTimeInterval: 1) // 1초 동안 대기
    print("Outer thread is cancelled:", Thread.current.isCancelled)
    
    // 외부 스레드에서 새 스레드를 만듦
    let innerThread = Thread {
        print("Inner thread started")
        print("Inner thread is cancelled:", Thread.current.isCancelled)
        Thread.sleep(forTimeInterval: 2)
        print("Inner thread finished")
    }
    innerThread.start() // 내부 스레드 실행
    
    guard !Thread.current.isCancelled else {
        print("Outer thread was cancelled")
        return
    }
    print("Outer thread finished")
}

thread.start()
Thread.sleep(forTimeInterval: 0.5) // 0.5초 대기 후
thread.cancel() // 외부 스레드를 취소함


```

```plaintext
Outer thread started
Inner thread started
Outer thread is cancelled: true
Inner threa is cancelled: false
```

위의 코드에서 스레드를 실행하게 된다면 외부 스레드(Outer thread)는 `thread.cancel()` 로 인해 종료되지만 내부에서 생성된 스레드(Inner thread)는 종료되지 않는다.  
즉, 외부 스레드를 취소하더라도 **취소 상태가 내부에 전달되지 않아** 내부 스레드는 계속 작업을 진행하게 된다.

위와 같은 문제들은 비동기 작업을 효율적으로 관리하는 데에 불편함을 야기한다.

## Problems: Expensiveness

만약 우리가 처리해야 할 내용이 많을 경우 스레드의 비용 측면에서 문제가 발생할 수 있다.

**✔️ 스레드 생성 비용과 경쟁**  
스레드를 생성하는데에 많은 비용이 든다. 스레드를 생성하게 되면 운영체제가 작업을 빨리 처리할 수 있도록 스케줄링하지만, 그러기 위해 다른 스레드와 경쟁을 하게 된다.  
만약 1000개의 스레드를 만들게 되면 많은 스레드가 서로 경쟁해서 운영체제가 시간을 적절히 할당하기 어려워진다.  
그렇기 때문에 작업이 느려지고 시스템이 비효율적으로 작동하게 된다.


**✔️ 스레드 호출 스택의 메모리 사용 문제**  
각 스레드는 고유한 호출 스택(작업을 처리하기 위해 필요한 메모리 공간)을 가지고 있으며, 이 스택의 크기는 0.5MB이다. 그렇기 때문에 1000개의 스레드를 생성하면 호출 스택만으로 500MB의 메모리가 필요하게 된다.


**✔️ 스레드 경쟁으로 인한 성능 문제**  
많은 스레드가 생성되어 각 스레드가 CPU를 사용하는 시간이 줄어들게 된다면 각 스레드가 작업을 처리할 수 있는 시간이 줄어들게 된다.

예를 들어, 1,000개의 스레드를 생성한 채로 다른 스레드에서 50,000번째 소수 구하기 작업을 수행한다고 해보자. 다른 경쟁 스레드가 없는 상황이라면 50,000번째 소수를 계산하는데는 0.025초가 걸리지만 경쟁 스레드가 1,000개가 있는 경우 2.93초로 100배 이상 처리 속도가 느려진다.


**✔️ Non-blocking 작업**  
> **non-blocking** 작업은 비동기 작업을 수행할 때 **CPU가 실제로 작업을 수행하고 있지 않은 대기 시간에 다른 작업을 처리할 수 있도록 처리하는 방식**을 의미한다. 예를 들어, 어떠한 작업을 지연시키거나, 네트워크 요청 시 서버로부터 데이터를 받을 때 까지 기다리는 경우 CPU는 처리할 작업이 많지 않기 때문에 non-blocking이 필요하게 된다.

이러한 경우 다른 스레드가 작업을 처리할 수 있도록 잠시 스레드를 포기했다가 준비가 되었을 때 다시 작업을 처리할 수 있어야 하지만 스레드에서는 이러한 처리를 하기 어렵다. 스레드는 CPU와 밀접하게 연결되어있고, CPU 코어에서는 연속적으로 작업을 처리하도록 설계되어 있기 때문에 잠시 멈추거나 다른 작업을 위해 대기하는 것이 어렵다.  
스레드에서 대기를 하기 위해 `Thread.sleep` 을 사용해볼 수 있는데 해당 방식은 매우 **비효율적**이다. 스레드 리소스를 사용하고 있지만, 실제로는 작업을 하고있지 않기 때문이다.  

이 보다 더 효율적인 대기 방식은 스레드를 멈추는 대신 운영체제에 일정 시간이 지나면 알려달라고 요청하는 것이다. 하지만 스레드 만으로는 이러한 작업을 수행할 수 없고, 스케쥴 된 작업을 확인하는 도구가 추가적으로 필요할 것이다.


**✔️ Thread pool**  
**Thread Starvation**이나 **Thread Explosion**를 해결하기 위해 **스레드 풀**을 만드는 방식을 주로 사용하게 된다.  

> **Thread Starvation**(스레드 기아 문제): 한정된 수의 스레드가 많은 작업을 처리하려 할 때, 일부 스레드가 오래 기다려야 하는 경우. 즉, 작업이 완료되지 않은 채로 계속 대기하는 상황
> **Thread Explosion**(스레드 과잉 생성 문제): 너무 많은 스레드가 생성되어 CPU 사용 시간 할당이 어려워 성능이 저하되고, 비효율적으로 작동하는 문제.

스레드 풀은 일정 수의 스레드를 미리 생성하고 관리하는 방식이다. 스레드가 필요할 때 새로 생성하는 대신 스레드 풀에 준비되어 있는 스레드를 요청하여 작업을 처리하게 된다. 스레드가 모두 사용 중일 경우에는 하나가 완료될 때 까지 기다리게 된다.  

스레드 풀은 몇가지 단점을 가지고 있다.  
1. 차단 작업에 도움 되지 않는다.
    여전히 타이머나 네트워크 요청 등의 대기 상태의 경우 CPU에서 다른 작업을 처리할 수 없는 상태가 된다.  
2. 국소적 해결책
    스레드 풀은 스레드를 과도하게 생성되지 않도록 할 수 있지만 직접 관리하는 코드 내에서만 가능하다. 외부 라이브러리나 프레임워크에서 자체적으로 스레드 풀을 사용하고 있을 수 있어 독립적인 스레드 풀이 여러 개 존재할 수 있다. 이는 결국 스레드 과잉 생성 문제를 다시 일으킬 수 있다.  
3. 협력적 동시성(cooperative concurrency)의 부족
    협력적 동시성은 여러 스레드가 자원을 공유하며 효율적으로 사용하는 방식을 의미하는데 스레드는 이러한 기능을 기본적으로 제공하지 않는다. 사용하기 위해서는 직접 만들어야 하며 글로벌한 협력적 솔루션을 제공하기 어렵다.  
  

## Problems: Data Races

스레드를 생성하여 작업을 하게되면 **Race Condition**이라는 문제를 마주하게 된다.
> Race Condition: 여러 스레드가 동일한 데이터를 동시에 읽거나 수정할 때 발생하는 문제
  
애플 프레임워크는 이러한 문제를 동기화하여 방지할 수 있도록 제공하는 도구가 있지만 스레드와 밀접하게 통합되어 있지 않아 개발자가 올바르게 사용해야 한다.  

**✔️ 데이터레이스 발생 원인**  
다음과 같은 클래스에서 가변 상태의 값을 여러 스레드에서 읽거나 수정하려고 한다.
```swift
class Counter {
  var count = 0
}
let counter = Counter()
```

그 후 1000개의 스레드를 생성하여 값을 증가시키게 되면
```swift
for _ in 0..<1_000 {
  Thread.detachNewThread {
    Thread.sleep(forTimeInterval: 0.01)
    counter.count += 1
  }
}
```
출력 값은 1000보다 작은 값이 실행할 때 마다 다르게 나올 것이다.  
이러한 결과가 나오는 이유는 값을 변경하는 작업이 **단일 원자적 연산이 아니기 때문**이다.  

`counter.count += 1` 은  
- 현재 count값 가져오기
- 값 증가시키기
- 값을 count에 할당하기
이렇게 세 단계로 동작하고 있다.  

우리가 원하는 동작 방식은 아래와 같다.  
```swift
스레드 1: var count1 = counter.count  // 0
스레드 1: count1 += 1                 // 1
스레드 1: counter.count = count1      // 1
스레드 2: var count2 = counter.count  // 1
스레드 2: count2 += 1                 // 2
스레드 2: counter.count = count2      // 2
```

하지만 실제로는 순서가 아래와 같이 섞여 실행될 가능성이 있다.
```swift
스레드 1: var count1 = counter.count  // 0
스레드 2: var count2 = counter.count  // 0
스레드 1: count1 += 1                 // 1
스레드 2: count2 += 1                 // 1
스레드 1: counter.count = count1      // 1
스레드 2: counter.count = count2      // 1
```

스레드 2는 count의 값을 가져올 때 1이 아닌 0의 값을 가져오게 된다.  
이러한 data race 문제를 해결하기 위해서는 접근을 **동기화**해야 한다.  

값을 변경하기 시작할 때 다른 스레드는 접근을 하지 못하고 수정이 완료된 후에 접근할 수 있도록 **락(lock)** 을 사용해야 한다.  


**✔️ 락(lock)**  
lock은 코드의 일부가 실행되는 동안 다른 스레드들이 해당 부분을 실행하지 못하도록 막는 기능을 한다.

`NSLock()` 을 이용하여 변경 작업 전 후에 락을 걸고 푸는 작업을 추가하는 메서드를 구현하여 해결할 수 있다.
```swift
class Counter {
  let lock = NSLock()
  private(set) var count = 0
  
  func increment() {
    self.lock.lock()
    defer { self.lock.unlock() }
    self.count += 1
  }
}

for _ in 0..<1_000 {
  Thread.detachNewThread {
    Thread.sleep(forTimeInterval: 0.01)
    counter.increment() // 증가 메서드 호출
  }
}
```

이렇게 실행하면 항상 일관된 1,000이라는 값을 얻을 수 있다.

위의 코드는 값을 변경하기 위해 락을 걸고 푸는 메서드를 따로 만들어야 하기 때문에 클로저를 통해 modify 메서드를 만들어 볼 수도 있다.

```swift
func modify(work: (Counter) -> Void) {
  self.lock.lock()
  defer { self.lock.unlock() }
  work(self)
}
counter.modify {
  $0.count += 1
}
```

위와 같은 코드도 좋지만 다음과 같이 getter, setter를 사용하는 것이 변경하려는 데이터를 더 명시적으로 표기할 수 있을 것이다.
```swift
private var _count = 0
var count: Int {
  get {
    self.lock.lock()
    defer { self.lock.unlock() }
    return self._count
  }
  set {
    self.lock.lock()
    defer { self.lock.unlock() }
    self._count = newValue
  }
}

for _ in 0..<1_000 {
  Thread.detachNewThread {
    Thread.sleep(forTimeInterval: 0.01)
	counter.count += 1
  }
}
// count = 539
```

하지만 위의 코드를 실행하면 또 다시 1,000이하의 값이 나오게 된다.  
위의 코드는 값을 읽고 쓰는 get과 set 부분에만 락이 걸리게 되며 전체 트랜잭션에는 락이 걸리지 않기 때문에 발생하는 문제이다.


**✔️ read-modify**  
*(비공식 지원 기능?)*  
read-modify는 동시성 관련 작업을 처리하는 기능으로, 속성의 읽기 및 수정 작업을 안전하게 처리할 수 있는 기능이다.

위의 코드를 read-modify를 사용하여 다음과 같이 수정해 볼 수 있다.
```swift
var count: Int {
  _read {
    self.lock.lock()
    defer { self.lock.unlock() }
    yield self._count
  }
  _modify {
    self.lock.lock()
    defer { self.lock.unlock() }
    yield &self._count
  }
}

counter.count += 1
```

해당 코드는 정확히 1,000의 값을 가지게 된다.  
`counter.count += 1` 과 같은 연산은 단일 트랜잭션으로 처리하게 된다.

하지만, read-modify에서도 문제가 발생할 가능성이 있다.  
```swift
counter.count += 1 + counter.count / 100
```
counter.count를 여러 번 참조하는 경우 문제가 발생하게 된다.  
읽기와 쓰기에 별도로 락이 걸리게 되면서 읽기와 쓰기 사이에 다른 스레드가 개입할 수 있게 된다. 그렇게 되면 해당 연산을 단일 트랜잭션으로 묶지 못하게 된다.  

위의 문제를 해결하기 위해선 결국 위에서 사용하였던 modify 메서드를 사용하여 연산 자체를 단일 트랜잭션을 묶어야 한다. 동기화 문제를 완전히 해결하기 위해서는 결국 상태를 변경하는 전용 메서드를 활용해야 한다.

이러한 문제점들은 멀티스레딩과 데이터레이스 문제의 까다로움을 보여준다.

락은 스레드와 분리되어 있는 도구라는 문제점이 있다. Swift의 새로운 동시성 도구는 스레드와 밀접하게 통합되어 있어서 더 쉽게 문제를 해결할 수 있다.

## **Thread의 한계**

과거 애플 플랫폼에서 스레드는 동시성과 비동기 작업을 처리하기 위한 주요 도구였다. 하지만 스레드는 다음과 같은 한계를 가지고 있다.

- 스레드는 자식스레드를 지원하지 않기 때문에 우선순위, 취소, 딕셔너리 등의 데이터가 하위 스레드에 전달되지 않는다.
- 스레드의 수가 과도하게 증가될 수 있다.
- 코드의 가독성이 떨어진다.
- 스레드 간 동기화를 위한 툴이 제한적이다.

