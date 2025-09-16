# 部署

您可以使用Neo生态系统中的不同工具部署智能合约，例如Neo区块链工具包。在这里，我们展示如何使用neow3j进行部署。在我们的智能合约项目中，我们喜欢使用单独的源集来保存与部署相关的代码。[环境搭建](https://neow3j.io/#/neo-n3/smart_contract_development/setup_and_compilation)指南中描述了如何添加这样的源集。

[样板](https://github.com/neow3j/neow3j-boilerplate)仓库展示了这可能的样子。

部署的核心类是`ContractManagement`，它在[这里](https://neow3j.io/#/neo-n3/dapp_development/smart_contracts?id=contractmanagement)介绍过，是neow3j SDK的一部分。因此，您需要模块`io.neow3j:contract`。有关如何导入它，请参阅[快速开始](https://neow3j.io/#/README?id=quickstart)部分。在调用`ContractManagement`合约之前，我们需要编译我们的合约代码。

```java
CompilationUnit res = new Compiler().compile(HelloWorldSmartContract.class.getCanonicalName(), substitutions);Copy to clipboardErrorCopied
```

然后我们可以通过调用其`deploy`方法将生成的NEF和清单传递给`ContractManagement`。

```java
AccountSigner signer = AccountSigner.none(deploymentAccount);
Hash160 owner = deploymentAccount.getScriptHash();
TransactionBuilder builder = new ContractManagement(neow3j)
        .deploy(res.getNefFile(), res.getManifest(), hash160(owner))
        .signers(signer);
Hash256 txHash = builder.sign().send().getSendRawTransaction().getHash();
Await.waitUntilTransactionIsExecuted(txHash, neow3j);Copy to clipboardErrorCopied
```

请注意，我们还将`hash160(owner)`作为参数传递给`ContractManagement`的`deploy`方法。该参数将传递给您的合约的`_deploy`方法，因此您可以使用它在部署时配置您的合约。`ContractManagement`合约将部署您的合约，然后在其上调用`_deploy`方法。您可以通过使用`@OnDeployment`注解方法将此方法添加到您的合约中（参见[这里](https://neow3j.io/#/neo-n3/smart_contract_development/devpack?id=_deploy)）。以下是此类方法的示例。

```java
@OnDeployment
public static void deploy(Object data, boolean update) {
        if (!update) {
                Storage.put(ctx, OWNER_KEY, (Hash160) data);
        }
}Copy to clipboardErrorCopied
```

上面我们将`hash160(owner)`作为部署参数传递，但是，如果您想传递多个参数，请使用`ContractParameter.array(...)`并将所有参数添加到其中。

最后，要获取新部署合约的哈希，您可以执行以下代码。哈希取决于部署交易的发送者、NEF校验和以及合约的名称。即使您将来更新合约，哈希也不会改变。

```java
Hash160 contractHash = SmartContract.getContractHash(
        a.getScriptHash(), 
        nefFile.getCheckSumAsInteger(), 
        manifest.getName());
```
