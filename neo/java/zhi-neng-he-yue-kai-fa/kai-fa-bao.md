# 开发包

Neow3j开发包提供了用Java编写智能合约所需的类、方法和注解。例如，如果您的智能合约需要验证交易签名，开发包就提供了相应的方法。或者，如果您希望在合约清单中发布关于合约的详细信息，您可以使用开发包的某个注解。以下部分描述了开发包的API、常用概念和构造。这里没有描述每个类。请也使用[Javadocs](https://javadoc.io/doc/io.neow3j/devpack/latest/overview-summary.html)来获取开发包功能的概览。

### [哈希](https://neow3j.io/#/neo-n3/smart_contract_development/devpack?id=hashes)

与Neow3j SDK一样，开发包为哈希提供了特殊类型。对合约和账户哈希使用`Hash160`，对交易和区块哈希使用`Hash256`。这两种类型的底层栈项都是NeoVM字节字符串，因此，与`ByteString`之间的转换不需要实际的转换。当您使用构造函数`Hash160(ByteString value)`或`Hash256(ByteString value)`时，开发包**不会**检查值是否为具有正确大小的有效哈希。如果您需要检查有效性，请对对象使用`Hash160.isValid(Object o)`或`Hash256.isValid(Object o)`。

### [存储](https://neow3j.io/#/neo-n3/smart_contract_development/devpack?id=storage)

Neo区块链上的每个智能合约都有自己的键值存储。这个存储通过所谓的存储上下文访问。上下文是访问合约存储的网关。您和存储之间的这个额外概念可能允许您将上下文传递给另一个合约，然后该合约可以直接访问您合约的存储。在开发包中，存储上下文由`io.neow3j.devpack.StorageContext`类表示。

访问存储的方法位于`io.neow3j.devpack.Storage`和`io.neow3j.devpack.StorageMap`类上。它们为不同的键和返回类型提供了许多`put`和`get`方法。由于这种方法调用总是需要存储上下文，因此使用`Storage.getStorageContext()`一次性检索`StorageContext`，将其存储在静态类变量中并在每次访问存储时重用它是有意义的。这可能在合约调用中节省GAS。如果您希望为特定目的保留存储的一个段，请使用`StorageMap`。存储映射使用前缀，该前缀会附加到该映射中使用的每个键上。

请注意，存储键和值的大小分别限制为64字节和65535字节。使用`StorageMap`时，映射前缀计入键大小。

### [智能合约接口](https://neow3j.io/#/neo-n3/smart_contract_development/devpack?id=smart-contract-interfaces)

您的智能合约可以通过`Contract.call(Hash160 scriptHash, String method, byte callFlags, Object[] arguments)`方法以临时的方式调用其他合约。但是，还有另一种方法，即使用我们所谓的合约接口。在开发包的上下文中，合约接口是为已部署合约提供接口的类。这里的“接口”一词不是按Java的含义使用，而是指作为通向区块链上实际合约实例的网关的含义。例如，`NeoToken`类包含原生NeoToken合约的所有方法，并允许您在自己的智能合约中调用它。这些合约接口类位于`io.neow3j.devpack.contracts`包中。Neo的所有原生合约都在这里表示，加上一些其他类，例如，用于访问代币合约的接口。

如果您需要调用原生合约的方法，例如，获取最新区块的哈希，请使用合约接口上的相应方法。

```java
Hash256 blockHash = new LedgerContract().currentHash();Copy to clipboardErrorCopied
```

除了现有的合约接口外，您还可以定义自己的接口。有效合约接口的要求是：

* 扩展`io.neow3j.devpack.contracts.ContractInterface`类。
* 创建一个带有单个`Hash160`参数的构造函数，并在构造函数中调用`super()`而不做任何其他逻辑。

自定义合约接口的最小版本如下所示。

```java
class MyContract extends ContractInterface {
    MyContract(Hash160 scriptHash) {
        super(scriptHash);
    }
}Copy to clipboardErrorCopied
```

或者，如果在编译时已知合约哈希，可以使用带有单个`String`参数的构造函数：

```java
class MyContract extends ContractInterface {
    MyContract(String scriptHash) {
        super(scriptHash);
    }
}Copy to clipboardErrorCopied
```

在这个最小形式中，该类只通过从`ContractInterface`继承的`getHash()`方法提供对合约哈希的访问。任何其他合约方法都必须根据其在合约清单中的签名添加。假设合约有一个带有`ByteString`参数和`ByteString`返回类型的方法`findElement`，您需要添加以下方法。请注意，该方法需要是native的。不需要方法体实现，因为这只是对区块链上实际合约实例的接口。

```java
public native ByteString findElement(ByteString key);Copy to clipboardErrorCopied
```

开发包提供已经包含遵循标准的合约API的抽象合约接口。例如，如果您想为同质化代币合约建立合约接口，可以扩展`FungibleToken`类。NEP-17代币合约的所有方法都已可用，您只需添加您的合约可能具有的额外方法（只需如前所示添加它们）。

```java
class MyTokenContract extends FungibleToken {
    public MyTokenContract(String contractHash) {
        super(contractHash);
    }

    public native int someCustomMethod(String arg);

}Copy to clipboardErrorCopied
```

如果您的同质化代币合约没有任何其他方法，或者您不需要它们，您可以简单地用您的代币合约哈希初始化一个`FungibleToken`并访问其NEP-17方法。

```java
new FungibleToken("0xef4073a0f2b305a38ec4050e4d3d28bc40ea63f5").balanceOf(owner);Copy to clipboardErrorCopied
```

查看`io.neow3j.devpack.contracts`包了解更多此类合约接口。

### [原生合约](https://neow3j.io/#/neo-n3/smart_contract_development/devpack?id=native-contracts)

#### [StdLib](https://neow3j.io/#/neo-n3/smart_contract_development/devpack?id=stdlib)

当使用`StdLib.jsonSerialize(Object o)`方法处理包含字节字符串或字节数组的值或对象（例如，`ByteString`、`byte[]`或`Hash160`）时，请确保首先对该值进行Base64编码。否则，它将被解释为UTF-8编码的字符串，这可能导致错误。它不会在JSON中以十六进制字符串的形式呈现。

#### [Neo名称服务](https://neow3j.io/#/neo-n3/smart_contract_development/devpack?id=neo-name-service)

Neo名称服务合约（NNS）技术上不是原生合约，但由Neo基金会维护和发行。开发包为NNS提供了合约接口，类为`io.neow3j.devpack.contracts.NeoNameService`。请注意，该类没有固定的脚本哈希，因为它不是原生合约。如果您想在合约中使用该类，可以简单地初始化它的实例。

```java
new NeoNameService("a92fbe5bf164170a624474841485b20b45a26047");Copy to clipboardErrorCopied
```

请确保脚本哈希等于NNS合约的当前脚本哈希。

### [事件](https://neow3j.io/#/neo-n3/smart_contract_development/devpack?id=events)

Neo智能合约可以触发事件。例如，它们会出现在合约调用的[应用程序日志](https://docs.neo.org/docs/n3/reference/rpc/getapplicationlog.html)中。合约可以触发的事件在其清单中列出。下面的JSON显示了这可能的样子。

```json
"events": [
    {
        "name": "transfer",
        "parameters": [
            {
                "name": "arg1",
                "type": "Integer"
            },
            {
                "name": "arg2",
                "type": "String"
            }
        ]
    }
]Copy to clipboardErrorCopied
```

事件由其名称和与其一起传递的状态参数定义。开发包允许您定义和使用最多16个状态参数的事件。表示这些事件的类位于[`io.neow3j.devpack.events`](https://javadoc.io/doc/io.neow3j/devpack/latest/io/neow3j/devpack/events/package-summary.html)包中。

事件在静态合约变量中声明，如以下代码片段所示。它们不能在方法体内或在非主合约类的类中声明。

```java
    @DisplayName("mint")
    private static Event1Arg<Integer> onMint;

    @DisplayName("transfer")
    private static Event2Args<Integer, String> onTransfer;Copy to clipboardErrorCopied
```

不需要用实际实例初始化变量。这对Java开发者来说是反直觉的，但是，这些变量并不意在拥有实际值。它们只是定义，具有名称以及状态参数的数量和类型。所有事件类都遵循命名模式`Event[n]Args`，其中`n`是事件接受的状态参数数量。`@DisplayName`注解是可选的，可用于为事件定义与变量名不同的名称。如果不使用，则变量名就是事件名。

一旦声明了事件，就可以在合约方法中通过调用其`fire(...)`方法来使用它。

```java
    public static boolean transfer() throws Exception {
        ...
        onTransfer.fire(transferAmount, "tokens transferred!");
        ...
        return true;
    }Copy to clipboardErrorCopied
```

> \*\*注意：\*\*不允许在[verify方法](https://neow3j.io/#/neo-n3/smart_contract_development/devpack?id=verify)中触发事件。

### [特殊合约方法](https://neow3j.io/#/neo-n3/smart_contract_development/devpack?id=special-contract-methods)

有几个合约方法具有特殊目的。neow3j开发包提供注解来在您的合约代码中标记它们。使用注解将使在代码中更容易发现这些方法，并允许编译器进行检查，帮助更快地发现这些方法中的错误。

#### [\_deploy](https://neow3j.io/#/neo-n3/smart_contract_development/devpack?id=_deploy)

该方法在合约部署或更新后立即被调用。更准确地说，当您&#x5728;_&#x43;ontractManagemen&#x74;_&#x4E0A;调用`deploy`或`update`时，_ContractManagemen&#x74;_&#x5408;约将在您的合约上调用该方法。您可以使用它在部署时设置和配置您的合约。在指定的方法上使用开发包的`io.neow3j.devpack.annotations.OnDeployment`注解。您的方法名不必为`_deploy`，可以是任何名称。尽管如此，在合约清单中它将以`_deploy`名称显示。该方法的签名必须为`void methodName(Object data, boolean isUpdate)`。

#### [verify](https://neow3j.io/#/neo-n3/smart_contract_development/devpack?id=verify)

如果您的合约以验证触发器调用，则会调用该方法。例如，当合约拥有代币并且您发出将这些代币转移到另一个账户/合约的提取交易时，合约会以验证触发器调用。换句话说，合约的`verify`方法被调用。大多数情况下，`verify`方法包含对合约所有者的简单见证检查。

在指定的方法上使用开发包的`io.neow3j.devpack.annotations.OnVerification`注解。您的方法名不必为`verify`，可以是任何名称。尽管如此，在合约清单中它将以`verify`名称显示。该方法必须返回布尔值，并可以有任意数量的参数。

> \*\*注意：\*\*Neo节点不允许verify方法触发任何事件。如果在此方法内触发事件，编译器将抛出异常。

#### [onNEP17Payment](https://neow3j.io/#/neo-n3/smart_contract_development/devpack?id=onnep17payment)

您的合约需要此方法才能从 NEP-17 合约接收代币，即同质化代币，如NEO或GAS。任何遵循 NEP-17 标准的合约在其代币转移到您的合约时都会在您的合约上调用此方法。

在指定的方法上使用开发包的`io.neow3j.devpack.annotations.OnNEP17Payment`注解。您的方法名不必为`onNEP17Payment`，可以是任何名称。尽管如此，在合约清单中它将以`onNEP17Payment`名称显示。该方法的签名必须为`void methodName(Hash160 sender, int amount, Object data)`。

#### [onNEP11Payment](https://neow3j.io/#/neo-n3/smart_contract_development/devpack?id=onnep11payment)

您的合约需要此方法才能从 NEP-11 合约接收代币，即非同质化代币。任何遵循 NEP-11 标准的合约在其代币转移到您的合约时都会在您的合约上调用此方法。在指定的方法上使用开发包的`io.neow3j.devpack.annotations.OnNEP11Payment`注解。您的方法名不必为`onNEP11Payment`，可以是任何名称。尽管如此，在合约清单中它将以`onNEP11Payment`名称显示。该方法的签名必须为`void methodName(Hash160 sender, int amount, ByteString tokenId, Object data)`。

### [权限、信任、组、安全方法和调用标志](https://neow3j.io/#/neo-n3/smart_contract_development/devpack?id=permissions-trusts-groups-safe-methods-and-call-flags)

Neo区块链上授权的基础是密码学签名。用户在合约调用中附加签名来证明他们有权在智能合约中执行某些操作。这种授权可能被智能合约滥用。恶意合约可以使用签名执行用户非意图的代币转移。为防止这种情况，Neo应用了见证范围，允许用户限制其见证/签名的使用。默认情况下，见证仅在作为调用入口点的合约中有效。范围可以扩展到特定合约、合约组或全局范围。

#### [组](https://neow3j.io/#/neo-n3/smart_contract_development/devpack?id=groups)

智能合约组由EC公钥指定。合约清单包含合约与组的附属关系，如下所示。

```json
"groups": [
    {
        "pubkey":"033a4d051b04b7fc0230d2b1aaedfd5a84be279a5361a7358db665ad7857787f1b",
        "signature":"DEA2r+67KlQc/dIvEdOXMIGCm7x+V5vXT5ZRtGRwOlDxBuqzur/OU8OSitiYn5f6tQywog8FziGp5S2VI2/5Wnxj"
    }
],Copy to clipboardErrorCopied
```

公钥标识组，签名是合约发起者拥有相应私钥材料的证明。签名从合约的哈希创建，需要进行Base64编码。因此，如果您想将合约添加到组中，您首先需要编译它，计算合约哈希，在该哈希上创建签名，并用组的公钥和生成的签名扩展合约清单。请注意，合约哈希取决于用于部署合约的账户。即，您需要提前知道将使用哪个账户来部署合约。要检索合约哈希并生成签名，您可以使用neow3j SDK，如以下示例代码所示。

```java
Hash160 sender = ...;
ContractManifest manifest = ...;
NefFile nefFile = ...;
Hash160 contractHash = SmartContract.calcContractHash(sender, nefFile.getCheckSumAsInteger(), manifest.getName());

ECKeyPair keyPair = ...;
Sign.SignatureData sig = Sign.signMessage(contractHash.toArray(), keyPair);
String encSig = Base64.encode(sig.getConcatenated());Copy to clipboardErrorCopied
```

您将必须手动修改合约清单JSON文件，并将生成的编码签名和公钥添加到`groups`部分。

#### [权限](https://neow3j.io/#/neo-n3/smart_contract_development/devpack?id=permissions)

除了见证范围外，智能合约安全性通过合约开发者可为其合约定义的权限和信任系统得到改善。权限定义了您的合约被允许调用哪些合约。它们是主动强制的，意味着一旦在合约清单中定义，从您合约内对权限中不包含的合约和方法的任何调用都会失败。要定义权限，请在合约类的类级别使用`io.neow3j.devpack.annotations.Permission`注解。默认情况下，您的合约将没有权限。

以下是一个示例配置。它允许您的合约调用哈希为`726cb6e0cd8628a1350a611384688911ab75f51b`的合约的任何方法，哈希为`d2a4cff31913016155e38e474a2c06d08be276cf`的合约的`getBalance`和`transfer`方法，以及公钥为`033a4d051b04b7fc0230d2b1aaedfd5a84be279a5361a7358db665ad7857787f1b`的组中任何合约的`commonMethodName`方法。

```java
@Permission(contract = "726cb6e0cd8628a1350a611384688911ab75f51b", methods = "*")
@Permission(contract = "d2a4cff31913016155e38e474a2c06d08be276cf", methods = {"getBalance", "transfer"})
@Permission(contract = "033a4d051b04b7fc0230d2b1aaedfd5a84be279a5361a7358db665ad7857787f1b", methods = "commonMethodName")
public class MyContract {Copy to clipboardErrorCopied
```

要为原生合约设置权限，您可以在注解中使用`nativeContract`字段和枚举`NativeContract`。正如您可能注意到的，上面示例代码中的第二个权限指的是原生GasToken合约。由于它是原生合约，您也可以使用以下注解，结果完全相同。

```java
@Permission(nativeContract = NativeContract.GasToken, methods = {"getBalance", "transfer"})Copy to clipboardErrorCopied
```

如果您想允许您的合约调用任何其他合约，请使用通配符权限`@Permission(contract = "*", methods = "*")`。

#### [信任](https://neow3j.io/#/neo-n3/smart_contract_development/devpack?id=trusts)

信任定义了哪些合约可以调用您的合约，但是，与权限不同，它们不被强制执行。即，您不能阻止其他合约调用您的合约。信任只是一个定义，钱包和其他dApp可以使用它在调用不信任调用合约的合约时告知用户。

默认情况下，信任属性为空，即没有合约被信任。使用`io.neow3j.devpack.annotations.Trust`注解来定义信任，如以下示例所示。第一个条目基于单个智能合约哈希，第二个基于合约组的公钥。

```java
@Trust(contract = "acce6fd80d44e1796aa0c2c625e9e4e0ce39efc0")
@Trust(contract = "033a4d051b04b7fc0230d2b1aaedfd5a84be279a5361a7358db665ad7857787f1b")
public class MyContract {Copy to clipboardErrorCopied
```

要信任原生合约，而不是将其哈希添加到`contract`属性，您可以使用`nativeContract`属性，方式与[上面](https://neow3j.io/#/neo-n3/smart_contract_development/devpack?id=permissions)指定的`@Permission`注解中的使用方式相同。在以下示例中，原生StdLib被信任：

```java
@Trust(nativeContract = NativeContract.StdLib)
public class MyContract {Copy to clipboardErrorCopied
```

如果您想信任任何合约，请使用通配符选项`@Trust("*")`。

#### [安全方法](https://neow3j.io/#/neo-n3/smart_contract_development/devpack?id=safe-methods)

不改变合约状态且不触发事件的方法可以在只读模式下安全调用。要向Neo网络发信号，您可以在方法级别使用`io.neow3j.devpack.annotations.Safe`注解。该方法将在合约清单中被标记为安全。

如果您的`@Safe`注解方法确实改变了状态或触发了事件，该方法的调用将失败。

#### [调用标志](https://neow3j.io/#/neo-n3/smart_contract_development/devpack?id=call-flags)

调用标志允许您限制在合约内调用的合约的操作。例如，您可以拒绝进一步调用其他合约、改变区块链状态或触发事件。

可能的标志在`io.neow3j.devpack.constants.CallFlags`中定义和记录，旨在在`io.neow3j.devpack.Contract`类的`call`方法中使用。

### [占位符替换](https://neow3j.io/#/neo-n3/smart_contract_development/devpack?id=placeholder-substitution)

开发包提供了在编译前替换合约中使用的字符串的可能性。主合约类中的任何字符串字面量都可以被替换，甚至包括注解值。目前，在合约类中使用的辅助类不支持此功能。

您必须以编程方式编译合约才能使用此功能。查看[此](https://neow3j.io/#/neo-n3/smart_contract_development/setup_and_compilation?id=programmatic-compilation)部分了解如何以编程方式进行编译。`Compiler`提供了一个接受`Map<String, String>`参数的`compile`方法。这是替换映射，告诉编译器哪些字符串是占位符字符串（映射的键）以及它们应该被替换为哪些值（映射的值）。合约中的占位符必须遵循语法`"${*}"`，其中`*`将用作替换映射中的键。

```java
    Map<String, String> substitutionMap = new HashMap<>();
    substitutionMap.put("contract_hash", "da65b600f7124ce6c79950c1772a36403104f2be");
    substitutionMap.put("event_name", "TheEvent");
    ...
    CompilationUnit res = new Compiler().compile(YourSmartContract.class.getName(), substitutionMap);Copy to clipboardErrorCopied
```

以下是使用占位符的智能合约可能的样子示例。

```java
    @Permission(contract = "${contract_hash}", methods = "*")
    static class PlaceholderSubstitutionContract {

        static final String aString = "${a_string}";
        static final Hash160 OWNER = StringLiteralHelper.addressToScriptHash("${account_address}");

        @DisplayName("${event_name}")
        static Event1Arg<String> event;

        public static String method() {
            String s2 = "${another_string}";
            return s1 + s2;
        }

        public static Hash160 getOwner() {
            return OWNER;
        }
    }Copy to clipboardErrorCopied
```

> \*\*注意：\*\*如果在编译时替换映射中没有指定占位符，将使用合约中存在的字符串（给定该值不限制于任何格式）。例如，`"${account_address}"`必须替换为有效地址，否则编译会失败，而如果`"event_name"`不是替换映射中的键，因此没有提供替换，则`event`将被称为`"${event_name}"`。

占位符替换功能也适用于合约测试。有关更多信息，请参阅[此](https://neow3j.io/#/neo-n3/smart_contract_development/testing?id=deployment-configuration)部分。
