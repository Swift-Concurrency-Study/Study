# 목차

- [Actor 등장 배경](#actor-등장-배경)
- [Actor 란?](#actor-란)
- [Actor의 특징(ft. isolated)](#actor의-특징-ft-isolated)
  - [Actor isolation](#actor-isolation)
    - [Task와 isolation](#task와-isolation)
    - [Protocol 준수와 isolation](#protocol-준수와-isolation)
    - [Closures와 isolation](#closures와-isolation)
    - [Data와 isolation](#data와-isolation)
  - [Sendable](#sendable)
    - [Sendable 조건](#sendable-조건)
    - [Adopting Sendable](#adopting-sendable)
    - [@Sendable functions](#sendable-functions)
  - [Actor Reentrancy](#actor-reentrancy)
  - [Actor Hopping](#actor-hopping)
- [Global Actor](#global-actor)
  - [함수에서의 자동 변환](#함수에서의-자동-변환)
  - [Protocol 적용시 주의사항](#protocol-적용시-주의사항)
- [Main Actor](#main-actor)
  - [Main DispatchQueue](#main-dispatchqueue)
  - [Main Actor 내부 Task 생성](#main-actor-내부-task-생성)
  - [Main Actor Hopping](#main-actor-hopping)
- [QnA](#qna)
- [참고자료](#참고자료)

# Actor 등장 배경

Data race는 동시성 프로그래밍을 정말 어렵게 만드는 오류 중 하나이다.

Data race는 두 개의 서로 다른 스레드가 동시에 mutable한 데이터에 접근하고 그 중 하나가 데이터를 수정하는 과정에서 발생한다.

Data race는 발생하기는 굉장히 쉽지만 디버깅하기는 굉장히 까다로운데, 이는 Data race를 유발하는 데이터 접근이 프로그램의 다른 부분에서 이루어질 확률이 높고 이에 따른 비지역적 추론이 필요하기 때문이다..

이러한 Data race를 피할 수 있는 방법은 값 타입을 사용하여 공유 가능한 mutable state를 제거하는 것이다. 특히나 struct와 같은 값 타입에서 let 프로퍼티를 사용하면 완전히 immutable하기 때문에 Data race가 발생할 일이 전혀 없어진다.

값 타입을 사용하면 프로그램의 추론이 더 쉬워지고 동시성 프로그래밍을 하면서도 더욱 안전하게 사용할 수 있기 때문에 Swift에서는 값 타입의 사용을 권장하고 있다.

하지만 저대로 우리가 원하는 바를 다 ~ 이룰 수 있다면 Actor가 나오지 않았겠죠?

결국 우리는 Data Race가 일어나지 않으면서 공유 가능한 mutable state를 필요로한다.

기존의 Swift엔 저런 것.. 존재하지 않았다..

<img src="https://github.com/user-attachments/assets/0d0e1193-9ee1-48d9-9b2f-ea71d7f9e5a1"> ~~만들어 줘.~~

그래서 Apple에서 만들어준게 바로 **Actor**이다.

# Actor 란?

actor는 Swift의 새로운 타입으로 struct, enum, class와 사용법이 동일하다

```swift
actor Counter {
    var value: Int = 0
    
    func increment() -> Int {
        value += 1
        return value
    }
}
```

actor의 특징은 아래와 같다.

- class와 같은 **reference type**
    
    → actor의 목적이 shared mutable state의 표현이기 때문..
    
- class와는 다르게 **상속 불가능**
- Property, Method, Initializer, subscripts등 모두 가질 수 있고 protocol을 채택할 수도, extension을 써서 확장을 할 수도 있음

추가로 Actor에서 가장 주요하게 class와 구별되는 특성은 바로 **인스턴스 데이터를 나머지 프로그램으로부터 분리**하고 해당 **데이터에 대한 동기화된 접근을 보장**한다는 것이다.

이에 관해서는 아래서 자세히 다뤄보자.

# Actor의 특징 (ft. isolated..)

<p align="center">
<img width=600 src="https://github.com/user-attachments/assets/c9515a95-ed89-4da5-9076-fce07570a684">
</p>

**Actor**는 **공유 가능한 mutable state의 동기화**를 위한 **동기화 메커니즘**을 제공한다.

Actor의 동기화 메커니즘은 해당 actor의 상태로 동시에 다른 두 개의 코드가 접근하지 않는다는 것을 보장한다. 이러한 특성은 `Locks`나 `Serial dispatch queue`와 같은 **상호 배제 속성(mutual exclusion property)** 을 언어 수준에서 제공한다.

또한 **컴파일러 수준**에서 **데이터 격리**를 제공한다. 이를 통해 가변 상태의 데이터를 보호할 수 있게 된다.

Actor의 기본 원리는 저장된 인스턴스 속성에 `self`만 접근을 허용하는 것이다.

Actor는 자신만의 상태를 가지고 있으며 그 상태는 나머지 프로그램으로부터 독립적인 상태이다. 그리고 이러한 Actor의 상태는 Actor 자기 자신을 통해서만 접근할 수 있다.

예제를 통해 살펴보자.

```swift
actor BankAccount { 
		let accountNumber: Int
		var balance: Double
}

extension BankAccount {
  enum BankError: Error {
    case insufficientFunds
  }
  
  func transfer(amount: Double, to other: BankAccount) throws {
    if amount > balance {
      throw BankError.insufficientFunds
    }

    print("Transferring \(amount) from \(accountNumber) to \(other.accountNumber)")

    balance = balance - amount
    other.balance = other.balance + amount  // error: actor-isolated property 'balance' can only be referenced on 'self'
  }
}
```

위 코드에서 중요한 부분은 actor isolated가 같은 타입의 서로 다른 인스턴스 사이에도 적용된다는 점이다.

위 메서드에서 `balance = balance - amount`은 `self` 이기 때문에 내부에서 격리된 상태에 접근하는 것으로 취급된다. 따라서 별다른 문제가 되지 않는다.

그러나 `other.balance = other.balance + amount` 부분은 `other` 라는 매개변수에 접근하는 것이기 때문에 `self`의 ‘**외부**’에 해당한다.

따라서 저 부분에서는 컴파일 에러가 발생한다.

근데 `other.accountNumber`를 사용하는 print문에서는 에러가 발생하지 않는다. 이유가 무엇일까??

이는 밑에서 설명할 `Sendable`과 관련이 있는데, `accountNumber` 프로퍼티는 불변 프로퍼티이다. 따라서 정의상 Data Race가 일어나지 않기 때문에 다른 actor 객체의 프로퍼티에 직접 접근하더라도 에러가 발생하지 않는 것이다.

지금은 조금 이해가 안되더라도 쭉쭉 읽다보면.. 이해가 될 것…임..

## Actor isolation

actor isolation은 actor에서 아주아주아주 중요한 개념 중 하나이다.

<p align="center">
<img width=200 src="https://github.com/user-attachments/assets/5d053fd5-d9c7-4fe0-adcb-4ed95ba5bc50"> 별이 5개 .ᐟ.ᐟ
</p>

actor에서 선언된 대부분의 요소들(property, method 등..)은 특별히 명시하지 않는 이상 **actor에 격리**되어있다.

아 ~~ 아까부터 격리 격리 하는데 격리가 뭔데요;;

격리(isolated)되었다는 것은 쉽게말해 **외부에서 직접적으로 조작할 수 없고 `self`을 통해서만 조작할 수 있다는 것**을 말한다.

이를 **actor의 경계선 안에 있다**고 표현한다.

이러한 actor isolation은 actor 타입의 근간이 되는 동작이다. 여기서부턴 actor isolation이 어떻게 Swift의 다른 언어적 특성들과 상호작용 하는지 코드를 통해 알아보자.

### Task와 isolation

Task와 Actor 간의 격리는 매우 중요한 개념이다. 

우리가 actor를 동작시키려면 특정 Task를 사용해서 actor에 접근해야하는데, 이때 actor의 상태 격리를 Task 내부에서도 유지해야한다.

즉, Task 내부에서의 상태와 actor 자체의 상태가 격리되어야 하는데, 이를 어떻게 구분할까?

**actor의 격리**는 **해당 코드가 존재하는 context에 따라 결정**된다. 이렇게 얘기하면 이해가 안될테니 아래 예시를 통해 이해해보자.

<img width=600 src="https://github.com/user-attachments/assets/ab6bd9e4-e7dd-4e2c-bee2-47db79860c20">

일단 **actor의 프로퍼티와 메서드**는 **해당 actor로 격리**된다.

`reduce`로 전달된 틀로저와 같이 **Sendable하지 않은 클로저**의 경우 **actor-isolated context 내부에 있을 때 actor-isolated** 된다.

**Task initializer** 또한 **context에서 actor isolation을 상속**하므로 생성된 Task는 **처음 시작된 actor와 동일한 actor에 의해 관리**된다.

반면, **detached된 Task**의 경우 **actor isolation을 상속하지 않음으로 actor로 격리되지 않는다**. 따라서 해당 작업은 actor 외부에 존재하는 것으로 간주되어 actor에 격리된 프로퍼티 혹은 메서드에 접근하기 위해 `await` 키워드를 사용해야한다.

### Protocol 준수와 isolation

아래와 같은 예시가 있다고 하자.

```swift
actor LibraryAccount {
    let idNumber: Int
    var booksOnLoan: [Book] = []
}

extension LibraryAccount: Equatable {
    static func ==(lhs: LibraryAccount, rhs: LibraryAccount) -> Bool {
        lhs.idNumber == rhs.idNumber
    }
}
```

`LibraryAccount` actor는 `Equatable` 프로토콜을 채택하고있고, static equality(==) 메서드는 두 개의 library account를 ID를 기준으로 비교하고 있다.

이 메서드는 static이며, `self` 인스턴스가 없으므로 actor에게서 독립되어있지 않다.

대신 두 개의 actor 타입의 파라미터를 가지고 있고, 이 static 메서드는 그 두 개의 바깥에 존재하며 구현체는 **각각의 actor의 불변 상태에만 접근**하고 있으므로 **에러가 발생하지 않는다.**

그럼 위 예시를 확장시켜서 `LibraryAccount`가 `Hashable`을 준수하고 있다면 어떻게 될지 알아보자.

```swift
actor LibraryAccount { ... }

extension LibraryAccount: Hashable {
    func hash(into hasher: inout Hasher) {
        hasher.combine(idNumber) // Compile Error !
    }
}
```

이번 코드에선 컴파일 에러가 발생한다 .ᐟ.ᐟ 

아니.. 위에 함수랑 동일하게 불변 상태에만 접근하고 있는데 왜 에러가…?

위와 같은 방식으로 `Hashable`을 준수한다는 것은 `hash(into:)` 메서드가 액터 외부에서 호출될 수 있다는 것을 의미한다.

하지만 `hash(into:)`는 비동기 함수가 아니기 때문에 actor isolation을 유지할 방법이 없다.

그럼 이거 어케고침 ?! actor는 Hashable 채택 못함 ?!

노노 ~ `nonisolated` 키워드를 붙이면 해결된다.

```swift
actor LibraryAccount { ... }

extension LibraryAccount: Hashable {
    nonisolated func hash(into hasher: inout Hasher) {
        hasher.combine(idNumber) // Compile OK !
    }
}
```

**`nonisolated`** 키워드는 **해당 메서드가 actor에 내부에 명시되어 있더라도, actor 외부에 있는 것처럼 처리됨**을 의미한다.

대신 `nonisolated` 메서드는 actor 외부에 있는 것으로 처리되기 때문에 actor의 mutable한 상태에는 접근할 수 없다.

### Closures와 isolation

위에선 프로토콜을 준수하는 과정에서 actor isolation과 protocol간의 상호작용에 대해 알아봤다.

이번엔 closure다.

```swift
actor LibraryAccount {
    let idNumber: Int = 0
    var booksOnLoan: [Book] = []
}

extension LibraryAccount {
    func readSome(_ book: Book) -> Int { ... }
    
    func read() -> Int {
        booksOnLoan.reduce(0) { prev, book in
            readSome(book)
        }
    }
}
```

`read` 함수 내부에서 `readSome` 메서드를 호출할 때, `await` 키워드를 붙이지 않고 있다. 이는 **actor isolated한 `read` 함수 내부에 형성된 클로저 또한 actor isolated하기 때문**이다.

그럼 아래와 같은 예제를 보자.

```swift
extension LibraryAccount {
    ...
    
    func readLater() {
        Task.detached {
            await self.read()
        }
    }
}
```

`readLater` 라는 메서드는 메서드 내부에서 Task 블럭을 생성하고있다.

그리고 이 task 블럭은 `detached`이기 때문에 actor가 하는 다른 일들과 동시적으로 실행된다. 그러므로 이 클로저는 actor nonisolated 한 상태라고 할 수 있다.

즉, 위와 같은 detached Task 내부에서 `read` 메서드를 호출하려면 `await` 키워드를 통해 `read` 메서드가 비동기적으로 실행되게끔 해줘야한다.

### Data와 isolation

위 예제 코드를 보면 `booksOnLoan` 에 들어간 `Book` 이 무슨 타입인지에 관해서는 이야기하지 않았다.

`Book`을 struct 타입이라고 가정해보자.

<img width=600 src="https://github.com/user-attachments/assets/67108769-cacb-4ced-8565-1663c8a6c106">

그럼 위 상황과 같이 책의 title과 같은 프로퍼티를 바꿔주더라도 actor에는 아무런 영향을 끼치지 않을 수 있다.

하지만 만약 `Book`이 class 타입이라면?

<img width=600 src="https://github.com/user-attachments/assets/d0a1d1d7-011b-4184-80a4-eefe57ee3e30">

그럼 이제 book의 title을 업데이트 해줬을 때, 참조값 자체가 바뀜으로 data race가가 발생할 수 있게 된다.

값 타임이나 actor는 동시성에서 안전하게 사용할 수 있지만, class 타입은 여전히 문제가 된다. 그렇다고 여태까지 class로 쓰던 모든 타입을 actor로 변경해? 그건 에바잖아요 .ᐟ.ᐟ

그래서 또 Apple에서 동시적으로 사용해도 안전하다는 것을 명시해줄 수 있는 **`Sendable`** 이라는 친구를 만들어줬다.

## Sendable

우리는 격리만 하기 위해서 actor를 쓰는게 아니다. 우리가 actor를 사용하는 목적은 shared mutable state의 사용임을 잊지 말아야한다.

즉, **actor의 경계를 넘어(cross-actor reference)** actor 내부 요소를 사용하기 위해선 그 격리된 경계를 넘어야한다.

여기엔 2가지 방법이 있다.

1. 불변 상태 요소 사용
    
    → actor가 정의된 것과 동일한 모듈의 어느 곳에서든 불변 상태에 대한 교차 actor 참조가 허용되는 이유는 한 번 초기화되면 해당 상태가 actor의 내부 또는 외부에서 수정할 수 없으므로 **정의상 데이터 경합이 존재하지 않기 때문**이다.
    
2. 비동기 함수 호출로 수행
    
    비동기 함수를 호출하면 actor가 해당 작업을 실행하도록 요청하는 ‘메시지’로 변환한다. 그렇게 변환된 메시지는 actor에 의해 한 번에 하나씩만 처리된다.
    
    DispatchQueue와 다른 점은 이 작업이 queue처럼 FIFO 방식으로 진행되지 않는다는 점이다.
    
    즉, **메시지의 실행 순서 ≠ 메시지의 도착 순서**이다.
    

여기서 또 주의해야 할 사항이 있다.

바로, 2번 방법을 통해 값을 외부로 넘기기 위해서는 **경계를 넘겨 보낼 수 있는 값**이어야 한다는 것이다.

이러한 타입을 **Sendable** 타입이라고 한다.

즉, Sendable은 **격리된 도메인(actor)의 경계를 넘길 수 있는 자격 조건**이다.

### Sendable 조건

Sendable가 될 수 있는 것은 아래와 같은 것들이 해당된다.

- Value types
- Actor types
- Immutable Classes (완전한 불변상태의 클래스)
- Internally-synchronized Class (내부적으로 동기처리가 된 클래스)
    - 개발자가 직접 mutex 등을 이용하여 관리하는 경우를 말한다.
    - 이 경우 컴파일러는 이러한 상황을 모르기 때문에 `@unchecked Sendable` 키워드를 붙여줘야한다.
- `@Sendable` function types

### Adopting Sendable

`Sendable`은 프로토콜이라 다른 프로토콜들과 같이 그냥 채택해주기면 하면된다.

struct의 경우 해당 struct 내부에 모든 저장 프로퍼티들이 `Sendable` 타입이면 그 struct 또한 `Sendable`을 채택할 수 있다.

```swift
struct Book: Sendable {
		var title: String
		var authors: [Author] // 만약 Author이 non-Sendable이면 컴파일 에러 !
}
```

제네릭에서도 `Sendable`을 쓸 수 있는데 아래 예제와 같이 여러 제네릭 타입이 있다면 그 모든 타입들이 모두 `Sendable`일 때, 상위 제네릭 타입도 `Sendable` 타입이 될 수 있다.

```swift
struct Pair<T, U> {
		var first: T
		var second: U
}

extension Pair: Sendable where T: Sendable, U: Sendable { ... }
```

### `@Sendable` functions

프로퍼티 뿐만 아니라 함수도 Sendable 이 될 수 있다.

함수가 Sendable이 되기 위해선 아래와 같은 조건을 만족해야한다.

- mutable 캡쳐가 없어야 함
- 캡쳐가 Sendable 타입이어야 함
- 동기적인 Sendable 클로저는 actor-isolated할 수 없음
    
    > Sendable이 붙었다는 것은 **actor 경계를 넘나들 수 있음을 뜻**하고 동기적으로 작동한다는 것은 함수가 **언제 어디서 호출되어도 바로 실행 가능**하다는 뜻이다. 따라서 **actor만의 분리된 상태를 가지는 값이 있을 수 없기에 actor-isolated 할 수 없다.**
    > 

Sendable 함수 타입은 **어디서 동시 실행이 일어날 수 있는지 나타내주고**, 이를 통해 **data race를 예방**할 수 있다.

## Actor Reentrancy

**actor는 한 번에 하나의 Task 실행만 허용**하는 방식으로 Data Race 문제를 해결해왔다. 비동기 함수의 경우 오래걸리는 작업을 하며 다른 작업이 actor에 진입하는 것을 계속해서 막는 것은 비효율적이다.

따라서 **잠재적 중단지점인 await에서 actor 점유를 내려놓고 다른 task가 actor에 진입할 수 있게** 하며, 이것을 **actor reentrancy(재진입)** 라고 한다.

하지만 여기서 actor 내부의 원자성을 보존하기 힘들어진다는 문제점이 발생한다.

```swift
actor DecisionMaker {
  let friend: Friend
  
  // actor-isolated opinion
  var opinion: Decision = .noIdea

  func thinkOfGoodIdea() async -> Decision {
    opinion = .goodIdea                       // <1>
    await friend.tell(opinion, heldBy: self)  // <2>
    return opinion // 🤨                      // <3>
  }

  func thinkOfBadIdea() async -> Decision {
    opinion = .badIdea                       // <4>
    await friend.tell(opinion, heldBy: self) // <5>
    return opinion // 🤨                     // <6>
  }
}

let goodThink = Task.detached { await decisionMaker.thinkOfGoodIdea() }  // runs async
let badThink = Task.detached { await decisionMaker.thinkOfBadIdea() } // runs async

let shouldBeGood = await goodThink.get()
let shouldBeBad = await badThink.get()

await shouldBeGood // could be .goodIdea or .badIdea ☠️
await shouldBeBad
```

위 예시 코드를 보면, `goodThink`와 `badThink` 각각의 task가 비동기적으로 실행됨을 알 수 있다.

`thinkOfGoodIdea` 와 `thinkOfBadIdea` 각각의 메서드 내부의 코드는 분명 순차적으로 진행되지만 메서드 내부에서 `await` 키워드를 만나면서 함수 실행 중 중단이 된다. 그럼 이후 actor 내부 프로퍼티인 `opinion`값을 반환할 때엔 이 다른 값으로 바뀔 수 도 있게 되는 것이다.

즉, actor가 Data Race는 해결해 줄 수 있더라도 **Race Condition과 원자성을 보장해주지 않는다**는 뜻이다.

이러한 점 때문에 actor를 사용할 때에는 아래와 같은 점들을 주의해야한다.

- 상태의 변경은 동기 코드에서 실행시킬 것
- 중단점에서 actor의 상태가 변화할 수 있음을 인지하고 있을 것
- `await` 키워드 이후의 상태를 생각할 것

여기까지 보면 재진입이 단점만 발생시키는 actor의 기능인 것 같은데, 재진입 덕분에 가지는 장점도 있다.

바로 GCD의 단점 중 하나였던 priority inversion 문제를 해결해준다는 것이다.

<img width=700 src="https://github.com/user-attachments/assets/3883eca5-9b48-41ef-b811-18aa1eb9cfd5">

GCD의 serial queue의 경우 엄격한 FIFO 방식을 따른다.

이러한 특성 때문에 위와 같은 상황에서 priority가 더 높은 B 작업을 수행하기 위해서는 priority가 더 낮은 1,2,3,4,5의 작업을 먼저 수행해야하는데, 이를 priority inversion이라고 한다.

<img width=700 src="https://github.com/user-attachments/assets/81b9bfb8-e8f7-4ba2-a177-6c2d38f05650">

위 사진에서 Database, Sports feed 모두 actor를 나타낸다. 

위와 같은 상황에서 Sports feed actor가 Database actor를 호출하면 uncontended 상태이기 때문에 Database actor에 pending된 작업이 있더라도 Database actor로 hop할 수 있다. (Hopping에 대해서는 아래서 다룰 거임요 ^0^)

그리고 Sports feed의 호출에 의한 `database.save` 작업을 수행하기 위해 이 전의 작업과는 별개의 work item(`D2`)을 생성하고 그걸 실행한다.

이런식으로 actor reentrancy는 엄격한 FIFO 방식으로 작업이 진행되지 않고 나중에 생긴 작업(`D2`)이 먼저 시작될 수 있는 것을 알 수 있다.

추가로 Apple이 actor reentrancy를 만들게 된 이유를 생각해보면 Dead Lock 발생 가능성 제거하기 위해서라는 이유가 존재할 것 같다. (여긴 약간 뇌피셜 섞임 주의)

Dead Lock은 런타임에서만 검증이 가능한데 이 점이 Swift Concurrency와는 방향성이 맞지 않기 때문에 재진입이라는 기능을 도입하게 된 게 아닐까… 싶은.. ㅎㅅㅎ

## Actor Hopping

actor 내부에서 다른 actor의 메서드를 호출하면 어떻게 될까?

actor의 동작은 **cooperative thread pool에서 수행**되는데, **한 actor에서 다른 actor로 전환하는 것**을 **Actor Hopping**이라고 한다.

<img width=600 src="https://github.com/user-attachments/assets/6c1f267b-62d5-473d-9388-e88ece11ed71">

위와 같은 actor들이 있을 때 스레드 변화를 살펴보며 이해해보자.

1. Sports feed actor가 협력 스레드(cooperative thread) 1번에서 동작하다가 Database actor의 `save` 메서드를 호출했다.
    
    <img width=600 src="https://github.com/user-attachments/assets/f15ecd77-8e41-469d-ba6a-b819381dc11e">
    
2. 현재 Database actor를 아무도 사용하지 않으므로 경쟁이 없는 상태다. (이런 상태를 NonContention 상태라고 한다.) 따라서 Sports feed는 곧장 Database actor로 이동(hopping) 할 수 있다. 그리고 Sport feed는 `await database.save`에 의해 중단 상태가 된다.
3. Sports feed가 동작하던 스레드는 중단점에 의해 다른 작업이 올 수 있게 된다. 따라서 아래 그림처럼 Database actor가 해당 스레드에서 작업을 하게된다.
    
    <img width=600 src="https://github.com/user-attachments/assets/6d9dc0f4-234b-48c3-9a14-d533cebcaca1">
    
4. 이 상황에서 Weather feed actor가 다른 스레드에서 동작하다가 Database actor를 사용하려 한다면 Database actor는 D2 작업을 생성한다. 하지만 Database actor는 `D1` 작업을 실행하고 있기 때문에 `D2` 작업은 보류 상태가 된다.
    
    <img width=600 src="https://github.com/user-attachments/assets/184685f2-c1f7-4e54-8513-7d94cf37b7b9">
    
5. Weather feed actor가 동작하는 도중 중단점을 만나면 해당 스레드에는 다른 작업이 올 수 있다. 아래 그림과 같이 Health feed actor의 작업이 왔다고 가정해보자.
    
    <img width=600 src="https://github.com/user-attachments/assets/bc700c74-00d3-414e-ad9a-e579c038d1b9">
    
6. 시간이 지나 `D1` 작업이 종료되면 보류중인 `D2` 작업, 기존에 멈춘 `S1` 작업, `W1` 작업 중 우선순위가 높은 순서대로 작업을 계속해서 하게된다.
    
    <img width=600 src="https://github.com/user-attachments/assets/8e75ec7e-1ac6-4980-ad74-73f114d60f03">
    

위 과정들을 보면 알 수 있듯 actor hopping 이 가지는 두 가지 특징이 있다.

- Non-blocking Thread (스레드를 block하지 않음)
- No More Thread (추가적인 스레드를 필요로하지 않음)

~~개인적으로 continuation과 비슷하다…는 느낌은 받은 부분.. 움움..~~

# Global Actor

Global Actor(전역 actor)는 위 actor의 제한 사항을 좀 더 풀어주기 위해 만들어졌다.

actor가 격리 상태를 제공하여 데이터 경쟁을 피하게 해준다는 취지는 좋다. 하지만 만약 격리 상태가 필요한 코드 부분들이 여기저기 흩어져있으면 어떡할까?

예를 들어, UI는 MainActor에서 동작해야 하는데, 모든 class, property, function, delegate 등을 `extension MainActor { }`로 감싸는 것은 비현실적이다.

이것을 해결해줄 수 있는 것이 바로 Global Actor이다.

먼저 간단한 global actor의 사용법을 알아보자.

```swift
@globalActor
public struct SomeGlobalActor {
		public actor MyActor { }
		public static let shared = MyActor()
}
```

위 코드와 같이 `@globalActor` 키워드를 추가함으로써 custom global actor를 만들 수 있다. 이 `@globalActor` 키워드는 어떤 타입이든 붙을 수 있다.

어디서 많이 보던 문법 아닌가?

```swift
@MainActor
class CustomViewController: UIViewController { ... }
```

맞다! 아래서 설명할 것이지만, Main Actor가 바로 이 global actor를 활용하여 만들어진 actor이다.

```swift
@globalActor
public actor MainActor {
  public static let shared = MainActor(...)
}
```

### 함수에서의 자동 변환

만약 특정 global actor를 한정하여 선언한 곳에 아무 actor 제한이 없는 함수를 넣으면 자동으로 global actor에서 실행되도록 가정한다.

```swift
var callback: @MainActor (Int) -> Void

func acceptInt(_: Int) { } // 어떠한 actor 제한도 없음

callback = acceptInt // @MainActor (Int) -> Void로 변환되어 돌아감
```

하지만 역으로 적용시키면 에러가 발생한다.

```swift
let fn3: (Int) -> Void = callback // error: removed global actor `MainActor` from function type
```

<img width=600 src="https://github.com/user-attachments/assets/d43974e8-2471-493a-a39b-ec5899b923dc">

### Protocol 적용시 주의사항

```swift
protocol P {
  @MainActor func f()
}

struct X { }

extension X: P {
  func f() { } // 암시적 @MainActor
}

struct Y: P { }

extension Y {
  func f() { } // 암시적 @MainActor이 되지 않는다.
			   // 왜냐하면 Protocol P의 준수와 분리된 extension에 정의되어있기 때문이다.
			   // 이 경우 error가 나지는 않고 MainActor로 돌아가지 않는다.
}
```

특정 프로토콜이 global actor를 준수하도록 요구하면 해당 프로토콜을 준수하는 scope(extension?)에서 구현해야 요구사항을 만족할 수 있다.

# Main Actor

main actor는 단어부터 대놓고 알려주고 있는대로 **메인 스레드를 나타내는 특별한 Global Actor**를 **Main Actor**라고 한다.

기본 actor 코드는 백그라운드 스레드에서 실행되지만 Main Actor로 격리된 코드는 무조건 메인 스레드에서 실행된다.

## Main DispatchQueue

main actor는 main GCD를 통해 모든 동기화를 수행한다.

actor의 executor 관점에서 main actor의 executor는 main GCD에 해당한다.

```swift
DispatchQueue.main.async {}
await MainActor.run {}
```

따라서 **`MainActor`는 `DispatchQueue.main`을 사용해 교체**할 수 있고, 반대로 **`DispatchQueue.main.async` 작업은 `MainActor.run`으로 대체**할 수 있다.

MainActor의 `run` 메서드의 정의는 아래와 같다.

```swift
static func run<T>(
		resultType: T. Type = T. self,
		body: @MainActor () throws -> T
) async rethrows -> T where T: Sendable
```

`run`은 비동기함수로 구현되어있는데, 이는 메인 스레드에서 동작할 수 있을 때까지 기다려야 할 수도 있기 때문에 중단점을 두어 해당 스레드에서 다른 작업이 실행될 수 있도록 하기 위함이다.

메인 스레드에서 한꺼번에 많은 작업이 이뤄지길 원하는 경우엔 아래와 같이 묶어서 호출해야 일시중단 없이 한 번에 실행 가능하다.

```swift
await MainActor.run {
 // UI 관련 코드 1
 // UI 관련 코드 2
}
```

## Main Actor 내부 Task 생성

<img width=600 src="https://github.com/user-attachments/assets/77910dc4-c0db-49ec-a08a-86d460474e7d">

<img width=600 src="https://github.com/user-attachments/assets/cb4235fd-9421-46a3-a1a9-b1cdf96015d2">

**actor 내부에서 Task를 생성**하는 시점에 도달하면 **Swift는 원래 scope과 동일한 actor에서 해당 작업이 실행되도록 스케줄링**한다.

따라서 위 코드에서 노란 네모 Task는 Main Actor에서 실행 될 것이다.

<img width=600 src="https://github.com/user-attachments/assets/6a66e02e-e9aa-4110-88ee-beefd900ef7c">

시스템은 호출자에게 즉시 반환하고, Task는 추후 메인 스레드에서 실행될 것이다.

Main Actor로 격리된 VC에서 Task를 생성해도, Task는 주변 컨텍스트를 이어가기 때문에 Task 내부 작업들은 메인 스레드에서 동작하게 된다.

그럼 만약 Task 내부에서 비동기 메서드를 호출하면 어떻게 될까?

<img width=700 src="https://github.com/user-attachments/assets/490229c8-1f14-4f69-b119-728d2a9abba9">

위와 같은 상황이 있다고 해보자.

위 코드에서 `download(url:)` 메서드도 과연 메인 스레드에서 동작할까?

이에 대한 정답은 해당 **비동기 메서드가 어디에 격리되어 있느냐**에 따라 다르다는 것이다.

위 상황을 그림으로 살펴보자.

<img width=700 src="https://github.com/user-attachments/assets/b460a7e9-693e-48fb-8a2c-7a73d8e23484">

main actor에서 Task를 사용하면 해당 작업도 main actor에 격리된다. 그러나 await 키워드를 만나면 메인 스레드 제어권을 시스템에게 넘긴다.

<img width=700 src="https://github.com/user-attachments/assets/d1f3eb97-7173-4525-b96f-c2abe4559be6">

시스템에 의해 `download` 메서드가 실행될 장소가 정해지는데, 이 메서드가 struct의 메서드라고 가정해보자.

그럼 어떠한 actor에도 격리되어 있지 않기 때문에 thread pool의 임의의 스레드에서 해당 비동기 메서드 작업이 실행될 수 있게 된다.

## Main Actor Hopping

<img width=600 src="https://github.com/user-attachments/assets/049812c7-9938-4205-bff4-7cc0c63fc4bc">

메인 스레드는 cooperative thread pool과 격리되어있다.

이는 즉, 아래 사진과 같이 **Main Actor와 다른 기본 actor들 간의 hopping이 일어날 때에는 Thread Context Switching이 발생**함을 의미한다.

<img width=700 src="https://github.com/user-attachments/assets/1718b249-2ee1-4a87-9c76-56aafef819fd">

그래서 코드를 아래와 같이 잘못 작성하면..

<img width=700 src="https://github.com/user-attachments/assets/58125668-4aea-4803-9dc7-f8ddbdec52b4">

이렇게 스레드간 컨텍스트 스위칭이 자주 발생하여 성능저하가 일어날 수 있다.

그러므로 위와 같은 상황에서는 Main Actor에서 할 작업을 일괄로 처리할 수 있도록 재구조화를 해야한다.

<img width=700 src="https://github.com/user-attachments/assets/41d1c27e-f101-4f4e-b37a-93d5e121b860">

이렇게 `Article`을 불러오는 작업(백그라운드 작업)을 한 번에 수행하고 UI 업데이트 작업(메인 스레드 작업)을 한 번에 수행하는 방식으로 바꿈으로써 컨텍스트 스위칭 비용을 줄일 수 있다.

# QnA

- `Sendable`을 채택하면 내부 데이터를 안전하게 보장할 수 있을까?
- Actor는 클래스와 어떻게 다를까?
- Actor는 왜 필요할까?

# 참고자료

https://developer.apple.com/news/?id=o140tv24

https://developer.apple.com/kr/videos/play/wwdc2021/10133/

https://developer.apple.com/videos/play/wwdc2021/10254

https://developer.apple.com/videos/play/wwdc2022/110350

https://developer.apple.com/videos/play/wwdc2022/110356

https://developer.apple.com/videos/play/wwdc2022/110351

https://developer.apple.com/videos/play/wwdc2023/10170

https://developer.apple.com/documentation/Swift/Copyable

https://forums.swift.org/t/a-roadmap-for-improving-swift-performance-predictability-arc-improvements-and-ownership-control/54206

https://github.com/swiftlang/swift-evolution/blob/main/proposals/0306-actors.md

https://github.com/swiftlang/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md

https://zeddios.tistory.com/1290

https://sujinnaljin.medium.com/swift-actor-%EB%BF%8C%EC%8B%9C%EA%B8%B0-249aee2b732d