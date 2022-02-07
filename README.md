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
