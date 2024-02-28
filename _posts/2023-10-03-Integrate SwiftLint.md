---
layout: post
title: "集成SwiftLint到Xcode项目"
date: 2023-10-03 20:12:00.000000000 +09:00
categories: [Swift]
tags: [Swift, SwiftLint, 代码规范]

---

## 集成SwiftLint到Xcode项目

### 前言

- [SwiftLint](https://github.com/realm/SwiftLint)是 [Realm](https://realm.io/) 推出的一款 Swift 代码规范检查工具, 基本上以 [Kodeco's Swift 代码风格指南](https://github.com/kodecocodes/swift-style-guide)为基础, 比较容易集成到Xcode. 支持自动纠正部分代码、支持自定义规则、可禁用或者开启某一些规则.
- 在个人开发中可以很好规范和改善代码质量；在团队开发中可以很好的规范团队的代码风格，避免语法层面的一些低级错误，提高代码质量，同时也提高团队Code Review的效率。
- SwiftLint 工作于 SourceKit 这一层，所以 Swift 版本发生变化时它也能继续工作。这也是 SwiftLint 轻量化的原因，因为它不需要一个完整的 Swift 编译器，它只是与已经安装在你的电脑上的官方编译器进行通信。

### 集成SwiftLint

+ 安装Homebrew

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

+ 使用Homebrew集成

  ```bash
  brew install swiftlint
  ```

+ 使用Cocoapods集成

  ```bash
  pod 'SwiftLint'
  ```

  在下一次执行 `pod install` 时将会把 SwiftLint 的二进制文件和依赖下载到 `Pods/` 目录下并且将允许你通过 `${PODS_ROOT}/SwiftLint/swiftlint` 在 Script Build Phases 中调用 SwiftLint。

  自从 SwiftLint 支持安装某个特定版本后，安装一个指定版本的 SwiftLint 是目前推荐的做法相比较于简单地选择最新版本安装的话（比如通过 Homebrew 安装的话）。

  请注意这会将 SwiftLint 二进制文件、所依赖的二进制文件和 Swift 二进制库安装到 `Pods/` 目录下，所以不推荐将此目录添加到版本控制系统（如 git）中进行跟踪。

+ 使用Mint集成

  ```bash
  mint install realm/SwiftLint
  ```

+ 使用安装包集成

  可以通过从[最新的 GitHub 发布地址](https://github.com/realm/SwiftLint/releases/latest)下载 `SwiftLint.pkg` 然后执行的方式安装 SwiftLint。

+ 编译源代码集成

  通过 clone SwiftLint 的 Git 仓库到本地然后执行 `make install` (Xcode 15.0+) 以从源代码构建及安装。

+ 使用 Bazel集成

  把这个放到你的 `MODULE.bazel`：

  ```
  bazel_dep(name = "swiftlint", version = "0.50.4", repo_name = "SwiftLint")
  ```

  或把它放到你的 `WORKSPACE`：

  然后你就可以在当前目录下使用这个命令运行 SwiftLint：

  ```bash
  bazel run -c opt @SwiftLint//:swiftlint
  ```

**Fastlane**

用[fastlane官方的SwiftLint功能](https://docs.fastlane.tools/actions/swiftlint)来运行 SwiftLint 作为你的 Fastlane 程序的一部分。

```
swiftlint(
    mode: :lint,                            # SwiftLint模式: :lint (默认) 或者 :autocorrect
    executable: "Pods/SwiftLint/swiftlint", # SwiftLint的程序路径 (可选的). 对于用CocoaPods集成SwiftLint时很重要
    path: "/path/to/lint",                  # 特殊的检查路径 (可选的)
    output_file: "swiftlint.result.json",   # 检查结果输出路径 (可选的)
    reporter: "json",                       # 输出格式 (可选的)
    config_file: ".swiftlint-ci.yml",       # 配置文件的路径 (可选的)
    files: [                                # 指定检查文件列表 (可选的)
        "AppDelegate.swift",
        "path/to/project/Model.swift"
    ],
    ignore_exit_status: true,               # 允许fastlane可以继续执行甚至是Swiftlint返回一个非0的退出状态(默认值: false)
    quiet: true,                            # 不输出像‘Linting’和‘Done Linting’的状态日志 (默认值: false)
    strict: true                            # 发现警告时报错? (默认值: false)
)
```

### SwiftLint命令

在终端输入 `swiftlint help` 可以查看所有可用的命令

```
OPTIONS:
  --version        Show the version.
  -h, --help       Show help information.
SUBCOMMANDS:
  analyze          Run analysis rules
  docs             Open SwiftLint documentation website in the default web browser
  generate-docs    Generates markdown documentation for selected group of rules
  lint (default)   Print lint warnings and errors
  reporters        Display the list of reporters and their identifiers
  rules            Display the list of rules and their identifiers
  version          Display the current version of SwiftLint

//查看所有命令
swiftlint help
//忽略空格导致的警告和错误
swiftlint autocorrect
//输出所有的警告和错误
swiftlint lint
//查看所有可获得的规则以及对应的 ID
swiftlint rules
//产看当前版本号
swiftlint version
```

### SwiftLint使用

集成 SwiftLint 到 Xcode 体系中，可以使警告和错误显示到 IDE 上，需要在 Xcode 中添加一个新的“Run Script Phase”并且包含如下代码即可：

```bash
if which swiftlint >/dev/null; then
swiftlint
else
echo "warning: SwiftLint not installed, download from https://github.com/realm/SwiftLint"
fi
exit 0
```

Xcode 15 对 Build Settings 进行了重大改动，它将 `ENABLE_USER_SCRIPT_SANDBOXING` 的默认值从 `NO` 更改为 `YES`。 SwiftLint 会遇到与缺少文件权限相关的错误，通常报错信息为：`error: Sandbox: swiftlint(19427) deny(1) file-read-data.`

要解决此问题，需要Build Settings手动将 `ENABLE_USER_SCRIPT_SANDBOXING` 设置为 `NO`，以针对 SwiftLint 配置的特定目标。

```
Build Settings -> Build Options -> User Script Sandboxing -> NO
```

Apple 芯片的 Mac 上通过 Homebrew 安装的 SwiftLint，可能会遇到这个警告:

> warning: SwiftLint not installed, download from https://github.com/realm/SwiftLint

因为 Homebrew 在搭载 Apple 芯片的 Mac 上将二进制文件默认安装到了 `/opt/homebrew/bin` 下。如果要让 Xcode 知道 SwiftLint 在哪，你可以在 Build Phase 中将 `/opt/homebrew/bin` 路径添加到 `PATH` 环境变量.

```
if [[ "$(uname -m)" == arm64 ]]; then
    export PATH="/opt/homebrew/bin:$PATH"
fi

if which swiftlint > /dev/null; then
  swiftlint
else
  echo "warning: SwiftLint not installed, download from https://github.com/realm/SwiftLint"
fi
```

> 注意: 在项目根目录下新建一个 `.swiftlint.yml` 文件，然后把官方的 [默认配置内容](https://github.com/realm/SwiftLint/blob/main/README_CN.md#配置) 复制到该文件中。将默认配置中 `included` 下面的 `- Sources` 改为你实际项目中的路径，尤其是主项目中代码所在的那个目录, 否则Xcode会报如下面的错误:

```
Linting Swift files in current working directory 
Error: No Lintable files found at paths: ''
Command PhaseScriptExecution failed with a nonzero exit code
```

或者可以创建一个指向在 `/usr/local/bin` 中实际二进制文件的符号链接：

```
ln -s /opt/homebrew/bin/swiftlint /usr/local/bin/swiftlint
```

如果是通过Cocoapods继承的，脚本应该是这样的：

```bash
${PODS_ROOT}/SwiftLint/swiftlint --fix
${PODS_ROOT}/SwiftLint/swiftlint lint
```

### SwiftLint规则

SwiftLint 已经包含了超过 200 条规则，可以通过命令行中执行 `swiftlint rules -e` 即可查看已启用的规则(`enabled in your config` 列)，也可以在 [这里](https://realm.github.io/SwiftLint/rule-directory.html) 找到规则的更新列表和更多信息，也可以查看 [Source/SwiftLintBuiltInRules/Rules](https://github.com/realm/SwiftLint/blob/main/Source/SwiftLintBuiltInRules/Rules) 目录来查看它们的实现。

**规则配置**

可以通过在你需要执行 SwiftLint 的目录下添加一个 `.swiftlint.yml` 文件的方式来配置 SwiftLint。可以被配置的参数有，包含的规则：

- `disabled_rules`: 关闭某些默认开启的规则。
- `opt_in_rules`: 一些规则是可选的。
- `only_rules`: 不可以和 `disabled_rules` 或者 `opt_in_rules` 并列。类似一个白名单，只有在这个列表中的规则才是开启的。

```
disabled_rules: # 执行时排除掉的规则
  - colon
  - comma
  - control_statement
opt_in_rules: # 一些规则是默认关闭的，所以你需要手动启用
  - empty_count # 你可以通过执行如下指令来查找所有可用的规则：`swiftlint rules`
# 或者，通过取消对该选项的注释来明确指定所有规则：
# only_rules：# 如果使用，请删除 `disabled_rules` 或 `opt_in_rules`
#   - empty_parameters
#   - vertical_whitespace

analyzer_rules: # `swiftlint analyze` 运行的规则
  - explicit_self

included: # 执行 linting 时包含的路径。如果出现这个 `--path` 会被忽略。
  - Sources
excluded: # 执行 linting 时忽略的路径。 优先级比 `included` 更高。
  - Carthage
  - Pods
  - Sources/ExcludedFolder
  - Sources/ExcludedFile.swift

# 如果值为 true，SwiftLint 将把所有警告都视为错误
strict: false

# 可配置的规则可以通过这个配置文件来自定义
# 二进制规则可以设置他们的严格程度
force_cast: warning # 隐式
force_try:
  severity: warning # 显式
# 同时有警告和错误等级的规则，可以只设置它的警告等级
# 隐式
line_length: 110
# 可以通过一个数组同时进行隐式设置
type_body_length:
  - 300 # warning
  - 400 # error
# 或者也可以同时进行显式设置
file_length:
  warning: 500
  error: 1200
# 命名规则可以设置最小长度和最大程度的警告/错误
# 此外它们也可以设置排除在外的名字
type_name:
  min_length: 4 # 只是警告
  max_length: # 警告和错误
    warning: 40
    error: 50
  excluded: iPhone # 排除某个名字
identifier_name:
  min_length: # 只有最小长度
    error: 4 # 只有错误
  excluded: # 排除某些名字
    - id
    - URL
    - GlobalAPIKey
reporter: "xcode" # 报告类型 (xcode, json, csv, checkstyle, codeclimate, junit, html, emoji, sonarqube, markdown, github-actions-logging)
```

**定义自定义规则**

用如下语法在你的配置文件里定义基于正则表达式的自定义规则：

```
custom_rules:
  pirates_beat_ninjas: # 规则标识符
    name: "Pirates Beat Ninjas" # 规则名称，可选
    regex: "([nN]inja)" # 匹配的模式
    match_kinds: # 需要匹配的语法类型，可选
      - comment
      - identifier
    message: "Pirates are better than ninjas." # 提示信息，可选
    severity: error # 提示的级别，可选
  no_hiding_in_strings:
    regex: "([nN]inja)"
    match_kinds: string
```

![rule](/assets/images/swiftlint-rule.png)

可以通过提供一个或者多个 `match_kinds` 的方式来对匹配进行筛选，它会将含有不包括在列表中的语法类型的匹配排除掉。这里有全部可用的语法类型：

- argument
- attribute.builtin
- attribute.id
- buildconfig.id
- buildconfig.keyword
- comment
- comment.mark
- comment.url
- doccomment
- doccomment.field
- identifier
- keyword
- number
- objectliteral
- parameter
- placeholder
- string
- string_interpolation_anchor
- typeidentifier

**嵌套配置**

SwiftLint 支持通过嵌套配置文件的方式来对代码分析过程进行更加细致的控制。

- 在你需要的目录引入 `.swiftlint.yml`。
- 在目录结构必要的地方引入额外的 `.swiftlint.yml` 文件。
- 每个文件被检查时会使用在文件所在目录下的或者父目录的更深层目录下的配置文件。否则根配置文件将会生效。
- `excluded` 和 `included` 在嵌套结构中会被忽略。

### 参考

[realm/SwiftLint](https://github.com/realm/SwiftLint/blob/main/README_CN.md)

[SwiftLint-Rules](https://realm.github.io/SwiftLint/rule-directory.html)

[Swift-Style-Guide](https://github.com/LeiHao0/swift-style-guide/blob/master/README_CN.md)