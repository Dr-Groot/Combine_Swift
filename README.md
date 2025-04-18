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

The Pattern

Finally, **Operators** are extensions of publishers. Operators are methods that are called by the publisher and return a value to the publisher. Operators can be used to transform or manipulate the publisher’s associated types before passing them on to the subscribers. There may be multiple operators for a single publisher, allowing for complex data transformations and manipulations.

![](https://miro.medium.com/v2/resize:fit:475/1*OaBFkoJdjDMfBQdvibWYFQ.png)

Depicting How to Attach Operators with Publishers and Subscribers

You can combine the output of multiple publishers and coordinate their interaction. For example, you can subscribe to updates from a text field’s publisher, and use the text to perform URL requests. You can then use another publisher to process the responses and use them to update your app.

By adopting Combine, you’ll make your code easier to read and maintain, by centralising your event-processing code and eliminating troublesome techniques like nested closures and convention-based callbacks.

**Part 1**

Now, let see **PassthroughSubject**, a subject that broadcasts elements to downstream subscribers.

```swift
// **PassthroughSubject Class**
final class PassthroughSubject<Output, Failure> where Failure : Error
```

If we want to send any string, we can create like this

```swift
// Example 1
// Defining Any Cancellable
var observers: Set<AnyCancellable> = []

// Declaring Boradbaster.
let checkMessage =  PassthroughSubject<String,Never>()

//Subscriber example 1
checkMessage.sink(receiveValue: { value in
              print(value)
}         ).store(in: &observers)

// Subscriber example 2
checkMessage.sink { value in
            print(value)
        }.store(in: &observers)

// Broadcasting.
checkMessage.send("My first example")
// O/P -> My first example

// Example 2
// Declaring Boradbaster.
let checkMessage2 = PassthroughSubject<String, Never>()

// Subscribing
checkMessage2.sink(receiveCompletion: {completion in
    switch completion {
    case .finished:
        print("FINISHED")
    case .failure:
        print("FAILURE")
    }
}, receiveValue: {value in
    print(value)
}).store(in: &observers)

// Broadcasting
checkMessage2.send("My second example")
// O/P -> My second example

checkMessage2.send(completion: .finished)
// O/P -> FINISHED

// Sending value and completion can be sepearte.
```

 

Basic counter

```swift
// Counter Class
class Counter {
    static var value = 0

    var counterPublisher = PassthroughSubject<Int, Never>()

    func increment() {
        Counter.value += 1
        counterPublisher.send(Counter.value)
    }
}

// Using the counter class 
let counter = Counter()
var observers: Set<AnyCancellable> = []

counter.increment()

counter.counterPublisher
            .sink {value in
                print(value)
 }.store(in: &observers)
```

Using PassthroughSubject with Errors

```swift
// Defining Error
enum MyError: Error {
    case somethingWentWrong
}

let subject = PassthroughSubject<String, MyError>()

let cancellable = subject
    .sink(receiveCompletion: { completion in
        switch completion {
        case .finished:
            print("Stream finished successfully")
        case .failure(let error):
            print("Stream failed with error: \(error)")
        }
    }, receiveValue: { value in
        print("Received value: \(value)")
    })
    
subject.send("Data loaded successfully")

subject.send(completion: .failure(.somethingWentWrong))
```

Using PassthroughSubject with Result Type

```swift
// Define a type for your success and failure cases
enum MyError: Error {
    case somethingWentWrong
}

// Create a PassthroughSubject that sends Result values
let subject = PassthroughSubject<Result<String, MyError>, Never>()

let cancellable = subject
    .sink { result in
        switch result {
        case .success(let data):
            print("Success with data: \(data)")
        case .failure(let error):
            print("Failure with error: \(error)")
        }
    }

// Sending a success event
subject.send(.success("Data loaded successfully"))

// Sending a failure event
subject.send(.failure(.somethingWentWrong))

// Using the PassthroughSubject for sending values we can ignore the default completion.
// Example: subject.send(completion: .finished)
 

```

**Extras:** we can define single observable using

```swift
var mainTableViewObserver: AnyCancellable?
```

**Part 2**

We have used **Future** publisher for fetching Api.

**Future** - A publisher that eventually produces a single value and then finishes or fails.

Use a future to perform some work and then asynchronously publish a single element. You initialise the future with a closure that takes a [`Future.Promise`](https://developer.apple.com/documentation/combine/future/promise); the closure calls the promise with a [`Result`](https://developer.apple.com/documentation/Swift/Result) that indicates either success or failure. In the success case, the future’s downstream subscriber receives the element prior to the publishing stream finishing normally. If the result is an error, publishing terminates with that error.

**Basic Use of Future**

```swift
**enum** error: Error {
    **case** somethingWentWrong
}

// Network Manager Class
**class** NetworkManager {
    //  Taking dummy data for api calling.
        **var** data: [String] = ["Alpha", "Beta", "Gama"]

        **func** fetchData() -> Future<[String], error> {
            **return** Future { promixe **in**
    //      API call and sending data with success
                promixe(.success(**self**.data))
            }
        }
}

**var** observers : [AnyCancellable] = []
**let** networkManager = NetworkManager()

networkManager.fetchData()
            .receive(on: DispatchQueue.main)
            .sink(receiveCompletion: { completion **in**
            **switch** completion {
            **case**.finished:
                print("Finished")
            **case** .failure(**let** error):
                print("Error: \(error)")
            }
        }, receiveValue: { value **in**
            print(value)
        }).store(in: &observers)

// We will recieve the value ["Alpha", "Beta", "Gama"]
// and then Finished will be print automatically.

// If we send promixe(.failure(.somethingWentWrong)) then it will only
// print Error: somethingWentWrong
```

**Part 3**

Revise basic API Call example

```swift
guard let url = URL(string: "https://api.openweathermap.org/data/2.5/weather?q=delhi&appid={YOUR-API-KEY}&units=metric") else {
            return
        }
        
        var request = URLRequest(url: url)
        request.httpMethod = "GET"
        
        URLSession.shared.dataTask(with: request) { data, response, error in
            guard data != nil else {
                print("data is nil")
                return
            }
            
            let decoder = JSONDecoder()
            let decodedData = try? decoder.decode(WeatherResponse.self, from: data!)
            
            DispatchQueue.main.async {
                self.temp = (decodedData?.main.temp)!
            }
            
        }.resume()
```

**DOTA HERO** API calling using Combine and Moya

```swift
// Moya Service
import Foundation
import Moya

enum DotaService {
    case heroStats
}

extension DotaService: TargetType {
    var baseURL: URL {
        guard let url = URL(string: "https://api.opendota.com") else { fatalError() }
        return url
    }

    var path: String {
        switch self {
        case .heroStats:
            return "/api/heroStats"
        }
    }

    var method: Moya.Method {
        switch self {
        case .heroStats:
            return .get
        }
    }

    var sampleData: Data {
        let json = ""
        return json.data(using: String.Encoding.utf8)!
    }

    var task: Moya.Task {
        switch self {
        case .heroStats:
            return .requestPlain
        }
    }

    var headers: [String : String]? {
        return nil
    }
}
```

```swift
// VM
import Foundation
import Moya
import Combine

class ViewModel {
    
    var observers: Set<AnyCancellable> = []
    let dotaServiceProvider = MoyaProvider<DotaService>(plugins: [ NetworkLoggerPlugin(configuration: .init(logOptions: .verbose))])
    var dotaHeroDataObserver = PassthroughSubject<HERO, Error>()
    
    func getDataSet() {
//        Set default data
        dotaHeroDataObserver.send([.default])
        
// Fetching data
// If it fails to load data from API it wouldn't go to reciveValue to decode.
       dotaServiceProvider.requestPublisher(.heroStats)
            .sink(receiveCompletion: { completion in
                switch completion {
                case .finished:
                    print("Fininshed data loading")
                case .failure(let error):
                    self.dotaHeroDataObserver.send(completion: .failure(error))
                }
            }, receiveValue: {response in
                do {
                    let responseData = try JSONDecoder().decode(HERO.self, from: response.data)
                    self.dotaHeroDataObserver.send(responseData)
                    self.dotaHeroDataObserver.send(completion: .finished)
                } catch (let error) {
                    self.dotaHeroDataObserver.send(completion: .failure(error))
                }
               
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

**Part 4**

Combine is similar to RxSwift but we don't have to install POD for that because Combine framework provides a declarative Swift API. If we have to call **DOTA HERO** API without using any POD we can do like this:

Basic API Calling using Combine and URLSession

```swift
func callSample() {
        guard let url = URL(string: "https://api.opendota.com/api/heroStats") else { return }
        URLSession.shared.dataTaskPublisher(for: url)
            .sink(receiveCompletion: { completion in
                switch completion {
                case .finished:
                    print("FINISHED")
                case .failure(let error):
                    print("FAILURE \(error)")
                }
            }, receiveValue: { data, response in
                if let httpResponse = response as? HTTPURLResponse {
                        print(httpResponse.statusCode)
                    }
                print(data)
            }).store(in: &observers)
    }
```

Advance API Calling using Combine and URLSession

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
