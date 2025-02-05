# 목차

- [GCD(Grand Central Dispatch)](#gcdgrand-central-dispatch)
  - [DispatchQueue](#dispatchqueue)
    - [SerialQueue](#serialqueue)
    - [ConcurrentQueue](#concurrentqueue)
    - [QoS (Quality of Service)](#qos-quality-of-service)
    - [GCD 사용시 주의 사항 !](#gcd-사용시-주의-사항-)
    - [main, global](#main-global)
- [DispatchGroup](#dispatchgroup)
- [DispatchWorkItem](#dispatchworkitem)
  - [DispatchWorkItem 기능](#dispatchworkitem-기능)
- [DispatchSemaphore](#dispatchsemaphore)
- [DispatchBarrier](#dispatchbarrier)
- [QNA](#qna)

# GCD(Grand Central Dispatch)

기존에는 개발자가 직접 스레드를 생성하고 작업(task)를 할당했다.

스레드를 만들어사용하고 없애는 책임까지 개발자에게 있었기에 효율성과 가용성 측면에서 좋지 않았다.

![요런식으로 스레드가 많이 생성될 수 있고 관리가 필요함 (내 컴터에서는 6093개가 한계인듯)](https://hanyo3477.notion.site/image/https%3A%2F%2Fprod-files-secure.s3.us-west-2.amazonaws.com%2F35745b71-fac8-4aef-ac72-46ac6d683435%2Ff0cc5538-eae8-4b75-9e08-7fcaec0c8658%2Fimage.png?table=block&id=17e17c78-8267-809b-8738-cfc5885046e5&spaceId=35745b71-fac8-4aef-ac72-46ac6d683435&width=480&userId=&cache=v2)

요런식으로 스레드가 많이 생성될 수 있고 관리가 필요함 (내 컴터에서는 6093개가 한계인듯)

이를 보완하고자 GCD가 등장했다.

GCD는 thread pool 패턴에 기반한 작업의 병렬 처리를 구현한 것이다.

## DispatchQueue

DispatchQueue 객체는 이를 실현하는 주된 방법이다.

DispatchQueue 객체를 생성하고 작업을 할당하면 알아서 스레드를 만들고 작업을 수행한 후 스레드를 지운다.

![https://hanyo3477.notion.site/image/https%3A%2F%2Fprod-files-secure.s3.us-west-2.amazonaws.com%2F35745b71-fac8-4aef-ac72-46ac6d683435%2Fe1ceabb8-a2a0-42c7-a1d9-1113fd9e40f3%2Fimage.png?table=block&id=17e17c78-8267-80e3-b89e-cb2f1c6eb1b7&spaceId=35745b71-fac8-4aef-ac72-46ac6d683435&width=1060&userId=&cache=v2](https://hanyo3477.notion.site/image/https%3A%2F%2Fprod-files-secure.s3.us-west-2.amazonaws.com%2F35745b71-fac8-4aef-ac72-46ac6d683435%2Fe1ceabb8-a2a0-42c7-a1d9-1113fd9e40f3%2Fimage.png?table=block&id=17e17c78-8267-80e3-b89e-cb2f1c6eb1b7&spaceId=35745b71-fac8-4aef-ac72-46ac6d683435&width=1060&userId=&cache=v2)

Synchronous 하게 실행하는 경우 worker에서 동작하던 DispatchQueue가 해당 작업을 넣은 스레드로 이동해서 작업을 수행한다.

![Worker의 점선 작업 == Thread의 보라색 작업](https://hanyo3477.notion.site/image/https%3A%2F%2Fprod-files-secure.s3.us-west-2.amazonaws.com%2F35745b71-fac8-4aef-ac72-46ac6d683435%2Fe66d7372-3cdf-4972-8223-17f1f23819a1%2Fimage.png?table=block&id=17e17c78-8267-80fd-9996-f652320631ca&spaceId=35745b71-fac8-4aef-ac72-46ac6d683435&width=1060&userId=&cache=v2)

Worker의 점선 작업 == Thread의 보라색 작업

```swift
// queue를 만들고
let queue = DispatchQueue(label: "example")

// async로 작업을 진행
queue.async {
	let smallImage = image.resize(to: rect)
	DispatchQueue.main.async {
		imageView.image = smallImage
	}
}
```

이런식으로 비동기 작업을 진행할 수 있다.

위의 코드는 큐를 serial하게 만들었을 때다. (default == serial)

### SerialQueue

기본적으로 DispatchQueue는 일단 Queue다.

큐 특성상 FIFO로 동작한다.

```swift
let queue = DispatchQueue(label: "com.example.imagetransform")
queue.async {
    Logger().log("1")
}
queue.async {
    Logger().log("2")
}
queue.async {
    Logger().log("3")
}
queue.async {
    Logger().log("4")
}
queue.async {
    Logger().log("5")
    sleep(2)
}
queue.async {
    Logger().log("6")
}
queue.async {
    Logger().log("7")
}
queue.async {
    Logger().log("8")
}
queue.async {
    Logger().log("9")
}
queue.async {
    Logger().log("10")
}
```

위와 같이 코드를 실행하면 1부터 10까지 순서대로 출력된다. (중간에 sleep을 호출하더라도)

### ConcurrentQueue

그럼 작업들을 동시에 실행하는 방법은 무엇일까?

DispatchQueue에 concurrent 옵션을 주는 것이다.

```swift
let queue = DispatchQueue(label: "com.example.imagetransform", attributes: .concurrent)

queue.async {
    Logger().log("1")
}
queue.async {
    Logger().log("2")
}
queue.async {
    Logger().log("3")
}
queue.async {
    Logger().log("4")
}
queue.async {
    Logger().log("5")
    sleep(2)
}
queue.async {
    Logger().log("6")
}
queue.async {
    Logger().log("7")
}
queue.async {
    Logger().log("8")
}
queue.async {
    Logger().log("9")
}
queue.async {
    Logger().log("10")
}
```

위의 코드를 실행하면 1~10의 숫자가 순서와 관계없이 출력된다.

### QoS (Quality of Service)

큐에 대한 우선순위라고 할 수 있다.

```swift
let queue = DispatchQueue(label: "com.example.imagetransform",
                          qos: .background, attributes: .concurrent)
```

QoS 종류는 다음과 같다. (우선순위가 높은 순)

1. `userInteractive`
2. `userInitiated`
3. `default`
4. `utility`
5. `background`
6. `unspecified`

위의 우선순위가 높은 작업을 먼저 실행하도록 한다.

그런데 작업 자체에도 우선순위를 부여할 수 있다.

```swift
queue.async(qos: .userInteractive) {
    Logger().log("1")
}
```

Priority Inversion을 방지하는 차원에서 큐의 우선순위는 큐의 우선순위와 작업의 우선순위 중 더 높은 우선순위를 따른다.

`DispatchQueue.global(qos: .background).async(.utility)`

Queue와 Task 각각 QoS가 다름, 둘 중 높은 걸 따라간다.

- Task QoS > Queue QoS
  Task의 우선순위가 큐보다 더 높은 경우 utility로 우선순위가 상승하게 된다.
- Task QoS < Queue QoS
  Queue QoS 따라간다.

<img width="658" alt="%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202025-01-17%20%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB%201 44 19" src="https://github.com/user-attachments/assets/f1484be5-d179-4da7-ae00-96531bcf1f9e" />


utility 작업이 먼저 큐에 들어가고, 이후 default 작업들이 들어간다.

그러나, utility가 끝날 때까지 queue의 QoS는 utility이다.

### GCD 사용시 주의 사항 !

GCD를 사용할 때 조심해야할 사항이 몇 가지 있다.

1. UI는 메인 스레드에서 처리해야한다.

   메인 스레드가 UI를 담당하는 것은 iOS에만 국한된 것이 아니라, 모든 OS에 적용되는 사항이다.

   만약 메인 스레드에서 돌아야하는 코드가 있는데 background 스레드에서 돌고있다면 보라색 경고창이 뜨게된다 !

   Main Thread Checker라는 친구가 해주는 건데, background 스레드에서 유효하지 않은 동작들을 잡아내는 친구이다. (Thread Sanitizer 과 비슷한 그친구요 ~)

2. `sync` 메소드에 대한 주의 사항 3가지

   2-1. 메인 큐에서 다른 큐로 작업을 보낼 때 `sync`를 사용하면 안된다.

   `sync`로 작업을 보낸다는 것은 “해당 작업이 끝날때까지 기다린다”는 것을 의미한다는 것을 모두가 .ᐟ.ᐟ 알 것이다. 메인 스레드는 UI를 업데이터 해줘야하는데 다른 작업들이 끝날때까지 기다린다..? 이건 곧 작업이 끝날때까지 UI 업데이트가 지연된다는 의미로 화면이 버벅여 보일 것이다.

   따라서 메인 스레드에서 작업을 넘길 때에는 항상 `async`(비동기)로 작업을 보내도록 하자 ~

   2-2. 현재와 같은 큐에 `sync`로 작업을 보내면 안된다.
	![image-8](https://github.com/user-attachments/assets/9d6d3c9d-9f36-41c2-bd21-7b5dce765a77)

   같은 큐에 동기적으로 작업을 보낸다는 건 위와 같은 코드를 의미한다.

   하나의 큐에서 사용하는 스레드 객체를 정해져있다. 따라서 같은 큐에 작업을 넣으면 같은 스레드에 해당 작업이 배치될 수 있는데, 해당 스레드가 sync로 인해 멈춰있는 상황이라면 교착상태(Dead Lock)가 발생한다.

   꼭 ! 발생하는 것은 아니지만 발생할 가능성이 있기 때문에 하지말하는 것 ~

   참고로 Global Queue는 QoS에 따라 각각 다른 큐 객체를 생성하므로 교착상태가 발생하지 않음 ~

   2-3. 메인 스레드에서 `DispatchQueue.main.sync`를 사용하면 안된다.

   2-2와 같은 이유인데, 메인 스레드에서 `sync`로 작업을 보내면 메인 스레드가 교착상태가 되어버린다 ! 이건 걍 무조건 교착상태가 되기 때문에 아예 컴파일 에러가 떠버려유 ~

3. 객체에 대한 캡처를 주의해야한다.

   동작해야 할 작업을 queue에 보낸다는 것은 결국 클로저를 보내는 것을 말한다.

   따라서 객체에 대한 캡처 현상이 발생할 수 있으며, 이는 자칫하면 순환참조 문제를 일으킬 수 있다.

   웬만하면 `[weak self]` 붙여주자 ~ ^^ 방어적 프로그래밍 굿뜨 ~

### main, global

DispatchQueue는 main과 global()을 통해 메인스레드에 작업을 할당하거나 바로 비동기 맥락으로 작업을 실행할 수 있다.

main은 프로세스에서 하나만 존재하며, iOS에서는 UI를 담당한다.

- UI이외의 작업이 메인 스레드에서 동작하는 것은 상관없다. (다만 UI응답시간이 길어짐)
- UI 작업이 메인 외의 스레드에서 동작하면 런타임에 오류를 던진다.

`global()`를 이용하면 시스템에서 제공하는 DispatchQueue를 이용하게 된다. 위에서는 custom queue를 만들어서 사용했다. 일반적으로 custom queue를 남발하면 스레드가 많아져서 context switch에서 오버헤드가 발생할 수 있으므로 `global()`사용을 권장한다.

다만, `global()`은 concurrent로 동작하기에 작업의 순서가 중요하다면 별도로 custom queue를 만들면 된다.

- 이때 target을 `global()`로 설정하면 시스템에서 제공하는 스레드 풀을 공유한다.
  - Target?
  - 서로 다른 Queue가 각자에 대한 순서를 유지하면서도 같은 맥락에서 함께 수행하는 것이 가능하다.
    ![image-9](https://github.com/user-attachments/assets/3ca7365f-07c9-44b2-afc4-e6ef726c8aac)
    ![image-10](https://github.com/user-attachments/assets/fb37c9c6-65ff-4134-8dcf-1e1ceafcee9e)
    Q1과 Q2는 순서가 유지되지만, 하나의 EQ라는 Queue에서 실행된다.
    이렇게 하면 Q1과 Q2간에 context switching을 줄이고 EQ하나로 통합할 수 있다.

# DispatchGroup

하나의 작업이 무거울 수 있다.

만약 아주 오래걸리는 작업을 하나의 task로 dispatch queue에 넣으면 해당 작업을 수행하는데 오래 걸릴 수 있다.

이럴 때 DispatchGroup을 이용하면 된다.

DispatchGroup은 여러개의 작업을 하나로 관리할 수 있다.

또한 작업의 완료 시점을 알고 원하는 작업을 수행할 수 있다.

```swift
// DispatchGroup 생성
let dispatchGroup = DispatchGroup()

// dispatchGroup에 작업 추가
DispatchQueue.global().async(group: dispatchGroup) {
    Logger().log("gergerg")
    sleep(3)
}

// 작업 종료 안내
dispatchGroup.notify(queue: DispatchQueue.main) {
    Logger().log("done")
}
```

이렇게 하면 작업이 끝났을 때 알림을 받고 안의 함수블럭을 실행한다.

# DispatchWorkItem

지금까지 큐에 작업을 넘길 때 클로저 안에 넣어서 처리했다.

```swift
DispatchQueue.global().async {
		print("Task 시작")
		print("Task 끝")
}
```

클로저를 묶어 클래스로 캡슐화한 것이 **DispatchWorkItem** 이다.

`DispatchWorkItem` 은 지금껏 클로저로 보내왔던 **작업이 캡슐화 된 class** 이다.

![image-11](https://github.com/user-attachments/assets/37a991c8-1e50-4597-b4c1-ffa76f22168f)

아래 예시 코드를 보면 알 수 있듯 `DispatchWorkItem`을 생성할 때 `qos` 파라미터를 통해 작업의 우선순위도 설정할 수 있다.

```swift
let defaultItem = DispatchWorkItem { // Task }
let utilityItem = DispatchWorkItem(qos: .utility) { // Task }
```

그리고 이렇게 정의 된 `DispatchWorkItem`은 `async(execute:)` 라는 `DispatchQueue`의 인스턴스 메소드를 통해 큐로 보낼 수 있다.

```swift
let utilityItem = DispatchWorkItem(qos: .utility) { // Task }

DispatchQueue.global().async(execute: utilityItem)
```

혹은 perform() 메소드를 통해 현재 스레드에서 sync하게 동작시킬 수 도 있다.

```swift
utilityItem.perform()
```

## DispatchWorkItem 기능

`DispatchWorkItem`은 아래 두 가지 기능을 제공한다.

1. 취소 기능

   `DispatchWorkItem` 은 `cancel()` 이라는 인스턴스 메소드를 가지고 있다.

   ```swift
   let item = DispatchWorkItem { }
   item.cancel()
   ```

   `cancel()` 메소드는 작업의 실행 여부에 따라 동작이 조금 달라진다.

   작업이 아직 큐에 있고 실행되기 전에 `cancel()` 을 호출하면 작업이 제거된다.

   만약 실행 중인 작업에 `cancel()` 을 호출하는 경우, 작업이 멈추지는 않고 `DispatchWorkItem`의 속성인 `isCancelled`가 `true`로 설정된다.

2. 순서 기능

   `notify(queue:execute:)` 라는 함수를 통해 작업 A가 끝난 후 작업 B가 특정 queue에서 실행되도록 지정할 수 있다.

   ```swift
   let itemA = DispatchWorkItem { }
   let itemB = DispatchWorkItem { }

   itemA.notify(queue: DispatchQueue.global(), execute: itemB)
   ```

# DispatchSemaphore

`DispatchSemaphore`는 iOS에서 세마포어를 사용하기 위해 쓰이는 객체이다 .ᐟ.ᐟ

```swift
// 공유 자원에 접근 가능한 작업 수를 2개로 제한
let semaphore = DispatchSemaphore(value: 2)
```

위와 같이 공유 자원에 접근 가능한 작업 수를 명시하고, 임계 구역에 들어갈 때에는 semaphore의 `wait()`을, 나올 땐 `signal()` 메소드를 호출한다.

```swift
for i in 1...3 {
		semaphore.wait() // semaphore 감소
		DispatchQueue.global().async() {
				// 임계 구역
				print("공유 자원 접근 start")
				sleep(3)

				print("공유 자원 접근 end")
				semaphore.signal() // semaphore 증가
		}
}

```

`DispatchSemaphore`는 두 가지 방식으로 사용할 수 있는데, 하나는 위와 같은 방식처럼 동시 작업 개수를 제한하는 것이다.

또 다른 하나는 **두 스레드가 특정 이벤트의 완료 상태를 동기화 하는 경우**에 유용하다.

이 용도로 사용할 때에는 `DispatchSemaphore`의 초기값을 0으로 설정하면 된다.

```swift
let semaphore = DispatchSemaphore(value: 0)

print("task A가 끝나길 기다리는 중..")

DispatchQueue.global().async() {
		// task A
		print("task A start..")
		print("task A end..")

		// task A 끝났다고 알려쥼 ~
		semaphore.signal()
}

// task A 끝날 때 까지는 value가 0이므로, task A 종료까지 block
semaphore.wait()
print("task A 완료 ~")

```

# DispatchBarrier

**DispatchBarrier**는 *“concurrent dispatch queue에서 실행되고 있는 task들에 대한 동기화 지점”* 라고 공식문서에서 말한다.

concurrent queue에 barrier를 추가하면, queue는 이전에 제출된 모든 작업이 실행을 마칠 때까지 barrier block의 실행을 지연시켰다가 앞에 제출된 작업들이 모두 끝나면 barrier block을 자체적으로 실행한다고 한다.

그럼 사용 방법은 다음과 같다.

dispatch queue의 인스턴스 메소드인 `async(group:qos:flags:execute:)` 를 사용해서 `flags` 파타미터에 `.barrier`를 넣어주면 된다.

```swift
concurrentQueue.async(flags: .barrier, execute: { })
```

> ### 효준이가 헷갈렸던 것
> 
> <img width="575" alt="%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202025-01-17%20%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB%201 07 35" src="https://github.com/user-attachments/assets/c3fc1da7-5c79-468a-8d50-3831c9b9b4b8" />
> 
> - 만약 프로세서가 1개라면 위 작업이 끝나면 `6+a`초 일까, `3+a` 초 일까?
  6 + a 초임
  결국 CPU는 한 번에 하나의 작업밖에 처리하지 못 하기 때문에 멀티 스레드로 돌려봤자 컨텍스트 스위칭하느라 6초에 a시간만큼 더 소요되는 것
  멀티코어 프로세서에서 멀티 스레드를 해야 효과가 있음
> - 그럼 단일 코어 프로세서에서 멀티 스레드를 하면 이점이 없나 ?
  답은 No
  단일 코어여도 멀티스레드는 장점이 있음
  작업을 비동기적으로 처리할 수 있으므로 I/O 작업에서 이점을 얻을 수 있음
  I/O 요청하고 자기 할 일 하러 가면 되니까

# QNA

- GCD에서 동기/비동기, 직렬/동시 큐에 대해 설명해주세요.
- GCD의 주요 구성 요소는 어떤 것들이 있나요.
- GCD에서 우선순위 역전을 방지하기 위해 어떻게 할까요.
- 동시성 프로그래밍은 무엇일까요.
- 병렬 프로그래밍과 동시성 프로그래밍의 차이점은?
- 동시성 프로그래밍을 하려면 코어나 프로세서를 사용하지 않고 어떻게 구현하나요?
- iOS에서 동시성 프로그래밍 방법은?
- GCD의 동작 원리에 대해 설명해주세요.
- GCD와 Operation Queue의 차이점은?
