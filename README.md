# Swift-Concurrency-Study
> Swift 6.0을 위한 동시성 스터디

### 스터디 목표
- Swift 6.0 동시성을 학습
- 학습한 내용을 기반으로 `기록소` 프로젝트 코드 품질 증가

<br>

## 📚 Swift Concurrency 커리큘럼

| 주차  | 학습 주제 | 활동 |
|------|---------|---|
| **Week 1** | Process & Thread 개념 | [활동 링크](https://github.com/Swift-Concurrency-Study/Study/blob/main/Week1/ProcessThread.md) |
| **Week 2** | 공유자원과 임계영역 | [활동 링크](https://github.com/Swift-Concurrency-Study/Study/blob/main/Week2/DataRace.md) |
|  | 직렬-동시, 동기-비동기 개념 | [활동 링크](https://github.com/Swift-Concurrency-Study/Study/blob/main/Week2/%EB%8F%99%EC%8B%9C%26%EC%A7%81%EB%A0%AC%2C%20%EB%8F%99%EA%B8%B0%26%EB%B9%84%EB%8F%99%EA%B8%B0.md) |
| **Week 3** | 동시성 프로그래밍 with GCD | [활동 링크](https://github.com/Swift-Concurrency-Study/Study/blob/main/Week3/GCD.md) |
| **Week 4** | Swift Concurrency 등장 배경 | [활동 링크](https://github.com/Swift-Concurrency-Study/Study/blob/main/Week4/swift%20concurrency.md) |
|  | 비동기 호출에서의 스레드 제어권 | [활동 링크](https://github.com/Swift-Concurrency-Study/Study/blob/main/Week4/swift%20concurrency.md#%EC%8A%A4%EB%A0%88%EB%93%9C-%EC%A0%9C%EC%96%B4%EA%B6%8C) |
|  | Task와 구조화된 동시성(= Structured Concurrency) | [활동 링크](https://github.com/Swift-Concurrency-Study/Study/blob/main/Week4/swift%20concurrency.md#async-let) |
| **Week 5** | Actor 개념 | [활동 링크](https://github.com/Swift-Concurrency-Study/Study/blob/main/Week5/Actor%2C%20Sendable.md#%EB%AA%A9%EC%B0%A8) |
|  | Sendable 프로토콜 | [활동 링크](https://github.com/Swift-Concurrency-Study/Study/blob/main/Week5/Actor%2C%20Sendable.md#sendable) |
|  | Main Actor 개념 | [활동 링크](https://github.com/Swift-Concurrency-Study/Study/blob/main/Week5/Actor%2C%20Sendable.md#global-actor) |
| **Week 6** | 얕은 복사 & 깊은 복사 (+ 클래스에서의 깊은 복사) | [활동 링크]() |
|  | NonCopyable 프로토콜 | [활동 링크]() |
|  | Generic과 Extension에서의 NonCopyable 활용 | [활동 링크]() |
| **Week 7, 8** | 기록소 프로젝트 코드 리팩토링 | [활동 링크](https://github.com/boostcampwm-2024/iOS10-MemorialHouse) |

<br>

## 👨🏻‍💻 스터디 방식

- **날짜**: 매주 금요일 9시 (+- 1시간)
- **진행 방식**:
  1. 스터디를 위해 조직/레포 생성
  2. 매주 각자 주제를 학습하고 노션에 정리
  3. 해당 주제와 관련된 면접 질문도 작성
  4. 스터디 날짜에 랜덤으로 2명 선정:
     - 1명: 발표 담당
     - 1명: 정리 담당
  5. 정리 담당자는 4명의 정리 내용을 취합해 최종본을 레포에 업로드
