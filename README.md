# Combine

The Combine framework provides a declarative Swift API for processing values over time. These values can represent many kinds of asynchronous events. Combine declares publishers to expose values that can change over time, and subscribers to receive those values from the publishers.

+ The **Publisher** protocol declares a type that can deliver a sequence of values over time. Publishers have operators to act on the values received from upstream publishers and republish them.
+ At the end of a chain of publishers, a **Subscriber** acts on elements as it receives them. Publishers only emit values when explicitly requested to do so by subscribers. This puts your subscriber code in control of how fast it receives events from the publishers it’s connected to.

You can combine the output of multiple publishers and coordinate their interaction. For example, you can subscribe to updates from a text field’s publisher, and use the text to perform URL requests. You can then use another publisher to process the responses and use them to update your app.

By adopting Combine, you’ll make your code easier to read and maintain, by centralizing your event-processing code and eliminating troublesome techniques like nested closures and convention-based callbacks.


## Project 1 - CombineIntro

We have used **Future** publisher for fetching APi Data from.

**Future** - A publisher that eventually produces a single value and then finishes or fails.

**Network Manager**
Taking dummy data for api calling.
```swift
    var data: [String] = ["Alpha", "Beta", "Gama"]
    
    func fetchData() -> Future<[String], Error> {
        return Future { promixe in
            promixe(.success(self.data))
        }
    }
```
