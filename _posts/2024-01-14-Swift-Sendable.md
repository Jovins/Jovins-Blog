---
layout: post
title: "Swift 并发框架之Sendable"
date: 2024-01-14 21:15:00.000000000 +09:00
categories: [Swift]
tags: [Swift, Concurrency, Sendable]
---

### 前言

在 Swift 中，`Sendable` 是一个协议，用于声明遵循该协议的类型是“可发送的”。具体来说，如果一个类型被声明为 `Sendable`，那么就可以安全地从一个任务（task）传递到另一个任务，而无需担心数据竞争或并发访问的问题。这样可以帮助开发者标识出哪些类型可以安全地在不同的并发执行上下文中传递和共享。

### Sendable

```swift
/// The Sendable protocol indicates that value of the given type can
/// be safely used in concurrent code.
public protocol Sendable {}
```

`Sendable` 是一个空协议，用于向外界声明实现了该协议的类型在并发环境下可以安全使用，更准确的说是可以自由地跨 `actor` 传递。`Sendable` 有一个专属名称 `Marker Protocols`，特征主要有:

+ 具有特定的语义属性 (semantic property)，并且是编译期属性而非运行时属性
  + `Sendable` 的语义属性就是要求并发下可以安全地跨 actor 传递
+ 协议体必须为空
+ 不能继承自 non-marker protocols
+ 不能作为类型名用于 `is`、`as?`等操作
+ 不能用作泛型类型的约束

比如值类型在 传递是是会执行拷贝操作的，也就是跨`Actor`传递是安全的，所以这些类型隐式遵守了`Sendable`.

+ 基础类型，`Int`、`String`、`Bool`
+ 所含元素类型符合 `Sendable` 协议的集合，如：`Array`、`Dictionary` 等
+ 不含有引用类型成员的 `struct`、引用类型关联值的 `enum`

所有 `actor` 也是自动遵守了`Sendable` 协议

```swift
@available(macOS 10.15, iOS 13.0, watchOS 6.0, tvOS 13.0, *)
public protocol Actor : AnyObject, Sendable {
    nonisolated var unownedExecutor: UnownedSerialExecutor { get }
}
```

`class` 遵循 `Sendable` 协议，并有以下限制 (确保实现了 `Sendable` 协议的类数据安全的)：

> + class 必须是 final，否则有 Warning: Non-final class 'X' cannot conform to 'Sendable'; use ' @unchecked Sendable'
> + class 的存储属性必须是 immutable，否则有 Warning: Stored property 'x' of 'Sendable'-conforming class 'X' is mutable
> + class 的存储属性必须都遵守 Sendable 协议，否则 Warning: Stored property 'y' of 'Sendable'-conforming class 'X' has non-sendable type 'Y'
> + class 的祖先类 (如有) 必须遵守 Sendable 协议或者是 NSObject，否则 Error: 'Sendable' class 'X' cannot inherit from another class other than 'NSObject'

比如一下示例:

```swift
class User {
  var name: String
  var age: Int
}
extension UserManager {
  func user() async -> User {
    // Warning: Non-sendable type 'User' returned by implicitly asynchronous call to actor-isolated instance method 'user()' cannot cross actor boundary
    return await bankAccount.user()
  }
}

// 修改
struct User {
  var name: String
  var age: Int
}
// 或者
final class User: Sendable {
  let name: String
  let age: Int
}
```

### @Sendable

`@Sendable` 是一个属性包装器，被 `@Sendable` 修饰的函数、闭包可以跨 `actor` 传递。通过将 `@Sendable` 应用于闭包类型，可以确保这些闭包可以安全地在不同的任务之间传递，并且不会引起数据竞争或并发访问的问题。

```swift
actor BankAccount {
    let accountNumber: Int
    var balance: Double

    init(accountNumber: Int, initialDeposit: Double) {
        self.accountNumber = accountNumber
        self.balance = initialDeposit
    }
 
    func addBalance(amount: Double, completion: @Sendable (Double) -> Void) {
        balance += amount
        completion(balance)
    }
}

class AccountManager {
    let bankAccount = BankAccount.init(accountNumber: 123456789, initialDeposit: 1000)
    func addAge() async {
        // Wraning: Non-sendable type '(Int) -> Void' passed in implicitly asynchronous call to actor-isolated instance method 'addAge(amount:completion:)' cannot cross actor boundary
        await bankAccount.addBalance(amount: 52, completion: { balance in
            print(balance)
        })
    }
}
```

用`@Sendable` 修饰 Closure，告诉 `Closure` 可能会在并发环境下调用，请注意数据安全！当然编译器会对 @Sendable Closure 的实现进行各种合规检查：

+ 不能捕获 actor-isolated 属性，否则 Error: Actor-isolated property 'x' can not be referenced from a Sendable closure
+ 不能捕获 `var` 变量，否则 Error: Mutation of captured var 'x' in concurrently-executing code
+ 所捕获对象必须实现 Sendable 协议，否则 Warning: Capture of 'x' with non-sendable type 'X' in a `@Sendable` closure。

### 总结

`Sendable` 和 `@Sendable` 提供了一种在并发代码中明确声明类型和闭包是否是“可发送的”的方式，从而帮助开发者编写更安全、更可靠的并发代码

+ `Sendable` 是一个 Marker Protocol，用于编译期的合规检查
+ 所有值类型都自动遵守 `Sendable` 协议
+ 所有遵守 `Sendable` 协议的类型都可以跨 actor 传递
+ `@Sendable` 用于修饰方法、闭包
+ 在并发环境下执行的闭包都应用 `@Sendable` 修饰
