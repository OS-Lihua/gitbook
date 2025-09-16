# 智能合约

要部署、调用或检索有关区块链上任何合约状态的信息，您可以使用`SmartContract`类。它是neow3j SDK中最通用的合约建模类。所有其他合约类都是它的子类。一方面，`SmartContract`提供了在区块链上调用合约的方法，从而导致实际的状态更改。另一方面，您可以使用它来模拟调用而不实际更改状态。后者也用于调用仅执行读取操作的情况。

唯一创建实际交易的方法是`invokeFunction(...)`。它根据提供的函数名和合约参数构建脚本，并使用它构造一个`TransactionBuilder`。然后可以配置返回的交易构建器，构建并发送交易，如[交易](https://neow3j.io/#/neo-n3/dapp_development/transactions)部分所述。此过程在下图中可视化。

```
         构建脚本           ->      配置交易       ->   交易准备签名和发送

        ---------------                  --------------------                -------------
       | SmartContract |        ->      | TransactionBuilder |      ->      | Transaction |
        ---------------                  --------------------                -------------Copy to clipboardErrorCopied
```

不改变区块链状态的方法都以单词“call”开头，以表示执行的调用只是对Neo JSON-RPC API的`invokefunction`或`invokescript`方法的调用。它们不产生交易构建器或交易。有几种这样的“call”方法，基本的一个是`callInvokeFunction(...)`。如果您知道合约调用将返回什么类型，可以使用更具体的调用方法之一，这些方法已经解包调用结果值，例如`callFunctionReturningScriptHash(...)`。

与返回类型相反，合约的每个方法都以特定的参数类型作为输入。要了解智能合约中存在哪些方法以及它需要哪些参数类型，每个合约都有一个清单，为您提供这些以及关于合约的更多信息。在以下部分中，我们将向您展示如何创建合约参数、获取清单，并确定您需要传递给要调用的函数的参数。

### [合约参数](https://neow3j.io/#/neo-n3/dapp_development/smart_contracts?id=contract-parameters)

当调用智能合约时，您将需要参数。neow3j SDK通过`ContractParameter`类表示参数。它提供了许多静态构造方法，涵盖所有可能的参数类型。如果您使用这些方法，neow3j将确保参数以正确的编码和正确的类型声明发送到合约。例如，如果您需要将NEO地址的脚本哈希作为参数传递，可以使用`ContractParameter.hash160(...)`方法。它将脚本哈希转换为预期的字节数组。

如果您调用一个以对象作为参数的合约，您需要使用类型为`Array`的合约参数。作为示例，假设下面的`Bongo`结构体类是预期的方法参数。

```java
@Struct
public class Bongo {

    public String lowNote;
    public String highNote;

    public Bongo(String lowNote, String highNote) {
        this.lowNote = lowNote;
        this.highNote = highNote;
    }
}Copy to clipboardErrorCopied
```

使用neow3j SDK，您需要构造以下参数来表示`Bongo`实例。

```java
ContractParameter.array(
    ContractParameter.string("C2"),
    ContractParameter.string("C5"));Copy to clipboardErrorCopied
```

当对象用于返回类型时，也适用相同的概念。换句话说，期望一个`Array`类型的返回值，该值按照对象变量在类中出现的顺序保存对象的变量。

### [合约调用](https://neow3j.io/#/neo-n3/dapp_development/smart_contracts?id=contract-invocation)

首先，指定您要调用的合约和函数。使用`io.neow3j.contract.Hash160`类表示合约的脚本哈希，并与`Neow3j`实例一起创建`SmartContract`。

```java
Hash160 scriptHash = new Hash160("0x1a70eac53f5882e40dd90f55463cce31a9f72cd4");
SmartContract smartContract = new SmartContract(scriptHash, neow3j);Copy to clipboardErrorCopied
```

然后，定义将传递给合约的参数。要确定所需的参数，您需要有关其方法的信息。您可以通过读取清单中合约的ABI来获取此信息。以下示例展示了如何检索智能合约的所有方法及其名称、参数和返回类型，以了解函数需要哪些参数以及您可以期望返回什么：

```java
List<ContractManifest.ContractABI.ContractMethod> methods = smartContract.getManifest().getAbi().getMethods();Copy to clipboardErrorCopied
```

在此示例中，我们调用一个名称服务合约，并使用域名和应在该域名下注册的地址调用`register`函数。

```java
Account account = Account.fromWIF("L3kCZj6QbFPwbsVhxnB8nUERDy4mhCSrWJew4u5Qh5QmGMfnCTda");
ContractParameter domainParam = ContractParameter.string("myname.neo");
ContractParameter accountParam = ContractParameter.hash160(account.getScriptHash());Copy to clipboardErrorCopied
```

使用智能合约实例、函数和参数，我们可以按如下方式构造调用。这还没有发送交易，但它构建了正确的脚本并使用它实例化了一个交易构建器以进行进一步配置。

```java
String function = "register";
TransactionBuilder txBuilder = smartContract.invokeFunction(function, domainParam, accountParam);Copy to clipboardErrorCopied
```

以下是包含交易构建器配置的完整代码。交易由在域名“myname.neo”下注册的同一账户签名。

```java
Neow3j neow3j = Neow3j.build(new HttpService("http://localhost:40332"));
Hash160 scriptHash = new Hash160("0x1a70eac53f5882e40dd90f55463cce31a9f72cd4");
SmartContract smartContract = new SmartContract(scriptHash, neow3j);

Account account = Account.fromWIF("L3kCZj6QbFPwbsVhxnB8nUERDy4mhCSrWJew4u5Qh5QmGMfnCTda");
ContractParameter domainParam = ContractParameter.string("myname.neo");
ContractParameter accountParam = ContractParameter.hash160(account.getScriptHash());
String function = "register";

NeoSendRawTransaction response = new SmartContract(scriptHash, neow3j)
        .invokeFunction(function, domainParam, accountParam)
        .signers(AccountSigner.calledByEntry(account))
        .sign()
        .send();Copy to clipboardErrorCopied
```

当然，也可以调用不需要任何参数的智能合约函数，例如，一个每次被调用时单纯地增加数字的合约。

```java
NeoSendRawTransaction response = new SmartContract(contract, neow3j)
        .invokeFunction("increment")
        .signers(AccountSigner.calledByEntry(account))
        .sign()
        .send();Copy to clipboardErrorCopied
```

### [测试调用](https://neow3j.io/#/neo-n3/dapp_development/smart_contracts?id=testing-the-invocation)

如果您需要在实际通过网络传播之前了解调用的效果，可以先进行测试调用。这也会调用RPC节点，但只模拟执行而不对区块链产生任何影响。继续上面域名合约的示例，测试调用将如下所示。请注意，根据您执行的调用，即使没有更改区块链状态，您也需要添加签名者。在这个具体的示例中，需要签名者是因为被调用的合约将验证注册的账户是否也是交易的发送者。

```java
Neow3j neow3j = Neow3j.build(new HttpService("http://localhost:40332"));

Hash160 scriptHash = new Hash160("0x1a70eac53f5882e40dd90f55463cce31a9f72cd4");
String function = "register";

Account account = Account.fromWIF("L3kCZj6QbFPwbsVhxnB8nUERDy4mhCSrWJew4u5Qh5QmGMfnCTda");
ContractParameter domainParam = ContractParameter.string("neo.com");
ContractParameter accountParam = ContractParameter.hash160(account.getScriptHash());
List<ContractParameter> params = Arrays.asList(domainParam, accountParam);

NeoInvokeFunction response = new SmartContract(scriptHash, neow3j)
        .callInvokeFunction(function, params, AccountSigner.calledByEntry(account));Copy to clipboardErrorCopied
```

`NeoInvokeFunction`包含有关合约执行中消耗的GAS数量、VM退出状态（例如HALT或FAULT）和VM的堆栈（即返回值）的信息。

### [合约接口](https://neow3j.io/#/neo-n3/dapp_development/smart_contracts?id=contract-interfaces)

有几个`SmartContract`的子类实现了一种与Neo原生合约和遵循代币标准的合约的接口类型。在这里，“接口”一词意味着这些类提供了与已部署合约交互的方式。代币合约，包括同质化和非同质化代币（如`NeoToken`和`GasToken`），在[代币合约](https://neow3j.io/#/neo-n3/dapp_development/token_contracts)部分单独讨论。下面讨论其他可用的合约类。

#### [合约管理](https://neow3j.io/#/neo-n3/dapp_development/smart_contracts?id=contractmanagement)

`ContractManagement`合约是一个Neo原生合约，正如其名称所示，可用于管理其他合约。更具体地说，它允许您部署、更新和删除合约。在neow3j SDK中，更新和删除方法无法使用，因为它们只能从另一个合约内部调用。然而，部署方法可用，允许您使用neow3j部署新合约。例如：

```java
Transaction tx = new ContractManagement(neow3j)
        .deploy(nef, manifest)
        .signers(AccountSigner.calledByEntry(account1)))
        .sign();Copy to clipboardErrorCopied
```

生成必要的NEF（Neo可执行格式）文件和合约清单在这里不做讨论，但它们是合约开发[部分](https://neow3j.io/#/)的一部分。

`ContractManagement`上还有两个其他方法。它们与最小部署费用有关。任何人都可以使用getter `getMinimumDeploymentFee()`。然而，setter `setMinimumDeploymentFee(...)`只有在交易由委员会成员签名时才能成功使用，这意味着对大多数开发者来说它没有用处。

#### [策略合约](https://neow3j.io/#/neo-n3/dapp_development/smart_contracts?id=policycontract)

`PolicyContract`保存有关Neo网络的几个设置的信息。您可以检索信息，如每交易字节的GAS费用、合约存储每字节的GAS价格，或某个账户是否被列入黑名单。

该合约也为所有这些值提供了setter，尽管它们只有在交易由委员会成员签名时才能成功使用。

#### [角色管理](https://neow3j.io/#/neo-n3/dapp_development/smart_contracts?id=rolemanagement)

`RoleManagement`合约用于为网络中的节点分配角色。节点可以是状态验证器、预言机节点或NeoFS“Alphabet”节点（负责NeoFS侧链上的共识）。

节点对角色的指定只能通过Neo委员会完成。然而，您可以使用`getDesignatedRole(...)`方法检查角色分配。

#### [Neo名称服务](https://neow3j.io/#/neo-n3/dapp_development/smart_contracts?id=neonameservice)

`NeoNameService`不是原生合约，而是由Neo核心团队管理的。此合约的脚本哈希与neow3j未知，必须在构造`NeoNameService`实例时由开发者提供。正如其名称所示，名称服务合约提供了将名称映射到所有者账户的可能性。您可以在官方[Neo文档](https://docs.neo.org/docs/n3/Advances/neons/index.html)中阅读更多相关信息。

neow3j SDK中的`NeoNameService`类为您提供了合约的所有方法。因此，您可以检查已注册的名称并注册您自己的名称到地址映射。其中一些只能由Neo委员会调用，例如`addRoot(...)`或`setPrice(...)`方法。
