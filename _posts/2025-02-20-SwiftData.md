---
title: "[iOS/WWDC23] SwiftData에 대해 알아보자"
date: 2025-02-20
categories:
  - iOS
  - SwiftData
tags:
  - WWDC23
  - SwiftData
comments: "true"
---

SwiftData는 데이터모델링과 관리 프레임워크이다.   
다른 포멧은 제외하고 코드에만 집중하며 Swift 매크로를 통해 매끄러운 API를 만든다.  

## Schema 구성요소

### @Model

- 스키마 정의하는 매크로
- 값 프로퍼티가 속성으로 사용되도록 적응시킨다.
- 기본 타입과 Struct, Enum Codable등 가능
- `@Attribute`를 통해 제약조건 추가 가능
- `@Relationship`으로 삭제 규칙 지정 가능
- `@Transient`로 특정 프로퍼티를 포함하지 않도록 할 수 있음

```swift
import SwiftData

@Model
class Trip {
  var name: String
  var destination: String
  var startDate: Date
  var bucketList: [BucketListItem]? = []
  var livingAccommodation: LivingAccommodation?
}
```

### @Attribute

**✔️ .unique**  
기본 키를 지정하기 위해서는` Attribute`의 `.unique` 옵션을 사용할 수 있다.  

**✔️ originalName**  
```swift
@Model
class Trip {
  @Attribute(.unique) var name: String
  var destination: String
  var start_Date: Date
  var end_Date: Date
}
```
위의 스키마에서 `start_Date` , `end_Date` 프로퍼티 이름의 밑줄을 없애고 싶어 변경하게 되면 이미 생성된 스키마에서 새로운 프로퍼티를 생성하게 된다.  

이 프로퍼티를 유지하면서 이름을 변경하고싶다면 기존 이름과 프로퍼티 이름을 **매핑**해줄 수 있다.  
```swift
@Model
class Trip {
  @Attribute(.unique) var name: String
  var destination: String
  @Attribute(originalName: "start_Date") var startDate: Date
  @Attribute(originalName: "end_Date") var endDate: Date
}
```

기존 이름을 매핑하여 데이터 손실을 막을 수 있고, 앱 업데이트 시 간단한 마이그레이션을 보장하게 된다.  

### @Relationship  
스키마 간의 관계를 설정하고, `deleteRule` 을 통해 **삭제 규칙을 적용**할 수 있다.  
- .cascade
    - 부모 객체가 삭제되면 관련된 자식 객체도 삭제된다.
- .nullify
    - 부모 객체가 삭제되면, 관계만 해제하고 자식 객체는 Nil이 된다.
- .deny
    - 자식 객체가 존재하면 부모 객체는 삭제될 수 없다.

```swift
@Relationship(deleteRule: .cascade)
var bucketList: [BucketListItem]? = []

@Relationship(deleteRule.cascade)
var livingAccommodation: LivingAccommodation?
```

### @Transient  
`@Transient` 로 지정된 프로퍼티는 데이터를 SwiftData에 저장하지 않고, **런타임 동안에만 유지**된다.  
```swift
@Transient
var tripView: Int = 0
```
반드시 기본 값이 지정되어 있어야 한다.  

## ModelContainer  

> 데이터 저장소 관리

- 스키마와 저장 데이터간의 관계를 관리  
- 데이터가 메모리 또는 디스크에 저장되도록 **저장 방식 관리 * *
- 버전 관리, 마이그레이션, 그래프 분리 같은 스토리지 운영 관리  
```swift
let container = try ModelContainer(
	for: [ Trip.self, LivingAccommodation.self ],
	configuration: ModelConfiguration(url: URL("path"))
)
```
- SwiftUI에서 `.modelContainer(for: )` modifier를 사용하여 컨테이너를 쉽게 주입할 수 있다.  
- WindowGroup에 ModelContainer를 설정하면 그 안의 모든 뷰가 같은 컨테이너를 사용할 수 있다.  

### ModelConfiguration  
- 데이터가 저장되는 위치를 제어  
    - 일시적인 데이터는 메모리에, 영구적인 데이터는 디스크에 저장  
- 특정 파일의 URL을 사용하거나, 그룹 컨테이너 권한 등 응용프로그램의 권한을 통해 URL 생성  
- **데이터를 읽기 전용 모드**로 로드하여 민감한 데이터나 템플릿 데이터의 수정을 방지  
- 두 개 이상의 CloudKit 컨테이너를 사용하는 응용프로그램은 `ModelConfiguration`의 일부로 지정할 수 있다.  

```swift
let fullSchema = Schema([
  Trip.self,
  BucketListItem.self,
  LivingAccommodations.self,
  Person.self,
  Address.self
])

let trips = ModelConfiguration(
  schema: Schema([
    Trip.self,
    BucketListItem.self,
    LivingAccommodations.self
  ]),
  url: URL(filePath: "/path/to/trip.store"),
  cloudKitDatabase: .private("com.example.trips")
)

let people = ModelConfiguration(
  schema: Schema([Person.self, Address.self]),
  url: URL(filePath: "/path/to/people.store"),
  cloudKitDatabase: .private("com.example.people")
) 

let container = try? ModelContainer(
  for: fullSchema, 
  configurations: trips, people
)
```
`fullSchema` 를 통해 프로젝트 내에서 사용하기 위한 스키마를 정의한다.  
`ModelConfiguration` 을 통해 서로 다른 저장 위치에서 스키마를 불러온다.  

최종적으로 스키마와 `configuration`을 결합하여 `ModelContainer`를 구성할 수 있다.  

위와 같이 형성한 `ModelContainer`는 `modifier`를 통해 SwiftUI 뷰에 주입할 수 있다.  
```swift
var body: some Scene {
  WindowGroup {
    ContentView()
  }
  .modelContainer(container)
}
```

## ModelContext  

> 데이터 변경을 처리하는 컨텍스트 

- **모델의 변경사항을 추적**하고, **데이터를 저장하거나 삭제하는 기능을 제공**한다.
- 컨테이너가 설정되면 `ModelContext`로 데이터를 가져와 저장할 준비를 할 수 있다.
- rollback과 reset기능을 지원하여 캐싱된 상태를 지울 수 있다.
- 자동 저장

### Context 주입
View나 Scene에서 `.modelContainer` 를 통해 주입하면, 컨테이너의 `mainContext` 에 `modelContext` 키를 바인딩 한다.  
- `MainContext`는 특수한 `MainActor` 정렬 모델 컨텍스트로, Scene이나 View에서 `ModelObject`와 함께 작동한다.  
```swift
struct ContextView: View {
	@Environment(\.modelContext) private var context
}
```

### Context 접근
`@Environment(\.modelContext)` 를 사용하여 SwiftUI View 코드에서 모델의 쿼리에 쓰인 컨텍스트에 쉽게 접근할 수 있다.  
- `@Environment(\.modelContext)` 는 기존 데이터의 변경사항을 추적할 수 있다.  
```swift
struct ContentView: View {
	@Query var trips: [Trip]
	@Environment(\.modelContext) private var context
	var body: some View {
		...
		Button {
		  modelContext.delete(trip)
		} label: {
		  Label("Delete", systemImage: "trash")
		}
	}
}
```
모델에 변경사항이 생기면 스냅샷으로 `ModelContext`에 기록된다. 계속해서 변경사항을 추적하다가 `context.save()` 를 호출하면 멈춘다.  
→ 즉, 데이터를 삭제하더라도 `context.save()` 를 하기 전 까지는 `ModelContext`에 남아있게 된다.  

### @Query  
`@State`와 비슷하게 동작하여, **모델에서 일어나는 모든 변화를 뷰에 업데이트**한다.  
`@Query` 를 사용하여 정렬, 필터와 같은 간단한 구문을 처리한 후의 데이터를 받아올 수 있다.  
```swift
@Query(sort: \.created) private var cards: [Card]
```

### Undo/Redo  
`.modelContainer()` 는 `isUndoEnable` 이라는 매개변수가 있다.  
```swift
.modelContainer(for: Trip.self, isUndoEnabled: true)
```
`true` 값을 주면 `MainContext`에 `undoManager`를 바인딩하여, 추가 코드 없이 변경사항을 취소 또는 복귀를 수행할 수 있다.  

### 자동 저장  
자동저장 기능이 활성화되면 `ModelContext`는 **시스템 이벤트(백그라운드 전환 또는 특정 주기) 발생 시 저장**된다. 자동저장은 기본으로 활성화되어있어, 불필요 시 비 활성화 할 수 있다.  
```swift
.modelContainer(for: Trip.self, isAutosaveEnabled: false)
```

## Model at scale  
백그라운드에서 데이터를 다루거나 원격 서버, 다른 영구 메커니즘과의 동기화, 배치 프로세스는 모두 모델 객체를 필요로한다.  
이러한 작업에서 필요한 객체 집합을 가져올 때 `ModelContext`의 `.fetch()` 메서드를 통해 가져온다.  

### FetchDescriptor  
`FetchDescriptor` 를 사용하여 복잡한 쿼리를 만들 수 있다.  
`#Predicate` 매크로를 스키마와 결합하여 컴파일러가 검증한 쿼리를 사용할 수 있다.  
파라미터로 offset, limt, faulting, perfetching 옵션을 사용할 수 있다.  

**✔️ \#Predicate**  
특정 조건을 기반으로 데이터를 필터링 할 때 사용할 수 있다.  
```swift
var predicate = #Predicate<Trip> { trip in
	trip.livingAccommodations.filter {
		hotelNames.contains($0.placeName)
	}.count > 0
}
var descriptor = FetchDescriptor(predicate: predicate)
var trips = try context.fetch(descriptor)
```

**✔️ SortDescriptor**  
`FetchDescriptor` 와 함께 사용하여 데이터를 가져올 때 정렬을 수행한 후 값을 가져오도록 할 수 있다.  
```swift
let descriptor = FetchDescriptor<Trip>(
  sortBy: [SortDescriptor(\Trip.name)],
  perdicate: tripPredicate
)
let trips = try context.fetch(descriptor)
```

### Tuning Option
`ModelContext`의 `enumerate`함수에서 옵션들이 결합된다.  
```swift
context.enumerate(FetchDescriptor<Trip>()) { trip in

}
```

**✔️ batchSize 조절**  
`enumerate`함수를 통해 배치사이즈 크기를 조절할 수 있다.  
```swift
context.enumerate(
  descriptor,
  batchSize: 1000) {
}
```
배치 크기를 줄이면 메모리를 적게 사용하게 되지만 I/O가 증가한다.  

**✔️ mutation guard**  
데이터를 순회하는 동안 데이터가 변경됨을 감지하는 mutation guard를 내부적으로 포함하고 있다.
```swift
context.enumerate(descriptor) { trip in
    trip.name = "새로운 이름" // ❌ 예외 발생 가능!
}
```
기본적으로 `ModelContext`가 변형되는 것을 감지하여 예외를 발생시켜 데이터의 일관성으로 보장하려는 목적을 가지고 있다.  

하지만, **의도적으로 일부 데이터를 변경**하려고 할 수 있다. 이러한 경우 `allowEscapingMutations` 매개변수를 `true`로 설정하여 **데이터의 변경을 허용**할 수 있다.  
```swift
context.enumerate(
    descriptor,
    batchSize: 500,
    allowEscapingMutations: true
) { trip in
    // Remind me to make reservations for trip
}
```

## 참고
[WWDC23_SwiftData 만나보기](https://developer.apple.com/videos/play/wwdc2023/10187)
[WWDC23_SwiftData로 스키마 모델링하기](https://developer.apple.com/videos/play/wwdc2023/10195)
[WWDC23_SwiftData 자세히 살펴보기](https://developer.apple.com/videos/play/wwdc2023/10196)
