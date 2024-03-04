---
layout: post
title: "WWDC 2021 Swift中的Async & Await"
date: 2024-02-04 10:05:00.000000000 +09:00
categories: [Swift]
tags: [Swift, Concurrency, Async, Await, Task]
---

### 概览

在 WWDC 2021 中，Swift 迎来了一次非常重要的版本更新`Swift 5.5`。这次更新为 Swift 并发编程带来了很大的改变，通过Structured concurrency（[SE-0304](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fapple%2Fswift-evolution%2Fblob%2Fmain%2Fproposals%2F0304-structured-concurrency.md)）、 `async/await`（[SE-0296](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fapple%2Fswift-evolution%2Fblob%2Fmain%2Fproposals%2F0296-async-await.md)) 、以及 Actors （[SE-0306](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fapple%2Fswift-evolution%2Fblob%2Fmain%2Fproposals%2F0306-actors.md)），让开发者可以在更抽象的层面上思考并发场景的解决方式，同时保障了并发场景下的性能和安全性，避免了使用 GCD 等传统并发模型时可能出现的多线程问题。

在学习`async`&`await`之前，先了解一下什么是同步执行和异步执行。

+ 线程代码是按照先后顺序依次执行，前面代码未执行完成时不会执行后续的代码。
+ 执行异步线程代码的时候，会同时继续执行异步代码和后续的同步代码，当异步代码完成时线程就会调用回调函数中的代码。

> 注意: 如果想了解更详细可以看WWDC [Meet async/await in Swift](https://developer.apple.com/videos/play/wwdc2021/10132/) 视频。

### 异步如何取代闭包

![async](/assets/images/2024Swift/async00.png)

如上图是一个获取缩略图的流程，具体的代码如下所示。可用于多次使用回调处理，显示缩略图这个简单的任务对应的代码，但是这段代码是有点问题。

```swift
func fetchThumbnail(id: String, completion: @escaping (UIImage?, Error?) -> Void) {
   let request = thumbnailURLRequest(for: id) 
    let task = URLSession.shared.dataTask(with: request) { data, response, error in
        if let error = error {
            completionHandler(nil, error)
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

在两处`guard/return`代码里，没有针对发生错误的场景执行`completion`回调，其结果可能是图片的 loading 动画一直转，就是不显示图。从业务角度看，`fetchThumbnail`函数的任何一个 return 都应该执行`completion`回调；但从编译层面，Swift 没办法保证return前都执行了回调处理，即我们不能借助Swift的错误处理机制在编译时发现这样的问题。以下是修复 bug 之后的代码。

```swift
func fetchThumbnail(for id: String, completion: @escaping (UIImage?, Error?) -> Void) {
    let request = thumbnailURLRequest(for: id) 
    let task = URLSession.shared.dataTask(with: request) { data, response, error in
        if let error = error {
            completionHandler(nil, error)
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

在 Swift 中仍然可以使用完成闭包定义方法，但它有一些缺点，可以通过使用 async 来解决：

- 确保在每个可能的方法出口中调用完成闭包。不这样做可能会导致应用程序无休止地等待结果。
- 闭包更难阅读。与结构化并发相比，推断执行顺序并不那么容易。
- [使用弱引用](https://www.avanderlee.com/swift/weak-self/) 需要避免保留循环。
- 实施者需要切换结果才能得到结果。不可能从实现级别使用 try catch 语句。

### 什么是 Async & Await

`async` : enable a function to suspend

`await`: marks where an async function may suspend execution

Other work can happen during a suspension, Once an awaited async call completes, execution resumes after the await.

`Async` 代表异步，可以看作是方法属性，明确表明方法执行异步工作；`Await` 是用于调用异步方法的关键字，表明正在等待他的好友异步回调。

+ 当将一个函数标记为异步时，就可以允许它挂起
+ 当函数挂起自身时，它的调用者也会挂起，所以它的调用者也是异步的

+ 为了指出异步函数中何处可能挂起一次或者多次，使用了`await`关键字
+ 当异步函数被挂起时，线程不会被阻塞

用`async/await`这种方法改造的示例如下：

```swift
func fetchImages(id: String) async throws -> [UIImage] {
    let request = thumbnailURLRequest(for: id)
    let (data, response) = try await URLSession.shared.data(for: request)
    guard (response as? HTTPURLResponse)?.statusCode == 200 else { throw FetchError.badID }
    let maybeImage = UIImage(data: data)
    guard let thumbnail = await maybeImage?.thumbnail else { throw FetchError.badImage }
    return thumbnail
}
```

支持异步的函数需要标记为`async`，如果该函数可能失败，则 async 写在 throws 前面；否则写在函数返回值前的箭头前面。创建 URLRequest 没太多可说，接着我们使用`data(for: request)`处理从服务端下载图片数据，这里同时使用了`try`和`await`。由于该方法是`awaitable`的，我们可以使用 await，让线程在执行到这里后挂起，并释放出资源去执行其他任务，直到网络请求的结果返回时，再重新拾起并继续执行后续代码。

在获取图片缩略图`UIImage.thumbnail`时，`UIImage.thumbnail`是属性，并不是方法，这样我们可以利用`await`来简化异步处理。async 属性，需要有明确的getter，并且用 get async 修饰，在其内部可以用 await 返回结果。其次，async 属性不能有 setter，即只能是可读属性。具体示例如下面代码段所示：

```swift
extension UIImage {
    var thumbnail: UIImage? {
        get async {
            let size = CGSize(width: 40, height: 40)
            return await self.byPreparingThumbnail(ofSize: size)
        }
    }
}
```

### Async 异步原理

![async](/assets/images/2024Swift/async01.png)

如上图所示是一个普通函数的执行流程，当`fetchThumbnail`调用`thumbnailURLRequest`时，同时也将线程控制权交给了后者。而`thumbnailURLRequest`执行结束后，则会主动交回控制权给调用它的`fetchThumbnail`，从而继续执行前者的逻辑。普通函数交出对线程的控制权的唯一方式，是该函数执行结束。

![async](/assets/images/2024Swift/async02.png)

如上图所示调用 async 函数时，控制权的传递则与之不同。fetchThumbnail`调用 async 方法`data(for: request)`时，同时也将线程控制权交给了后者。`data(for: request)`在执行过程中，可能会挂起，并把控制权交给操作系统，而不是它的调用者`fetchThumbnail`，当 async 方法挂起时，其调用者同时也被挂起了。当async方法执行完毕后，它会把控制权再交还给它的调用者`fetchThumbnail`，并继续执行直到结束退出。

+ 当标记一个函数为`async`时，意味着它可以挂起。在 async 函数中，使用`await`关键词标记在哪里可以一次或多次挂起。
+ 当 async 函数挂起时，线程并未阻塞，系统会自由安排其他任务。
+ 当 async 函数恢复执行时，其返回的结果会自然融入到 async 函数的调用者，并在先前挂起的地方接续执行。

### Async 序列

在获取缩略图的例子中，如果把await 用在了 for 循环中，这种可以在循环中 await 的序列，我们称之为来 async 序列 (async sequence)。可以像使用普通序列那样使用 async 序列，唯一区别是它所提供的元素是通过异步的方式交付的。示例如下

```swift
for await id in staticImageIDsURL.lines {
    let thumbnail = await fetchThumbnail(for: id)
    collage.add(thumbnail)
}
let result = await collage.draw()
```

### Async & Await 运用

如下是运用回调完成图片显示的示例:

```swift
struct ThumbnailView: View {
    @ObservedObject var viewModel: viewModel
    var post: Post
    @State private var image: UIImage?
    var body: some View {
        Image(uiImage: self.image ?? placeholder)
            .onAppear {
                self.viewModel.fetchThumbnail(for: post.id) { result, _ in
                    self.image = result
                }
            }
    }
}
```

已经了解了`Async & Await` 之后，那么如何在项目中使用呢？把完成的回调直接去掉，使用`try?`和`await`来衔接`fetchThumbnail`的调用。但是编译的时候代码报错了(如下图)，提示是`async 方法不能使用在不支持并行的上下文中`。

![async](/assets/images/2024Swift/async03.png)

在同步执行中是不能接受使用await异步代码的，解决这个问题是使用`Task` 任务函数。`Task` 任务把要执行的工作包裹在闭包中，并把它发送给系统，等待下一个可执行任务的线程去立即执行。就像全局`DispatchQueue`的 async 方法一样。

```swift
struct ThumbnailView: View {
    @ObservedObject var viewModel: ViewModel
    var post: Post
    @State private var image: UIImage?

    var body: some View {
        Image(uiImage: self.image ?? placeholder)
            .onAppear {
                Task {
                    self.image = try? await self.viewModel.fetchThumbnail(for: post.id)
                }
            }
    }
}
```

### Async & Await XCTest

```swift
class MockViewModelSpec: XCTestCase {
    func testFetchThumbnail() throws {
        let expectation = XCTestExpectation(description: "mock thumbnails completion")
        self.mockViewMode.fetchThumbnail(for: mockID) { result, error in
            XCAssertNil(error)
            expectation.fulfill()
        }
        wait(for: [expectation], timeout: 5.0)
    }
}
```

如上示例是测试获取缩略图是否的成功的单元测试；过去测试异步代码是冗长繁琐的，要经历设置期望、调用需要测试的API接口、完成期望、等待一段任意长的时间。

现在只需要把方法标记为 async，用 try await 执行要测试的 API 接口，并用 XCTAssert 将其包起来，async 代码就像测试同步代码一样简单。如下:

```swift
class MockViewModelSpec: XCTestCase {
    func testFetchThumbnail() async throws {
        XCAssertNoThrow(try await self.mockViewMode.fetchThumbnail(for: mockID))
    }
}
```

### 如何写 Async 方法

前面介绍的都是用框架提供的 async 方法，如果想实现个 async 方法，要怎么做呢？

```swift
func getPersistentPosts(completion: @escaping ([Post], Error?) -> Void) {
    do {
        let req = Post.fetchRequest()
        req.sortDescriptors = [ NSSortDescriptor(key: "date", assending: true) ]
        let asyncRequest = NSAsynchronousFetchRequest<Post>(fetchRequest: req) { result in
            completion(result.finalResult ?? [], nil)
        }
        try self.managedObjectContext.execute(asyncRequest)
    } catch {
        completion([], error)
    }
}
```

调用`getPersistentPosts`时，调用会进入 Core Data，一段时间后，Core Data 会调用完成回调把结果传递回`getPersistentPosts`。这个过程是异步请求的过程，那么如何改造成 async 方法呢？

![async](/assets/images/2024Swift/async04.png)

Swift提供了让开发者能够安全便利的使用 continuation 的能力，下面代码中的`withCheckedThrowingContinuation` 把原本使用完成回调的方法转换成了async方法，所以在它使用 try await 之前，并将其返回值作为 persistentPosts 这个 async 方法的返回值。`continuation.resume`方法，则是连接上图右侧的 resume 断层的桥梁，它使得挂起的函数在合适的时候继续执行。注意下面代码中的`throwing`和`returning`两种形态。

```swift
func persistenPosts() async throws -> [Post] {
    typealias PostContinuation = CheckedContinuation<[Post], Error>
    return try await withCheckedThrowingContinuation { posts, error in
        self.getPersistentPosts {
            continuation.resume(throwing: error)
        } else {
            continuation.resume(returning: posts)
        }
    }
}
```

continuation 有个简单但是重要的原则，`resume`方法必须在每个路径上执行，有且只有一次。

![async](/assets/images/2024Swift/async05.png)

![async](/assets/images/2024Swift/async06.png)

### 总结

在Swift中，Async和Await是用于异步编程的关键字。它们使得编写和管理异步代码更加简单和直观。

- **Async**：`async`关键字用于标记一个函数、闭包或方法是一个异步操作。使用`async`关键字声明的函数可以在其中使用`await`来等待其他异步操作的结果，而不会阻塞当前线程。
- **Await**：`await`关键字用于暂停当前异步函数的执行，等待另一个异步操作的完成，并且获取其结果。这样可以让程序在等待异步操作的同时继续执行其他任务，提高了并发性和响应性。

使用Async和Await可以让我们更容易地编写和理解异步代码，避免了回调地狱和复杂的多线程同步问题。这些关键字使得异步编程更加像编写同步代码一样直观和易于维护。
