# Schedulers
---

## 🟡 개요

옵저버블, 연산자 체인에서 멀티스레딩을 적용할 때 → 스케줄러 사용하여 연산자/옵저버블 실행

- 멀티스레딩이 필요한 경우
    - 네트워크 요청         : 앱의 메인스레드 차단 없이 백그라운드에서 처리해야 함
    - 데이터 처리, 가공    : DB에서 데이터 가져와 가공, UI스레드차단 없이 데이터 처리할 때
    - 병렬작업                 : 이미지 동시 다운
    - 비동기 UI 업데이트  : 다른 비동기작업 대기할 필요 없이 UI 업데이트 해야할 때
    - "스레드" : 작업을 수행하는 단위
    - "스케줄러" : 실행 가능한 작업을 관리, 어떤 스레드에서 어느 작업 실행할지 결정하는 역할, 작업들을 예약, 조절하여 동시성과 병렬성을 제어
        
1. **Observable 및 일부 연산자의 스케줄링**
    
    일부 연산자는 사용할 스케줄러를 파라미터로 받는다.(아래두개, delay, timeout 등)
    
    파라미터로 받은 스케줄러 내에서 연산을 수행
    
    기본적으로 Observable과 연산자 체인은 스케줄러를 통해 작동
    
2. **연산자 SubscribeOn** 
    
    전달인자 : 옵저버블이 구독될 스케줄러를 지정
    
    연산자 체인에서 어떤 위치에서 호출하더라도 동작
    
    옵저버블 생성 시점과 관련
    
    옵저버블 생성을 특정 스케줄러에서 생성하고 싶을 때 사용
    
3. **연산자 ObserveOn** 
    
    전달인자 : 옵저버블의 결과를 처리할, 옵저버에 알림을 보낼 때 사용할, 스케줄러를 지정
    
    연산자 체인에서 호출된 위치에 따라 그 후에 호출되는 연산자에 영향을 줌(스케줄러가 바뀌므로)
    
    옵저버블의 이벤트를 수신하는 시점과 관련
    
    operator(map, filter, etc)와 subscribe 작업을 특정 스케줄러에서 사용하고 싶을 때 사용
    
    ```swift
    sequence1
      .observeOn(backgroundScheduler)     //현재스케줄러->🟠bg 지정
      .map { n in                         //🟠
          print("This is performed on the background scheduler")
      }
      .observeOn(MainScheduler.instance)  //🟠bg->🩷main 지정
      .map { n in                         //🩷
          print("This is performed on the main scheduler")
      }
    ```
    
- 전달인자에는 **SubscribeOn, ObserveOn** 모두 ****스케줄러타입이 들어감
    - 스케줄러는 **`SchedulerType`** 프로토콜을 준수하는 타입으로 나타냄
        - ex. **`MainScheduler`**, **`ConcurrentDispatchQueueScheduler`**, **`SerialDispatchQueueScheduler`**
    - 만약 특정 스레드로 지정한다면, 스레드를 나타내는 스케줄러로 기입

---

## 🟡 참고 이미지
<table>
  <tr>
    <td><img width=370 alt="이미지" src="https://github.com/lj-7-77/practice/assets/97071996/f3ea0875-d176-4e1f-96d4-0929424e5baf"  /></td>
    <td><img width=430 alt="이미지2" src="https://github.com/lj-7-77/practice/assets/97071996/df15cb56-c33c-4ea9-8b29-f7f9e35eb4ef"  /></td>
  <tr>
</table>

> 색상🟠🔵🩷 : 스케줄러<br>
박스 테두리 색 : 메서드가 호출이 이뤄지는 스케줄러
> <br>전달인자 삼각형 색: 해당 스레드로 전환, 실제 작업이 실행

- 예시 코드
  
```swift
import RxSwift

let disposeBag = DisposeBag()

// 🟠 백그라운드 스케줄러
let backgroundScheduler = ConcurrentDispatchQueueScheduler(qos: .background)
let othercheduler = ConcurrentDispatchQueueScheduler(qos: .custom) //🔵

let mainScheduler = MainScheduler.instance //🩷 메인 스케줄러

Observable.just(5)                      //🔵 옵저버블 선언
    .observeOn(backgroundScheduler)     //🔵->🟠 이벤트처리스케줄러 전환
    .map { value in                     //🟠 데이터 변환
        print("Mapping on Thread: \(Thread.current)")
        return value * 10 // 값 변환
    }
    .subscribeOn(othercheduler)    //호출🟠->실제작업🔵 초기구독 스케줄러를 other로 지정
    .observeOn(mainScheduler)            //🔵->🩷 이벤트처리스케줄러를 전환
    .subscribe(onNext: { transformedValue in //🩷
        print("Value: \(transformedValue), Thread: \(Thread.current)")
    })
    .disposed(by: disposeBag)
```
---

## 🟡 특징
- map과 subscribeOn의 호출순서는 상황에 따라 바뀔 수도 있다.
- subscribeOn 호출은 🟠스케줄러, 작업수행은 🔵스케줄러에서 진행되는 이유
    - subscribeOn은 스케줄러 지정을 하고, 지정을 하는 행위가 이루어지는 스케줄러가 어디든지 지정받은 특정 스케줄러는 변하지 않는다.
    - 작업 수행은 지정받은 스케줄러에서 진행되기 때문
    
    > 작업 수행을 다른 스케줄러에서 하고 싶다면 *`observeOn(scheduler)`* 를 사용해라.<br>
    sequence 생성과 dispose 호출을 특정 스케줄러에서 하고 싶다면 *`subscribeOn(scheduler)`* 를 사용하라([공식문서](https://github.com/ReactiveX/RxSwift/blob/main/Documentation/Schedulers.md))
    
    
    - subscribe 호출 전 : 옵저버블은 선언, 생성만 된 상태, 어떤 이벤트도 일어나지 않는다.
    - subscribe 호출 후 : sequence 생성
        
        ```swift
        import RxSwift
        
        let numberObservable: Observable<Int> //선언
        
        numberObservable = Observable.create { observer in  //생성
            observer.onNext(1)
            observer.onNext(2)
            observer.onNext(3)
            observer.onCompleted()
            return Disposables.create()
        }
        //이후 subscribe 호출하면 시퀀스 생성됨
        ```
        
- ❌ subscribeOn이 직접 subscribe에 영향을 주는것이 아님
    - subscribeOn → create클로저에 영향을 미친다.
        - create클로저에서는 옵저버블을 생성하는 과정 이루어진다.
        - subscribeOn이 지정한 스케줄러에서 create클로저 내부 작업이 실행된다.
    - observeOn → subscribe()에 영향을 미친다.
        - subscribe는 구독 과정(실제 옵저버블의 이벤트를 수신, 처리)을 맡은 메서드이다.
        - observeOn이 지정한 스케줄러에서 subscribe의 작업이 실행된다.
        
- 마지막에 observeOn 한번더 호출하여 다른 스케줄러로 전환하는 이유
    - 스케줄러 전환을 통해 작업을 효율적으로 처리하기 위해서
    - 스케줄러 전환의 장점
        - 병렬 처리 및 성능 향상
        - 메인스케줄러에서 이뤄져야하는 작업 - UI 업데이트
        - 백그라운드에서 처리를 요하는 작업 - 네트워크 요청, 파일 다운로드 등
        - 데드락 방지
          
- 기본적으로는 MainScheduler에서 실행되며, 코드 위치에 따라 상위 스케줄러의 영향을 받는다.

---

참고

https://reactivex.io/documentation/ko/scheduler.html

[https://sweepty.medium.com/rxswift-scheduler-제대로-알아보기-f2e26aeb829d](https://sweepty.medium.com/rxswift-scheduler-%EC%A0%9C%EB%8C%80%EB%A1%9C-%EC%95%8C%EC%95%84%EB%B3%B4%EA%B8%B0-f2e26aeb829d)

http://rx-marin.com/post/observeon-vs-subscribeon/
