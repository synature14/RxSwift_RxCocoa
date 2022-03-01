# RxSwift_RxCocoa

<img width="562" alt="SumUp_RxCocoa" src="https://user-images.githubusercontent.com/13548107/152744815-0f759ec9-0317-4605-a36b-340d2b17a6be.png">

## Using binding observables to display data
 Make your stream reusable and transform a single-use data source into a multi-use Observable.
 
 ```swift
 let search = searchCityName.rx.text.orEmpty
            .filter { !$0.isEmpty }
            .flatMapLatest { text in    // 최신값이 있으면 이전에 네트워크 요청하는 Observable들은 모두 cancel해버림
                ApiController.shared
                    .currentWeather(for: text)
                    .catchErrorJustReturn(.empty)
            }
            .share(replay: 1)       // observable을 reuse해보자..!
            .observeOn(MainScheduler.instance)
            
search.map { "\($0.temperature) 도" }
  .bind(to: tempLabel.rx.text)
  .disposed(by: disposeBag)
        
search.map(\.icon)
  .bind(to: iconLabel.rx.text)
  .disposed(by: disposeBag)
        
search.map { "\($0.humidity)%" }
  .bind(to: humidityLabel.rx.text)
  .disposed(by: disposeBag)
        
search.map(\.cityName)
  .bind(to: cityNameLabel.rx.text)
  .disposed(by: disposeBag)
  
 ```
 
 ## RxSwift - Combining Operators
 ### 1) merge
 단순히 모든 event를 결합 (단 순서는 번갈아 가며) -> 여러 stream의 event들을 각 stream의 순서 번갈아가며 event 결합
![merge](https://user-images.githubusercontent.com/13548107/153549336-c58dbf74-4c51-41ac-9b58-348e819538c4.png)

```swift
let disposeBag = DisposeBag()

let first = Observable.of(1, 2, 3)
let second = Observable.of(4, 5, 6)

Observable.merge(first, second)
    .subscribe(onNext: { print($0) })
    .disposed(by: dispseBag)

/* Prints:
1
4
2
5
3
6
*/
```


### 2) combineLatest
여러 stream 중에서 단 한 가지라도 이벤트를 방출하면, 각각 stream의 맨 마지막 값을 뽑아서 새로운 값을 방출.

한 번 값을 방출한 이후에는 클로저가 각각의 Observable이 방출했었던 최신 값을 받음

![combineLatest](https://user-images.githubusercontent.com/13548107/153549679-3d536f11-0c23-4baf-934a-905933269a4e.png)

* Q.언제쓰이나? 
>such as observing several text fields at once and combining their values, watching the status of multiple sources, and so on.

**ex) 이메일과 비밀번호가 변할 때마다 버튼의 enabled 를 계산할 때

```swift
let disposeBag = DisposeBag()

let first = Observable.of(1, 2, 3, 4)
let second = Observable.of("A", "B", "C")

Observable.combineLatest(first, second)
    .subscribe(onNext: { print("\($0)" + $1) })
    .disposed(by: dispseBag)

/* Prints:
1A
2A
2B
3B
3C
4C
*/
```




### 3) withLatestFrom
A.withLatestFrom(B) {  ($0, $1)  }  - $0는 A의 onNext값. $1은 B의 onNext값.

장점: 새로운 이벤트가 발생한 스트림에서 지정한 스트림의 가장 최신 아이템을 얻을 수 있음

* 조건1) B가 1번 이상 방출된 상태에서부터 시작! (그 전에는 모든 이벤트 무시 **그전에A값 방출되어도,,,,)
* 조건2) withLatestFrom 메소드를 호출한 observable 즉 A의 이벤트가 발생한 경우에 B 이벤트 방출.

 ![withLatestFrom](https://user-images.githubusercontent.com/13548107/153553677-4bb53c7f-2fe9-437b-a536-947957a7e885.png)


```swift
// withLatestFrom
oddNumber.withLatestFrom(evenNumber) { "\($0) \($1)"}
    .subscribe(onNext: { print("\($0)") })
    .disposed(by: bag)

oddNumber.onNext(1) // emit x
evenNumber.onNext(2) // emit x

evenNumber.onNext(4) // emit x
oddNumber.onNext(3) // 3 4
oddNumber.onNext(5) // 5 4
oddNumber.onNext(7) // 7 4
```



### 4) zip
발생 순서가 같은 이벤트만 발생 (순서가 다르면 발생하지 않음)
>They pair each next value of each observable at the same logical position (1st with 1st, 2nd with 2nd, etc.)

![zip](https://user-images.githubusercontent.com/13548107/153550421-5aedf6c0-d4e7-4991-92c8-8fc61aa5e105.png)
```swift
let disposeBag = DisposeBag()

let first = Observable.of(1, 2, 3, 4
let second = Observable.of("A", "B", "C")

Observable.zip(first, second)
    .subscribe(onNext: { print("\($0)" + $1) })
    .disposed(by: dispseBag)
    
/* Prints:
1A
2B
3C
*/
```

***

## RxSwift - Transforming Operators
### 1) flatMap
FlatMap은 구성 요소 하나하나 observable로 만들어서 방출합니다.


> Projects each element of an observable sequence to an observable sequence and merges the resulting observable sequences into one observable sequence.
> ![flatMap](https://user-images.githubusercontent.com/13548107/153555374-84333fc0-5f1f-4638-ac06-84b8ebefe484.png)



### 2) flatMapLatest
Observable이 새로 들어오면 이전에 생성된 observable을 dispose시키고 새로운 것을 이용


![flatMapLatest02](https://user-images.githubusercontent.com/13548107/153554532-c1732aca-387d-44a9-bc45-18ef099c6422.png)
> 위 그림에서 헷갈릴수 있으니 '01', '02', '03' Observable을 각각 'Blue', 'Green', 'Orange'라고 하자.
> 1) Blue observable에서 1을 방출했으므로 flatMapLatest{}의 최종 리턴 stream에 '1' 방출!
> 2) 이후에 Green observable에서 2를 방출했으므로 stream에 '2'방출! 하면서~ 기존 Blue observable에 대한 listening 끊음
> 3) 따라서 Blue에서 방출한 '3'값은 무시
> 4) Orange observable에서 4를 방출 -> 최종 리컨 stream에도 '4'방출! 하면서~~ 기존 Green observable에 대한 listening 끊음
> 5) 따라서 Green에서 방출한 '5' 무시
> 6) Orange observable에서 '6' 방출 -> 최종 리컨 stream에도 '6'방출!


```swift
struct Student {
  var score: BehaviorSubject<Int>
}

let disposeBag = DisposeBag()     

let ryan = Student(score: BehaviorSubject(value: 80))
let charlotte = Student(score: BehaviorSubject(value: 90))

let student = PublishSubject<Student>()

student
   .flatMapLatest {
       $0.score
}
   .subscribe(onNext: {
       print($0)
   })
   .disposed(by: disposeBag)

student.onNext(ryan)
ryan.score.onNext(85)

student.onNext(charlotte)
ryan.score.onNext(95) // 지금 구독하고 있는 최신 이벤트는 charlotte이므로 ryan에 이벤트는 무시
charlotte.score.onNext(100)

/* Prints:
 80 
 85 
 90 
 100
*/
```


