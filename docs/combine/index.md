# Description of Combine

Combine is a unified declarative framework for processing values over time. It is Apple's
framework that is built using the functional reactive concepts that can be found in other
languages. If you are already familar with ReactiveX extensions, there is [a pretty good cheat-sheet for translating the specifics between Rx and Combine](https://medium.com/gett-engineering/rxswift-to-apples-combine-cheat-sheet-e9ce32b14c5b),
built and inspired by the data collected at
[https://github.com/freak4pc/rxswift-to-combine-cheatsheet](https://github.com/freak4pc/rxswift-to-combine-cheatsheet).

Combine is Apple's functional reactive library. In Apple's words, it provides
> "a declarative Swift API for processing values over time".

## Core Concepts

Combine is built to process streams of events - one or more events, over time. It does so by
sourcing data from **publishers**, transforming the events through **operators**, which are
consumed by **subscribers**. These sequences, often called "streams" are composed and typically
chained together.

[Publisher](https://developer.apple.com/documentation/combine/publisher) and
[Subscriber](https://developer.apple.com/documentation/combine/subscriber) are defined as
protocols in Swift, and when defined in code are set up
with two associated types: an Output type and a Failure type. Subscribers have an Input and Failure
type defined, and these must align to the publisher types for the two to be composed together.

```text
Publisher <OutputType>, <FailureType>
              |  |          |  |
               \/            \/
Subscriber <InputType>, <FailureType>
```

Operators are used to transform types - both the Output and Failure type. Operators may also
split or duplicate streams, or merge streams, Operators must always be aligned by the combination
of Output/Failure types.

The interals of the system are all driven by the subscriber. Subscribers and Publishers
communicate in a well defined sequence:

- the subscriber is attached to a publisher: `.subscribe(Subscriber)`
- the publisher sends a subscription: `receive(subscription)`
- subscriber requests _N_ values: `request(_ : Demand)`
- publisher sends _N_ (or fewer) values: `receive(_ : Input)`
- publisher sends completion: `receive(completion:)`

Operators fit in between Publishers and Subscribers. They adopt the
[Publisher protocol](https://developer.apple.com/documentation/combine/publisher), subscribing
to one or more Publishers, and sending results to one (or more) Subscribers.

## Publishers

protocol documentation: [Publisher](https://developer.apple.com/documentation/combine/publisher)

A publisher defines how values (and errors) are produced, and allows the registration of a subscriber.

NotificationCenter.default.publisher -> `<Notification>`, `<Never>`

[Convenience publishers](https://developer.apple.com/documentation/combine/publishers)

- Just -> `<SomeType>`, `<Never>`
  - often used in error handling
  - provides a single result on a stream and then terminates
- Empty -> `<SomeType>`, `<Error>`
  - never publishes any values, and optionally finishes immediately
  - `Empty(completeImmediately: false)`
- Fail
  - A publisher that immediately gterminates with the specified failure
- Once
  - Generates an output to each subscriber exactly once then finishes or fails immediately
- Optional
  - generates a value exactly once for each subscriber, if the optional has a value
- Sequence
  - publishes a given sequence of elements
- Deferred
  - publisher waits for a subscriber before running the provided closure to create values for the subscriber

publisher -> `<SomeType>`, `<Never>`
- extracts a property from an object and returns it
- ex: `.publisher(for: \.name)`

BindableObject

- often linked with method `didChange` to publish changes to model objects
- `@ObjectBinding var model: MyModel`

@Published

- property wrapper that adds a Combine publisher to any property

Future

- you provide a closure that converts a callback/function of your own choosing into a promise.
- example:

```swift
return Future { promise in
  self.myFunctionCall(someVariable) { varname in
    promise(.success(varname ? username : nil))
  }
}
```

- can be used within a Flatmap in an operator sequence to do your own processing/logic within
  a stream, call out to an external service, etc.
- commonly used when making external service calls over the network.

## Subscribers

Cancellation:

Subscribers can support cancellation, which terminates a subscription early.

```swift
let trickNamePublisher = ... // Publisher of <String, Never>

let canceller = trickNamePublisher.sink { trickName in
}
```

Kinds of subscribers:

- key-path assignment
  - ex: `Subscribers.Assign(object: exampleObject, keyPath: \.someProperty)`
  - ex: `.assign(to: \.isEnabled, on: signupButton)`
  - Assigns the value of a KVO-compliant property from a publisher.
  - requires Failure to be `<Never>`

- Sink
  - you provide a closure where you process the results

- Subject
  - behave like both a publisher and subscriber
  - broadcasts values to multiple subscribers
  - `Passthrough` and `CurrentValue` subscribers
    - Passthrough doesn't maintain any state - just passes through provided values
    - CurrentValue remembers the current value so that when you attach a subscriber you can see the current value

- SwiftUI
  - SwiftUI provides the subscribers, you primarily fill in the publishers and operators

## Operators

The naming pattern of operators tends to follow similiar patterns on ordered collection types.

signature transformations

- eraseToAnyPublisher
  - when you chain operators together in swift, the object's type signature accumulates all the various
    types, and it gets ugly pretty quickly.
  - eraseToAnyPublisher takes the signature and "erases" the type back to the common type of AnyPublisher
  - this provides a cleaner type for external declarations (framework was created prior to Swift 5's opaque types)
  - `.eraseToAnyPublisher()`
  - often at the end of chains of operators, and cleans up the type signature of the property getting asigned to the chain of operators

functional transformations

- map
  - you provide a closure that gets the values and chooses what to publish
  - there's a variant `tryMap` that that transforms all elements from the upstream publisher with a provided error-throwing closure.

- compactMap
  - republishes all non-nil results of calling a closure with each received element.
  - there's a variant `tryCompactMap` for use with a provided error-throwing closure.

- prefix
  - Republishes elements until another publisher emits an element.
  - requires Failure to be `<Never>`

- decode
  - common operating where you hand in a type of decoder, and transform data (ex: JSON) into an object
  - can fail, so it returns an error type
  - Available when Output conforms to Decodable.
  -> `<SomeType>`, `<Error>`

- flatMap
  - collapses nil values out of a stream
  - used with error recovery or async operations that might fail (ex: Future)
  - requires Failure to be `<Never>`

- removeDuplicates
  - `.removeDuplicates()`
  - remembers what was previously sent in the stream, and only passes forward new values
  - there's a variant `tryRemoveDuplicates` for use with a provided error-throwing closure.

- encode
  - Encodes the output from upstream using a specified TopLevelEncoder. For example, use JSONEncoder.
  - Available when Output conforms to Encodable.

**list operations**

- filter
  - requires Failure to be `<Never>`
  - takes a closure where you can specify how/what gets filtered
  - there's a variant `tryFilter`for use with a provided error-throwing closure.

- merge
  - Combines elements from this publisher with those from another publisher of the same type, delivering an interleaved sequence of elements.
  - requires Failure to be `<Never>`
  - multiple variants that will merge between 2 and 8 different streams

- reduce
  - A publisher that applies a closure to all received elements and produces an accumulated value when the upstream publisher finishes.
  - requires Failure to be `<Never>`
  - there's a varient `tryReduce` for use with a provided error-throwing closure.

- contains
  - emits a Boolean value when a specified element is received from its upstream publisher.
  - variant `containsWhere` when a provided predicate is satisfied
  - variant `tryContainsWhere` when a provided predicate is satisfied but could throw errors

- drop
  - multiple variants
  - requires Failure to be `<Never>`
  - Ignores elements from the upstream publisher until it receives an element from a second publisher.
  - or `drop(while: {})`

- dropFirst

- count
  - publishes the number of items received from the upstream publisher

- comparison
  - republishes items from another publisher only if each new item is in increasing order from the previously-published item.
  - there's a variant `tryComparson` which fails if the ordering logic throws an error

- prepend
  - Prefixes a Publisher’s output with the specified sequence.
  - requires Failure to be `<Never>`

- append
  - Append a Publisher’s output with the specified sequence.
  - requires Failure to be `<Never>`

**error handling**

- assertNoFailure
  - Raises a fatal error when its upstream publisher fails, and otherwise republishes all received input.

- retry
  - requires Failure to be `<Never>`
  - multiple variants - once or by a provided count

- catch
  - Handles errors from an upstream publisher by replacing it with another publisher.

- mapError
  - Converts any failure from the upstream publisher into a new error.

- setFailureType

- breakpoint
  - Raises a debugger signal when a provided closure needs to stop the process in the debugger.
- breakpointOnError
  - Raises a debugger signal upon receiving a failure.

**thread or queue movement**

- receive(on:)
  `.receive(on: RunLoop.main)`

- subscribe(on:)

**scheduling and time**

- throttle
  - Publishes either the most-recent or first element published by the upstream publisher in the specified time interval.
  - requires Failure to be `<Never>`

- timeout
  - Terminates publishing if the upstream publisher exceeds the specified time interval without producing an element.
  - requires Failure to be `<Never>`

- debounce
  - `.debounce(for: 0.5, scheduler: RunLoop.main)`
  - collapses multiple values within a specified time window into a single value
  - often used with `.removeDuplicates()`

- delay
  - Delays delivery of all output to the downstream receiver by a specified amount of time on a particular scheduler.
  - requires Failure to be `<Never>`

- measureInterval
  - Measures and emits the time interval between events received from an upstream publisher.
  - requires Failure to be `<Never>`

**combining streams**

- zip
  - Combine elements from another publisher and deliver pairs of elements as tuples.
  - requires Failure to be `<Never>`

- combineLatest
  - brings inputs from 2 (or more) streams together
  - you provide a closure that gets the values and chooses what to publish

(operators to be organized and described):

- collect
  - multiple variants
    - buffers items
    - `collect()` Collects all received elements, and emits a single array of the collection when the upstream publisher finishes.
    - `collect(Int)` collects N elements and emits as an array
    - `collect(.byTime)` or `collect(.byTimeOrCount)`

- max
  - Available when Output conforms to Comparable.
  - Publishes the maximum value received from the upstream publisher, after it finishes.

- min
  - Publishes the minimum value received from the upstream publisher, after it finishes.
  - Available when Output conforms to Comparable.

- allSatisfy
  - Publishes a single Boolean value that indicates whether all received elements pass a given predicate.
  - there's a variant `tryAllSatisfy` when the predicate can throw errors

- replaceError
  - requires Failure to be `<Never>`

- replaceEmpty
  - requires Failure to be `<Never>`

- replaceNil
  - requires Failure to be `<Never>`
  - Replaces nil elements in the stream with the proviced element.

- abortOnError

- ignoreOutput

- switchToLatest

- scan

- handleEvents

- first
  - requires Failure to be `<Never>`
  - publishes the first element to satisfy a provided predicate

- last
  - requires Failure to be `<Never>`
  - publishes the last element to satisfy a provided predicate

- log

- print
  - Prints log messages for all publishing events.
  - requires Failure to be `<Never>`

- output