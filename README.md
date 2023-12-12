# Combine

The Combine framework provides a declarative Swift API for processing values over time. These values can represent many kinds of asynchronous events. Combine declares publishers to expose values that can change over time, and subscribers to receive those values from the publishers.

+ The **Publisher** protocol declares a type that can deliver a sequence of values over time. Publishers have operators to act on the values received from upstream publishers and republish them.
+ At the end of a chain of publishers, a **Subscriber** acts on elements as it receives them. Publishers only emit values when explicitly requested to do so by subscribers. This puts your subscriber code in control of how fast it receives events from the publishers it’s connected to.

You can combine the output of multiple publishers and coordinate their interaction. For example, you can subscribe to updates from a text field’s publisher, and use the text to perform URL requests. You can then use another publisher to process the responses and use them to update your app.

By adopting Combine, you’ll make your code easier to read and maintain, by centralizing your event-processing code and eliminating troublesome techniques like nested closures and convention-based callbacks.


## Project 1

We have used **Future** publisher for fetching APi Data from.

**Future** - A publisher that eventually produces a single value and then finishes or fails.

**Network Manager**

```swift
//  Taking dummy data for api calling.
    var data: [String] = ["Alpha", "Beta", "Gama"]
    
    func fetchData() -> Future<[String], Error> {
        return Future { promixe in
//      API call 
            promixe(.success(self.data))
        }
    }
```

Handling data from network manager class by using sink.

```swift
var observers : [AnyCancellable] = []

NetworkManager.shared.fetchData()
            .receive(on: DispatchQueue.main)
            .sink(receiveCompletion: { completion in
            switch completion {    
            case.finished:
                print("Finished")
            case .failure(let error):
                print("Error: \(error)")
            }
        }, receiveValue: { value in
            self.alpha = value
            self.mainTableView.reloadData()
        }).store(in: &observers)
```

Now, let see **PassthroughSubject**, a subject that broadcasts elements to downstream subscribers.
```swift
final class PassthroughSubject<Output, Failure> where Failure : Error
```

if we want to send any string, we can create like this
```swift
let sendMessage = PassthroughSubject<String, Never>()let sendMessage = PassthroughSubject<String, Never>()
sendMessage.send("Buttton Was pressed")
```
and for receiving data ->
```swift
var observers : [AnyCancellable] = []
sendMessage.sink { string in
            print(string)
        }.store(in: &observers)
```

**Extras:** we can define sigle observable using
```swift
var mainTableViewObserver: AnyCancellable?
```

## Project 2

Basic counter

```swift
class Counter {
    static var value = 0
    
    var counterPublisher = PassthroughSubject<Int, Never>()

    func increment() {
        Counter.value += 1
        counterPublisher.send(Counter.value)
    }
}
```

```swift
let counter = Counter()
var observers: [AnyCancellable] = []

counter.increment()

counter.counterPublisher
            .sink {value in
                print(value)
            }.store(in: &observers)
```

## Project 3

**DOTA HERO** API calling using combine.

```swift
// VIEW MODEL

import Foundation
import Moya
import Combine

class ViewModel {
    
    var observers: Set<AnyCancellable> = []
    let dotaServiceProvider = MoyaProvider<DotaService>()
    var dotaHeroDataObserver = PassthroughSubject<HERO, Error>()
    
    
    func getDataSet() {
//        Set default data
        dotaHeroDataObserver.send([.default])
        
//        Fetching data
       dotaServiceProvider.requestPublisher(.heroStats)
            .sink(receiveCompletion: { completion in
                switch completion {
                case .finished:
                    print("Fininshed data loading")
                case .failure(let error):
                    self.dotaHeroDataObserver.send(completion: .failure(error))
                }
            }, receiveValue: {response in
                let responseData = try! JSONDecoder().decode(HERO.self, from: response.data)
                self.dotaHeroDataObserver.send(responseData)
                self.dotaHeroDataObserver.send(completion: .finished)
            }).store(in: &observers)
    }
}
```

```swift
// VIEW CONTROLLER

import UIKit
import Combine

class ViewController: UIViewController {
    
    let viewModel = ViewModel()
    var observers: Set<AnyCancellable> = []

    override func viewDidLoad() {
        super.viewDidLoad()
        
        configDependency()
        viewModel.getDataSet()
    }
    
    private func configDependency() {
        viewModel.dotaHeroDataObserver
            .sink(receiveCompletion: {completetion in
                switch completetion {
//                  It is called at last when data is recieved.
                case .finished:
//                  called when successfully recieved data
                    print("FINISHED")
                case .failure(let error):
//                    called when recieving error from api calling.
                    print("ERROR: \(error)")
                }
            }, receiveValue: {data in
                print(data)
            }).store(in: &observers)
    }
}
```

## Project 4
Combine is similar to RxSwift but we don't have to install POD for that because Combine framework provides a declarative Swift API.
If we have to call **DOTA HERO** API without using any POD we can do like this:

```swift
// VIEW MODEL

import Foundation
import Combine

enum HTTPError: LocalizedError {
    case statusCode
}

class ViewModel {
    
    var observers: Set<AnyCancellable> = []
    var dotaHeroDataObserver = PassthroughSubject<HERO, Error>()
    let url = URL(string: "https://api.opendota.com/api/heroStats")!
    
    func getDataSet() {
        dotaHeroDataObserver.send([.default])
      
        URLSession.shared.dataTaskPublisher(for: url)
        .tryMap { output in
            guard let response = output.response as? HTTPURLResponse, response.statusCode == 200 else {
                throw HTTPError.statusCode
            }
            return output.data
        }
        .decode(type: HERO.self, decoder: JSONDecoder())
        .eraseToAnyPublisher()
        .sink(receiveCompletion: { completion in
            switch completion {
            case .finished:
                self.dotaHeroDataObserver.send(completion: .finished)
            case .failure(let error):
                self.dotaHeroDataObserver.send(completion: .failure(error))
            }
        }, receiveValue: { heros in
            self.dotaHeroDataObserver.send(heros)
        }).store(in: &observers)
        

    }
}
```

```swift
// VIEW CONTROLLER

import UIKit
import Combine

class ViewController: UIViewController {
    let viewModel = ViewModel()
    var observers: Set<AnyCancellable> = []

    override func viewDidLoad() {
        super.viewDidLoad()
        
        configDependency()
        viewModel.getDataSet()
    }
    
    private func configDependency() {
        viewModel.dotaHeroDataObserver
            .sink(receiveCompletion: {completetion in
                switch completetion {
//                  It is called at last when data is recieved.
                case .finished:
//                  called when successfully recieved data
                    print("FINISHED")
                case .failure(let error):
//                    called when recieving error from api calling.
                    print("ERROR: \(error)")
                }
            }, receiveValue: {data in
                print(data)
            }).store(in: &observers)
    }
}
```
