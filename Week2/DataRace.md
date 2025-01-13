# 목차
- [경쟁 상태 (Race Condition)](#경쟁-상태-race-condition)
    - [상호 배제 (Mutual Exclusion)](#상호-배제-mutual-exclusion)
        - [Busy Waiting](#busy-waiting)
        - [Sleep and Wakeup](#sleep-and-wakeup)
        - [뮤텍스(Mutex)](#뮤텍스-mutex)
        - [세마포어(Semaphore)](#세마포어-semaphores)
        - [뮤텍스 vs 세마포어](#뮤텍스-vs-세마포어)
    - [교착상태 (Dead Lock)](#교착상태-dead-lock)
        - [교착상태 조건](#교착상태-조건)
        - [교착상태 해결책](#교착상태-해결책)
- [Data Race](#data-race)
    - [Data Race가 위험한 이유?](#data-race가-위험한-이유)
    - [iOS에서 Data Race 해결책](#ios에서-data-race-해결책)
- [QnA](#qna)

# 경쟁 상태 (Race Condition)

**경쟁 상태(Race Condition)** 란 두 개 이상의 흐름에서 어떤 기능의 실행되는 시점, 순서에 따라 공유자원의 상태(결과)가 달라지는 상태를 의미한다.

예를 들어 아래와 같은 코드가 있다고 하자.

```swift
bool a = NULL;
Thread1.do { a = true; }
Thread2.do { a = false; }
```

위 코드에서 Thread1과 Thread2는 거의 동시에 동작한다. 

이를 계속해서 실행하면 a의 상태가 언제는 true, 언제는 false로 달라지는 것을 확인할 수 있다. 

이것은 프로그램을 사용하는 입장에서 예측 가능한 결과를 보장해주지 않는다.

이렇게 Race Condition이 발생하는 코드를 **임계 구역 (Criticial Section)**이라고 한다.

이러한 임계 구역이 나타나는 이유는 코드를 실행할 때 원자적으로 수행되지 않고 실제로는 아래와 같이 3단계의 명령어로 나뉘어 실행되기 때문이다.

<img width=300 src="https://github.com/user-attachments/assets/6ad4a482-1305-40d1-b5e6-ed41f7b2685a">

## 상호 배제 (Mutual Exclusion)

위에서 언급한 임계 구역을 해결하기 위해서는 어떻게 해야할까?

간단하게 생각하면 먼저 접근하는 쪽에서 접근 중이라는 표시를 해주고 뒤에 들어온 작업은 먼저 들어온 작업이 끝날 때까지 기다리면 된다.

이렇게 하나의 스레드 혹은 프로세스만 임계 구역을 실행하도록 보장하는 기법을 **상호 배제(Mutual Exclusion)** 라고한다.

상호 배제는 아래 규칙을 지켜야 성공적인 솔루션을 제공한다고 볼 수 있다.

1. 임계 구역에 하나의 프로세스 혹은 스레드만 진입해야 한다.
2. 프로세스 혹은 스레드는 진입을 위해 영원히 대기해서는 안된다. → 교착상태 (DeadLock)
3. 임계 구역 외부의 프로세스 혹은 스레드가 다른 프로세스 혹은 스레드를 막아서는 안된다.
4. CPU의 성능과는 독립적이어야 한다.

그럼 상호 배제 기법들을 알아보자.

### Busy Waiting

> 임계 구역을 다른 프로세스 혹은 스레드가 사용 중인지 여부를 while문을 통해 지속적으로 확인하는 방법
> 

확실하긴 하지만, 할당받은 CPU 사용시간을 낭비한다는 단점이 있다. (지양해야댐..)

Busy Waiting의 알고리즘은 아래 3가지가 존재한다.

1. Strict Alternation
    
    <img width=500 src="https://github.com/user-attachments/assets/1a8905f0-e095-43f8-a1bb-4c4b757ae734">
    
    해당 알고리즘은 임계 구역에 2개의 스레드가 들어갈 수 있다. 또한, 임계 구역 외부 while문에서 block되고 있으므로 3번 규칙을 어기게된다.
    
2. Peterson’s solution
    
    <img width=500 src="https://github.com/user-attachments/assets/bfcbe24d-5993-42fd-aa8c-4fc0d985a7bc">
    
    다른 프로세스가 임계구역에 접근하는지, 누구의 차례인지를 검사하여 임계 구역에 들여보내는 방식이다.
    
3. TSL instruction
    
    <img width=500 src="https://github.com/user-attachments/assets/50bb6aa5-c5fe-4424-8cd0-847223ed4244">
    
    해당 방식은 TSL 레지스터, LOCK(Test and Set Lock)이라는 어셈블리 명령어를 사용한다.
    
    프로그램 수준이 아닌 어셈블리 수준에서 상호배제를 보장하는 것이다.
    
    락을 설정하는 데에 있어 인터럽트를 할 수 없게 한다는 장점이 있다.
    

이렇게 많은 알고리즘들이 있지만 결국 busy waiting의 한계(기다리는 시간 낭비)를 벗어날 순 없다.

### Sleep and Wakeup

Sleep and WakeUp은 System call을 통해서 sleep을 호출한 프로세스 혹은 스레드를 재우고, 다른 프로세스가 wakeup을 해주는 구조이다.

**생산자 - 소비자(Producer - Consumer)문제**는 Sleep and Wakeup과 함께 알아두면 좋은 문제이다. 우리가 데이터를 교환하는 것을 보여 누구는 데이터를 만들고 누구는 그 데이터를 사용한다는 점을 알 수 있다. 이를 생산자(Producer)와 소비자(Consumer)에 빗대어 표현한 것이다.

데이터를 무한히 버퍼에 저장할 순 없다. 즉, 언젠가는 생산/소비를 멈춰야한다.

여기서 Sleep and Wakeup 개념이 사용된다.

생산자는 버퍼가 꽉차면 sleep 후 소비자를 깨우고, 소비자는 버퍼가 비면 sleep 후 생산자를 깨우는 방식이다.

하지만 이 방식의 경우 잘못하면 모두가 자버릴 수 있다.

<img width=500 src="https://github.com/user-attachments/assets/fb0cb704-b046-4d00-b112-69225c51c2f8">

위 코드에서 count가 N일 때 생산자가 if (count == N)분기까지 하고 interrupt되고 소비자가 wakeup 코드를 실행해버리면 생산자는 영원한 잠에 빠진다.

이처럼 다른 프로세스 혹은 스레드가 영원히 잠에 빠질 수 있는 것이다.

### 뮤텍스 (Mutex)

프로세스나 스레드가 공유자원을 `Lock()` 으로 잠금 설정하고 Lock 소유권을 얻어 임계 구역에서 작업하는 방법을 말한다.

<img width=400 src="https://github.com/user-attachments/assets/0c921ae5-222a-4e86-a689-c266ae358607">

작업이 끝난 후 해당 프로세스 혹은 스레드가 `Unlock()`을 통해 잠금을 해제하고 다른 프로세스 혹은 스레드가 접근할 수 있게 된다.

잠금이 설정되어있는 동안은 공유자원에 다른 프로세스나 스레드는 접근할 수 없다.

Lock을 기다리는 프로세스 혹은 스레드들은 Busy Waiting에 빠지게 되는데, 이 때 Sleep을 하여 대기 큐에 넣고, 사용 가능할 때 프로세스 혹은 스레드를 깨운다.

### 세마포어 (Semaphores)

일반화된 뮤텍스로, 간단한 정수 값과 `wait`, `signal`로 공유 자원에 대한 접근을 처리한다.

<img width=400 src="https://github.com/user-attachments/assets/a206ad8e-c045-4203-890a-1f6cc7ef912a">

프로세스나 스레드가 공유 자원에 접근하면 세마포어에서 `wait()` , 접근이 끝나면 `signal()` 을 수행한다.

세마포어에는 조건 변수가 없고, 세마포어 값 수정 시 다른 프로세스는 동시 수정이 불가능하다.

### 뮤텍스 vs 세마포어

뮤텍스는 세마포어에서 접근 가능 개수를 2개로 설정한 것과 비슷해 보이는데 어떤 차이가 있을까?

가장 중요한 차이점은 **공유 자원의 Lock에 대한 소유권**이다.

일반적인 세마포어의 경우 Lock에 대한 소유권이 없다. 따라서 다른 프로세스 혹은 스레드가 signal 등을 통해 변수를 조작하여 임계 구역에 대한 접근 권한을 획득할 수 있다는 잠재적인 위험요소가 존재한다.

하지만 뮤텍스의 경우 해당 임계 구역에 진입한 프로세스 혹은 스레드가 Lock에 대한 소유권을 갖는다. 따라서 Unlock을 하는 것도 진입한 프로세스 혹은 스레드만이 가능하다.

## 교착상태 (Dead Lock)

**교착상태 (Dead Lock)** 은 두 개 이상의 프로세스 혹은 스레드가 동시에 공유자원에 접근하려다 잠겨서 서로 아무것도 하지 않는 상태를 의미한다.

<img width=300 src="https://github.com/user-attachments/assets/24090f9a-ddeb-4726-9ac5-b37ef032232c">

### 교착상태 조건

1. 상호 배제 (Mutual Exclusion)
2. 점유 상태로 대기 (Hold and wait): 임계 구역을 점유한 채로 대기
3. 선점 불가 (No preemption): 다른 프로세스 혹은 스레드에 CPU 양보 안함
4. 순환성 대기 (Circular wait): 서로가 서로를 기다림

### 교착상태 해결책

1. 예방 (Prevention)
    
    교착상태 조건 중 하나 이상을 부정하면 된다. (확실하지만 자원 소모 심함;;)
    
    - 상호 배제 부정 : Race Condition을 허용하게 된다.
    - 점유 및 대기 부정
        - 모든 자원을 미리 할당한다 → 자원 낭비
        - 자원 점유를 안 할 때에만 요청을 수락한다 → 기아 상태
    - 비선점 부정
        - 카메라, 키보드 등 I/O 자원을 공유하는 것도 문제가 될 수 있다.
    - 순환선 대기: 번호 달고 대기 → 자원낭비
2. 회피 (Avoidance)
    
    운영체제가 프로세스 혹은 스레드가 교착상태에 빠질 가능성을 계산하는 것이다.
    
    주로 은행원 알고리즘이 이곳에 해당한다.
    
    > 은행원 알고리즘: 총 자원의 양과 현재 할당한 자원의 양을 기준으로 안정, 불안정 상태로 나누고 안정 상태로 가도록 자원을 할당하는 것
    > 
3. 탐지 (Detection)
    
    현실적으로 모든 상황을 회피할 순 없으니 어디서 교착상태가 발생했는지 알아내는 작업이다.
    
4. 복구 (Recovery)
    
    탐지를 하고 나서 교착상태를 해결하려면 프로세스 및 스레드를 종료하거나 자원을 회수해야한다.
    
    그런데 교착상태가 발생한 프로세스가 운영체제라면? 우짬요? 응 재부팅해야돼 ~ ㅋㅋ
    

# Data Race

휴 ~ 드디어 이번 주차의 주제가 나왔다.

Data Race는 동기화 기법이 적용되지 않은 상황에서 공유 데이터에 읽기와 쓰기가 동시에 발생하는 경우를 의미한다.

위에서 나온 Race Condition과 많이 헷갈려하는데 보통 Race Condition이 조금 더 넓은 범위로 사용된다.

## Data Race가 위험한 이유?

1. 디버깅이 어렵다
    - 테스트 환경에서는 문제가 드러나지 않다가, 실제 환경에서만 발생할 수 있다.
    - 항상 동일한 결과를 재현하지 어렵다.
2. 비결정성
    - 같은 코드를 실행해도 결과가 달라져 안정적인 시스템 설계가 어렵다.
3. 데이터 손상
    - 데이터 구조나 상태가 손상되면 프로그램 전체에 예기치 않은 동작을 초래할 수 있다.
4. 보안 취약점
    - 공격자가 Data Race를 악용해 메모리 상태를 의도적으로 조작 및 탈취할 수 있다.

## iOS에서 Data Race 해결책

Data Race를 만들지 않기 위해 예방하는 방법에는 크게 4가지 정도가 존재한다.

- NSLock
    - `NSLock` 이라는 클래스의 객체를 만들어두고 `.lock()`, `.unlock()` 메서드를 통해 변수를 건드리는 작업을 할 때마다 락을 걸었다 풀었다는 해주는 방법
    - `NSLock`을 사용할 때 주의할 사항은 Lock을 걸었던 스레드에서 Lock을 풀어줘야하는 것이다.
    - 예시 코드
        
        ```swift
        final class SynchronizedInteger {
            private var _value: Int
            private let lock = NSLock()
            
            var value: Int {
                get {
                    lock.lock()
                    let value = _value
                    lock.unlock()
                    return value
                }
                set {
                    lock.lock()
                    _value = newValue
                    lock.unlock()
                }
            }
            
            init(_ value: Int) {
                self._value = value
            }
            
            func increment() {
                lock.lock()
                _value += 1
                lock.unlock()
            }
        }
        ```
        
- DispatchQueue barrier
    - DispatchQueue에 작업을 넣을 때 flag를 지정함으로써 배리어 블럭처럼 작동하게 하는 것이다.
    - 배리어 블럭이 DispatchQueue에 추가되면 큐는 배리어 블럭의 실행을 배리어 블럭 이전에 들어간 모든 작업이 끝날 때까지 미루게 된다.
    - 예시 코드
        
        ```swift
        final class BarrierInteger {
            private var _value: Int
            private let queue = DispatchQueue(label: "BarrierInteger", attributes: .concurrent)
            
            var value: Int {
                get {
                    var value: Int!
                    queue.sync {
                        value = _value
                    }
                    return value
                }
                set {
                    queue.async(flags: .barrier) {
                        self._value = newValue
                    }
                }
            }
            
            init(_ value: Int) {
                self._value = value
            }
            
            func increment() {
                queue.async(flags: .barrier) {
                    self._value += 1
                }
            }
        }
        ```
        
- DispatchSemaphore
    - GCD의 일부로 Semaphore가 구현되어있는데 Binary Semaphore(동시에 접근할 수 있는 스레드의 개수가 1개인 세마포어)로 사용할 수 있다.
    - `NSLock`과 비슷하지만 세마포어의 특성상 `wait()` 메서드를 통해 락을 건 스레드가 아닌 다른 스레드에서도 `signal()` 메서드를 호출하여 걸려있는 락을 풀 수 있다.
    - 예시 코드
        
        ```swift
        final class SemaphoredInteger {
            private var _value: Int
            // 동시에 접근할 수 있는 스레드는 1개
            private let semaphore = DispatchSemaphore(value: 1)
            
            var value: Int {
                get {
                    semaphore.wait()
                    let value = _value
                    semaphore.signal()
                    return value
                }
                set {
                    semaphore.wait()
                    _value = newValue
                    semaphore.signal()
                }
            }
            
            init(_ value: Int) {
                self._value = value
            }
            
            func increment() {
                semaphore.wait()
                _value += 1
                semaphore.signal()
            }
        }
        ```
        
- Actor
    - Swift Concurrency에서 등장한 타입으로 actor 타입을 사용하면 actor가 제공하는 동기화 메커니즘을 사용할 수 있다. (~~이건 이제.. 추후 알아볼 예정이니 간단하게만 적어두겠다 ^^~~)

# QnA

- Data Race와 Race Condition의 차이를 설명해주세요.
- Swift에서 Data Race를 예방할 수 있는 방법을 설명해주세요.
- 경쟁 조건이 발생하는 이유와 이를 방지하기 위한 방법에는 어떤 것들이 있을까요?
- 뮤텍스와 세마포어의 차이점을 설명해주세요.
- 교착상태(데드락)가 발생하는 조건을 설명하고 이를 해결하는 방법에는 어떤 것들이 있는지 설명해주세요.
- 상호 배제는 Race Condition과 Data Race를 해결할 수 있나요?
- 교착상태는 무엇이며 조건이 무엇인가요?