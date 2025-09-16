# Meme治理 App

此项目应该展示如何使用Java和React/NextJS在Neo区块链上构建去中心化应用(dApp)。

该dApp实现了一个治理协议，用户可以：

* 创建添加新meme的提案。
* 创建删除现有meme的提案。
* 在指定时间范围内对现有提案（添加或删除）进行投票。
* 执行在投票中被接受的提案。
* 获取当前保存的memes。

> 注意：任何持有GAS的用户都可以参与Meme治理协议。

### [合约](https://neow3j.io/#/neo-n3/tutorials_and_examples/meme_governance_dapp?id=contracts)

本部分中介绍的智能合约的GitHub仓库可以在这里找到：

> https://github.com/AxLabs/meme-governance-contracts

有两个合约，`MemeContract`和`GovernanceContract`。`GovernanceContract`是`MemeContract`的所有者，因此是唯一有权更改`MemeContract`状态的实体。

`GovernanceContract`内置了投票机制，因此`MemeContract`上的每个更改都必须通过投票。用户可以投赞成票或反对票。要接受一个提案，必须满足以下条件：

* 投票时间范围需要结束（参见[getVotingTime](https://neow3j.io/#/neo-n3/tutorials_and_examples/meme_governance_dapp?id=getvotingtime)）。
* 提案需要最低数量的赞成票（参见[getMinVotesInFavor](https://neow3j.io/#/neo-n3/tutorials_and_examples/meme_governance_dapp?id=getminvotesinfavor)）。
* 提案需要赞成票多于反对票。

当提案通过投票时，它可以被执行（参见[execute](https://neow3j.io/#/neo-n3/tutorials_and_examples/meme_governance_dapp?id=execute)），从而在`MemeContract`上持久化。

#### [GovernanceContract规范](https://neow3j.io/#/neo-n3/tutorials_and_examples/meme_governance_dapp?id=specification-governancecontract)

[**getMemeContract**](https://neow3j.io/#/neo-n3/tutorials_and_examples/meme_governance_dapp?id=getmemecontract)

```javascript
{
  "name": "getMemeContract",
  "safe": true,
  "parameters": [],
  "returntype": "Hash160"
}Copy to clipboardErrorCopied
```

返回底层`MemeContract`的地址。

[**getVotingTime**](https://neow3j.io/#/neo-n3/tutorials_and_examples/meme_governance_dapp?id=getvotingtime)

```javascript
{
  "name": "getVotingTime",
  "safe": true,
  "parameters": [],
  "returntype": "Integer"
}Copy to clipboardErrorCopied
```

返回提案创建后的投票时间范围（区块数量）。

[**getMinVotesInFavor**](https://neow3j.io/#/neo-n3/tutorials_and_examples/meme_governance_dapp?id=getminvotesinfavor)

```javascript
{
  "name": "getMinVotesInFavor",
  "safe": true,
  "parameters": [],
  "returntype": "Integer"
}Copy to clipboardErrorCopied
```

获取提案被接受所需的最少赞成票数。

[**proposeNewMeme**](https://neow3j.io/#/neo-n3/tutorials_and_examples/meme_governance_dapp?id=proposenewmeme)

```javascript
{
  "name": "proposeNewMeme",
  "safe": false,
  "parameters": [
      {
          "name": "memeId", // A unique id/name for this new meme.
          "type": "String"
      },
      {
          "name": "description", // A description of the meme.
          "type": "String"
      },
      {
          "name": "url", // An image url that points directly to the meme file.
          "type": "String"
      },
      {
          "name": "imageHash", // The SHA-256 hash of the meme file found under the provided url.
          "type": "ByteArray"
      }
  ],
  "returntype": "Void"
}Copy to clipboardErrorCopied
```

创建一个添加新meme的提案。

**要求：**

* 不存在具有相同`memeId`的开放（投票进行中）提案。
* 不存在具有相同`memeId`的已关闭**且**已接受的提案。（未被接受的已关闭提案可以被覆盖。）
* 不存在具有相同`memeId`的meme。

[**proposeRemoval**](https://neow3j.io/#/neo-n3/tutorials_and_examples/meme_governance_dapp?id=proposeremoval)

```javascript
{
  "name": "proposeRemoval",
  "safe": false,
  "parameters": [
      {
          "name": "memeId", // The id/name of an existing meme to be removed.
          "type": "String"
      }
  ],
  "returntype": "Void"
}Copy to clipboardErrorCopied
```

创建一个删除现有meme的提案。

**要求：**

* 存在具有相同`memeId`的现有meme。

[**vote**](https://neow3j.io/#/neo-n3/tutorials_and_examples/meme_governance_dapp?id=vote)

```javascript
{
  "name": "vote",
  "safe": false,
  "parameters": [
      {
          "name": "memeId", // The id/name of the meme that this proposal is about.
          "type": "String"
      },
      {
          "name": "voter", // The voter's script hash.
          "type": "Hash160"
      },
      {
          "name": "inFavor", // True to vote in favor and false to vote against a proposal.
          "type": "Boolean"
      }
  ],
  "returntype": "Void"
}Copy to clipboardErrorCopied
```

对提案进行投票（赞成或反对）。

**要求：**

* 存在给定`memeId`的开放提案。
* 投票者必须是签名者（具有called by entry作用域）。

[**execute**](https://neow3j.io/#/neo-n3/tutorials_and_examples/meme_governance_dapp?id=execute)

```javascript
{
  "name": "execute",
  "safe": false,
  "parameters": [
      {
          "name": "memeId", // The meme id/name that the proposal was about.
          "type": "String"
      }
  ],
  "returntype": "Boolean"
}Copy to clipboardErrorCopied
```

执行一个已完成的提案。如果提案是关于创建meme的，则在`MemeContract`上创建该meme及其属性。如果提案是关于删除meme的，则从`MemeContract`中删除该meme。

> **注意**：如果提案未被接受，则将其删除。

**要求：**

* 存在指定`memeId`的已关闭提案。

[**getProposals**](https://neow3j.io/#/neo-n3/tutorials_and_examples/meme_governance_dapp?id=getproposals)

```javascript
{
  "name": "getProposals",
  "safe": true,
  "parameters": [
      {
          "name": "startingIndex", // The first index in the iterator on the contract.
          "type": "Integer"
      }
  ],
  "returntype": "Array"
}Copy to clipboardErrorCopied
```

获取尚未执行的提案列表。返回的数组包含在[下面](https://neow3j.io/#/neo-n3/tutorials_and_examples/meme_governance_dapp?id=meme-and-proposal-structure)详细说明的结构中的提案。

> **注意：** 返回列表的大小是有限的。此方法旨在供RPC使用，由于RPC不适合处理大量数据，部署的合约在返回数组中限制为100个条目。如果合约持有超过100个提案，您可以通过多次调用并为每个RPC将`startingIndex`增加100来获取数据。例如，使用0获取前100个提案。

#### [MemeContract规范](https://neow3j.io/#/neo-n3/tutorials_and_examples/meme_governance_dapp?id=specification-memecontract)

[**getMeme**](https://neow3j.io/#/neo-n3/tutorials_and_examples/meme_governance_dapp?id=getmeme)

```javascript
{
  "name": "getMeme",
  "safe": true,
  "parameters": [
    {
      "name": "memeId", // The id/name of the meme.
      "type": "String"
    }
  ],
  "returntype": "Array"
}Copy to clipboardErrorCopied
```

返回指定meme ID的meme属性。返回的数组在[下面](https://neow3j.io/#/neo-n3/tutorials_and_examples/meme_governance_dapp?id=meme-and-proposal-structure)详细说明。

[**getMemes**](https://neow3j.io/#/neo-n3/tutorials_and_examples/meme_governance_dapp?id=getmemes)

```javascript
{
  "name": "getMemes",
  "safe": true,
  "parameters": [
    {
      "name": "startingIndex", // The first index in the iterator on the contract.
      "type": "Integer"
    }
  ],
  "returntype": "Array"
}Copy to clipboardErrorCopied
```

获取现有memes的列表。返回的数组包含在[下面](https://neow3j.io/#/neo-n3/tutorials_and_examples/meme_governance_dapp?id=meme-and-proposal-structure)详细说明的结构中的memes。

> **注意：** 起始索引的处理方式与函数`getProposals`相同（参见[上面](https://neow3j.io/#/neo-n3/tutorials_and_examples/meme_governance_dapp?id=getproposals)）。

#### [附加说明](https://neow3j.io/#/neo-n3/tutorials_and_examples/meme_governance_dapp?id=additional-notes)

两个合约在部署治理合约时相互关联。两个合约都是分别部署的。首先部署meme合约，然后以meme合约的哈希作为数据参数部署治理合约。部署治理合约时，执行以下步骤：

* 将`MemeContract`上的所有者设置为`GovernanceContract`的地址。
* 在`GovernanceContract`上设置`MemeContract`哈希，以供所有未来调用。

> **注意：** 您可以通过调用`MemeContract.getOwner()`和`GovernanceContract.getMemeContract()`来检查两个合约是否正确初始化。

#### [Meme和提案结构](https://neow3j.io/#/neo-n3/tutorials_and_examples/meme_governance_dapp?id=meme-and-proposal-structure)

meme的属性以以下结构的数组形式传递：

```javascript
{
    "type": "Array",
    "value": [
        {
            "name": "id", // The unique id/name of the meme.
            "type": "String"
        },
        {
            "name": "description", // The description of the meme.
            "type": "String"
        },
        {
            "name": "url", // The url of the meme.
            "type": "String"
        },
        {
            "name": "imageHash", // The sha-256 hash of the image of the above url.
            "type": "ByteArray"
        }
    ]
}Copy to clipboardErrorCopied
```

提案以以下结构的数组形式返回：

```javascript
{
    "type": "Array",
    "value": [
        {
            "name": "meme", // Reference to the meme that this proposal refers to (structure as above).
            "type": "Array"
        },
        {
            "name": "create", // Whether the proposal is about to create or remove the above meme.
            "type": "Boolean"
        },
        {
            "name": "voteInProgress", // True, if this proposal can be voted for, false, if the voting is closed.
            "type": "Boolean"
        },
        {
            "name": "finalizationBlock", // The last block number that accepts any vote.
            "type": "Integer"
        },
        {
            "name": "votesInFavor", // The number of votes in favor of the proposal.
            "type": "Integer"
        },
        {
            "name": "votesAgainst", // The number of votes against the proposal.
            "type": "Integer"
        }
    ]
}Copy to clipboardErrorCopied
```

### [前端](https://neow3j.io/#/neo-n3/tutorials_and_examples/meme_governance_dapp?id=frontend)

本部分中介绍的前端的GitHub仓库可以在这里找到：

> https://github.com/AxLabs/meme-governance-frontend

前端使用[NextJS](https://nextjs.org/)构建，由以下部分组成：

* 登陆页面：关于Meme治理dApp的一般信息
  * 在`/`路径上提供服务，例如`http://localhost:8080/`
* dApp：实际的dApp实现，向用户暴露所有操作
  * 在`/dapp`路径上提供服务，例如`http://localhost:8080/dapp`
  * 🚀 **与**[**NeoLine**](https://neoline.io/en/)**浏览器钱包完全集成**

此项目旨在作为样板使用，以快速和简单的方式引导项目。

#### [开始使用](https://neow3j.io/#/neo-n3/tutorials_and_examples/meme_governance_dapp?id=getting-started)

在本地环境中运行以下命令：

```shell
git clone --depth=1 https://github.com/AxLabs/meme-governance-frontend.git my-project-name
cd my-project-name
npm installCopy to clipboardErrorCopied
```

如果您是开发者，并希望在**带有实时重载的开发模式**下本地运行：

```shell
npm run devCopy to clipboardErrorCopied
```

如果您希望创建**优化的生产构建**，则执行：

```shell
npm run build-prodCopy to clipboardErrorCopied
```

要提供优化的生产构建服务，请运行：

```shell
npm run start
```
