# 介绍

Neo区块链与其他区块链的不同之处在于，它允许您使用多种知名编程语言（如C#、Go和JavaScript）来实现智能合约。Neow3j为此做出了贡献，并添加了对Java作为智能合约语言的支持。Neow3j中与智能合约开发相关的部分被称为**Neow3j开发包(devpack)**。它由三个模块组成。

* `io.neow3j:devpack` 是您的智能合约项目将依赖的模块。它是一个Java库，包含合约开发所需的Neo特有注解、方法和类。您将在合约中使用它，例如获取当前区块的信息或发送通知。该模块的API在[开发包](https://neow3j.io/#/neo-n3/smart_contract_development/devpack)部分进行了描述。
* `io.neow3j:compiler` 包含从Java类生成NeoVM代码的编译器。您可以通过在Java程序中调用它来以编程方式使用它，但很可能您不需要直接访问它。
* `io.neow3j:gradle-plugin` 实现了一个Gradle插件，您可以将其应用到智能合约项目中，它提供了一个简单的Gradle任务来进行合约编译

### [面向Neo虚拟机的Java](https://neow3j.io/#/neo-n3/smart_contract_development/introduction?id=java-for-the-neo-virtual-machine)

重要的是要理解，尽管表面上您编写的是Java代码，但生成的字节码和执行的虚拟机与Java无关。您的智能合约代码被编译为在Neo虚拟机(NeoVM)上运行的字节码，而不是在Java虚拟机(JVM)上运行。由于JVM和NeoVM之间的差异，使用Java开发Neo智能合约的编程体验是不同的。因此，我们将面向Neo的Java定义为Java的一种变体或子集，并将其命名为NeowJava。[NeowJava](https://neow3j.io/#/neo-n3/smart_contract_development/neowjava)章节讨论了NeowJava与Java的偏差以及使用NeowJava时的陷阱。

### [开发环境](https://neow3j.io/#/neo-n3/smart_contract_development/introduction?id=development-environment)

您可以在任何您想要的编辑器或IDE中编写智能合约。有一些环境，特别是Visual Studio Code和IntelliJ IDEA，为您提供额外的支持以改善开发体验。

使用[**Visual Studio Code**](https://code.visualstudio.com/)，您可以利用[Neo区块链工具包](https://marketplace.visualstudio.com/items?itemName=ngd-seattle.neo-blockchain-toolkit)扩展。它支持合约调试、轻松的合约设置，并提供了一个GUI来操作本地的neo-express实例。除了工具包之外，我们建议安装[Java扩展包](https://marketplace.visualstudio.com/items?itemName=vscjava.vscode-java-pack)和[Gradle扩展包](https://marketplace.visualstudio.com/items?itemName=richardwillis.vscode-gradle-extension-pack)。

[**IntelliJ**](https://www.jetbrains.com/idea/download/)显然是一个不错的选择，因为它已经对Java有很好的支持。为IntelliJ也实现了一个[Neo插件](https://plugins.jetbrains.com/plugin/17195-neo)。它通过IntelliJ的UI提供对neo-express的基本控制。有关使用说明和简短视频，请访问插件的[GitHub页面](https://github.com/irshadnilam/intellij-neo)。使用IntelliJ无法调试智能合约。

### [环境要求](https://neow3j.io/#/neo-n3/smart_contract_development/introduction?id=requirements)

* 智能合约编译需要本地安装[**Java 8 SDK**](https://adoptopenjdk.net/)(或更高版本)。
* 运行智能合约测试需要本地安装[**Docker**](https://www.docker.com/products/docker-desktop)。
