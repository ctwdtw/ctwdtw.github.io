---
title: "Swift 中的 Type Erasure 與 Protocols With Associated Types"
date: 2023-06-26T08:56:34+08:00
draft: false
---
### 摘要：
本篇筆記介紹如何使用 `Wrapper Object` 和 `Method Dispatch` 的技巧來實現 `Type Erasure`, 解決 `Protocols With Associated Types` 不能被當作型別來使用的問題。筆記在最後將提出的解法和 Swift 5.1 與 Swift 5.7 提供的 `Opaque Type` 和 `Boxed Protocol Type` 做了一個類比。

### 狀況：
`Protocols With Associated Types` 是 Swift 在 `Protocol` 中支持 `Generic` 的語法特性之一，在一些情況下，他可能會很有用，例如我們要對一個 `Scrum Team` 的 Members 做建模，`Protocols With Associated Tyes` 可以幫我們對不同的 Team Member 建立統一的抽象：
```Swift
protocol Worker {
    associatedtype Skill
    func doWork()
}
```
不同的 Team Member 都可以遵循這個統一的 `Protocol`:
```Swift

struct iOS {
    ...
    ...
}

struct JunioriOSDeveloper: Worker {
    typealias Skill = iOS

    static func new() -> JunioriOSDeveloper {
        JunioriOSDeveloper()
    }

    func doWork() {
        print("***: I am good at \(Skill.self) programming.")
    }
    
    func sayWTF() {
        print("***: What The Fuck, this code sucks!")
    }
}

struct SenioriOSDeveloper: Worker {
    typealias Skill = iOS

    static func new() -> SenioriOSDeveloper {
        SenioriOSDeveloper()
    }

    func doWork() {
        print("***: I am good at \(Skill.self) programming, and I know how to properly use software design principles")
    }
    
    func fixDiryCodeBase() {
        print("***: This code sucks, and I now how to fix it!")
    }
}

struct CoordinateResource {
    ...
    ...
}

struct ScrumMaster: Worker {
    typealias Skill = CoordinateResource

    static func new() -> ScrumMaster {
        ScrumMaster()
    }

    func doWork() {
        print("***: I am gootd at \(Skill.self), and that's my main responsibility.")
    }
}
```
然而，當我們想把 `Worker` `protocol` 當作陣列的型別，以便將 `SenioriOSDeveloper`, `ScrumMaster` 等物件放入陣列中使用 forEach 的方式對陣列中對每個物件做 `doWork()` 的訊息傳遞時，Swift 5.1 以前的版本會在 compiler time 報出 `Protocol can only be used as a generic constraint because it has Self or associatedType requirements` 的錯誤訊息：
```Swift
 var scrumTeams = [Worker]() // compiler error
```
Swift 5.1 和 Swift 5.7 分別引入了 `some` 和 `any` 這兩個關鍵字來修正 `protocol with associated type` 不能用來當作型別的限制，本篇筆記著重在使用 `Wrapper Object` 和 `Method Dispatch` 的方法來解決以上問題，並會將結果稍微和 `some`, `any` 稍微做一點比較。

### 行動：
既然 `Worker` 不能被當作型別，只能用來做泛型的 constraint, 而我們的目標又是能夠對物件做 `doWork()` 的訊息傳遞，我們就可以引入以下的 `Generic Wrapper Object`, 利用這個 `Genric Wrapper Object` 來當作 Array 的型別：
```Swift
struct SomeWorker<T: Worker> {
    private let _doWork: () -> Void
    init(_ worker: T) {
        _doWork = worker.doWork
    }

    func doWork() {
        _doWork()
    }
}

var scrumTeam: [SomeWorker] = [SomeWorker(JunioriOSDeveloper())] // build success
```
然而，當我們想更進一步在 scrumTeam 中加入 `SenioriOSDeveloper` 的時候，compiler 會再次向我們抱怨：
`Conflicting arguments to generic parameter 'T' ('JunioriOSDeveloper' vs. 'SenioriOSDeveloper')` 原因是 `SomeWorker` 的 generic type T 應該是在 compiler time 要被被唯一決定的，在陣列任中給定兩個不同的 generic type T 會導致 compiler 無法給出唯一的型別推斷，於是 compiler 再次向我們抱怨。

既然我們關心的是訊息的傳遞，不是具體的型別本身，而 compiler 又向我們抱怨型別不唯一，我們的目標就是把它擦去 (本篇標題 Type Erasure), 自然的做法，就是把 genric constraint 從泛型結構中拿掉，移至建構子中：
```Swift
struct AnyWorker {
    private let _doWork: () -> Void
    init<W: Worker>(_ worker: W) {
        _doWork = worker.doWork
    }
    
    func doWork() {
        _doWork()
    }
}
```
於是，我們得到一個非泛型的結構 `AnyWorker` 來當作 scrumTeam 陣列的型別資訊，並且藉由建構子的 generic constraint, 我們能夠捕獲並轉傳不同具體型別的 `doWork()` 訊息，而不需要讓陣列容器知道這些訊息對應的具體型別為何 (被我們擦去了)：
```Swift
var scrumTeam: [AnyWorker] = [AnyWorker(JunioriOSDeveloper()), AnyWorker(SenioriOSDeveloper())]
scrumTeam.append(AnyWorker(ScrumMaster()))
scrumTeam.forEach {
    $0.doWork()
}

// output
***: I am good at iOS programming.
***: I am good at iOS programming, and I know how to properly use software design principles
***: I am gootd at CoordinateResource, and that's my main responsibility.

```

### 結果：
藉由引入 `AnyWorker` 我們將 `Worker` 提供的抽象上再加入一層抽象，擦去了遵循 `Worker` 具體物件的型別，並讓具體型別的 `doWork()` 訊息可以被 `AnyWorker` 轉傳，解決了 `Worker` 這種有 `associated type` 的 `Protocol` 不能被當作 `Type` 的問題。

`SomeWorker` 也可以做到類似的效果，解決 `Protocol With Associated Type` 不能被當作 `Type` 的問題。但因為 `SomeWorker` 是一個泛型結構，會保留建構子傳入的具體型別的資訊，因此無法將不同具體型別的 `SomeWorker` 放入陣列中。

事實上，如果閱讀 Swift 的[官方文件](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/opaquetypes)，我們會發現 `AnyWorker` 和 `SomeWorker` 的一些特性 Swift 5.1 以及 Swift 5.7 引入的 `Opaque Type` (the `some` keyword) 與 `Boxed Protocol Type` (the `any` keyword)有些類似：

> opaque type hides its return value’s type information. Instead of providing a concrete type as the function’s return type, the return value is described in terms of the protocols it supports. Opaque types preserve type identity — the compiler has access to the type information, but clients of the module don’t.

> boxed protocol type can store an instance of any type that conforms to the given protocol. Boxed protocol types don’t preserve type identity — the value’s specific type isn’t known until runtime, and it can change over time as different values are stored.

Opaque Type 保留了 type identity, compiler 知道 type information, 但是 client code 不知道，這和 `SomeWorker` 轉傳 `doWork()` 訊息，但 compiler 知道底層物件的具體型別的特性類似。

Boxed Protocol type 不保留 type identity 的資訊，這和 `AnyWorker` 轉傳 `doWork()` 訊息，且 compiler 不再乎底層物件的具體型別的特性類似。

然而，以上說明僅是類比，並非嚴格的一一對應關係，關於 `Opaque Type` 和 `Boxed Protocol Type` 的理解，將在之後的筆記討論。

### 參考資料：
Opaque and Boxed Types - https://docs.swift.org/swift-book/documentation/the-swift-programming-language/opaquetypes

David Lin on Type Erasure - https://davidlinnn.medium.com/type-erasure-1c74ba7130d5