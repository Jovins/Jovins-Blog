---
layout: post
title: "SwiftData 学习"
date: 2023-11-10 21:05:00.000000000 +09:00
categories: [Swift]
tags: [Swift, SwiftData]
---

### 关于SwiftData

SwiftData 框架于 WWDC 2023 上发布。它旨在改变 Swift 应用程序中存储和管理数据的过程。该框架建立在 Core Data 的基础上，并提供了便捷的API，使开发人员能够轻松地执行数据库查询、插入、更新和删除操作。此外，SwiftData还提供了数据模型映射功能，方便将数据库中的表映射到Swift对象上。使用SwiftData可以加快iOS应用程序的开发速度，并简化与数据库交互的复杂性。

#### SwiftData 是什么

SwiftData 是专门为 Swift 设计的持久化框架。它提供了一种声明性方式来定义数据模型、关系和迁移。 SwiftData 负责底层存储细节，可以专注于应用程序逻辑。

#### SwiftData 特点

+ 富有表现力的数据模型定义
+ 自动数据迁移
+ 并发支持
+ iClound云同步
+ 安全性

#### SwiftData 与Core Data比较

![01](/assets/images/2024Swift/swiftdata-01.webp)

### SwiftData 使用

SwiftData 提供了`@Model`宏来直接定义数据的 Schema. 下面是SwiftData使用步骤:

> 1.导入`import SwiftData`
> 2.使用`@Model`修饰模型
> 3.将模型注册到`modelContainer`
> 4.获取模型上下文`@Environment(\.modelContext) private var modelContext`
> 5.使用模型上下文进行增删改操作
> 6.使用`@Query`修饰模型数组，获取查询的数据

SwiftData默认支持的类型:

+ 基础值类型. 比如 `String`、`Int`、`CGFloat`、`Float`等
+ 复杂类型. 比如 `Struct`、`Enum` 、`Codable`、集合等
+ 数据模型之间的关系`Relationship`和数据集合

#### 1.@Model声明模型

````swift
@Model
class Person {
    @Attribute(.unique) 
    var id: String
    var name: String
    @Transient
    var age: Int = 0
  	@Relationship(deleteRule: .cascade)
    var students: [Student]? = []
    init(name: String) {
        self.id = UUID().uuidString
        self.name = name
    }
}
````

> `@Attribute`: 指定模型的属性的自定义行为
> `@Transient`: 控制单个属性不被持久化
> `@Relationship`: 指定两个模型之间的关系
>
> `@Relationship(deleteRule: .cascade)`: 表示在数据删除时自动删除所有的关联
>
> 注意: @Attribute 还支持
> + @Attribute(.externalStorage): 将数据以二进制的形式持久化在 Store 之外
> + @Attribute(.transform): 支持存储类型和内存类型的转换
> + @Attribute(.spotlight): 自动集成 Spotlight
> + @Attribute(hashModifier: ""): 修改 Hash
> + `@Attribute(.unique)`: (主键)保证属性是唯一的

#### 2.构建ModelContainer

通过`ModelContainer`直接初始化

```swift
let container = try ModelContainer(for: Person.self)
```

用配置`ModelConfiguration`初始化

```swift
let container = try ModelContainer(
    for: Person.self,
    configurations: ModelConfiguration(url: URL("path"))
)
```

通过View 和 Scene 的修饰器来快速关联一个 ModelContainer

```swift
struct SwiftDataDemoApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(for: Person.self)
    }
}
```

#### 3.获取上下文context

通过Environment 来访问到 modelContext

```swift
struct ContextView : View {
    @Environment(\.modelContext) private var context
}
```

直接获取共享的主Actor context

```swift
let context = container.mainContext
```

直接初始化一个新的Context

````swift
let context = ModelContext(container)
````

#### 4.增删改查

**增删改**

```swift
let person = Person(name: "Lily", age: 10)
// Insert a new person
context.insert(person)
// Delete an existing person
context.delete(person)
// Manually save changes to the context
try context.save()
```

**查询**

Model Context 使用了 Swift 原生的 Predicate 宏、 Fetch Descriptor、Sort Descriptor 来进行自定义数据获取。在 iOS 17 中，`Predicate` 支持 Swift 原生类型并利用宏来简化使用。相较于 `NSPredicate`, `Predicate` 有完整的类型检查与完善的 Xcode 补全。

```swift
// 创建查询条件
let personPredicate = #Predicate<Person> {
    $0.name == "Lily" &&
    $0.age == 10
}
// 查询
let descriptor = FetchDescriptor<Person>(predicate: personPredicate)
let persons = try? context.fetch(descriptor)
```



### SwiftData 数据迁移

经过每个版本迭代，确保数据能够正常使用，SwiftData提供`VersionedSchema` 与 `SchemaMigrationPlan` 进行迁移。

迁移步骤:

+ 模型封装在 `VersionedSchema` 中, 让 SwiftData 知道具体发生了什么变化
+ 将各个 `VersionedSchema` 按迁移顺序排序，创建一个 SchemaMigrationPlan
+ 迁移阶段包含两种类型：
  - 轻量迁移：正如其名，轻量迁移不需要额外代码配置，可以自动生效。
  - 自定义迁移：轻量迁移中不支持的都需要进行自定义迁移，如上述的 `@Attribute(.unique)`。

定义每个版本的 **VersionedSchema**：

```swift
enum StudentSchemaV1: VersionedSchema {
    static var models: [any PersistentModel.Type] {
        [Student.self, Teacher.self]
    }

    @Model
    final class Student {                                             
        var name: String
        var destination: String
        var start_date: Date
        var end_date: Date
    }
  // 这个版本中的其他模型...
}

enum StudentSchemaV2: VersionedSchema {
    static var models: [any PersistentModel.Type] {
        [Student.self, Teacher.self]
    }

    @Model
    final class Student {                                             
        @Attribute(.unique) var name: String   
        var destination: String
        var start_date: Date
        var end_date: Date
    }
  // 这个版本中的其他模型...
}

enum StudentMigrationPlan: SchemaMigrationPlan {
    static var schemas: [any VersionedSchema.Type] {
        [StudentSchemaV1.self, StudentSchemaV2.self]
    }

    static var stages: [MigrationStage] {
        return [migrateV1toV2]                                                  
    }

    static let migrateV1toV2 = MigrationStage.custom(                                    // 3️⃣
        fromVersion: StudentSchemaV1.self,
        toVersion: StudentSchemaV2.self,
        willMigrate: { context in
            let student = try? context.fetch(FetchDescriptor<StudentSchemaV1.Trip>())
            // 对同名 Trip 去重...
            try? context.save() 
        }, didMigrate: nil)
}
```

在 `ModelContainer` 中配置 `migrationPlan` 即可让 App 在启动后开始迁移。

```swift
struct SwiftDataDemoApp: App {
    let container = ModelContainer(
        for: Trip.self, 
        migrationPlan: StudentMigrationPlan.self
    )

    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(container)
    }
}
```

SwiftData 是一个开源的数据库框架，专门为 Swift 语言而设计。它提供了一种简单而灵活的方法来处理数据存储和管理，使开发人员能够轻松地在其应用程序中实现数据持久化和管理。该框架提供了便利的 API 和丰富的功能，如查询构建器、数据筛选和排序等，以帮助开发人员更轻松地操作数据。此外，SwiftData 还支持异步操作和事务处理，确保数据操作的安全性和性能。

总的来说，SwiftData 是一个强大而灵活的数据库框架，可以帮助开发人员轻松处理应用程序中的数据存储和管理任务。





