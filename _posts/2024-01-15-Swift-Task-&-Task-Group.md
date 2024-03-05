---
layout: post
title: "Swift 并发框架之Task & Task Group"
date: 2024-01-15 21:15:00.000000000 +09:00
categories: [Swift]
tags: [Swift, Concurrency, Task, Task Group]
---

### 前言

Swift 中的 `Task` 和 `Task Group` 是 WWDC 2021 引入并发编程的新特性，它们是 Swift 5.5 中引入的异步/并发编程模型的一部分。`Task`允许我们从非并发方法创建并发环境，使用 async/await 调用方法。

### Task

- `Task` 是一个表示异步操作的类型，可以通过 `async` 关键字来创建。
- 它可以用于执行异步操作，比如网络请求、文件 I/O 等。
- 可以使用 `await` 来等待 `Task` 的完成，并获取其结果。

Task 有 3 种状态：

+ **暂停 (suspended)** — 有 2 种情况会导致 Task 处于暂停状态：
  + Task 已准备就绪等待系统分配执行线程；
  + 等待外部事件，如 Task 遇到 suspension point 后可能会进入暂停状态并等待外部事件来唤醒。

> ps. 异步函数 (`A`) 调用另一个异步函数 (`B`)时，调用方会暂停，并不意味着整个 Task 会暂停。
>
> 从函数 `A` 的视角看，其会暂停等待函数 `B` 返回；
>
> 但从 Task 视角看，其不一定会暂停，可能会继续在其上执行被调用的函数 `B`；
>
> 当然，Task 也可能会被暂停，如果被调用的函数要在不同的并发上下文中执行。

+ **运行中 (running)** — Task 当前正在某个线程上运行，直至完成，或遇到 suspension point 而进入暂停状态；

+ **已完成 (completed)** — Task 所有工作都已完成。

Task 是线程的高级抽象，用于执行一项任务，Task 提供了一些高级抽象能力：

- Task 可以携带调度信息，如：任务优先级；
- Task 作为正在执行的任务的句柄 (Handle)，可以用于 cancel 等；
- Task 可以携带用户提供的 task-local data。

#### Task 创建和执行

```swift
let testTask = Task {
    return "This is a test task"
}
print(await testTask.value)
// Prints: This is a test task
```

在同步方法中去执行异步方法是有问题的，如下所示:

![task](/assets/images/2024Swift/task01.png)

```swift
func executeTask() async {
    let basicTask = Task {
        return "This is the result of the task"
    }
    print(await basicTask.value)
}
```

`executeTask() `可以通过在新的Task中调用该方法来解决上述错误：

```swift
var body: some View {
    Text("Hello, world!")
        .padding()
        .onAppear {
            Task {
                await executeTask()
            }
        }
}
```

该任务创建了一个并发支持环境，我们可以在其中调用异步方法`executeTask()`

#### Task 中 cancel

`Task`的`cancel`方法用于取消一个正在运行的任务。当调用cancel方法时，任务会收到一个取消请求，并有机会清理任何必要的资源，任务本身必须检查取消状态并相应地停止正在进行的工作。

如果任务是使用`async/await`语法创建的，可以在调用任务的`cancel`方法后立即调用get方法来等待任务完成或取消。另外，也可以在任务上调用cancel操作之前检查任务是否已经被取消，以避免不必要的取消请求。总之，Task的cancel方法提供了一种方式来请求任务取消，并且任务有责任处理这个取消请求以便安全地停止正在进行的工作。

```swift
struct ContentView: View {
    @State var image: UIImage?

    var body: some View {
        VStack {
            if let image = image {
                Image(uiImage: image)
            } else {
                Text("Loading...")
            }
        }.onAppear {
            Task {
                do {
                    image = try await fetchImage()
                } catch {
                    print("Image loading failed: \(error)")
                }
            }
        }
    }

    func fetchImage() async throws -> UIImage? {
        let imageTask = Task { () -> UIImage? in
            let imageURL = URL(string: "https://source.unsplash.com/random")!
            print("Starting network request...")
            let (imageData, _) = try await URLSession.shared.data(from: imageURL)
            return UIImage(data: imageData)
        }
        return try await imageTask.value
    }
}
```

在`imageTask`创建后立即取消它:

```swift
func fetchImage() async throws -> UIImage? {
    let imageTask = Task { () -> UIImage? in
        let imageURL = URL(string: "https://source.unsplash.com/random")!
        print("Starting network request...")
        let (imageData, _) = try await URLSession.shared.data(from: imageURL)
        return UIImage(data: imageData)
    }
    // Cancel the image request right away:
    imageTask.cancel()
    return try await imageTask.value
}

// 出现打印的结果是:
Starting network request...
Image loading failed: Error Domain=NSURLErrorDomain Code=-999 "cancelled"
```

如果在 `Task`中写了`try Task.checkCancellation()`, 那么 `Task`在检测到取消时抛出错误来停止执行当前任务：

```swift
let imageTask = Task { () -> UIImage? in
    let imageURL = URL(string: "https://source.unsplash.com/random")!

    /// Throw an error if the task was already cancelled.
    try Task.checkCancellation()

    print("Starting network request...")
    let (imageData, _) = try await URLSession.shared.data(from: imageURL)
    return UIImage(data: imageData)
}
// Cancel the image request right away:
imageTask.cancel()

// 出现打印的结果:
Image loading failed: CancellationError()
```

我们可以通过`Task`的属性 `isCancelled`判断 `Task`是否已经取消再去执行其它代码:

```swift
let imageTask = Task { () -> UIImage? in
    let imageURL = URL(string: "https://source.unsplash.com/random")!

    guard Task.isCancelled == false else {
        // Perform clean up
        print("Image request was cancelled")
        return nil
    }

    print("Starting network request...")
    let (imageData, _) = try await URLSession.shared.data(from: imageURL)
    return UIImage(data: imageData)
}
// Cancel the image request right away:
imageTask.cancel()
```

> 注意：在结构化并发中 cancel 操作会从父任务传递给所有子任务。

#### Task 设置优先级

每个`Task`都可以有其优先级，可以应用的值类似于在使用[调度队列](https://www.avanderlee.com/swift/concurrent-serial-dispatchqueue/)时可以配置[的服务质量级别](https://developer.apple.com/documentation/dispatch/dispatchqos)，低、中、高优先级看起来与[操作](https://www.avanderlee.com/swift/operations/)设置的优先级类似。配置优先级有助于防止低优先级任务避开高优先级任务的执行。

![task](/assets/images/2024Swift/task02.png)

### Task Group

Task Group是一种用于并行处理多个异步任务的结构。它允许你将多个异步任务分组，并在它们全部完成时执行一个闭包或者获取它们的结果。可以将任务组视为动态添加的多个子任务的容器。子任务可以并行或串行运行，但任务组只有在其子任务完成后才会被标记为已完成。

#### Task Group 是什么

**Task Group**:

- `Task Group` 是用来管理一组任务的结构，可以通过 `withTaskGroup` 函数创建。
- 通过 `addTask` 方法向 `Task Group` 中添加任务。
- 可以使用 `await` 来等待所有任务完成，并获取它们的结果。

通过Task Group，可以创建一个新的任务组，然后在其中添加多个异步任务。这些任务会并行执行，而不需要手动管理线程或队列。当所有任务都完成时，你可以使用`await`关键字来等待所有任务的完成，并在闭包中处理它们的结果。比如照片库下载多张图片:

```swift
await withTaskGroup(of: UIImage.self) { taskGroup in
    let photoURLs = await listPhotoURLs(inGallery: "Amsterdam Holiday")
    for photoURL in photoURLs {
        taskGroup.addTask { await downloadPhoto(url: photoURL) }
    }
}
```

#### Task Group使用

我们可以通过多种方式对任务进行分组，包括处理错误或返回最终结果集合。比如可以重写上述示例并返回图片集合：

```swift
let images = await withTaskGroup(of: UIImage.self, returning: [UIImage].self) { taskGroup in
    let photoURLs = await listPhotoURLs(inGallery: "Amsterdam Holiday")
    for photoURL in photoURLs {
        taskGroup.addTask { await downloadPhoto(url: photoURL) }
    }
    var images = [UIImage]()
    for await result in taskGroup {
        images.append(result)
    }
    return images
}
```

我们还可以通过`AsyncSequence`异步序列来等待图片加载完成并将图片附加到结果集合中，通过使用reduce运算符重写上面的代码：

```swift
let images = await withTaskGroup(of: UIImage.self, returning: [UIImage].self) { taskGroup in
    let photoURLs = await listPhotoURLs(inGallery: "Amsterdam Holiday")
    for photoURL in photoURLs {
        taskGroup.addTask { await downloadPhoto(url: photoURL) }
    }
    return await taskGroup.reduce(into: [UIImage]()) { partialResult, name in
        partialResult.append(name)
    }
}
```

#### Task Group处理错误

图像下载方法在失败时抛出错误是很常见的。可以重写示例来处理这些情况，重命名`withTaskGroup`为`withThrowingTaskGroup`：

```swift
let images = try await withThrowingTaskGroup(of: UIImage.self, returning: [UIImage].self) { taskGroup in
    let photoURLs = try await listPhotoURLs(inGallery: "Amsterdam Holiday")
    for photoURL in photoURLs {
        taskGroup.addTask { try await downloadPhoto(url: photoURL) }
    }
                                                                                          
    var images = [UIImage]()
    /// Note the use of `next()`:
    while let downloadImage = try await taskGroup.next() {
        images.append(downloadImage)
    }
    return images
}
```

#### Task Group取消

可以通过取消正在运行的任务或调用`cancelAll()`组本身的方法来取消一组任务。当使用将任务添加到已取消的组时`addTask()`，它们将在创建后直接取消。它将根据该任务是否[正确地尊重取消而](https://www.avanderlee.com/concurrency/tasks/#handling-cancellation)直接停止其工作。或者，您可以使用它`addTaskUnlessCancelled()`来阻止任务启动。

#### Task Group结果生成器

```swift
let photoURLs = try await listPhotoURLs(inGallery: "Amsterdam Holiday")
let images = try await withThrowingTaskGroup {
    for photoURL in photoURLs {
        Task { try await downloadPhoto(url: photoURL) }
    }
}
```

### Async let

`async let`是一种用于在异步上下文中声明并初始化变量的机制。使用`async let`可以在异步函数或闭包内等待一个异步操作完成，并将结果绑定到一个新的变量上。

当我们使用`async let`时，编译器会自动生成一个临时的 `Task` 来处理异步操作，并在该操作完成后将结果赋给相应的变量。这样就可以在异步任务执行过程中直接处理结果，而不需要在外部进行等待或者处理异步操作。

```swift
func fetchData() async -> String {
    return "Some data fetched from the network"
}

func processData() async {
    async let data = fetchData()
    print("Processing data: \(await data)")
}
```

`async let` 是 Swift 中的一种便捷方式，允许我们在异步上下文中方便地等待异步操作完成，并直接处理结果。

+ 对异步函数的调用不用 `await`，而是在赋值表达式的最左边加上 `async let` (第 `7~8` 行)，称之为 `async let binding`；

+ 在需要使用 `async let` 表达式的结果时要用 `await`，如结果可能会抛出错误，还需要处理错误 (第 `11~12` 行)；

+ `async let` 只能出现在异步上下文中 (Task closure、async function 以及 async closure)。

### Unstructured tasks

Unstructured tasks（非结构化任务）是一种利用底层原语直接创建和管理的任务。与使用 async/await 或者 Task API 创建结构化任务不同，非结构化任务允许开发人员以更自由的方式处理并发操作。

通过非结构化任务可以使用低级别的任务API手动创建和管理任务的执行。这意味着你可以更细粒度地控制任务的行为，例如设置特定的调度选项、传递自定义参数等。非结构化任务通常用于需要更高度定制化或者复杂性较高的并发场景。

```swift
func performCustomTask() {
    let handle = Task.runDetached {
        // 在这里编写自定义的并发操作逻辑
    }
    // 可以在之后等待任务完成或者进行其他操作
}
```

使用 `Task.runDetached` 方法创建了一个非结构化任务，该方法直接运行一个任务而不需要在 async 函数中创建。这样就可以按照具体需求自由地编写并发操作逻辑，并且可以选择是否需要等待任务的完成。

### Detached Task

Detached Task（分离任务）是一种创建和执行非结构化任务的方式。与结构化任务不同，分离任务以分离的方式执行，意味着它们可以独立于当前代码块或函数继续执行，不需要等待其完成。

通过 Detached Task，你可以创建一个任务并让它在后台运行，而无需显式地等待其完成。这使得 Detached Task 特别适合于那些需要在后台执行、并且不需要等待结果的场景。

```swift
Task.detached(priority: .background) {
    // Runs asynchronously
}
```

闭包内的代码将从父上下文异步执行，比如下面示例:

```swift
await asyncPrint("Operation one")
Task.detached(priority: .background) {
    // Runs asynchronously
    await self.asyncPrint("Operation two")
}
await asyncPrint("Operation three")

func asyncPrint(_ string: String) async {
    print(string)
}

// Prints:
// Operation one
// Operation three
// Operation two
```

> 注意: 分离运行的任务将创建一个新的操作上下文，不会继承父任务的优先级和任务本地存储，并且如果父任务被取消，它们也不会取消.

### 总结

![task](/assets/images/2024Swift/task03.webp)
