# 测试

全面测试智能合约需要在运行的Neo实例上部署它，并可能在特定的链设置下调用其方法。Neow3j提供了一个测试库，允许在JUnit之上方便地进行智能合约测试。该库位于 `io.neow3j:devpack-test` 模块中。

按照[环境搭建](https://neow3j.io/#/neo-n3/smart_contract_development/setup_and_compilation)指南中描述的用Gradle设置智能合约项目后，您也就为编写合约测试做好了准备。

查看[样板仓库](https://github.com/neow3j/neow3j-boilerplate)，了解合约测试类的简单示例。

### [测试配置](https://neow3j.io/#/neo-n3/smart_contract_development/testing?id=test-configuration)

合约测试必须用 `@ContractTest` 注解。除了添加JUnit相关功能外，它还允许您配置测试。以下是一个示例：

```java
@ContractTest(contracts = HelloWorldSmartContractTest.class, blockTime = 1)
public class HelloWorldSmartContractTest {Copy to clipboardErrorCopied
```

**合约**

最重要的是，您需要通过 `contracts` 属性指定要测试的合约。指定的合约将在此测试类中所有测试之前自动编译和部署。

**区块时间**

智能合约测试在底层的Neo区块链实现上运行，该实现以固定时间间隔生成区块。您可以使用`blockTime`属性更改该间隔。

**批处理文件**

一些Neo区块链实现（如neo-express）具有**批处理**功能，允许您执行一系列命令，在运行之前改变区块链状态。因此，您可以在运行测试之前将区块链初始化为所需状态。将批处理文件放&#x5728;_&#x72;esource&#x73;_&#x76EE;录（&#x5373;_&#x73;rc/test/resources_）中，并在`batchFile`属性中设置文件名。批处理文件在所有测试之前应用一次。

查看neo-express官方[文档](https://github.com/neo-project/neo-express/blob/master/docs/command-reference.md#neoxp-batch)，了解如何为neo-express编写批处理文件。

**检查点文件**

一些Neo区块链实现（如neo-express）具有**检查点**功能，允许您加载区块链状态（检查点），该状态是之前从另一个区块链实例导出的。您可以在测试中使用此功能，在运行测试之前应用这样的检查点。区块链将从检查点的状态开始运行。将检查点文件放&#x5728;_&#x72;esource&#x73;_&#x76EE;录（&#x5373;_&#x73;rc/test/resources_）中，并在`checkpoint`属性中设置文件名。检查点文件在所有测试之前应用一次。如果也配置了批处理文件，则首先应用检查点。

查看neo-express官方[文档](https://github.com/neo-project/neo-express/blob/master/docs/command-reference.md#neoxp-checkpoint)，了解如何为neo-express生成检查点文件。

**链配置**

底层区块链实例可以通过配置文件进行配置。例如，neo-express通过 `.neo-express` 配置文件进行配置，该文件指定了共识节点的密钥材料、预设账户和区块链参数等。devpack-test库包含一个默认的neo-express配置文件，其中有一个共识节点和一个控制共识节点私钥的账户。您可以通过将配置文件添加&#x5230;_&#x72;esource&#x73;_&#x76EE;录（&#x5373;_&#x73;rc/test/resources_）并在 `neoxpConfig` 属性中设置文件名来使用自定义配置。默认配置将被覆盖。作为文件模板，您可以使用库的默认neo-express配置文件[这里](https://github.com/neow3j/neow3j/blob/master-3.x/test-tools/src/main/resources/default.neo-express)。

查看neo-express官方[文档](https://github.com/neo-project/neo-express/blob/master/docs/settings.md)，了解neo-express配置文件中可以使用哪些设置。

### [测试扩展](https://neow3j.io/#/neo-n3/smart_contract_development/testing?id=test-extension)

除了 `@ContractTest` 注解外，您还必须将 `ContractTestExtension` 添加到您的测试类中。它是一个JUnit 5测试扩展，钩入测试类的前后阶段。它配置并启动测试区块链，应用批处理和检查点文件，编译和部署您的合约，并在所有测试运行后清理所有内容。

```java
    @RegisterExtension
    private static ContractTestExtension ext = new ContractTestExtension();Copy to clipboardErrorCopied
```

`ContractTestExtension` 有第二个构造函数，它接受一个 `TestBlockchain` 实例。这为测试中使用的底层Neo区块链实现提供了灵活性。目前只存在一个 `TestBlockchain` 的实现。它在docker容器中使用neo-express。未来可能会添加其他实现，例如，直接在测试网上运行测试，但与neo-express实例相比功能有限。

`ContractTestExtension` 提供了几种方法来控制区块链实例。您可以暂停、恢复和快进区块生产，创建新账户或检索创世账户及其关联的私钥。此外，它提供对配置为向底层区块链进行RPC方法调用的 `Neow3j` 的访问，以及对表示已部署合约的 `SmartContract` 对象的访问。您可以在所有测试之前的设置方法中检索这两者，并将它们设置为测试类的静态变量。

```java
    @BeforeAll
    public static void setUp() {
        neow3j = ext.getNeow3j();
        contract = ext.getDeployedContract(HelloWorldSmartContract.class);
    }Copy to clipboardErrorCopied
```

### [部署配置](https://neow3j.io/#/neo-n3/smart_contract_development/testing?id=deployment-configuration)

`devpack-test` 允许您通过为每个合约添加静态配置方法来配置被测试的智能合约的部署。这些方法必须用`@DeployConfig` 注解，注解的值必须设置为此配置所针对的合约类。该方法必须返回`DeployConfiguration`，并且不接受参数或接受一个 `DeployContext` 类型的可选参数。配置在`DeployConfiguration` 对象上进行。目前，它允许您设置部署参数、占位符字符串替换的映射（[这里](https://neow3j.io/#/neo-n3/smart_contract_development/devpack?id=placeholder-substitution)有描述）以及部署交易的签名者（单签和多签账户）。

`DeployContext` 参数允许您访问在部署顺序中位于前面的合约。它还提供部署交易哈希。

```java
    @DeployConfig(ExampleContract.class)
    public static DeployConfiguration config2(DeployContext ctx) {
        DeployConfiguration config = new DeployConfiguration();
        SmartContract sc = ctx.getDeployedContract(AnotherContract.class);
        config.setDeployParam(ContractParameter.hash160(sc.getScriptHash()));

        config.setSubstitution("owner_address", "NXXazKH39yNFWWZF5MJ8tEN98VYHwzn7g3");
        config.setSubstitution("contract_hash", "ef4073a0f2b305a38ec4050e4d3d28bc40ea63f5");

        config.setSigner(AccountSigner.calledByEntry(anAccount));
    }Copy to clipboardErrorCopied
```

示例显示，如果其他被测试的合约在部署顺序中位于前面，您可以访问它们。这允许您获取合约哈希并将其用作另一个合约的部署参数。

请注意，您只能设置一个部署参数。这是根据智能合约中标准化的 `_deploy` 方法。它只接受一个部署参数。因此，如果您想传递多个参数，请将它们打包到一个数组参数中。

另外请注意，如果您使用带有 `@BeforeAll` 注解的 `setUp` 方法，请注意部署配置方法在这样的 `setUp` 方法之前被调用。因此，您在那里设置的内容在配置方法中还不可用。如果您在配置方法中依赖账户，请在测试类的静态构造函数中设置它们。
