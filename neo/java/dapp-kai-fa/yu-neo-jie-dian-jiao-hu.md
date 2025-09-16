# 与Neo节点交互

### [设置连接](https://neow3j.io/#/neo-n3/dapp_development/interacting_with_a_node?id=setting-up-a-connection)

由于Neow3j不是Neo节点实现，与外部节点交互对于任何涉及从区块链读取或写入的操作都至关重要。与Neo节点交互的主要组件是 `io.neow3j.protocol.Neow3j` 类。该类为Neo节点支持的所有JSON-RPC方法提供了Java等价物。

您可以按以下方式实例化它：

```java
Neow3j neow3j = Neow3j.build(new HttpService("http://localhost:40332"));Copy to clipboardErrorCopied
```

这假设Neo节点在`http://localhost:40332`监听。用您想要连接的节点地址替换该URL。通过这种方式实例化`Neow3j`对象，它将使用默认设置进行配置。一些`Neow3j`配置设置将基于节点的协议设置，这些设置在实例化`Neow3j`对象期间获取，例如，轮询间隔默认设置为每个区块的毫秒数。如果您想使用自定义值，可以额外传递一个已配置的`Neow3jConfig`对象。例如：

```java
long pollingInterval = 30000; // in milliseconds
Neow3j neow3j = Neow3j.build(new HttpService("http://localhost:40332"),
                             defaultNeow3jConfig().setPollingInterval(pollingInterval));Copy to clipboardErrorCopied
```

`Neow3jConfig` 类包含 `Neow3j` 特定的配置值/对象，如轮询间隔、使用哪个ScheduledExecutorService，或使用哪个Neo Name Service Resolver合约。

> <details>
>
> <summary>地址版本</summary>
>
> 作为例外， `Neow3jConfig` 也有一个静态变量，用于保存neow3j中使用的地址版本。它被设计为静态的，因为在没有 `Neow3j` 实例可访问的上下文中需要地址版本。确保地址版本与您正在使用的地址类型匹配，并在必要时使用 `Neow3jConfig.setStaticAddressVersion(byte version)` 进行调整。这确保了与您正在交互的特定地址格式的兼容性。
>
> </details>

> <details>
>
> <summary>离线Neow3j实例</summary>
>
> 如果您想构建一些需要 `Neow3j` 实例但您没有节点或不希望执行任何RPC的东西，您可以在 `build()` 函数中使用 `OfflineService` ，或直接实例化一个具体的 `JsonRpc2_0Neow3j` 对象并使用自定义协议。前者将拒绝任何RPC，而后者将跳过初始RPC来获取节点的协议值以实例化 `Neow3j` 实例的配置。
>
> </details>

现在我们已经设置了一个`Neow3j` 实例，我们可以开始探索与Neo区块链的潜在交互。`Neow3j` 上的大多数方法构造并返回一个 `Request` 对象，该对象指定请求和期望的响应格式。在此请求上使用 `send()` 实际将其发送到Neo节点。返回的类型将是 `Response` 的子类。

为了避免遇到意外的 `NullPointerException`，您可以在响应对象上使用 `hasError()`、`getError()` 或`throwOnError()` 等方法，在访问任何其他响应数据之前平滑地处理错误。这些方法有助于确保在通过neow3j与Neo区块链交互时进行健壮的错误处理。

`Neow3j` 上可用的另一组方法基于[RxJava](https://github.com/ReactiveX/RxJava)并返回您可以订阅的 `Observable` 。这些方法将在下一节中简要探讨，该节涵盖区块链监控。这种方法使您能够使用neow3j异步观察和响应来自Neo区块链的事件和数据。

### [Monitoring the Blockchain](https://neow3j.io/#/neo-n3/dapp_development/interacting_with_a_node?id=monitoring-the-blockchain)

区块链应用程序中的一个常见用例涉及跟踪新区块及其内容。`Neow3j` 提供了几种追赶和订阅新区块的方法。从Neo节点检索的任何区块都将转发给您的订阅者。以下示例演示了从区块索引100开始获取所有区块并订阅新生成的区块。布尔参数控制您是否希望接收每个区块的完整交易数据。

```java
neow3j.catchUpToLatestAndSubscribeToNewBlocksObservable(new BigInteger("100"), true)
        .subscribe((blockReqResult) -> {
            System.out.println("blockIndex: " + blockReqResult.getBlock().getIndex());
            System.out.println("hashId: " + blockReqResult.getBlock().getHash());
            System.out.println("confirmations: " + blockReqResult.getBlock().getConfirmations());
            System.out.println("transactions: " + blockReqResult.getBlock().getTransactions());
        });Copy to clipboardErrorCopied
```

如果您对历史区块不感兴趣，并且希望从最新区块开始订阅：

```java
neow3j.subscribeToNewBlocksObservable(true)
        .subscribe((blockReqResult) -> {
            System.out.println("blockIndex: " + blockReqResult.getBlock().getIndex());
            System.out.println("hashId: " + blockReqResult.getBlock().getHash());
            System.out.println("confirmations: " + blockReqResult.getBlock().getConfirmations());
            System.out.println("transactions: " + blockReqResult.getBlock().getTransactions());
        });Copy to clipboardErrorCopied
```

### [检查交易](https://neow3j.io/#/neo-n3/dapp_development/interacting_with_a_node?id=inspecting-a-transaction)

您可以从前面部分的区块订阅中检索交易信息，但您也可以通过获取单个交易的信息来更加具体。例如，如果您向节点发送了一笔交易，现在想检查它在区块链上的状态。

```java
Hash256 txHash = new Hash256("da5a53a79ac399e07c6eea366c192a4942fa930d6903ffc10b497f834a538fee");
NeoGetTransaction response = neow3j.getTransaction(txHash).send();
if (response.hasError()) {
    throw new Exception("Error fetching transaction: " + response.getError().getMessage());
}
Transaction tx = response.getTransaction();Copy to clipboardErrorCopied
```

`Transaction` 对象将包含关于交易的所有信息，如为其支付的费用、它被包含在的区块，或自交易被包含在区块中以来已添加了多少个区块。如果您需要原始字节数组形式的交易，可以使用 `getRawTransaction` 方法代替。此方法将提供表示交易字节的Base64编码字符串。

```java
NeoGetRawTransaction response = neow.getRawTransaction(txHash).send();
String tx = response.getRawTransaction();Copy to clipboardErrorCopied
```

对于大多数交易，dApp对调用输出感兴趣。您可以使用 `getApplicationLog` 方法检索调用的结果。

```java
Hash256 txHash = new Hash256("da5a53a79ac399e07c6eea366c192a4942fa930d6903ffc10b497f834a538fee");
NeoGetApplicationLog response = neow.getApplicationLog(txHash).send();
if (response.hasError()) {
    throw new Exception("Error fetching transaction's app log: " + response.getError().getMessage());
}
// Get the first execution. Usually there is only one execution.
NeoApplicationLog.Execution execution = response.getApplicationLog().getExecutions().get(0);
// Check if the execution ended in a NeoVM state FAULT.
if (execution.getState().equals(NeoVMStateType.FAULT)) {
    throw new Exception("Invocation failed");
}
// Get the result stack.
List<StackItem> stack = execution.getStack();
StackItem returnValue = stack.get(0);

// Get the notifications fired by the transaction.
List<NeoApplicationLog.Execution.Notification> notifications = execution.getNotifications();Copy to clipboardErrorCopied
```

应用程序日志中包含的堆栈将包含调用返回的所有堆栈项。通常，返回堆栈由位于索引0处的一个返回值组成。了解调用返回的堆栈项类型对于正确解释它很重要。

除了返回值外，您还可以检查由交易触发的通知。应用程序日志中的通知提供了跟踪智能合约活动的方法。目前，没有直接监控智能合约的直接方法。相反，您需要订阅新区块、检查交易应用程序日志，并通过将合约哈希与 `notification.getContract()` 进行比较来检查合约触发的通知。

### [在节点上使用钱包](https://neow3j.io/#/neo-n3/dapp_development/interacting_with_a_node?id=using-a-wallet-on-the-node)

如果您运行自己的Neo全节点，可以利用直接存储在该节点上的钱包。Neow3j提供了与这些钱包交互和使用的必要方法。

首先，您需要打开一个钱包：

```java
NeoOpenWallet response = neow3j.openWallet("/path/to/wallet.json", "walletPassword").send();
if (response.hasError()) {
    throw new Exception("Failed to open wallet. Error message: " + response.getError().getMessage());
}

if (response.getOpenWallet()) {
    System.out.println("Successfully opened wallet.");
} else {
    System.out.println("Wallet not opened.");
}Copy to clipboardErrorCopied
```

一旦钱包打开，您可以列出该钱包中的帐户：

```java
NeoListAddress response = neow3j.listAddress().send();
if (response.hasError()) {
    throw new Exception("Failed to fetch wallet accounts. Error message: " + response.getError().getMessage());
}
List<NeoAddress> listOfAddresses = response.getAddresses();Copy to clipboardErrorCopied
```

检查钱包余额：

```java
NeoGetWalletBalance response = neow3j.getWalletBalance(NeoToken.SCRIPT_HASH).send();
if (response.hasError()) {
    throw new Exception("Failed to get wallet balance. Error message: " + response.getError().getMessage());
}
String balance = response.getWalletBalance().getBalance();Copy to clipboardErrorCopied
```

最后，关闭钱包：

```java
NeoCloseWallet response = neow3j.closeWallet().send();
if (response.hasError()) {
    throw new Exception("Failed to close the wallet. Error message: " + response.getError().getMessage());
}Copy to clipboardErrorCopied
```

### [Neo-Express](https://neow3j.io/#/neo-n3/dapp_development/interacting_with_a_node?id=neo-express)

`io.neow3j.protocol.Neow3jExpress` 类扩展了`Neow3j` 的方法，并添加了特定于neo-express的方法。[Neo-express](https://github.com/neo-project/neo-express)是一个开发者工具，促进了简化的工作流程，提供了用于管理和配置私有网络的开发工具。

Neo-express相比普通Neo节点暴露了额外的RPC方法，可通过 `Neow3jExpress` 访问。使用 `Neow3jExpress` 的方式与 `Neow3j` 类似，但使用指向neo-express实例的URL。

```java
Neow3jExpress neow3j = Neow3jExpress.build(new HttpService("http://localhost:40332"));Copy to clipboardErrorCopied
```

此API对于作为本地Neo网络设置的一部分为neo-express创建工具的开发者特别有用。
