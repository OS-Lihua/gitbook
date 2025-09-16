# 调试

默认情况下，`neow3jCompile` Gradle任务将输出一个`.nefdbgnfo`文件，其中包含编译智能合约的调试信息。该文件与NEF和合约清单放在同一文件夹中，即通常在`./build/neow3j`。该文件实际上是一个包含JSON文件的zip存档，如果您想查看内部内容。

调试信息旨在与[Neo调试器](https://github.com/neo-project/neo-debugger)一起使用，该调试器是VS Code扩展包[Neo区块链工具包](https://marketplace.visualstudio.com/items?itemName=ngd-seattle.neo-blockchain-toolkit)的一部分。在IntelliJ中无法进行智能合约调试。从这里开始，我们假设您在VS Code中工作并已安装了Neo区块链工具包扩展。

为了让VS Code知道要调试哪个合约，您需要添加启动配置。启动配置位于项目目录中的`.vscode/launch.json`文件中。如果还没有这样的文件，您可以通过从“运行”菜单中选择“添加配置”来生成它。在`launch.json`中，将光标移动到“configurations”部分内并使用自动完成，这样将出现一个输入框，其中包含一个项目“_Neo Contract: Launch_”。选择它，将添加一个新配置。类似于以下示例调整配置。`program`属性必须指向由neow3j Gradle插件输出的NEF文件。有关如何为Neo调试器配置启动配置的更多信息，请访问创建者的[示例](https://github.com/devhawk/safe-purchase-sample/blob/master/.vscode/launch.json)之一。

以下是neow3j样板项目的摘录。

```json
{
    "name": "HelloWorldSmartContract",
    "type": "neo-contract",
    "request": "launch",
    "program": "${workspaceFolder}/build/neow3j/HelloWorld.nef",
    "neo-express": "${workspaceFolder}/default.neo-express",
    "invocation": {
        "operation": "getOwner",
        "args": []
    },
    "storage": [
        {
            "key": "owner",
            "value": "@NNSyinBZAr8HMhjj95MfkKD1PY7YWoDweR"
        }
    ]
}Copy to clipboardErrorCopied
```

配置就位后，您可以通过按F5或从命令面板使用“Debug: Start Debugging”来开始调试。

> **注意** 目前，仅在合约类内部支持调试。如果类调用开发包中的方法或工作区中其他自定义类中的方法，调试器将不会进入这些方法。
