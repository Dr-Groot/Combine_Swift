# Combine

The Combine framework provides a declarative Swift API for processing values over time. These values can represent many kinds of asynchronous events. This means that we can use the framework’s APIs to handle *closures, delegates, KVO, and notifications* more efficiently, and transform, filter, and combine data streams in powerful ways. Combine declares publishers to expose values that can change over time, and subscribers to receive those values from the publishers.

For example, if we need to handle a chain of events, such as waiting for a network request to complete before updating the UI, we can use the Combine framework to write more concise and readable code. We can create a publisher that emits a value when the network request completes, and subscribe to that publisher to update the UI. This can make our code more streamlined and easier to reason about.

In the rest of this tutorial, we’ll explore the basics of the Combine framework and show you how to use it to write API calls.

**How Does Combine Work?**

Combine framework works on the basis of following three key concepts

1. Publisher

2. Subscriber

3. Operator

Combine is mainly working as publisher and subscriber model.

The **Publisher** is a protocol that defines a value type with two associated types: **Output** and **Failure**. Publishers allow the registration of subscribers and emit values to those subscribers. Publishers can be thought of as the source of the data stream.

The **Subscriber** is also a protocol, but it is a reference type. Subscribers can receive values from publishers and can be notified when the publisher has finished emitting values. The subscriber has two associated types: **Input** and **Failure**. Subscribers can be thought of as the destination of the data stream.

Here is the pattern for the Combine publisher and subscriber model.

![](https://miro.medium.com/v2/resize:fit:700/1*ZwMj-2ofoo1MU3Xfw8j2Eg.png)


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
