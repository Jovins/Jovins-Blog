---
layout: post
title: "Swift 并发框架之Actor"
date: 2024-01-05 16:25:00.000000000 +09:00
categories: [Swift]
tags: [Swift, Concurrency, Actor]

---

### 前言

Swift Actors 是 Swift 5.5 中是一项新的语言特性，旨在帮助开发人员更容易地编写并发代码。Actors 可以让多个任务同时访问一个对象，同时保证线程安全和数据完整性。本文将详细介绍 Swift 中的 Actors，包括如何定义、如何使用以及如何避免数据竞争。

### 什么是Actor

在受到 [Actor Model](https://link.juejin.cn/?target=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FActor_model) 的启发后，Swift 在 [SE-0306](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fapple%2Fswift-evolution%2Fblob%2Fmain%2Fproposals%2F0306-actors.md) 提案引入了 Actor，其主要的目的是解决数据竞争问题。

当多个线程在没有同步的情况下访问同一内存，并且至少有一个访问是写的时候，就会发生数据竞争。数据竞争会导致不可预测的行为、内存损坏、不稳定的测试和奇怪的崩溃等等问题。那么可能会遇到无法解决的崩溃问题，因为你不知道它们何时发生，如何重现它们，或者如何根据理论来修复它们。

`Actor` 是一种支持并发操作的对象，它封装了一些数据和行为，并且可以被多个任务同时访问。与传统的共享内存并发模型不同，`Actor` 模型使用消息传递来实现并发，每个 `Actor` 都有自己的状态，在处理消息时不会影响其他 Actors 的状态。Actors 不仅提供了并发安全，还可以有效地降低锁的使用，提高程序的性能。

在 Swift 中，`Actor` 被定义为一个类或结构体，并使用 `actor` 关键字修饰 (不能继承)。`Actor` 类或结构体中包含一些属性和方法，这些属性和方法只能由 actor 自身或者其他 actor 访问。非 actor 对象无法直接访问 Actor 的属性和方法。

### 怎样定义 Actor

定义一个 Actor 很简单，只需要在类或结构体前面加上 `actor` 关键字即可，如下示例:

```swift
actor TestActor {
    var count = 0
    func increment() async {
        count += 1
    }
}
```

### 如何防止数据竞争

`Actor` 通过创建对其隔离数据的同步访问来防止数据竞争。在Actors之前，我们会使用各种锁来创建相同的结果。这种锁的一个例子是并发调度队列与处理写访问的屏障相结合。下面是一个银行账户存款、取款的示例:

```swift
final class BankWithQueue {
    let name = "test"

    private var _money: Int = 0
    var money: Int {
        queue.sync {
            _money
        }
    }
    private var queue = DispatchQueue(label: "bank.queue", attributes: .concurrent)
 
    func depositMoney(count: Int) {
        queue.sync(flags: .barrier) {
            _money += count
        }
    }
 
    func drawalMoney(count: Int) {
        queue.sync(flags: .barrier) {
            _money -= count
        }
    }
}
```

如果我们使用`Actor` 来定义银行取款、存款来实现上述的例子，示例如下:

```swift
actor BankDeposit {
	let name: String = "test"
  var money: Int = 0

  func depositMoney(count: Int) {
    money += count
  }
 
  func drawalMoney(count: Int) {
    money -= count
  }
}
```

上述示例可以看到，这个实例更简单，更容易阅读。所有与同步访问有关的逻辑都被隐藏在Swift标准库中的实现细节里。但是在同步线程中读取`BankDeposit`里面的属性或者方法时，都会报这个错误。

![actor](/assets/images/2024Swift/actor01.png)

### Actor 使用

可以使用 `await` 关键字来调用 Actor 的异步方法，例如：

```swift
Task {
    let bank = BankDeposit()
    await bank.depositMoney(count: 100)
    print(await bank.money)
}
```

![actor](/assets/images/2024Swift/actor02.png)

尽管 Actors 可以提供并发安全，但在实际使用中仍然需要注意一些细节，以避免数据竞争和其他并发问题。

### 为什么会出现数据竞争

为什么在使用 Actors 时仍会出现数据竞争？

当在代码中持续使用 `Actor`时，这样肯定会降低遇到数据竞争的风险。创建同步访问可以防止与数据竞争有关的奇怪崩溃。然而，你显然需要持续地使用它们来防止你的应用程序中出现数据竞争。但是如果在两个线程使用 `await`正确地访问我们的 Actor 的数据：

```swift
queueOne.async {
    await bank.depositMoney(count: 100)
}
queueTwo.async {
    print(await bank.money)
} 
```

这里的竞争条件定义为：“哪个线程将首先开始隔离访问？”。所以基本上有两种结果：

- 队列一在先，增加吃食的鸡的数量。队列二将打印：100
- 队列二在先，打印出吃食的鸡的数量，该数量仍为：0

在不同之处修改数据时不再访问数据。如果没有同步访问，在某些情况下这可能会导致无法预料的行为。

### Actor 关键字

Swift 中的 Actor 是一种并发编程模型，用于管理并发代码执行。Actor 允许我们在保持数据封装的同时，将并发操作限制在特定的作用域内。

通常我们使用`Actor`时会遇到以下错误:

+ Actor-isolated property ‘balance’ can not be referenced from a non-isolated context

+ Expression is ‘async’ but is not marked with ‘await’

> 根本原因：Actor 隔离对其属性的访问以确保互斥访问。

#### isolated关键字

isolation 用于标记 Actor 的成员属性和方法是否可以从其他地方访问，以确保线程安全性。使用 isolation 关键字可以明确指定哪些代码是可被异步调用的，从而帮助开发者更好地控制并发代码的执行。

比如银行账户转账为例:

```swift
actor BankAccountActor {
    enum BankError: Error {
        case insufficientFunds
    }
   
    var balance: Double
    
    init(initialDeposit: Double) {
        self.balance = initialDeposit
    }
    
    func transfer(amount: Double, to toAccount: BankAccountActor) async throws {
        guard balance >= amount else {
            throw BankError.insufficientFunds
        }
        balance -= amount
        await toAccount.deposit(amount: amount)
    }
    
    func deposit(amount: Double) {
        balance = balance + amount
    }
}
```

Actor 方法默认是隔离的，但没有明确标记为隔离。如果在方法前面用`isolated`标记时就会报错:

```swift
isolated func transfer(amount: Double, to toAccount: BankAccountActor) async throws {
    guard balance >= amount else {
        throw BankError.insufficientFunds
    }
    balance -= amount
    await toAccount.deposit(amount: amount)
}

isolated func deposit(amount: Double) {
    balance = balance + amount
}

// ‘isolated’ may only be used on ‘parameter’ declarations
```

只能是将`Actor`参数标记隔离，对参数使用隔离关键字可以很好地使用更少的代码来解决特定问题。

```swift
// 编译器目前禁止但允许使用多个隔离参数
func transfer(amount: Double, to toAccount: isolated BankAccountActor) async throws {
    guard balance >= amount else {
        throw BankError.insufficientFunds
    }
    balance -= amount
    toAccount.balance += amount
}
```

#### nonisolated 关键字

nonisolated 关键字可以应用于 Actor 的非隔离成员属性和方法，以明确指示它们允许从 Actor 外部进行异步调用。可以更加灵活地控制并发代码的执行方式，同时仍然确保了数据封装性和线程安全性。

```swift
actor BankAccountActor {
    let accountHolder: String
    let bank: String
    var details: String {
        "Bank: \(bank) - Account holder: \(accountHolder)"
    }
    // ...
}
```

帐户持有人是不可变的，可以安全地从非隔离环境访问，但是如果引入计算属性`details`访问不可变属性，编译器就会识别出一下错误:

> Actor-isolated property ‘details’ can not be referenced from a non-isolated context

accountHolder 和 bank 都是不可变属性，我们可以引入关键字 `nonisolated`标记 details 属性来解决错误:

```swift
actor BankAccountActor {
    let accountHolder: String
    let bank: String
    nonisolated var details: String {
        "Bank: \(bank) - Account holder: \(accountHolder)"
    }
    // ...
}
```

### MainActor

在Swift编程语言中，`MainActor` 是在并发编程中引入的一个新概念。它是用于标识代码片段应该在主线程上执行的一种机制。在 Swift 5.5 中引入了 `MainActor`，通过将它应用到特定的函数、方法或属性上，可以确保这些被标记的代码片段只会在主线程上执行，从而帮助开发者更轻松地处理多线程编程中的问题，避免由于线程不安全而导致的 bug 和异常情况。

#### 什么MainActor

MainActor 是一个全局唯一的 Actor，在主线程上执行任务，作为其全局 Actor 的一个例子，继承了`GlobalActor`协议；全局 Actor 注释的类的子类必须与同一个全局 Actor 隔离。

```swift
@MainActor
final class MyMainActor {
    @MainActor
    var mainThreadData: Int {
        get {
            // 从主线程获取数据
            return 5
        }
    }
    @MainActor
    func performTask() async {
        // 在这里执行需要在主线程上完成的任务
    }
}
let actorInstance = MyMainActor()
await actorInstance.performTask() // 确保在主线程上执行
let data = actorInstance.mainThreadData // 确保从主线程获取数据
```

### 总结

在Swift中，`Actor` 是一种并发编程模型，用于管理共享状态。`Actor` 允许你定义一个包含异步方法和属性的实体，这些方法和属性只能由一个线程同时访问。在Actor内部，所有的操作都是串行执行的，从而避免了常见的并发问题，如数据竞争和死锁。通过使用 `Actor`，可以更轻松地编写安全的并发代码，并且在处理共享状态时不需要显式地使用诸如锁或信号量之类的同步原语。