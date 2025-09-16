# 交易

交易是更改区块链状态的必要手段。每个更改都是由交易发起的。Neo的交易是相当通用的；它们没有特定的类型。相反，它们的影响由附加的脚本决定。脚本是一系列NeoVM指令，因此可以采用各种形式，尽管交易脚本通常涉及对智能合约的一次或多次调用。您可以在官方Neo文档中找到[这里](https://docs.neo.org/docs/n3/foundation/Transactions.html)描述的Neo交易的完整定义。

neow3j SDK在其`io.neow3j.transaction.Transaction`类中表示Neo交易。它提供了所有必要的功能来序列化为字节数组并将交易发送到Neo节点。然而，手动构建有效交易将是一项繁琐的任务。因此，neow3j提供了`io.neow3j.contract.TransactionBuilder`，它简化了构建有效交易的过程，并尽早地识别错误配置。

### [构建交易](https://neow3j.io/#/neo-n3/dapp_development/transactions?id=building-transactions)

`TransactionBuilder`  和 `Transaction`需要与Neo节点的连接来获取信息（如费用）或发送交易。因此，`TransactionBuilder`使用`Neow3j`对象实例化。一旦构建，构建器允许您设置Neo交易可以具有的几乎所有属性。系统费用（您的交易将消耗的GAS数量）和网络费用（基于您的交易大小和验证所需的努力）都在构建交易时自动获取。

```java
Neow3j neow3j = Neow3j.build(new HttpService("http://127.0.0.1:40332"));
TransactionBuilder builder = new TransactionBuilder(neow3j);Copy to clipboardErrorCopied
```

如果未明确设置，名&#x4E3A;_&#x6E;once_、_validUntilBloc&#x6B;_&#x548C;_versio&#x6E;_&#x7684;属性也将被自动设置。_nonc&#x65;_&#x9632;止重放攻击，即复制并再次发送完全相同的交易。其默认值由交易构建器随机设置。_validUntilBloc&#x6B;_&#x503C;决定交易在需要被包含在区块中之前将保持有效多久。如果在该时间之前未被包含，它将被网络丢弃。默认值设置为连接网络支持的最大值。交&#x6613;_&#x76;ersio&#x6E;_&#x9ED8;认设置为0，您很可能不需要更改。

最感兴趣的属性是交易的脚本和其签名者。如前所述，脚本决定交易对区块链状态的实际影响。这里不讨论构建脚本本身的过程。Neow3j在`contract`模块中提供了许多类，这些类将为您提供一个包含预配置脚本的`TransactionBuilder`。

```java
builder.script(script);Copy to clipboardErrorCopied
```

交易签名者的使用具有双重目的。列表中的第一个签名者指定将支付交易费用的账户，称为交易的`发送者`。此外，所有签名者（包括列表中的第一个）都用于授权脚本内依赖于账户签名的特定操作。例如，从账户A向账户B转移代币通常需要账户A的批准。

您可以使用交易构建器中的`signers(Signer... signers)`方法设置交易的签名者。请注意，参数中提供的第一个签名者将用作交易的发送者。如果您已经指定了所有签名者但希望将发送者更改为另一个签名者，可以使用`firstSigner(Hash160 account)`方法明确设置。

```java
// Import the Account class from the io.neow3j.wallet package
import io.neow3j.wallet.Account;

// Instantiate the Account class as follows
Account account = Account.fromWIF("L24Qst64zASL2aLEKdJtRLnbnTbqpcRNWkWJ3yhDh2CLUtLdwYK2");
builder.signers(AccountSigner.calledByEntry(account));Copy to clipboardErrorCopied
```

签名者与见证范围相关联，该范围限制了签名者的见&#x8BC1;_&#x5728;调用期间如何使用。例如，如果签名者仅用于支付交易费用，则见证范围可以设置&#x4E3A;_&#x4E;one\*。需要了解的有两个签名者类别：`AccountSigner`和`ContractSigner`。对于由账户支持的签名者，请使用`AccountSigner`，它为所有见证范围提供静态构建器方法。

如果签名者是智能合约，请使用`ContractSigner`。这种类型的签名者不需要签名，但在交易执行期间调用合约的`verify(...)`方法。由于合约签名者无法支付交易费用，`ContractSigner`类不&#x4E3A;_&#x4E;on&#x65;_&#x89C1;证范围提供构建器方法。此外，如果合约的`verify(...)`方法有额外参数，`ContractSigner`的构建器方法可以接受合约参数。

**注意：** 交易需要至少一个`AccountSigner`来支付费用。

> 见证是由调用和验证脚本组成的脚本对。两个脚本都是NeoVM指令序列。在账户签名者的情况下，调用脚本包括将签名数据推送到NeoVM堆栈上的指令。在验证脚本中，相应的公钥数据被推送到NeoVM堆栈上，然后是验证调用脚本中提供的签名的指令。
>
> 有关见证和见证范围的更多信息，请参阅[NeoSPCC的Medium文章](https://neospcc.medium.com/thou-shalt-check-their-witnesses-485d2bf8375d)。

见证只能添加到`Transaction`对象，而不能添加到构建器，因为它们通常依赖序列化的交易来生成签名。如果您在交易的签名者中使用`单一签名`账户，`sign()`方法将自动为您生成见证。

以下是如何构建、签名和发送交易的简单示例：

```java
Neow3j neow3j = Neow3j.build(new HttpService("http://127.0.0.1:40332"));
Account account = Account.fromWIF("L24Qst64zASL2aLEKdJtRLnbnTbqpcRNWkWJ3yhDh2CLUtLdwYK2");

byte[] script = new ScriptBuilder().contractCall(NeoToken.SCRIPT_HASH, "symbol", null)
                                   .toArray();

TransactionBuilder builder = new TransactionBuilder(neow3j)
    .script(script)
    .signers(AccountSigner.calledByEntry(account));

// Build the transaction
Transaction tx = builder.sign()
                        .send();Copy to clipboardErrorCopied
```

要手动向交易添加见证，请继续阅读下一节。

### [签名交易](https://neow3j.io/#/neo-n3/dapp_development/transactions?id=signing-transactions)

如[构建交易](https://neow3j.io/#/neo-n3/dapp_development/transactions?id=%2fneo-n3%2fdapp_development%2ftransactions%3fid)部分所示，`TransactionBuilder`提供了一个`sign()`方法，根据提供的签名者添加正确的签名。如果您使用带有私钥的`AccountSigner`，`sign()`方法将自动生成所需的签名/见证。然而，如果账户没有密钥（例如，对于多重签名账户），您需要手动提供见证。

要手动签名，请在构建器上使用`getUnsignedTransaction()`来检索没有见证的交易。然后，通过从序列化的交易字节创建见证来添加必要的见证：

```java
Transaction tx = builder.getUnsignedTransaction();
Account account = Account.fromWIF("L24Qst64zASL2aLEKdJtRLnbnTbqpcRNWkWJ3yhDh2CLUtLdwYK2");
ECKeyPair keyPair = account.getECKeyPair();
byte[] txBytes = tx.getHashData();
Witness witness = Witness.create(txBytes, keyPair);
tx.addWitness(witness);
tx.send();Copy to clipboardErrorCopied
```

对于需要来自多重签名账户的见证的更高级用例，您可以创建一个未签名的`Transaction`并使用`addMultiSigWitness(...)`方法手动附加见证。以下是一个由三个参与者组成的多重签名账户的示例，需要两个人的签名：

```java
Neow3j neow3j = Neow3j.build(new HttpService("http://127.0.0.1:40332"));

ECKeyPair.ECPublicKey pubKey1 = ...;
ECKeyPair.ECPublicKey pubKey2 = ...;
ECKeyPair.ECPublicKey pubKey3 = ...;
int threshold = 2;

// Create the multi-sig account with its verification script
Account multiSigAccount = Account.createMultiSigAccount(Arrays.asList(pubKey1, pubKey2, pubKey3), threshold);

byte[] script = ...; // Define your script

// Create and get an unsigned transaction
Transaction tx = new TransactionBuilder(neow3j)
    .script(script)
    .signers(AccountSigner.calledByEntry(multiSigAccount))
    .getUnsignedTransaction();

// Sign the transaction's hash data externally
byte[] rawSig1 = ...; // Signature from participant 1
byte[] rawSig2 = ...; // Signature from participant 2

Sign.SignatureData sigData1 = Sign.SignatureData.fromByteArray(rawSig1);
Sign.SignatureData sigData2 = Sign.SignatureData.fromByteArray(rawSig2);

// Map signature data to public keys
HashMap<ECKeyPair.ECPublicKey, Sign.SignatureData> signatureMap = new HashMap<>();
signatureMap.put(pubKey1, sigData1);
signatureMap.put(pubKey2, sigData2);

// Add a multi-sig witness to the transaction based on the verification script and signatures
tx.addMultiSigWitness(multiSigAccount.getVerificationScript(), signatureMap);

// Send the transaction
NeoSendRawTransaction response = tx.send();Copy to clipboardErrorCopied
```

如果您需要手动签名交易并需要合约签名者的见证，请使用`Witness.createContractWitness(List<ContractParameter> verifyParams)`为合约签名者创建见证，并使用`addWitness(Witness witness)`将其添加到交易中。

### [跟踪交易](https://neow3j.io/#/neo-n3/dapp_development/transactions?id=tracking-transactions)

要跟踪已发出的交易，请在交易实例上使用`track()`方法。然后，您可以使用`getApplicationLog()`在交易被包含在区块中后立即检索应用程序日志：

```java
tx.track().subscribe(blockIndex -> {
    System.out.println("Transaction included in block " + blockIndex + ".");
    NeoApplicationLog log = tx.getApplicationLog();
    System.out.println("Transaction exited with state " + log.getExecutions().get(0).getState() + ".");
});Copy to clipboardErrorCopied
```

如果您在交易被包含在区块中之前调用`getApplicationLog()`，它将抛出`RpcResponseErrorException`，表示交易未知或找不到。

### [添加额外网络费用](https://neow3j.io/#/neo-n3/dapp_development/transactions?id=adding-additional-network-fees)

交易涉及两种类型的费用：系统费用和网络费用。系统费用涵盖在NeoVM上执行脚本所消耗的资源，而网络费用基于交易大小和签名验证所需的努力。您可以使用`additionalNetworkFee()`添加额外的网络费用以获得优先级：

```java
Transaction tx = new TransactionBuilder(neow3j)
        .script(script)
        .signers(AccountSigner.calledByEntry(acc))
        .additionalNetworkFee(1_000_000L) // Adds 1,000,000 GAS fractions (0.01 GAS)
        .sign();Copy to clipboardErrorCopied
```
