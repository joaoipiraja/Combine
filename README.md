# Combine
Combine -  Proof Of Concept 


```swift


struct Food{
    var id: Int
    var name: String
    var repetition: Int = 1
}

let foodBank: Publishers.Sequence<[Food], Never> = [
    Food(id: 0, name: "apple"),
    Food(id: 1, name: "bread", repetition: 3),
    Food(id: 2, name: "orange"),
    Food(id: 2, name: "milk")
].publisher

var timer = Timer.publish(every: 2, on: .main, in: .common).autoconnect()
var cancellables = Set<AnyCancellable>()

```

### Mixing Publishers 
- `.zip(publisher)` ou `.merge(publisher)`

```swift

foodBank.zip(timer)
foodBank.merge(//same output)

```
### Mapping: Transforming a Sequence
- `.map()` 
- `.tryMap()` 
- `.flatMap()`: convert in 1 array
- `.compactMap()`: non optional

```swift

foodBank.map(\.name)
foodBank.map({ (food, timestamp) in
    return "\(food) at \(timestamp)"
})


foodBank.flatMap{
  return Array(repeating: $0, count: $0.repetition)
}


enum SomeError: Swift.Error {
    case test
}

func toEmoji(from:String) throws -> String{
    let emojiDict: [String : String] =
        ["apple": "🍎", "bread": "🍞", "orange": "🍊", "milk": "🥛"]
    guard let emoji = emojiDict[from] else {
        throw SomeError()
    }
    return emoji
}

foodBank.tryMap{
  try toEmoji(from: $0.name)
}

```

### Subjects

#### CurrentValueSubject & PassthroughSubject

```swift
CurrentValueSubject<Int,Never>(0)
PassthroughSubject<Int,Never>()



//sink & send

.sink{
    print("completion \($0)")
} receiveValue: { value in
    print("receive value \(value)")
}.store(in: &cancellables)

.send(completion: .finished)
.send(completion: .failure(ERROR))
.send()

```

### Assign 

```swift

class MyClass{
    var anInt: Int = 0 {
        didSet{
            print("\nanInt = \(anInt)")
        }
    }
}

var myObject = MyClass()
let myRange = Range(0...10)
let subscription = myRange.publisher.assign(to: \.anInt, on: myObject)

```

### Receive
- `Receive(on:_)`

```swift 

let subscription = myRange.publisher.receive(on: DispatchQueue.main).assign(to: \.anInt, on: myObject)

```

### Subscriber Pattern

```swift

class MyClass {
    var anInt: Int = 0 {
        didSet {
            print("didSet \(anInt)")
        }
    }
}

let obj = MyClass()
let pub = (0...2).publisher //publisher
let subscriber = Subscribers.Assign(object: obj, keyPath: \MyClass.anInt) //subscriber

pub.receive(subscriber: subscriber)

```
### Mulitple Publishers

```swift

class ViewModel{
    private let userNamesSubject = CurrentValueSubject<[String], Never>(["Bill"])
    var userNames: AnyPublisher<[String], Never> //read only
    let newUserNameEntered = PassthroughSubject<String, Never>()

    init(){
      userNames = userNamesSubject.eraseToAnyPublisher()
      newUserNameEntered.sink {
          print("completion \($0)")
        } receiveValue: { userName in
            self.userNamesSubject.send(self.userNamesSubject.value + [userName])
        }.store(in: &subscriptions)
    }
}

```
### URL Request

```swift

var baseURL = URL(string: "https://jsonplaceholder.typicode.com")!
let postEndpoint = baseURL.appendingPathComponent("posts")


struct Post: Codable {
    let id: Int
    let title: String
    let userId: Int
    let body: String
}


//https://jsonplaceholder.typicode.com/posts/1 

let request = (1...10).publisher
    .map({
        postEndpoint.appendingPathComponent(String($0))
    })
    .flatMap({ url -> URLSession.DataTaskPublisher in
        return URLSession.shared.dataTaskPublisher(for: url)
    })
    .map(\.data)
    .receive(on: DispatchQueue.main)
    .decode(type: Post.self, decoder: JSONDecoder())
    .sink(receiveCompletion: { completion in
    print("completion: \(completion)")
  }) { data in
    print("received data: \(data)")
  }


```

### Erase To Any Publisher 

```swift

let subject = PassthroughSubject<Int, Never>()
let erasedSubject: AnyPublisher<Int, Never> = subject.eraseToAnyPublisher()

```

### Other publishers

- Just: Once
- Fail: finishes when fail
- Empty: finishes when empty
- Record

```swift
    Record(
        output: Array(1...3),
        completion: .failure(SomeError.test)
    )
    .scan(0) { $0 + $1 }
    .sink(
        receiveCompletion: { print($0) },
        receiveValue: { print($0) }
    )

```

### Future

A publisher that EVENTUALLY produces a single value and then finishes

```swift

let publisher = Future{ promise in 
  DispatchQueue.global(qos: .background).asyncAfter(deadline: .now() + 10){
      promise(Result.sucess("teste"))
  }
}.eraseToAnyPublisher()

publisher
.sink {
  print("completion \($0)")
} receiveValue: {
  print("received quote \($0)")
}

```
### NotificationCenter

[UPDATES SOON]

A notification dispatch mechanism that enables the broadcast of information to registered observers.

```swift

let textField = UITextField()

let sub = NotificationCenter.default
    .publisher(for: UITextField.textDidChangeNotification, object: textField)
    .sink { value in
        print("val \(value)")
      //  textMessage.value = value
}
    
DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
    print("2")
    textField.text = "hello"
}    

```

```swift

let myNotification = Notification.Name("NewItem")
let publisher = NotificationCenter.default
    .publisher(for: myNotification, object: nil)
    .sink { (value) in
        print("receive value \(value)")
        if let text = value.object as? String {
            print("object: \(text)")
        }
    }

NotificationCenter.default.post(name: myNotification, object: "great stuff")

```

```swift

var isPortraitMode: Bool = false

let center = NotificationCenter.default
let cancellable = center
    .publisher(for: UIDevice.orientationDidChangeNotification)
    .filter() { _ in UIDevice.current.orientation == .portrait }
    .sink() { _ in
        print ("Orientation changed to portrait.")
        isPortraitMode = UIDevice.current.orientation == .portrait
    }

center
    .post(Notification(name: UIDevice.orientationDidChangeNotification, object: nil, userInfo: nil))

```
