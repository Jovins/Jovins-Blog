---
layout: post
title: "浅谈Clean Architecture"
date: 2023-11-22 23:10:00.000000000 +09:00
categories: [Swift]
tags: [Swift, Architecture, MVVM, Clean]
---

## 前言

自从 2008 年发布 iOS SDK 以来，我们就用UIKit来构建应用程序。在此期间，一直在不懈地寻找最适合其应用程序的架构。一切始于`MVC`，但后来我们见证了`MVP`、`MVVM`、`VIPER`等的兴起。在本文中，我们将探讨 MVVM Clean Architecture如何帮助我们构建强大且可扩展的 iOS 应用程序。

![clean](/assets/images/2024Swift/cleanmvvm01.png)

在了解`Clean Architecture`之前，我们先回顾一下 `MVC`、`MVP`、 `MVVM` 和`VIPER`架构是什么样的.

## MVC

![clean](/assets/images/2024Swift/cleanmvvm02.png)

MVC是一种经典的软件架构模式，用于设计和组织应用程序的结构。MVC代表Model-View-Controller，将应用程序分为三个主要部分：

1. **Model（模型）**：模型代表应用程序的数据结构和业务逻辑。它负责管理数据的存储、检索和处理，与数据库或其他数据源进行交互，但不直接处理用户界面。
2. **View（视图）**：视图是用户界面的可视化部分，负责展示数据给用户并接收用户的输入操作。视图通常是在UI框架中实现的，如网页、移动应用界面等。
3. **Controller（控制器）**：控制器作为模型和视图之间的中介，负责处理用户的输入并相应地更新模型和视图。控制器从视图中获取用户的输入，然后更新模型的状态，最后通知视图更新以反映模型的变化。

MVC的核心思想是分离应用程序的数据、表示和用户交互。这种分层结构使得开发人员可以更好地管理代码，并更容易扩展和维护应用程序。通过MVC模式，开发人员可以将应用程序分成独立的组件，从而实现单一职责原则，降低耦合度，提高代码的清晰度和可维护性。总的来说，MVC架构是一种有效的设计模式，帮助开发人员将应用程序划分为不同的组件，使得应用程序的开发、测试和维护更加简单和高效。

> // 优点:
>
> 1. **分离关注点**：MVC将应用程序划分为模型、视图和控制器三个部分，每个部分负责不同的功能，使代码易于管理和维护。
> 2. **可扩展性**：由于MVC模式具有低耦合性，因此可以更容易地扩展或修改应用程序的特定部分而不影响其他部分。
> 3. **代码复用**：MVC允许开发人员重用模型、视图和控制器中的代码，提高了代码的可重用性。
> 4. **易于测试**：由于MVC模式将业务逻辑、数据表示和用户交互分开，因此可以更轻松地对每个部分进行单元测试。
> 5. **更好的团队协作**：通过清晰地定义每个组件的职责，MVC使团队成员更容易理解和协作开发应用程序。
>
> // 缺点
>
> 1. **复杂性**：MVC模式可能增加了应用程序的复杂性，因为需要对代码进行划分和管理，引入了额外的结构。
> 2. **学习曲线**：对于新手开发人员来说，理解和正确实现MVC模式可能需要一定的学习曲线。
> 3. **过度分层**：在小型应用程序中，使用MVC可能会导致过度分层，增加了开发成本和时间。
> 4. **过多的交互**：在MVC模式中，视图、控制器和模型之间可能会有过多的交互，增加了系统的复杂性和维护成本。
> 5. **性能问题**：在某些情况下，由于每次请求都会经过控制器的处理，可能会导致性能问题。

## MVP

![clean](/assets/images/2024Swift/cleanmvvm03.png)

MVP是一种软件架构模式，用于设计用户界面和业务逻辑之间的交互。MVP代表Model-View-Presenter，它将应用程序分为三个主要部分：

1. **Model（模型）**：模型代表应用程序的数据结构和业务逻辑。它负责管理数据的存储、检索和处理，与数据库或其他数据源进行交互，但不直接处理用户界面。
2. **View（视图）**：视图是用户界面的可视化部分，负责展示数据给用户并接收用户的输入操作。视图通常是在UI框架中实现的，如网页、移动应用界面等。
3. **Presenter（呈现者）**：呈现者作为模型和视图之间的中介，负责从模型中获取数据，并将数据格式化后传递给视图展示。同时，呈现者也负责接收视图产生的用户操作，然后更新模型的状态。

MVP的核心思想是分离关注点，使得视图只负责显示数据和接收用户输入，而业务逻辑则放在呈现者中进行处理。这种分离使得代码更易于维护和测试，同时也有利于团队协作开发。总的来说，MVP架构通过将应用程序划分为不同的组件，并定义清晰的交互方式，提高了应用程序的可维护性和可测试性。

## MVVM

![clean](/assets/images/2024Swift/cleanmvvm04.png)

MVVM是一种软件架构模式，主要用于设计用户界面。MVVM代表Model-View-ViewModel，它将应用程序的用户界面分为如下几个部分：

- **Model** 数据层，读写数据，保存 App 状态
- **View** 页面层，提供用户输入行为，并且显示输出状态
- **ViewModel** 逻辑层，它将用户输入行为，转换成输出状态
- **ViewController** 主要负责数据绑定

MVVM架构的核心思想是解耦视图和业务逻辑，使得它们可以独立开发和测试。通过使用数据绑定机制，视图模型可以自动更新视图中的数据，同时视图模型也能够将视图中的用户交互传递给模型进行处理，实现了视图、模型和视图模型之间的双向绑定。

> MVVM架构的优点：
>
> 1. **松耦合性**：MVVM通过数据绑定机制实现视图和视图模型之间的关联，降低了它们之间的耦合度，使得代码更易于维护和扩展。
> 2. **可测试性**：由于视图模型包含了视图所需的数据和行为，并且与视图和模型之间的通信逻辑分离，因此可以更容易地对视图模型进行单元测试。
> 3. **重用性**：MVVM架构鼓励将业务逻辑和显示逻辑分离，使得视图模型中的部分代码可以被不同的视图共享，提高了代码的重用性。
> 4. **分工明确**：MVVM清晰地定义了每个组件的角色和职责，有利于团队协作开发，并且使得前端与后端的开发可以更好地分工合作。
> 5. **用户界面与逻辑分离**：MVVM架构使得用户界面和业务逻辑分离，使得设计师和开发人员可以并行工作，而无需过多干扰彼此。
>
> MVVM架构的缺点：
>
> 1. **学习成本**：对于新手开发人员来说，理解MVVM模式可能需要一定的学习曲线，尤其是对于数据绑定等概念。
> 2. **过度抽象**：有时候MVVM架构可能会引入过多的抽象层次，增加了开发的复杂性。
> 3. **性能问题**：在某些情况下，过多的数据绑定以及视图模型的复杂逻辑可能导致性能问题，需要谨慎设计和优化。
> 4. **不适合小规模应用**：对于小规模应用程序，使用MVVM可能会显得繁琐和过度设计。
> 5. **双向绑定复杂性**：在涉及复杂的双向数据绑定时，可能需要处理额外的复杂性和注意事项。

## VIPER

![clean](/assets/images/2024Swift/cleanmvvm05.png)

`VIPER`架构是从`View-Interactor-Presenter-Entity-Routing`架构演变而来，主要用于构建iOS应用程序。VIPER架构包括以下几个组件：

1. **View（视图）**：视图负责用户界面的展示和用户交互。在VIPEr中，视图通常会将用户的操作传递给Presenter，并且接收Presenter返回的数据进行显示。
2. **Interactor（交互器）**：交互器包含应用程序的业务逻辑，负责处理数据的获取、转换和存储等任务。它与外部数据源进行交互，如数据库、网络请求等。
3. **Presenter（呈现者）**：呈现者充当了视图和交互器之间的中介，负责对交互器获取的数据进行格式化后提供给视图进行展示，同时也会处理视图产生的用户操作并将其传递给交互器进行处理。
4. **Entity（实体）**：实体表示应用程序中的数据模型，包含应用程序的核心数据结构。
5. **Routing（路由）**：路由器负责管理屏幕之间的导航和跳转，处理模块之间的通信和导航逻辑。

`VIPER架构`强调了模块之间的清晰分离和单一职责原则，使得各个组件之间的依赖性更加清晰，有利于代码的可维护性和可测试性。`VIPER架构`通过将应用程序划分为不同的组件，并定义明确的职责和交互方式，帮助开发人员更好地设计和构建iOS应用程序。

## Clean Architecture

![clean](/assets/images/2024Swift/cleanmvvm06.png)

MVVM Clean Architecture是一种将MVVM（Model-View-ViewModel）架构模式与Clean Architecture相结合的设计理念。通过结合MVVM的双向数据绑定和Clean Architecture的分层设计，MVVM Clean Architecture旨在提供一个更具可维护性和可测试性的应用程序架构，并使开发人员更容易构建复杂的用户界面应用程序。

![clean](/assets/images/2024Swift/cleanmvvm07.png)

从上图中我们可以得到 **Presentation Layer、Domain Layer 和 Data Layer**:

**Presentation Layer -> Domain Layer <- Data Layer**

+ `Presentation Layer`: ViewModels(Presenters) + Views(UI)
+ `Domain Layer`: Entities + Use Cases + Repositories + Interfaces
+ `Data Layer`:  Repositories Implementations + API(Network) + Persistence DB

> 1. **Presentation Layer** : 视图由执行一个或多个用例的 ViewModel（Presenters）协调。`Presentation Layer`仅依赖于 `Domain Layer `。
> 2. **Domain Layer (业务逻辑)**:  是不依赖其他层，完全独立的。Domain Layer不应包含来自其他层的任何内容（例如*演示 - UIKit 或 SwiftUI*或数据层 - Mapping *Codable*）
> 3. **Data Layer** : 负责协调来自不同数据源的数据，数据源可以是远程的或本地的（例如持久数据库）。`Data Layer` 仅依赖于 `Domain Layer`。

![clean](/assets/images/2024Swift/cleanmvvm08.png)

**Clean Architecture with MVVM的主要特点:**

1. **分层架构**：遵循Clean Architecture的原则，将应用程序分为不同的层级，包括实体层、用例层、接口适配器层和框架和驱动层，以保持各层之间的清晰分离。
2. **松耦合性**：通过MVVM模式的数据绑定机制以及Clean Architecture的分层设计，实现了各个模块之间的松耦合，降低了代码的依赖性，使得系统更加灵活和易于维护。
3. **单一职责原则**：每个组件都具有明确的职责，模型负责数据管理和业务逻辑，视图负责用户界面的展示，而视图模型则负责连接视图和模型，并处理视图的显示逻辑。
4. **可测试性**：MVVM Clean Architecture强调对代码进行单元测试和集成测试，由于各个组件之间的清晰划分，使得测试工作更加容易。
5. **可扩展性**：应用MVVM Clean Architecture的应用程序更容易扩展和修改，新增功能或调整需求时，可以更灵活地对系统进行改动而不会影响其他部分。

### 实践

```
-- Presentation
------ Views
------ ViewModels
-- Domain
------ Entities
------ UseCases
------ Interfaces
-- Data
------ API
------ Repositories
```

#### 1.定义Model

```swift
struct Report: Codable, Hashable {
		let id: String?
    let date: Date?
    var status: Status?
    let skuId: String?
    let sku: String?
    let units: String?
    let createdAt: Date?
    let updatedAt: Date?
}
```

#### 2.创建ViewModel

```swift
final class ReportViewModel: BaseViewModel {
    
    private(set) var reports: [Report] = []
    private var page: Int = 1
    private var size: Int = 20
    
    func fetchReports(refreshType: RefreshType, status: String) {
        ReportUseCases().fetchReports(status: status, page: page, size: size)
            .handleEvents(receiveSubscription: { _ in
            }, receiveOutput: { [weak self] output in
                if let self,
                   !output.pageContext.isNoMore {
                    page += 1
                }
            }, receiveCompletion: { [weak self] completion in
                guard let self else { return }
                self.showErrorHud(refreshType: refreshType, completion: completion)
            }).map { model in
                if refreshType == .refresh {
                    self.reports = model.items
                } else {
                    self.reports.append(contentsOf: model.items)
                }
                return model.pageContext
            }.eraseToAnyPublisher()
    }
}
```

#### 3.设计Views and View Controllers

```swift
import UIKit
import Combine

final class ReportViewController: UIViewController {

    private lazy var tableView: UITableView = {
        let tableView = UITableView(frame: CGRect(), style: .plain)
        tableView.backgroundColor = .white
        tableView.separatorStyle = .none
        tableView.dataSource = self
        tableView.delegate = self
        tableView.allowsMultipleSelectionDuringEditing = true
        tableView.register(ReportTableViewCell.self)
        return tableView
    }()
    private var viewModel: ReportViewModel = ReportViewModel()
    private let cancelBag = CancelBag()

    override func viewDidLoad() {
        super.viewDidLoad()
        setupUI()
    }

    private func setupUI() {
        view.backgroundColor = .white
        navigationItem.title = "Report"
        view.addSubview(tableView)
        tableView.snp.makeConstraints {
            $0.edges.equalToSuperview()
        }
      	tableView.addRefreshAndLoadMore { [weak self] in
            guard let self else { return }
            self.loadData(refreshType: .more)
        }
        tableView.refreshAction = { [weak self] in
            guard let self else { return }
            self.resetPrefetchedData()
            self.loadData(refreshType: .refresh)
        }
        tableView.loadMoreAction = { [weak self] in
            guard let self else { return }
            self.loadData(refreshType: .more)
        }
        loadData(refreshType: .refresh)
    }
  
  	private func loadData(refreshType: RefreshType) {
        viewModel.fetchReports(refreshType: refreshType, status: status)
            .sink { [weak self] completion in
                guard let self else { return }
                if case .failure(_) = completion,
                   refreshType == .more {
                    self.tableView.showErrorFooterView()
                }
                self.tableView.endRefresh()
            } receiveValue: { [weak self] pageContext in
                guard let self else { return }
                if pageContext.isNoMore {
                    self.tableView.setNoMoreData(showFooter: !pageContext.noData)
                } else {
                    self.tableView.resetNoMoreData()
                }
                self.tableView.reloadData()
            }.store(in: cancelBag)
    }
}

// MARK: - UITableViewDataSource
extension IdahBatchReportViewController: UITableViewDataSource, UITableViewDelegate {

    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return viewModel.reports.count
    }

    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(IdahReportApprovalTableViewCell.self, for: indexPath)
        cell.setupData(model: viewModel.reports[indexPath.row])
        return cell
    }

    func tableView(_ tableView: UITableView, heightForRowAt indexPath: IndexPath) -> CGFloat {
        return UITableView.automaticDimension
    }
}
```

#### 4.实现数据绑定

使用数据绑定机制（例如 KVO、ReactiveSwift 或 Combine）在 ViewModel 和视图之间建立双向关系。这样，只要 ViewModel 的属性发生变化，就可以自动更新 UI。

#### 5. 与数据层集成

实现数据层以从各种来源获取和保存数据。这可以使用网络请求、本地数据库或其他数据存储机制来完成。ViewModel 应通过接口或协议与数据层交互，以促进松散耦合。

```swift
struct ReportUseCases {
    func fetchReports(status: String, page: Int, size: Int) -> AnyPublisher<Reports, Error> {
        return ReportRepository().fetchReports(status: status, page: page, size: size)
    }
}

struct ReportRepository {
    func fetchReports(status: String, page: Int, size: Int) -> AnyPublisher<Reports, Error> {
        return IdahNetwork.decodableRequest(ReportAPI(status: status, page: page, size: size))
    }
}

struct ReportAPI {
    let status: String?
    let page: Int
    let size: Int
}

extension ReportAPI: DecodableTargetType {

    typealias ResultType = Reports

    var baseURL: URL {
        return Host.domain
    }

    var path: String {
        return "/reports/search"
    }

    var method: Moya.Method {
        return .post
    }

    var task: Moya.Task {
        var parameters = [String: Any]()
        if status != "全部" {
            parameters["status"] = status
        }
        parameters["endDate"] = Date().formateCurrentDateToYMD()
        parameters["page"] = page
        parameters["size"] = size
        return .requestParameters(parameters: parameters, encoding: JSONEncoding.default)
    }

    var headers: [String : String]? { nil }
}

```

#### 6.单元测试

为 ViewModel 层编写单元测试，确保其正确性和可靠性。模拟数据层依赖关系以隔离 ViewModel 的逻辑并验证其在不同场景下的行为。

## 结论

在本文中，我们探讨了 MVVM Clean Architecture 如何帮助我们构建强大且可扩展的 iOS 应用程序。通过分离关注点并促进松散耦合，此架构模式增强了代码的可测试性、可维护性和可重用性。如果正确实施，MVVM Clean Architecture 可以使 iOS 开发人员创建易于维护、扩展和测试的应用程序，从而带来更好的用户体验。