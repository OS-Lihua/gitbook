# 钱包和账户

账户和钱包是区块链中的重要概念。Neo账户的基本形式由一个EC密钥对组成。从这个EC密钥对可以导出`脚本哈希`和 `地址`，用于标识账户。

除了这种简单形式外，还有多重签名（multi-sig）账户，由多个参与账户的公钥组成。Neow3j使用相同的`Account`类来表示单一和多重签名账户。

钱包是一个或多个账户的集合。

### [钱包](https://neow3j.io/#/neo-n3/dapp_development/wallets_and_accounts?id=wallets)

创建新钱包的最简单方法是使用静态创建方法之一。

```java
Wallet w = Wallet.create();Copy to clipboardErrorCopied
```

这会创建一个带有新账户（带有新密钥对）的钱包。此方法还有其他版本，允许您立即加密新私钥或在创建后直接将钱包写入文件。

[NEP-6](https://github.com/neo-project/proposals/blob/master/nep-6.mediawiki)是Neo的钱包标准。如果您有从其他钱包软件导出的NEP-6钱包文件，可以使用`fromNEP6Wallet(...)`从 NEP-6文件中读取钱包信息。

```java
String absoluteFileName = "/path/to/your/NEP6.wallet";
Wallet w = Wallet.fromNEP6Wallet(absoluteFileName)
        .name("NewName");Copy to clipboardErrorCopied
```

> **注意：** 从 NEP-6钱包文件读取钱包时，包含的账户的私钥将被加密，直到您在钱包上调用`decryptAllAccounts(String password)`。加密的账户无法用于签名交易。

如果您已经有`Account`对象，可以使用静态方法`withAccounts(...)`创建钱包。此外，您可以手动设置钱包的名称、版本和[scrypt](https://en.wikipedia.org/wiki/Scrypt)参数。如果什么都不设置，将使用默认值。

```java
Wallet w = Wallet.withAccounts(Account.create())
        .name("MyWallet")
        .version("1.0");Copy to clipboardErrorCopied
```

> **注意：** 如果您想提取钱包实例，例如作为数据传输对象（DTO），请确保创建`NEP6Wallet`实例并使用该实例。为此，您需要先加密钱包，然后调用`toNEP6Wallet`。这将自动为钱包中的所有账户初始化`NEP6Account`实例。

### [账户](https://neow3j.io/#/neo-n3/dapp_development/wallets_and_accounts?id=accounts)

neow3j中的账户可以在有或没有EC密钥材料的情况下创建。如果账户的私钥可用，该账户可用于签名任意数据，包括交易或其他原始数据。

以下方法创建具有可用私钥的账户：

* `create()`
* `new Account(ECKeyPair ecKeyPair)`
* `fromNEP6Account(NEP6Account nep6Account)` - Requires decryption of the private key with `decryptPrivateKey(...)`.
* `fromWIF(String wif)`

也可以在没有私钥的情况下创建账户。例如，这适用于只知道公钥或其派生验证脚本的多重签名账户。您可以使用以下静态方法创建没有私钥的账户：

* `fromAddress(String address)`
* `fromScriptHash(Hash160 scriptHash)`
* `fromPublicKey(ECPublicKey publicKey)`
* `fromVerificationScript(VerificationScript script)`

> **注意：** 如果您正在使用Neo地址或`仅观察`账户（即没有私钥），可以使用`Hash160`类。但是，如果您正在使用钱包并使用NEP-6钱包文件，可以使用`Account`类。

账户可以持有一个标签，默认设置为其地址。如果需要，您可以使用`label(String)`方法将其更改为自定义标签。

```java
Account a = Account.fromWIF("L3kCZj6QbFPwbsVhxnB8nUERDy4mhCSrWJew4u5Qh5QmGMfnCTda")
        .label("MyAccount");Copy to clipboardErrorCopied
```

> **注意：** 当您想要提取单个账户实例，例如作为数据传输对象（DTO）时，请确保创建`NEP6Account`实例并使用该实例。为此，您需要先加密账户，然后调用`toNEP6Account`。当从钱包实例创建`NEP6Wallet`时，会为每个账户自动执行此转换。

### [多重签名账户](https://neow3j.io/#/neo-n3/dapp_development/wallets_and_accounts?id=multi-sig-accounts)

多重签名账户可以使用以下方法创建：

* `fromVerificationScript(VerificationScript script)`
* `fromNEP6Account(NEP6Account nep6Account)`
* `createMultiSigAccount(List<ECPublicKey> publicKeys, int signingThreshold)`
* `createMultiSigAccount(String address, int signingThreshold, int numberOfParticipants)`

前两种方法可用于单一和多重签名账户。NEP6Account脚本中可用的验证脚本包含关于多重签名账户的所有所需信息。这包括签名阈值、参与者数量和涉及的公钥。

在后两种方法中，没有可用的验证脚本。因此，必须明确指定签名阈值和/或参与者数量中的一个或两个，以便多重签名账户可以在交易中用作签名者。`TransactionBuilder`需要签名阈值和参与者数量来确定使用该账户签名必须支付的网络费用。

多重签名账户不持有EC密钥对。这将违背多重签名账户的目的，因为它们的密钥材料应该分散在多个实体中。

在以下示例中，从三个公钥创建了一个新的多重签名账户。其签名阈值为2，即，为了使由该账户签名的交易成功，三个参与者中至少两个必须签名。

```java
List<ECPublicKey> publicKeys = Arrays.asList(
        ECKeyPair.createEcKeyPair().getPublicKey(),
        ECKeyPair.createEcKeyPair().getPublicKey(),
        ECKeyPair.createEcKeyPair().getPublicKey()
);

Account a2 = Account.createMultiSigAccount(publicKeys, 2)
        .label("MyMultiSigAccount");Copy to clipboardErrorCopied
```

### [账户余额](https://neow3j.io/#/neo-n3/dapp_development/wallets_and_accounts?id=account-balances)

要检查账户的NEP-17余额，请使用以下方法。

```java
Neow3j neow3j = Neow3j.build(new HttpService("http://localhost:40332"));
Map<Hash160, BigInteger> nep17Balances = a.getNep17Balances(neow3j);Copy to clipboardErrorCopied
```

这返回一个包含账户所有NEP-17代币余额的映射。
