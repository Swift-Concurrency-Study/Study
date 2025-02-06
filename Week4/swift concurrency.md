# 목차

- [Swift Concurrency 등장 배경](#swift-concurrency-등장-배경)
  - [가독성 & 에러처리 관점](#가독성--에러처리-관점)
    - [가독성](#가독성)
    - [에러 핸들링 안정성](#에러-핸들링-안정성)
  - [성능적 관점](#성능적-관점)
    - [스레드 관점](#스레드-관점)
    - [우선순위 역전](#우선순위-역전)
  - [CompletionHandler → Async/await](#completionhandler--asyncawait)
- [스레드 제어권](#스레드-제어권)
  - [스레드 제어권 관점](#스레드-제어권-관점)
    - [sync 에서의 스레드 제어권](#sync-에서의-스레드-제어권)
    - [async 에서의 스레드 제어권](#async-에서의-스레드-제어권)
- [스택 프레임의 변화](#스택-프레임의-변화)
  - [sync 방식의 Stack Frame](#async-방식의-stack-frame)
  - [async 방식의 Stack Frame](#async-방식의-stack-frame)
    - [add가 호출된 상황](#add가-호출된-상황)
    - [await database.save가 호출된 경우](#await-databasesave가-호출된-경우)
    - [save 메소드 종료 후 Return 과정](#save-메소드-종료-후-return-과정)
- [async let](#async-let)
  - [async-let 예제](#async-let-예제)
- [Task](#task)
- [Structured Concurrency](#structured-concurrency)
- [Task Group](#task-group)
- [AsyncSequence/Stream](#asyncsequencestream)
- [협력적 취소](#협력적-취소)
- [Continuation](#continuation)
- [Swift Concurrency](#swift-concurrency)
- [참고자료](#참고자료)

# Swift Concurrency 등장 배경

> swift concurrency란?
WWDC 2021년에 소개된 동시성 프로그래밍 API
> 

async와 await 키워드로 비동기 태스크 종료 후 코드를 작성할 수 있다.

await로 중지되면, 이후 사용해야 하는 데이터를 Heap 영역에 저장하고, 이후 다시 돌아오면 꺼내서 사용한다.

GCD와 비교하며 왜 등장하게 되었는지 살펴보겠다.

## 가독성 & 에러처리 관점

### 가독성

<img width="542" alt="image" src="https://github.com/user-attachments/assets/10a019ef-e09b-42fd-ad55-c2e96a26d13c" />


기존의 GCD는 비동기 작업이 끝났는 지의 여부를 Completion Closure를 통해 알려준다.

그러면 A 작업이 끝나면 B, B 작업이 끝나면 C, … 이를 비동기로 처리한다면 무수히 많은 Depth가 생겨 들여쓰기에 의해 가독성이 낮아질 것이다.

반면 Swift Concurrency는 아래와 같이 동작한다.

<img width="552" alt="image 1" src="https://github.com/user-attachments/assets/6ed1c6d2-4f61-48ed-b836-b8037657cd32" />


위 사진들의 코드는 동일한 로직인 것.

await 키워드를 통해 실제 비동기 코드이지만, **동기처럼** 보이게 하는 효과를 지녀 가독성을 증가시킬 수 있다.

### 에러 핸들링 안정성

URLSession을 통해서 이미지를 다운 받는 메소드가 있다고 하자

이미지 내려받는 걸 실패했을 때 예외처리하는 상황으로 둘을 비교해보겠다.

<img width="555" alt="image 2" src="https://github.com/user-attachments/assets/125767ce-3f46-48d6-8614-894489e59339" />


GCD는 이미지를 성공적으로 내려받으면 컴플리션 핸들러의 첫 번째 파라미터로 이미지를 넘겨준다.

그러나 상태코드가 200이 아니거나, 내려받은 data가 Nil인 경우 nil을 줘야 한다.

개발자가 실패했을 때에 대한 에러처리를 잘 하면 문제가 없지만, 휴먼 에러등의 이유로 컴플리션 클로저 호출을 빼먹으면 문제가 될 수 있다.

매번 확인해야 하는 번거로움이 있음

추가로, Result를 쓰면 가독성은 더 심각해짐

<img width="741" alt="image 3" src="https://github.com/user-attachments/assets/dc6df136-be69-491d-9797-5dd3c98d47bb" />


그래서 Swift Concurrency에서는 컴플리션 핸들러를 사용하지 않는다.

대신 do-catch 혹은 gaurd에 의해 Error를 던져주는 식으로 처리를 할 수 있는 것.

이러면 실패했을 때 컴플리션 핸들러를 빼먹어도 문제가 되지 않는다.

+ 콜백을 안 해도 되니까 가독성도 좋아짐

## 성능적 관점

> 스레드 생성량과 Context Switching 수를 비교해 보는 과정
> 

### 스레드 관점

GCD는 Thread Explosion을 조심해야 한다. (폭발이 아니라 너무 많이 생성되는 것 ㅇㅇ)

스레드를 너무 많이 만들면 컨텍스트 스위칭이 많아지고, 성능이 오히려 저하된다.

너무 많은 스레드 블록에서의 메모리 오버헤드, 스케줄링 오버헤드 등이 문제라 Thread Explosion을 예방하는 안전한 코드 작성이 필요함

반면, Swift Concurrency에서의 동작

await로 중단되었을 때 CPU가 컨텍스트로 스위칭하는 게 아니라, 같은 스레드에서 다음 함수를 실행시킨다.

**이로 인해 하나의 코어가 하나의 스레드 실행을 보장함**

이러면 컨텍스트 스위칭으로 할 작업들을 같은 스레드에서 함수로 호출하니까 비용이 발생하지 않음

<img width="737" alt="image 4" src="https://github.com/user-attachments/assets/86958565-d0c4-4bd8-8176-b98a2cafcc08" />

Swift Concurrency에서는 Actor가 Thread를 재활용하고,

Thread의 개수를 Core의 개수와 동일하게 제한해서 이 문제를 해결한다.

### 우선순위 역전

GCD로 동시성 프로그래밍을 할 경우, 우선순위 역전이 발생할 수 있다고 한다.

하나의 큐에서 QoS가 각기 다른 작업이 담길 수 있는 것.

Background QoS인 작업이 큐에 추가되고, User Initiated 작업이 추가됐다고 가정하겠다.

그러면 **background 작업들의 우선순위를 User Initiated로 올려서** 새로 추가된 태스크가 너무 기다리지 않게 함

이게 FIFO 방식이라 그런듯

반면 Swift Concurrency는 FIFO가 아니므로 우선순위가 높은 애들을 먼저 처리해줄 수 있음

Task에 priority를 부여해서 앞에 작업이 쌓여있더라도 높은 우선순위 작업이 들어오면 해당 작업 먼저 수행시킬 수 있다.

<img width="718" alt="image 5" src="https://github.com/user-attachments/assets/08fddff9-3fdc-4bd6-a848-4b5b83d3955c" />


## CompletionHandler → Async/await

서버에서 이미지 리스트를 불러오고 이미지에 대한 썸네일을 화면에 보여주는 과정이 있다고 해보자. 서버에서 가져온 정보를 `UIImage`로 변환하는 과정에는 아래와 같은 일련의 과정이 필요하다.

<img width="451" alt="image 6" src="https://github.com/user-attachments/assets/fa92c7b0-3fe3-41fc-82dc-dcc2cf6b6fcf" />


해당 과정을 살펴보면 하위 과정이 실행되기 위해서는 상위 과정에 대한 결과값이 필요하다. 즉, 위 과정들은 모두 차례대로 진행되어야함을 의미한다.

`thumbnailURLRequest`나 데이터를 UIImage로 전환하는 `UIImage(data:)` 와 같은 메서드들은 결과가 매우 빠르게 도출되기 때문에 어떤 스레드에서 호출되어도 상관없으며 동기적으로 실행되어도 괜찮다.

하지만 `dataTask(with:completion:)` 이나 `prepareThumbnail(of:completionHandler:)` 와 같은 함수들은 실행하고 결과가 나오기까지 시간이 조금 걸린다. 따라서 SDK에선 비동기 함수를 제공하며 위와 같은 함수들은 비동기로 실행되어야한다.

그럼 위 과정을 이제 기존의 `completionHandler`를 통한 비동기 처리 방식으로 코드를 짜보자.

<img width="621" alt="image 7" src="https://github.com/user-attachments/assets/f8c93e43-3a5b-4c3f-86ea-c72607c7a84d" />


먼저 `thumbnailURLRequest(for:)` 메소드를 호출한다. 위에서 말했듯 이 함수는 동기적으로 호출되는 함수이기 때문에 빠르게 처리가 된다.

<img width="609" alt="image 8" src="https://github.com/user-attachments/assets/dcd6196f-d1a7-4a23-b1f0-bf8475d7068f" />

이후 `URLSessionDataTask` 를 동기적으로 만들고 비동기 작업을 시작하기 위해 따로 `task.resume()` 를 호출해야한다.

데이터를 다운로드 받는 것은 시간이 걸리는 작업이며, 그 동안 스레드가 block되지 않게 하기 위해서는 위와 같이 비동기 작업으로 처리해주는 것이 매!우! 중요하다.

<img width="612" alt="image 9" src="https://github.com/user-attachments/assets/6713c76e-e798-4eb8-954e-a333dda0b4b3" />


다운로드 요청이 완료되면 completionHandler를 통해 data, response, error 값들이 옵셔널하게 도착한다.

만약 error가 발생했다면, completionHandler를 호출하여 에러 처리를 해줘야한다.

<img width="611" alt="image 10" src="https://github.com/user-attachments/assets/249b73c7-d330-4b64-b708-86f478cdd2d1" />


값이 잘 도착했다면, `UIImage(data:)` 를 호출하여 동기적으로 데이터를 `UIImage`로 변환시켜준다.

<img width="615" alt="image 11" src="https://github.com/user-attachments/assets/6323969c-8ad6-4ecd-9c0a-ce871241b9e8" />

이미지가 잘 생성이 되었다면, 우리는 `prepareThumbnail` 메소드를 호출하고 또 completionHandler를 통해 값을 전달한다. 해당 과정이 이루어지는 동안 스레드는 unblocked되고 다른 작업을 할 수 있게 된다.

간단한 과정인데 일단 굉장히 장황하게 설명되었다.. 그럼 위 코드는 이제 완-벽 한걸까??

노노 .ᐟ.ᐟ

<img width="636" alt="image 12" src="https://github.com/user-attachments/assets/4c6c71ae-97ad-4318-98bf-1ac4cb76ff13" />


위 `guard-let` 구문을 보면 에러에 대한 처리 없이 그냥 함수를 종료시켜버린다 ! 따라서 UIImage를 생성하거나 썸네일을 생성하는데 실패했더라도, `fetchThumbnail` 의 호출부는 이를 알 수 없고, 이미지는 영영 업데이트 되지 않게된다..

<img width="543" alt="image 13" src="https://github.com/user-attachments/assets/fa331428-5453-4cb1-9005-1c989cc9d4bc" />


이를 해결하기 위해선 모든 함수 return 경로에 error를 담은 completion을 호출해야한다.. 여기선 Swift의 기본 에러 핸들링 메커니즘을 사용할 수 없는 것이다. (error throw하는거 못 함;)

이렇게 completionHandler를 사용한 두 개의 동기, 두 개의 비동기 처리를 하는 함수를 완성시켰다.

근데 20줄 따리의 코드 중 무려 5줄의 미묘한 버그가 끼어있을 수 있는 에러를 담은 completionHandler가 끼어있다. ㅋㅋ 이게 맞냐고 ~

이걸 아 ~ 주 조금은 안전하게 만들 수 있다. 바로 Result 타입을 활용하는 방식이다.

<img width="611" alt="image 14" src="https://github.com/user-attachments/assets/5315bffd-3959-4d4e-8b02-75b02414959b" />

음.. 근데 그냥 코드가 조금 더 길어지고 못생겨짐..

자 ~ 그럼 이제 이 못나고 불안전한 코드를 async/await을 활용하여 리팩토링해보자 !

<img width="620" alt="image 15" src="https://github.com/user-attachments/assets/06955a33-c976-4c72-bf7a-68e91b18ad5e" />

먼저 함수를 작성할 때, `throws` 키워드 전에 `async` 키워드를 붙여준다. 에러를 던지지 않는 함수라면 그냥 화살표 전에 `async`를 붙여주면 된다.

그럼 이제 깔꼼하게 `fetchThumbnail` 함수는 UIImage를 반환하고, 에러가 발생하면 throw를 할 수 있게 되었다 !

<img width="619" alt="image 16" src="https://github.com/user-attachments/assets/97d121b8-e206-48f7-9b39-3f85cdfa0ad6" />

맨 처음 `fetchThumbnail` 이 호출되면, 이 전과 같이 `thumbnailURLRequest` 를 호출한다. 이 함수는 동기함수로, 해당 작업을 하는 동안은 스레드가 block 된다.

<img width="615" alt="image 17" src="https://github.com/user-attachments/assets/8f1c972d-1d6c-422e-a434-5175eadd293a" />

다음으론 `data(for:)` 메소드를 호출하여 데이터를 다운받기 시작한다. `data(for:)` 메소드는 `dataTask` 와 같이 Foundation에서 제공하는 메소드로, 비동기적으로 처리된다. 하지만 `dataTask`와는 달리, `data(for:)` 메소드는 **awaitable**하다. 해당 함수가 호출되면, 빠르게 중단되고, 스레드는 unblocked 되며 다른 작업을 할 수 있게 된다.

`throws` 키워드가 붙은 함수를 호출하기 위해서 `try`를 붙여야 하는 것처럼, `async` 키워드가 붙은 함수를 호출하기 위해선 `await` 키워드를 붙여줘야한다.

`dataTask`와 `data` 두 버전 모두 값과 에러에 대한 처리를 제공하고 있다. 하지만 awaitable 버전은 훨씬훨씬 코드가 간단해진 것을 확인할 수 있다 !

<img width="615" alt="image 18" src="https://github.com/user-attachments/assets/f876637c-f7ac-4f62-898b-8f5610f3c273" />

이후 데이터를 `UIImage`로 변환시키고 thumbnail 프로퍼티에 접근하면 썸네일이 렌더링되기 시작한다. 썸네일이 생성되기 시작하면, 스레드는 또 다시 unblocked 되며 다른 작업을 할 수 있게 된다. 그리고 썸네일이 잘 생성되었다면 그걸 반환하고, 실패했다면 error를 throw하게 된다.

껄껄.. completionHandler로는 20줄이었던 코드가 단 6줄로 변-신 ~

심지어 depth가 깊어지지도 않은 완전 straight한 코드임..

위 코드에서 확인할 수 있듯, `async` 키워드는 함수에만 붙을 수 있는게 아니고 프로퍼티, 이니셜라이저 등에도 모두 붙일 수 있다.

저 `thumbnail`이라는 프로퍼티는 기본제공이 아니고 따로 만든 프로퍼티인데 그 코드를 살펴보자.

<img width="644" alt="image 19" src="https://github.com/user-attachments/assets/120fad08-5a78-4109-8f52-fc434348fb23" />

UIImage의 extension에 구현해줬는데, 구현부는 굉장히 짧다. thumbnail 프로퍼티는 CGSize를 만들고 `byPreparingThumbnail(ofSize:)` 의 결과를 기다린다.

프로퍼티가 async 키워드를 가지기 위해 필요한 사항이 몇 가지 있다.

첫 번째로, 명시적 `getter` (explicit getter)가 있어야 한다는 점이다. async 키워드를 붙이기 위해선 `getter`를 명시적으로 적어줘야한다. 추가로 Swift 5.5부터는 `getter`에도 `throws` 키워드가 붙일 수 있다.

두 번째로는, 프로퍼티가 `setter`를 가져서는 안된다. 프로퍼티에 `async` 키워드가 붙기 위해서는 read-only 여야만 한다.

함수나 프로퍼티, 이니셜라이저에서 `await` 키워드는 함수가 어디서 스레드를 unblock할 것인지를 나타낸다. `await` 키워드는 다른 곳에서도 사용될 수 있다.

<img width="627" alt="image 20" src="https://github.com/user-attachments/assets/1528e7b6-5f43-490c-ae39-3416de449db1" />

`async` 시퀀스를 반복하기위한 반복문에서 위와 같이 사용할 수 있다. 비동기 시퀀스는 각각의 요소들을 비동기적으로 제공한다는 점을 제외하고는 일반 시퀀스와 같다. 따라서 다음 요소를 가져오기 위해선 `await` 키워드가 붙어야하며, 이는 해당 요소가 `async`임을 나타낸다.

(여기서 이제 AsyncSequence에 대해 더 알고싶으면 [Meet AsyncSequence](https://developer.apple.com/videos/play/wwdc2021/10058/)로, 많은 비동기 작업들을 병렬적으로 실행하는 것을 알고싶다면 [Structed concurrency in Swift](https://developer.apple.com/videos/play/wwdc2021/10134/)로..)

# 스레드 제어권

await로 비동기 메소드를 호출하는 경우, Potential Suspension Point로 지정된다.

<img width="664" alt="image 21" src="https://github.com/user-attachments/assets/4e2a1e88-4d29-488d-b7c5-51a29e872670" />

생각해보면 당연하다

await로 URLSession의 data 비동기 메소드를 호출하면 그 아래 작업들은 data 메소드가 끝날 때 까지 기다리게 된다.

이 지점을 `Suspension Point` 라고 한다.

이를 통해 fetchThumbnail의 메소드는 더 이상 할 일이 없으니, 해당 작업을 처리하던 스레드가 다른 동작을 할 수 있게끔 제어권을 놓아주는 행위를 할 수 있다.

스레드를 멈추는 것이 아닌, 다른 작업을 할 수 있게 제어권을 넘기는 것 말이다.

Suspend 된다 = 해당 스레드에 대한 제어권을 포기한다

라고 봐도 무방할듯

## 스레드 제어권 관점

### sync 에서의 스레드 제어권

A 함수에서 B라는 sync 동기 함수를 호출하면, A 함수가 실행되던 스레드의 제어권을 B 함수에게 전달한다.

A 함수는 동기적으로 호출했기 때문에 B가 끝날 때까지 아무것도 하지 못 한다.

<img width="667" alt="image 22" src="https://github.com/user-attachments/assets/6a9516bd-dd69-4bc8-8fe9-d8d5ab4caf61" />

따라서 하나의 스레드에서 A가 동작하다가 B 작업을 하고, B가 끝나면 A 작업으로 다시 돌아온다.

이게 sync의 스레드 점유권의 흐름이다.

### async 에서의 스레드 제어권

개요에서 말했듯 A 작업을 하다 B라는 비동기 메소드를 호출하면 A는 스레드 제어권을 B에게 넘겨준다.

왜냐하면 A 작업은 어차피 B를 호출한 시점부터 그 아래 코드들은 B가 끝날 때까지 아무것도 못 하기 때문이다.

<img width="665" alt="image 23" src="https://github.com/user-attachments/assets/3e135760-8290-4c9f-91a3-79f3b49103c3" />

그러면 Suspension Point를 만난 순간부터 스레드 제어권을 포기하면,

해당 **스레드에 대한 제어권은 시스템에게 가고** 시스템은 스레드를 사용해서 다른 작업을 수행할 수 있게 된다.

우선순위에 따라 여러 작업을 멀티 스레드로 처리할 것이다.

그러다 **멈췄던 내 작업이** 가장 중요하다고 판단되는 순간에 **해당 함수를 재개(resume)**하고, 비동기 함수는 할당받은 스레드를 **다시 제어하며 작업**할 수 있게 된다.

1. await로 async 함수를 호출하는 순간(= Suspension Point) 해당 스레드 제어권 포기
2. async 작업 및 같은 블록 아래의 코드들은 실행 불가능
3. 스레드 제어권을 시스템에게 넘기면서, 1번의 호출된 async도 스케줄 대상이 됨
4. 시스템은 작업 우선순위를 따지며 작업들을 처리하고,
이때 1번이 실행되던 스레드에서 다른 작업을 먼저 실행할 수도 있음
5. 그러다 1번의 호출된 async 작업이 중요해지는 순간(= 내 차례)
다시 작업하라고 resume을 하고, 이 때 특정 스레드의 제어권을 줘서 마저 실행이 된다.
    
    `중요한 건 이때 Resume되는 스레드는 1번 스레드와 다를 수 있음`
    

await한다고 **무조건** Suspension Point가 되는 건 아니지만,

위처럼 await 키워드를 통해 코드 블럭이 하나의 트랜잭션으로 처리되지 않을 수 있음

# 스택 프레임의 변화

## sync 방식의 Stack Frame

모든 스레드는 함수 호출을 위한 자신만의 독립된 스택 영역을 갖는다.

<img width="669" alt="image 24" src="https://github.com/user-attachments/assets/56cf48ad-b25f-4cb9-a3f4-4ae5ab56803f" />

스레드가 함수 호출을 실행하면 새 프레임이 스택에 푸쉬,

해당 스택 프레임은 스택의 Top에 쌓이고 이에는 로컬 변수, 리턴 주소값 등이 포함되어 있다.

쌓인 스택 프레임은 함수가 끝나면 Pop 되어 사라진다.

## async 방식의 Stack Frame

<img width="656" alt="image 25" src="https://github.com/user-attachments/assets/1299619e-58cb-40f2-842f-910202c9752c" />

1. 비동기 메소드인 `updateDatabase`를 호출
2. `updateDatabase` 내에서 비동기 메소드인 `add` 호출
3. add 내에서 비동기 메소드인 `database.save` 호출

Flow는 위와 같다.

### add가 호출된 상황

먼저 2번, add가 호출된 상황부터 보면

스택 메모리 관점에서는 add 메소드가 호출됐으니 add 메소드에 대한 스택 프레임을 스택 영역에 적재한다.

`중요`

이때, add 스택 프레임에는 사용할 필요가 없는 Local 변수를 저장한다.

무슨 뜻이냐면, suspension point 때문에 사용되지 않을 (= await 아래) 지역 변수를 스택 프레임 저장한다는 것이다.

그럼 위 사진과 같이 `(id, article)`이 스택 프레임에 담기게 될 것이다.

왜 이렇게 하냐면, **await 전/후로 모두 사용되는 정보를 저장하기 위한 공간이 필요**하다.

그럼 await 전에 존재하는 지역변수(= 파라미터) `newArticle`은 따로 저장 공간이 필요할 것이다.

**Suspension Point를 만나서 다른 스레드로 작업을 이어갈 때,** 이 전 내용들을 기억하기 위해 **Heap 메모리 영역에 저장**한다.

<img width="676" alt="image 26" src="https://github.com/user-attachments/assets/95e42429-b473-4e83-9d48-1ab3373b866a" />

위와 같이 말이다.

### await database.save가 호출된 경우

이어서 3번, add 함수에서 await database.save를 호출한 경우 이 곳이 Suspension Point가 된다.

그러면 스택 영역 제일 위에 있던 add 스택 프레임의 변화를 보자

이론 상, A 메소드가 호출되다가 B 메소드를 호출하면 A 스택 프레임 위에 B가 쌓이게 된다. 그러면서 B 메소드가 동작이 끝나면 다시 A로 돌아와서 기존 작업을 한다.

그러나, 비동기 메소드에서 비동기 메소드의 경우 add 스택 프레임 위에 쌓이는 것이 아니라, **add 스택 프레임이 save 스택 프레임으로 대체**된다.

**중요**

이렇게 동작하는 이유는 **await 전후로 사용될 코드가 Heap 영역의 async frame에 저장**되어 있기 때문에 스택에 필요하지 않는 것이다.

그리고 **스택에 있어봤자 스레드 점유권을 다시 얻을 때 해당 스레드로 돌아온다는 보장이 없기 때문**이다.

Suspension Point에서 모든 정보가 Heap에 저장되니, 다시 점유권을 얻었을 때 작업 수행이 가능한 것이다.

이 async 프레임 목록은 Continuation에 대한 런타임의 표현이다.

<img width="677" alt="image 27" src="https://github.com/user-attachments/assets/5c93d5bd-6b9b-4303-89a3-99ee26955f9e" />

따라서 3번이 끝나면, 위 사진처럼 save 스택 프레임이 add 스택 프레임을 대체하게 된다. 

### save 메소드 동작 중 await

<img width="671" alt="image 28" src="https://github.com/user-attachments/assets/838132e9-6c84-4cd0-a6d9-5e1390b7d42a" />

save 메소드 내부에서 만약 await로 비동기 메소드를 호출해서 Suspend가 되었다고 가정하겠다.

그러면 해당 스레드는 스레드 점유권을 내주고 되고 해당 스레드는 다른 작업을 수행할 수 있게 된다.

### save 메소드 종료 후 Return 과정

save 메소드가 동작할 차례가 되어 continuation에 의해 Heap에 있던 save 비동기 프레임이 스택에 쌓이게 되고,

save 메소드 수행 후 작업을 마치면 [ID]를 반환한다.

<img width="680" alt="image 29" src="https://github.com/user-attachments/assets/7d39f093-5f73-44f2-b606-6488c5d21fa0" />

save 메소드의 동작이 끝나면 [ID]를 반환하고, save를 위한 스택 프레임은 add 메소드를 위한 스택 프레임으로 대체된다.

# async let

> `동시 바인딩`을 지원하고자 나온 Task의 한 종류
> 

```swift
func fetchOneThumbnail(withID id: String) async throws -> UIImage {
    let imageReq = imageRequest(for: id), metadataReq = metadataRequest(for: id)
    async let (data, _) = URLSession.shared.data(for: imageReq)
    async let (metadata, _) = URLSession.shared.data(for: metadataReq)
    guard let size = parseSize(from: try await metadata),
          let image = try await UIImage(data: data)?.byPreparingThumbnail(ofSize: size)
    else {
        throw ThumbnailFailedError()
    }
    return image
```

위의 코드처럼 async let 을 쓰면 해당 변수를 사용할 때 await으로 기다린 후에 사용할 수 있다.

왜냐하면 언제 작업이 끝날지 모르기 때문임!

<img width="1684" alt="%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-01-24_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_8 47 07" src="https://github.com/user-attachments/assets/6044ab34-5272-4793-9fbf-aecc52563f5a" />

이 경우 Swift는 자식 작업을 생성한 후 빈 값을 변수에 넣고 계속 진행시킨다.

그 후 해당 변수가 필요해질 때 실제로 자식 작업을 기다리게 된다.

- 사실 우리는 변수를 r-value로 사용할 때 해당 변수의 get 함수를 실행한다.
- 그런데 await하고 r-value를 사용한다는 의미는 무엇일까?
- 즉, get의 async 버전이 존재한다는 것이다.

```swift
class A {
    var a: Int {
        get async {
            return 1 // 대충 오래걸리는 작업
        }
    }
}
```

### async-let 예제

<img width="432" alt="image 30" src="https://github.com/user-attachments/assets/90d5e428-c13e-4da9-ad50-cb0fed73a213" />

두 가지 다른 URL로부터 데이터를 다운로드하는 예제가 있다.

현재 코드는 순차적 바인딩이다.

하나는 이미지를 받는 거고, 하나는 이미지에 대한 메타 데이터용.

이러면 imageReq를 통해 이미지를 받아올 때까지 기다리고,

그 후에 metadata를 받아올 때까지 기다려서 이미지를 만들고 반환을 하게 된다.

그리고 오류 가능성이 있기 때문에 `try await` 을 사용해서 호출해야 한다.

async-let 도입

<img width="501" alt="image 31" src="https://github.com/user-attachments/assets/80b921b2-2a64-4811-8b6b-c25a79e5f2a1" />

두 다운로드가 동시에 이루어질 수 있게 async-let을 사용하여 동시 바인딩을 한다.

이러면 Child Task에서 작업이 발생하기 때문에 try await을 사용하지 않아도 됨

<img width="497" alt="image 32" src="https://github.com/user-attachments/assets/871226b1-b57a-4132-aa00-b23370af42cd" />

이제 아래 블록들에서 data와 metadata 변수를 사용하기 전에 try await을 한다.

<img width="1019" alt="%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-01-24_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_8 57 52" src="https://github.com/user-attachments/assets/b73e663e-ecae-42b7-9ffc-93870c44ce18" />

한 비동기 메소드에서 다른 비동기를 호출할 때마다 동일한 Task를 사용해서 호출한다.

fetch~ 메소드에서 두 개의 **async-let으로 두 개의 자식 Task를 만든 것**.

이러니까 트리 구조가 되는 거고, 상위에서는 await를 할 필요 없이 작업이 완료된 경우에 만 부모가 작업을 완료할 수 있다고 말함

# Task

위에서 만든 async 함수들을 어떻게 사용할까?

그냥 맨땅에 `await function()`하면 컴파일 에러가 발생한다.

비동기 함수를 호출하려면 비동기 맥락에서 사용해야 한다.

왜냐하면 await자체가 관리 권한을 포기할 수 있다는 것인데 main스레드처럼 sync환경에 실행하면 해당 실행이 정지할 수 있기 때문이다.

이를 위해 Task라는 특별한 구조체가 필요하다.

```swift
Task {
 // 비동기 코드 실행
}

```

Task를 생성하면 클로저로 전달된 작업이 바로 실행된다. (비동기로 실행됨!)

이 때 Task는 우선순위를 부여하여 생성할 수 있고 취소도 가능하다.

```swift
let task = Task(priority: .background) {
    <#code#>
}
task.cancel()

```

이렇게 취소가 가능하다.

- task 변수에 할당하지 않아도 비동기 프로그램은 정상적으로 실행된다. 다만, cancel등의 관리를 할 수 없다는 단점이 생긴다. ⇒ 취사선택

Task의 다른 특징 중 하나는 주변 환경을 캡쳐한다는 것이다.

이는 actor등과 같은 격리된 환경과 자신을 호출한 자료형에 대한 참조도 포함한다.

- 그러나 Task에서는 그 실행이 종료되면 바로 self에 대한 레퍼런스를 내려놓는다.
- 따라서 weak self로 캡쳐해야 하는 경우가 거의 없다.

만약 주변환경을 캡쳐하기 싫다면 → `Task.detached`를 사용하자.

# Structured Concurrency

Task하나만으로 비동기 코드를 실행할 수 있다니! 참 좋은데 말입니다…

그런데 Task 안에서도 다른 Task를 만들 수 있지 않을까?

비동기 작업 하나에 대해 오래 걸리는 작업을 또 분리하고 싶은 요구가 있을 수 있다.

```swift
let task = Task {
    Task {

    }
}

```

요런식으로 말이다.

그런데 이렇게 하면 문제가 있다.

- 첫번째 task의 경우 task외부에서 관리할 수 있다. 그러나 중첩된 Task는 외부에서 관리하기 힘들다.
- 그리고, 이렇게 생성된 Task는 자신을 생성한 Task와는 별도로 동작한다.

# Task Group

새로운 Task가 기존 Task에 종속된 관계를 갖게 할 수 없을까?

그래서 Task Group이 필요하다.

```swift
Task {
    let arr: [Int] = await withTaskGroup(of: Int.self) { group in
        var arr = [Int]()
        group.addTask {
            1
        }
        group.addTask {
            1
        }
        for await int in group {
            arr.append(int)
        }
//-----
//      while let int = await group.next() {
//
//      }
//-----
//      var it = group.makeAsyncIterator()
//      while let int = await it.next() {
//
//      }
        return arr
    }
}

```

TaskGroup은 비동기 함수라서 비동기 맥락 안에서 실행되어야 한다.

또한 2가지 정보가 필요하다. (자식 작업의 return 타입, 그룹 작업의 return 타입)

그룹 작업의 return 타입의 경우 대개 타입추론으로 해결해주는데 필요한 경우 직접 적어야 한다.

이렇게 하면 비슷한 작업에 대해 자식 작업을 만들어서 동시에 여러 자료를 취합할 수 있게 된다.

이 방식을 `구조적 동시성`이라고 한다.

- 계층적으로 부모 - 자식 관계를 형성하고, 부모는 자식작업이 끝날때까지 기다린다.
- 작업의 우선순위 = max(부모, 자식)

구조적 동시성 종류는 다음과 같다.

- Task Group
- async-let

TaskGroup에는 Throwing할 수 있는 ThrowingTaskGroup이 별도로 있다.

| **특성** | **TaskGroup** | **ThrowingTaskGroup** |
| --- | --- | --- |
| **정의** | 일반 작업 그룹으로, 작업이 성공적으로 완료되면 결과를 반환. | 예외를 던질 수 있는 작업 그룹으로, 작업 도중 에러를 발생시킬 수 있음. |
| **결과 타입** | **Non-throwing Result** (`T`) | **Throwing Result** (`T`) |
| **작업 실패 시 처리** | 작업 실패가 발생하지 않음. | 작업 중 하나라도 에러가 발생하면 그룹 전체가 중단됨. |
| **에러 처리 필요 여부** | 에러 처리가 필요 없음. | 에러 처리(`try`, `catch`) 필요. |
| **사용 예** | - 독립적인 작업 처리.- 작업 실패 가능성이 없는 경우. | - 네트워크 요청, 파일 처리 등 에러 발생 가능성이 있는 작업. |
| **`addTask` 메서드 사용 가능 여부** | 가능 | 가능 |
| **`await` 사용 시** | 단순히 결과를 기다림. | 결과와 함께 에러를 처리해야 함. |
| **에러 전파** | 없음. | 에러가 발생하면 호출자에게 전파. |

자식작업을 많이 만들어서 일을 더 잘게 분해하는 게 좋은 것같지만 꼭 그렇지는 않다.

Task는 자식작업이 모두 완료되는 것을 기다리기 때문에 해당 부모작업의 종료시까지 자식작업들의 메모리를 들고 있게 된다.

그 결과를 사용하기 위해서라면 필요하지만 경우에 따라 자식 작업의 결과가 중요하지 않을 수도 있다.

이 경우 `with(Throwing)DiscardingTaskGroup()`을 사용하면 된다.

이것을 사용하면 자식 작업은 반환되자마자 메모리에서 해제된다.

# AsyncSequence/Stream

위의 코드에서 `for await int in group`를 보았을 것이다.

이것은 이번에 새롭게 추가된 `for-await-in` 문법이다.

이것은 AsyncSequence를 다루기 위해 등장한 신문법이다.

기존 Sequence와 유사하며 여기에 비동기 특성을 부여한 프로토콜이다.

- Sequence가 제공하던 고차함수들 대부분 사용 가능

내가 가진 자료형이 AsyncSequence를 채택하고, next()와 makeAsyncIterator()를 구현하면 사용할 수 있다.

AsyncStream의 경우 기존 콜백함수나 delegate함수를 async하게 사용할 수 있도록 도와준다.

```swift
class QuakeMonitor {
    var quakeHandler: (Quake) -> Void
    func startMonitoring()
    func stopMonitoring()
}

let quakes = AsyncStream(Quake.self) { continuation in
    let monitor = QuakeMonitor()
    monitor.quakeHandler = { quake in
        continuation.yield(quake) // continuation 인스턴스를 통해서 소통함
    }
    continuation.onTermination = { @Sendable _ in
        monitor.stopMonitoring()
    }
    monitor.startMonitoring()
}

let significantQuakes = quakes.filter { quake in
    quake.magnitude > 3
}

for await quake in significantQuakes {
    ...
}

```

continuation 인스턴스를 통해 yield메서드로 값을 반환하기만 하면 사용 가능하다.

# 협력적 취소

<img width="1218" alt="image 33" src="https://github.com/user-attachments/assets/8a3fe6fe-0473-4f3d-925d-2b1f087a9d79" />

취소면 취소지 뭔 협력적취소…?

말그대로 취소에 “협력”한다는 것이다.

우리가 Task에 대해 취소 명령을 날리면 그 즉시 취소(함수 return)되는 것이 **아니다**.

다만, Task는 최대한 빨리 취소가 될 수 있도록 “**협력**”하는 것이다.

이렇게 하는 이유는 Task와 그 하위 작업들에게 취소가 되었을 경우 어떻게 할 것인지 여지를 주기 위해서다.

만약 바로 함수를 종료시키면 취소가 되었을 경우 어떻게 해야하는지를 가이드할 수 없다.

그러나 Task가 종료되었을 경우에 어떤 행동을 취할지 개발자에게 여지를 줌으로써 더 유연한 개발이 가능하다.

```swift
func fetchThumbnails(for ids: [String]) async throws -> [String: UIImage] {
    var thumbnails: [String: UIImage] = [:]
    for id in ids {
        try Task.checkCancellation()
        // if Task.isCancelled { break }
        thumbnails[id] = try await fetchOneThumbnail(withID: id)
    }
    return thumbnails
}

```

비동기 함수를 작성할 때 Task의 타입 메서드로 `checkCancellation()`과 타입계산속성 `isCancelled`를 사용할 수 있다.

- `checkCancellation()`: 현 작업이 취소되었는지 확인한 후 취소일 경우 에러를 던짐
- `isCancelled`: 현 작업이 취소되었으면 true, 아니면 false 반환

이를 통해 보통 긴 작업을 시작하기 전에 적절히 작업 취소 여부를 확인하여 코드를 작성하면 된다.

이러한 방식을 하나도 구현 안하면 작업이 취소되어도 자신의 작업을 계속한다.

그러기 때문에 협력적 취소라고 부르는 것이기도 하다. (비협조적이면 취소가 안된다…)

위의 Task 타입메서드/속성을 사용하면 현재 자신의 Task를 자동으로 추적한다.

> 협력적 취소는 구조적 동시성에서 유용하다!
> 
- 자식 작업이 오류등을 날리면 다른 작업들도 멈추게 된다.

# Continuation

이렇게 몸에도 좋고 맛도 좋은 SwiftConcurrency지만…

기존코드에 당장 적용하기에는 너무 부담이 되는 것도 사실이다.

이를 간편하게 해결해주고자 짜잔~ continuation이 있답니다…

Continuation에는 Checked와 Unsafe 2개가 있다.

CheckedContinuation의 설명은 다음과 같다.

- 동기 코드(synchronous)와 비동기(asynchronous) 코드 사이의 인터페이스 제공
    - 비동기 상황(어떤 것이 먼저 실행될지 모름, thread를 누가 차지할지 모름)에서도 순서대로 실행될 수 있도록 heap에서 관리함
- 정확성 위반(correctness violations) 기록
    - Continuation을 사용할 때, resume은 반드시 1번 불려야한다. (무조건 한번)
    - UnsafeContinuation은 이것을 안한다. (나머지 기능은 같음)

```swift
// 원래 함수
func getPersistentPosts(completion: @escaping ([Post], Error?) -> Void) {
    do {
        let req = Post.fetchRequest()
        req.sortDescriptors = [NSSortDescriptor(key: "date", ascending: true)]
        let asyncRequest = NSAsynchronousFetchRequest<Post>(fetchRequest: req) { result in
            completion(result.finalResult ?? [], nil)
        }
        try self.managedObjectContext.execute(asyncRequest)
    } catch {
        completion([], error)
    }
}

// Async스타일로 변경한 함수
func persistentPosts() async throws -> [Post] {
    typealias PostContinuation = CheckedContinuation<[Post], Error>
    return try await withCheckedThrowingContinuation { (continuation: PostContinuation) in
        self.getPersistentPosts { posts, error in
            if let error = error {
                continuation.resume(throwing: error)
            } else {
                continuation.resume(returning: posts)
            }
        }
    }
}

```

위의 코드처럼 바꿔줄 수 있다.

resume을 해야 기존에 await를 했던 부분을 다시 진행한다.

# Swift Concurrency

이렇게 async-await의 등장부터 이를 사용할 수 있는 환경을 제공하는 Task, TaskGroup과 반복을 위한 AsyncSequence, 그리고 Continuation을 통한 기존 API 통합을 살펴보았다.

아직 다루지 못한 Sendable, Actor 등은 차치하고, Swift Concurrency가 달성하고 싶었던 목표 중 하나인 가독성을 살펴보았다.

그렇다면 드는 의문이 *효율면에서 다른점은 없나?*이다.

Apple은 그렇다고 한다. 그러면 어떻게 달성되었을까?

우리가 여태 하는 이야기가 비동기였다. 이는 Thread와 관련이 깊다.

Thread의 효율적인 사용이 문제인데, 이는 이전 CS 배울 때 다루었다.

⇒ “프로그램이 잘 실행되도록 하는 것이 목표”

즉, context-switching을 줄이는 것이 Thread에게 있어서의 과제가 될 것이다.

이는 어떻게 반영되었을까?

<img width="1597" alt="image 34" src="https://github.com/user-attachments/assets/49c13619-4165-4800-a853-c2d3ad5db457" />

기존 GCD환경에서는 위에서처럼 여러 Thread가 CPU자원을 번갈아 가면서 사용되었다.

이럴 때 Thread가 많아지면 많아질수록 스케줄링이나 lock과 관련해서 대기 시간이 길어질 수 있었다.

<img width="1597" alt="image 35" src="https://github.com/user-attachments/assets/cea7b50c-08b8-4e25-830d-d56c9edecb74" />

하지만, SwiftConcurrency에서는 CPU당 Thread를 하나씩 할당한다.

그리고 Continuation이라는 객체를 통해서 각각의 실행 맥락을 보존한다.

이를 통해 우리가 지불해야하는 비용은 함수 실행 비용 밖에 없다.

실제 비동기 함수를 실행할 때를 살펴보자.

<img width="1355" alt="image 36" src="https://github.com/user-attachments/assets/bedeb6ce-22de-4a93-a561-6db8fc4e5b0d" />

비동기 함수를 실행할 때 우리는 await을 붙이고 호출한다.

이때 시스템에게 제어권을 넘기게 되고 함수의 실행이 끝나면 원래 함수를 호출한 쪽으로 돌아와서 resume(계속진행)한다.

좀 더 자세히 보자.

<img width="1506" alt="image 37" src="https://github.com/user-attachments/assets/779c79d2-70c3-4855-be71-30dcdfa8d784" />

Stack과 Heap이 나온다.

우리는 함수를 호출하면 변수 등 관련정보를 모아서 Stack에 저장한다는 것을 알고있다.

이를 Continuation이라는 객체에 담아서 Heap에 보관했다고 생각하면 된다.

그리고 Thread는 현재 Stack만 관리하는 것이다.

Heap에 Continuation으로 저장하면 Stack에서 제거한다.

왜 Heap에 저장하냐? → 여러 스레드에서 공유되기 때문! (어떤 스레드가 실행할지 모르니까)

> 우선순위 역전
> 

기존 DispatchQueue의 경우 서로 다른 낮은우선순위와 높은 우선순위가 있을때 높은 우선순위에 우선순위를 일치시켰다.

왜냐하면 Queue (선입선출)이기 때문!

<img width="1597" alt="image 38" src="https://github.com/user-attachments/assets/ebe117e7-1cb7-4ad0-b77f-9069f83854ce" />

하지만 SwiftConcurrency에서는 Heap에 보관된 Continuation에서 취사선택하면 되므로 우선순위가 높은 것을 먼저 실행 가능하다!

<img width="1597" alt="image 39" src="https://github.com/user-attachments/assets/ef9af320-54a0-4492-9257-dd289da4d651" />

요롷게!

# 참고자료

https://sujinnaljin.medium.com/swift-async-await-concurrency-bd7bcf34e26f

https://engineering.linecorp.com/ko/blog/about-swift-concurrency

https://developer.apple.com/videos/play/wwdc2021/10254/?source=post_page-----bd7bcf34e26f--------------------------------

https://developer.apple.com/videos/play/wwdc2021/10134?time=243&source=post_page-----bd7bcf34e26f--------------------------------

https://developer.apple.com/videos/play/wwdc2022/110351
