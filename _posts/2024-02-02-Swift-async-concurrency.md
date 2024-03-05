---
layout: post
title: "Swift结构化并发"
date: 2024-02-02 10:05:00.000000000 +09:00
categories: [Swift]
tags: [Swift, Concurrency, Async, Await, Task, Actor]

---

### 前言

在认识`Swift`中的异步与并发时，先回顾一下多线程.

同步和异步的区别：

- 同步：只能在当前线程中执行任务，不具备开启新线程的能力
- 异步：可以在新的线程中执行任务，具备开启新线程的能力(异步主队列不具备)

队列

- 并发队列
  - 可以让多个任务并发（同时）执行（自动开启多个线程同时执行任务）
  - 并发功能只有在异步（dispatch_async）函数下才有效
- 串行队列
  + 让任务一个接着一个地执行（一个任务执行完毕后，再执行下一个任务）

> 注意点:
>
> 同步和异步主要影响：能不能开启新的线程
>
> - 同步：只是在当前线程中执行任务，不具备开启新线程的能力
> - 异步：可以在新的线程中执行任务，具备开启新线程的能力
>
> 并发和串行主要影响：任务的执行方式
>
> - 并发：允许多个任务并发（同时）执行
> - 串行：一个任务执行完毕后，再执行下一个任务
>
> 异步 + 主队列(不具备开启新的线程) 
>
> 同步 + 主队列（死锁）
>
> - 注意 : 如果dispatch_sync方法是在主线程中调用的，并且传入的队列是主队列，那么会导致死锁
> - sync函数是在主线程中执行的，sync里面的block也是在主线程中执行。sync在主队列的前面block在后面，因为block在主队列中会等sync执行完才可以执行，但是执行sync时会执行里面的block的，但是block要等sync执行完才可以执行，所以就会导致死锁。

### Swift 异步

异步编程在 swift 开发里是一个比较常见的操作，比如经常需要在网络请求回来之后更新数据模型和视图。当异步操作嵌套时，不仅容易出现`completion` 回调逻辑错误。比如在获取缩略图的例子: 

```swift
func fetchThumbnail(id: String, completion: @escaping (UIImage?, Error?) -> Void) {
    let request = thumbnailURLRequest(for: id)
    let task = URLSession.shared.dataTask(with: request) { data, response, error in
        if let error = error {
            completion(nil, error)
        } else if (response as? HTTPURLResponse)?.statusCode != 200 {
            completion(nil, FetchError.badID)
        } else {
            guard let image = UIImage(data: data!) else {
                return
            }
            image.prepareThumbnail(of: CGSize(width: 40, height: 40)) { thumbnail in
                guard let thumbnail = thumbnail else {
                    return
                }
                completion(thumbnail, nil)
            }
        }
    }
    task.resume()
}
```

在上面的例子很容易发现在获取到image的处理中，如果没有image时是直接return了，并没有调用completion，这样外部调用就无法处理所有可能的情况。而且不调用 `completion` 是一个合法（但不符合预期）的行为，编译器不会产生错误, 补全代码后如下:

```swift
func fetchThumbnail(id: String, completion: @escaping (UIImage?, Error?) -> Void) {
    let request = thumbnailURLRequest(for: id)
    let task = URLSession.shared.dataTask(with: request) { data, response, error in
        if let error = error {
            completion(nil, error)
        } else if (response as? HTTPURLResponse)?.statusCode != 200 {
            completion(nil, FetchError.badID)
        } else {
            guard let image = UIImage(data: data!) else {
                completion(nil, FetchError.badImage)
                return
            }
            image.prepareThumbnail(of: CGSize(width: 40, height: 40)) { thumbnail in
                guard let thumbnail = thumbnail else {
                    completion(nil, FetchError.badImage)
                    return
                }
                completion(thumbnail, nil)
            }
        }
    }
    task.resume()
}
```

为了处理异步函数的结果，这里使用很多completion来通知调用方，并且调用方需要判断 `UIImage?` 和 `Error?` 的 4 种组合结果，但实际上有效的只有 2 种。调用方可以知道结果，要么成功，要么失败。为了优化异步函数调用，Swift 5.5 引入了 `async` 和 `await`。

为了解决 `completion` 被忽略调用的情况，可以尝试使用 Swift 5.5 新增的 `async` 和 `await` 来解决这个问题，代码如下：

```swift
func fetchThumbnail(id: String) async throws -> UIImage {
    let request = thumbnailURLRequest(for: id)
    let (data, response) = try await URLSession.shared.data(for: request)
    guard (response as? HTTPURLResponse)?.statusCode == 200 else { throw FetchError.badID }
    let maybeImage = UIImage(data: data)
    guard let thumbnail = await maybeImage?.thumbnail else { throw FetchError.badImage }
    return thumbnail
}
```

![01](/assets/images/2024Swift/structure-concurrency-1.png)

上图当 `thumbnailURLRequest` 调用完成之后，返回 `fetchThumbnail` 继续执行后续代码，如果这个函数是一个耗时任务，那么当前线程就会持续等待，直到完成。

![02](/assets/images/2024Swift/structure-concurrency-2.png)

上图是一个异步函数调用，当 `data(for:)` 调用后，函数被挂起进行等待，当任务完成后，系统恢复 `data(for:)` 的调用，返回 `fetchThumbnail` 继续执行后续代码。

> `async` / `await` ：
>
> 1. `async` 允许一个函数被挂起；
> 2. `await` 标记一个异步函数的潜在暂停点；
> 3. 在挂起期间可能会进行其他工作；
> 4. 一旦等待的异步调用完成，在 `await` 之后恢复执行。

### swift 并发

当我们了解完Swift 异步函数处理后，这一节我们就开始了解处理多个缩略图，如下代码.

```swift
func fetchThumbnails(ids: [String], completion handler: @escaping ([String: UIImage]?, Error?) -> Void) {
    guard let id = ids.first else { return handler([:], nil) }
    let request = thumbnailURLRequest(for: id)
    let dataTask = URLSession.shared.dataTask(with: request) { data, response, error in
        guard let response = response,
              let data = data
        else {
            return handler(nil, error)
        }
        UIImage(data: data)?.prepareThumbnail(of: thumbSize) { image in
            guard let image = image else {
                return handler(nil, ThumbnailFailedError())
            }
            fetchThumbnails(for: Array(ids.dropFirst())) { thumbnails, error in
                // 添加图片到 thumbnails
            }
        }
    }
    dataTask.resume()
}
```

上述代码看起来非常离谱，这里是异步获取到第一张图后移除掉ids数组第一个id，然后继续调用**fetchThumbnails**方法，代码可能性非常差，也比较难维护。

使用`async` 和 `await` 处理获取多个缩略图，代码如下:

```swift
func fetchThumbnails(ids: [String]) async throws -> [String: UIImage] {
    var thumbnails: [String: UIImage] = [:]
    for id in ids {
        let request = thumbnailURLRequest(for: id)
        let (data, response) = try await URLSession.shared.data(for: request)
        try validateResponse(response)
        guard let image = await UIImage(data: data)?.byPreparingThumbnail(ofSize: thumbSize) else {
            throw ThumbnailFailedError()
        }
        thumbnails[id] = image
    }
    return thumbnails
}
```

### Swift 结构化并发

在上述例子中的`thumbSize`是本地提供的，处理起图片大小是比较简单的；如果`thumbSize`也是从网络接口获取的话，那么生成缩略图的过程中有两个请求。代码如下:

```swift
func fetchOneThumbnail(id: String) async throws -> UIImage {
    let imageReq = imageRequest(for: id)
  	let metadataReq = metadataRequest(for: id)
    let (data, _) = try await URLSession.shared.data(for: imageReq)
    let (metadata, _) = try await URLSession.shared.data(for: metadataReq)
    guard let size = parseSize(from: metadata),
          let image = await UIImage(data: data)?.byPreparingThumbnail(ofSize: size)
    else {
        throw ThumbnailFailedError()
    }
    return image
}
```

在 `imageReq` 请求完成之后，才发起 `metadataReq` 请求，只有两个请求都完成之后，才能执行之后的代码。如果在两个请求和使用他们的返回结果之间有其他的无关任务，无关任务的执行必须等到两个请求完成之后，这无疑是一种资源浪费。这里引入`async-let` 来解决这个问题，代码如下:

```swift
func fetchOneThumbnail(id: String) async throws -> UIImage {
    let imageReq = imageRequest(for: id)
  	let metadataReq = metadataRequest(for: id)
    async let (data, _) = URLSession.shared.data(for: imageReq)
    async let (metadata, _) = URLSession.shared.data(for: metadataReq)
    guard let size = parseSize(from: try await metadata),
          let image = try await UIImage(data: data)?.byPreparingThumbnail(ofSize: size)
    else {
        throw ThumbnailFailedError()
    }
    return image
}
```

使用 `async let` 标记 `data` 和 `metadata`，使用 `await` 进行访问，如果函数可以抛出错误，那么还需要使用 `try` 关键字。两个网络请求会同时进行，并且执行后续代码，即上文提到的无关任务。当需要访问结果的时候，系统会进行等待直到完成或者抛出错误。这样便能更高效地完成整个任务。

### Task & Task Group

+ `Task`代表一个异步执行的任务单元。开发者可以创建、取消、等待和管理任务，`Task` 提供了对异步操作的抽象和控制.
+ `Task Group`关键字用于创建一个任务组，该任务组可以同时运行多个任务，并且可以等待它们全部完成后再进行下一步操作。使用`Task Group`关键字可以更方便地管理和协调多个并发任务。

在`for-in` 内部的逻辑就是处理单个缩略图的逻辑，我们希望任务取消后返回之前已经处理的缩略图，可以使用 `isCancelled` 进行判断，代码如下：

```swift
func fetchThumbnails(ids: [String]) async throws -> [String: UIImage] {
    var thumbnails: [String: UIImage] = [:]
    for id in ids {
        if Task.isCancelled { break }
        thumbnails[id] = try await fetchOneThumbnail(withID: id)
    }
    return thumbnails
}
```

除了用Task来处理获取单个缩略图的逻辑外，我们还可以使用任务组进行处理，即使图片数据和元数据已经可以同时请求了，但是对于整个获取缩略图这个任务而言，仍然是一个接一个进行的，如果我们想要同时进行多个任务，就需要引入任务组进行并发编程，代码如下：

```swift
func fetchThumbnails(ids: [String]) async throws -> [String: UIImage] {
    var thumbnails: [String: UIImage] = [:]
    try await withThrowingTaskGroup(of: (String, UIImage).self) { group in
        for id in ids {
            group.async {
                return (id, try await fetchOneThumbnail(withID: id))
            }
        }
        for try await (id, thumbnail) in group {
            thumbnails[id] = thumbnail
        }
    }
    return thumbnails
}
```

使用任务组可以非常方便地执行并发任务，线程切换和派发任务都由 Swift 进行管理，不需要我们编写复杂的控制逻辑。

### Actor

+ `Actor`代表一个并发安全的实体，其中包含了自己的状态和行为。`Actor` 可以确保其内部状态在并发环境下的安全访问，并且提供了一种结构化并发编程的方式。

在了解了Swift结构化并发之后，语法简洁而且高效，但是在并发的过程中，如果没有采取手段加以控制，很容易会产生数据竞争的问题。

```swift
class Counter {
    var value = 0
    func increment() -> Int {
        value = value + 1
        return value
    }
}
let counter = Counter()
let queue1 = DispatchQueue(label: "queue_1")
let queue2 = DispatchQueue(label: "queue_2")

queue1.async {
    print(counter.increment())
}
queue2.async {
    print(counter.increment())
}
```

上述例子有可能是`1,1`、`1,2`、`2,1`等，是取决于value的写入和读取的时机。那么共享的可变状态引发的数据竞争，通常我们使用锁来解决数据竞争的问题，但是如果锁使用不当的话会引发死锁。所以可以使用 Swift 5.5 新增的 `Actor` 来解决数据竞争的问题。

`Actor` 是一种并发模型，由状态（State）、行为（Behavior）、邮箱（Mailbox）三者组成，它可以保证在并发环境中，可变状态被访问的安全性，它只在内部维护自己的可变状态，外部访问 Actor 内部数据的时候，会按顺序来处理。

在 Swift 中 `Actor` 是和 `Class` 一样是引用类型，也有属性，方法，下标，遵守协议等功能，但是在静态方法中是没有 self 的，所以不涉及到数据的隔离。`Actor` 和 `Class` 区别在于 Actor 遵守了 Actor 协议，可以保证在并发环境下操作数据的安全性，而且 Actor 不支持继承，使得 Actor 使用起来很简单。所以 Actor 没有重写方法的能力。

+ `State`状态：`Actor` 持有的变量，由自身管理，避免并发环境下的锁问题
+ `Behavior`行为：`Actor` 中的计算逻辑，通过 `Actor` 接收到的消息来改变自身的状态
+ `Mailbox`邮箱：`Actor` 之间通讯的桥梁，内部使用 `FIFO` 队列来存储和处理消息，接收方从邮箱中获取消息

`Actor` 模型所有 `actor` 状态都是本地的，外部是无法访问的；`actor` 之间必须通过消息传递进行通讯；一个 `actor` 可以响应消息、退出新的 `actor`、改变内部状态、把消息发送给一个或多个 `actor`；`actor` 可能会阻塞自己但是不应该阻塞运行的线程。

使用Actor来解决数据竞争问题，代码如下:

```swift
actor Counter {
    var value = 0
    func increment() -> Int {
        value = value + 1
        return value
    }
}
let counter = Counter()

asyncDetached {
    print(await counter.increment())
}
asyncDetached {
    print(await counter.increment())
}
```

### 总结

+ 由于异步代码的控制流难以编写，难以阅读，Swift 引入 `async` / `await` 解决异步编程的问题。
+ `async` / `await` 并不具有并发性，异步不代表并发，Swift 引入结构化并发让 `async` / `await` 编写的代码执行并发任务，也解决一些控制流上的问题。
+ 异步与并发虽好，随之而来的是数据竞争，Swift 引入 `actor` 来解决共享可变状态的问题，状态被隔离在 `actor` 内部，修改只能通过向 `actor` 发送信息并等待结果，消息被 `actor` 以同步的方式进行处理，避免同一时间对数据进行修改。