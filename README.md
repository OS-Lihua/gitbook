---
description: '原作者: RareSkill'
---

# Tornado Cash 的工作原理（开发者逐行解读）

<figure><img src="https://rareskills.io/wp-content/uploads/2024/09/935a00_8fb7ab406b7c430dafcdff2f004b56c7~mv2.png" alt=""><figcaption></figcaption></figure>

### Tornado Cash 简介

Tornado Cash 是一个加密货币智能合约混合器，允许用户通过一个地址存入加密货币，并通过另一个地址提取，而不会在这两个地址之间创建可追踪的链接。

Tornado Cash 可能是最具标志性的零知识智能合约应用，因此我们将以足够低的级别解释其工作原理，以便程序员能够复现该应用。

假设读者了解梅克尔树（Merkle Trees）的工作原理，并且反向计算加密哈希是不可行的。读者也应至少具备中级的 Solidity 编程水平（因为我们将阅读源代码片段）。

Tornado Cash 是一个相当复杂的智能合约，因此如果您对该语言还不熟悉，请先查看我们的 [Solidity 教程](https://rareskills.io/learn-solidity)。

### 关于 Tornado Cash 的快速警告

#### 法律问题

Tornado Cash 目前受到美国政府的制裁，与其互动可能会“污染”您的钱包，并导致其后与集中交易所的交易被标记。

**2024 年更新**：截至 2024 年 11 月 28 日，制裁已被美国上诉法院解除。

#### Tornado Cash 被黑客攻击

在 5 月 27 日，[Tornado Cash 治理智能合约](https://github.com/tornadocash/tornado-governance)（而非我们将要审查的智能合约）遭到黑客攻击，攻击者通过提交恶意提案获得了大多数 ERC20 投票代币的治理权 [ERC20 voting tokens for governance](https://rareskills.io/post/erc20-votes-erc5805-and-erc6372)。他们随后已将控制权归还，保留了一小部分代币。

### 零知识如何工作（对讨厌数学的程序员）

您可以在不深入了解零知识证明算法的情况下理解 Tornado Cash 的工作原理，但您必须了解零知识证明的工作方式。 **与它们的名字和许多流行示例相反，零知识证明证明的是计算的有效性，而不是某一事实的知识。它们并不执行计算。它们接受一个约定的计算、证明该计算已被执行的证明，以及计算的结果，然后确定证明者是否真正运行了该计算并产生了输出。零知识的部分来自于可选特性，即证明可以以不泄露输入信息的方式呈现。**

例如，您可以证明您知道一个数字的素因子，而不揭示它们，使用 [RSA 算法](https://rareskills.io/post/solidity-rsa-signatures-for-aidrops-and-presales-beating-ecdsa-and-merkle-trees-in-gas-efficiency)。RSA 不声称“我知道两个秘密数字”这件事；它证明您将两个隐藏的数字相乘并生成了公共数字。RSA 加密实际上是零知识证明的一个特例。在 RSA 中，您证明您知道两个秘密的质数，它们相乘得到您的公钥。在零知识中，您将这种“魔法乘法”推广到任意算术和布尔运算。一旦我们为基本操作建立了零知识运算，我们就可以为更复杂的事情构建零知识证明，例如证明我们知道哈希函数的前像、梅克尔根，甚至一个完全功能的虚拟机。

另一个要点是：**零知识证明的验证并不执行计算，它只是验证某人确实执行了计算并产生了所声称的输出。**

还有一个有用的推论是：**要生成，但不验证某个计算的零知识证明，您必须实际执行该计算。**

这很有道理，对吧？您怎么能证明您知道哈希的前像，而不实际对前像进行哈希呢？因此，证明者实际上执行计算并创建一些辅助数据，称为证明，以证明他们确实正确地执行了计算。

当您验证 RSA 签名时，您并不会乘以其他人的私钥因素来生成他们的公钥，那样会违背目的。您只需验证签名和消息是否通过单独的算法检查即可。因此，一个计算由以下内容组成：

```
verifier_algorithm(proof, computation, public_output) == true
```

在 RSA 的上下文中，您可以认为公钥是计算的结果，签名和消息是零知识证明。

计算和输出是公开的。我们同意使用某个哈希函数并得到了某个输出。证明“隐藏”了所使用的输入，只证明哈希函数已执行并生成了输出。

<figure><img src="https://rareskills.io/wp-content/uploads/2024/09/935a00_84da717f7427457d8018eed3bfc6a9ee~mv2.png" alt=""><figcaption></figcaption></figure>

仅凭证明，验证者无法计算 `public_output`。验证步骤不进行计算，只验证基于证明的计算是否产生了所声称的结果。

我们不会在本文中教授零知识算法，但只要您能够接受我们可以证明某个计算发生而无需自己进行该计算，我们就可以继续。

### 匿名加密货币转账的工作原理：混合

从根本上说，Tornado Cash 的匿名化策略是混合，类似于其他匿名加密货币如 Monero。多个用户将他们的加密货币存入一个地址，将他们的存款“混合”在一起。然后他们以这样的方式提取，以至于存款人和取款人无法连接在一起。

想象一下，100 个人把一美元放入一堆钱中。然后 100 个不同的人在一天后到达并每人提取一美元。在这个方案中，我们无法知道最初的存款人试图将钱发送给谁。

<figure><img src="https://rareskills.io/wp-content/uploads/2024/09/935a00_9d9def80d2ed462ead091b08af0b9996~mv2.png" alt=""><figcaption></figcaption></figure>

显然有一个问题，如&#x679C;_&#x4EFB;何&#x4EBA;_&#x90FD;可以从这堆现金中取出，那么它会很快被盗。但是如果我们试图留下某些元数据来指明谁被允许提取，那么这将揭示存款人试图将钱发送给谁。

#### 混合永远无法完全私密

当您将以太币发送到 Tornado Cash 时，这完全是公开的。当您从 Tornado Cash 提取时，这也是完全公开的。未公开的是这两个地址之间的关联（假设有足够多的其他存款人和取款人）。

所有人只能知道一个地址是“这个地址从 Tornado Cash 获得了以太币”或“这个其他地址存入了 Tornado Cash”。当一个地址从 Tornado Cash 提取时，人们无法知道加密货币来自哪个存款人。

### 没有零知识的 Tornado Cash：许多哈希前像证明的逻辑 OR

让我们尝试在不考虑隐私的情况下解决这个问题。

存款人创建两个秘密数字，将它们连接起来，并在存入以太币时将其哈希值放在链上（稍后我们将讨论为什么我们生成两个秘密数字而不是一个）。当几个人存款时，有几个公共哈希在一个智能合约中，我们不知道它们的前像。

取款人出现时，取款人揭示某个哈希的前像（这两个秘密数字）并提取他们的存款。

这显然是失败的，因为它显示存款人与取款人之间在链下沟通了秘密数字。

但是，如果取款人能够证明他们知&#x9053;_&#x67D0;个哈&#x5E0C;_&#x7684;前像，而不揭&#x793A;_&#x54EA;个哈&#x5E0C;_&#x4EE5;&#x53CA;_&#x4E0D;揭示哈希的前像_，那么我们就有了一个功能性的加密货币混合器！

这个简单的解决方案是创建一个计算：

```sh
zkproof_preimage_is_valid(proof, hash_{1}) OR 
zkproof_preimage_is_valid(proof, hash_{2}) OR
zkproof_preimage_is_valid(proof, hash_{3}) OR
...
zkproof_preimage_is_valid(proof, hash_{n-1}) OR
zkproof_preimage_is_valid(liproof, hash_{n}) 
```

请记住，验证者并不会实际执行上述计算，因此我们不知道哪个哈希是有效的。验证者（Tornado Cash）只是验证证明者是否进行了上述计算并返回了 true。它只会在证明者知道某个前像的情况下返回 true，而如何返回 true 是无关紧要的。

此时必须注意：**所有存款哈希都是公开的**。当用户存款时，他们提交的哈希是两个秘密数字的哈希，且该哈希是公开的。我们想要隐瞒的是取款人知道哪个哈希的前像。

但这是一个非常大的计算。对于大量存款的巨型循环来说，成本很高。\[1]

我们需要一个数据结构来紧凑地存储大量哈希，幸运的是我们有：梅克尔树（Merkle Trees）。

### 使用梅克尔树存储大量哈希

与其循环遍历所有哈希，我们可以说“我知道某个哈希的前像”，并且“该哈希在梅克尔树中”。这与指向一个很长的哈希数组并说“我知道这些哈希中的一个的前像”效果相同，但效率更高。

梅克尔证明在树的大小上是对数级的，因此相比于我们之前的大循环，它不需要太多额外工作。

当加密货币存款时，用户生成两个秘密数字，将它们连接、哈希并将哈希放入梅克尔树中。

在提取时，取款人生成一个叶子哈希前像，然后通过梅克尔证明证明该叶子哈希在树中。

这当然将存款人与取款人联系在一起，但**如果我们以零知识的方式同时进行梅克尔证明和叶子前像验证，那么链接就被打破了！**

零知识证明让我们证明**我们**生成了有效的梅克尔证明，并且相对于公共梅克尔根以及叶子的前像——而不显示**我们是如何**进行该计算的。

仅提供梅克尔证明和生成根的零知识证明并不安全，取款人还必须证明他们知道该叶子的前像。

梅克尔树的叶子都是公开的。每当有人存款时，他们提供的哈希会被公开存储。由于梅克尔树是完全公开的，任何人都可以计算出任何叶子的梅克尔证明。

因此，证明某个叶子在树中是不够的，以防止通过伪造证明进行盗窃。

取款人还必须证明他们知道该叶子的前像，而不揭示该叶子。请记住，该叶子本身是两个数字的哈希。

您可以在 [存款函数](https://github.com/tornadocash/tornado-core/blob/master/contracts/Tornado.sol#L55) 中看到 `_commitment` 参数是公开的。`_commitment` 变量是添加到树中的叶子，它是两个秘密数字的哈希，存款人并未发布。

```solidity
/**
  @dev 将资金存入合约。调用者必须发送（对于以太币）或批准（对于 ERC20）等于该实例的价值或 `denomination`。
  @param _commitment 票据承诺，即 PedersenHash(nullifier + secret)
**/
function deposit(bytes32 _commitment) external payable nonReentrant {
    require(!commitments[_commitment], "承诺已提交");
​
    uint32 insertedIndex = _insert(_commitment);
    commitments[_commitment] = true;
    _processDeposit();
​
    emit Deposit(_commitment, insertedIndex, block.timestamp);
}
```

实际上，提取的证明包括证明以下计算已被执行：

```solidity
processMerkleProof(merkleProof, hash(concat(secret1, secret2))) == root
```

其中 `processMerkleProof` 接受梅克尔证明和叶子作为参数，而 `hash(concat(secret1, secret2))` 生成该叶子。

在 Tornado Cash 的上下文中，验证者是 Tornado Cash 智能合约，将资金释放给提供有效证明的人。

证明者是取款人，能够证明他们进行了哈希计算，以生成某个叶子。通常，唯一可以提取的人是同一个存款人，因为他们是唯一能够证明知道哈希前像的一方。当然，这个用户必须使用一个不同且完全无关联的地址进行提取！

取款人实际上执行上述计算（梅克尔证明和叶子哈希生成），生成零知识证明，证明他们正确地执行了计算，然后将此证明提交给智能合约。

`merkleProof` 和 `{secret1, secret2}` 在证明中是隐藏的，但通过证明计算，验证者可以验证取款人确实执行了计算以正确生成叶子和梅克尔根。

让我们总结一下：

* 存款人：
  * 生成两个秘密数字，并从它们的连接中创建承诺哈希
  * 提交承诺哈希
  * 向 Tornado Cash 转移加密货币
* Tornado Cash：
  * 在存款阶段：
    * 将承诺哈希添加到梅克尔树中
* 取款人：
  * 为梅克尔根生成有效的梅克尔证明
  * 生成由两个秘密数字构成的承诺哈希
  * 生成上述计算的零知识证明
  * 将证明提交给 Tornado Cash
* Tornado Cash：
  * 在提取阶段：
    * 验证证明与梅克尔根的匹配
    * 将加密货币转移给取款人

### 防止多次提取

上述方案有一个问题：是什么防止我们多次提取？可以推测，我们必须“移除”梅克尔树中的叶子，以便考虑已提取的存款，但那样就会泄露哪个存款是我们的！

Tornado Cash 通过永远不从梅克尔树中移除叶子来处理这个问题。一旦叶子被添加到梅克尔树中，它就会永远保留。

为了防止多次提取，智能合约使用了一种称为“无效化器方案”的机制，这在零知识应用程序和协议中相当常见。

#### 无效化器方案

零知识中的无效化器方案像一种奇特的随机数，为匿名性提供了一层保护。

这就是为什么构成叶子的两个秘密数字，而不是一个的原因。

用于存款哈希的两个数字分别是无效化器和秘密，叶子是 `concat(nullifier, secret)` 的哈希，顺序如此。

在提取时，用户必须提交无效化器的哈希，即 `nullifierHash`，以及证明他们连接无效化器和秘密并对其进行哈希以生成某个叶子。智能合约随后可以通过零知识算法验证发送者确实知道无效化器哈希的前像。

这就是为什么我们需要两个秘密数字。如果我们揭示了这两个数字，那么我们就能够知道取款人正在针对哪个叶子！通过仅揭示其中一个组成数字，无法确定它与哪个叶子相关联。

请记住，零知识证明验证不能计算出无效化器，给出的输入只能验证计算、输出和证明是否一致。这就是用户必须提交一个公共的 `nullifierHash` 和证明他们通过隐藏的无效化器进行了计算的原因。

您可以在 Tornado Cash [提取函数](https://github.com/tornadocash/tornado-core/blob/master/contracts/Tornado.sol#L79) 中看到这一逻辑。

[tornado cash withdraw function source code](https://rareskills.io/wp-content/uploads/2024/09/935a00_036bdf19c4094319aac7a3d0b0b5a2f3~mv2.png)

让我们总结一下，用户必须证明：

* 他们知道叶子的前像
* 无效化器未被重复使用（这是一个简单的 Solidity 映射，而不是零知识验证步骤）
* 他们可以生成无效化器哈希和无效化器的前像

这里有几种可能的结果：

* 用户提供错误的无效化器：检查无效化器和无效化器前像的零知识证明将不通过
* 用户提供错误的秘密：叶子前像的零知识证明将不通过
* 用户提供错误的无效化器哈希（以绕过第 86 行的检查）：无效化器和无效化器前像的零知识证明将不通过

### 增量梅克尔树是一个节省 gas 的梅克尔树

您可能注意到，在上述解释中，我们进行了一个快速的处理。我们如何在链上更新梅克尔树而不耗尽 gas？显然，存款很多，重新计算整个树是不可承受的。

增量梅克尔树通过一些巧妙的优化来解决这些限制。但在讨论优化之前，我们需要理解这些限制。

增量梅克尔树是一个固定深度的梅克尔树，每个叶子开始时都是零值，非零值通过从左到右逐个替换零叶子进行添加。

下面的动画演示了一个深度为 3 的增量梅克尔树，它最多可以容纳 8 个叶子。在 Tornado Cash 的术语中，我们称这些为“承诺”。

<figure><img src="https://rareskills.io/wp-content/uploads/2024/09/935a00_b974bc021cd64fb3860ef2d886ea72b8~mv2.gif" alt=""><figcaption></figcaption></figure>

增量梅克尔树的一些重要特性：

* 梅克尔树的深度固定为 32。这意味着它不能处理超过 `2^32 - 1` 的存款。（这是 Tornado Cash 选择的任意深度，但需要保持不变）。
* 梅克尔树开始时是一个所有叶子
*   的值都是 `hash(bytes32(0))` 的树。

    * 随着存款的进行，最左侧未使用的叶子一个一个被替换为承诺哈希。存款以“从左到右”的方式添加到叶子中。
    * 一旦存款被加入到梅克尔树中，就无法被移除。
    * 每次存款时，都会存储一个新的根。Tornado Cash 称之为“具有历史的梅克尔树”。因此，Tornado Cash 实际上存储的是一组梅克尔根，而不是单个根。显然，随着成员的添加，梅克尔根会发生变化。

    现在我们面临一个问题：如何在链上构建一个深度为 `2^32 - 1` 的梅克尔树而不耗尽 gas？仅计算第一层将需要超过 400 万次迭代，这显然是不可行的。

    但增量梅克尔树的限制使得两个关键不变性得以实现，这允许智能合约进行大规模计算的快捷方式：最新成员右侧的所有内容都是零值梅克尔子树，且左侧的所有内容可以缓存而不必重新计算。

    #### 巧妙的快捷方式 1：最新成员右侧的所有子树都是全零的梅克尔子树

    **全零叶子的梅克尔子树具有可预测的根，可以预先计算。**

    由于所有的叶子最初都是零，构建梅克尔树时会涉及大量计算，其中包括计算所有叶子为零的梅克尔树的根。

    请看下图，注意当所有叶子都是零时，计算的重复工作有多少：

    \[numbered leaves for an incremental merkle tree]

    大多数叶子对将是 `bytes32(0)` 和 `bytes32(0)` 的连接。然后，哈希将与来自姐妹子树的相同哈希连接，依此类推。

    Tornado Cash 预先计算了深度为零的树（只有一个 `bytes32(0)` 叶子的哈希）、两个叶子的零子树的根、四个叶子的零子树的根、八个叶子的零子树的根，依此类推。

    这意味着我们可以根据一个全零的梅克尔树计算梅克尔根。

    **关于“零根”的技术细节**

    Tornado Cash 实际上并不使用 `hash(bytes32(0))` 作为空值，而是使用 `hash("tornado")`。这并不影响算法，因为这只是一个常量。然而，使用零的概念讨论增量梅克尔树更为简便。

    #### 巧妙的快捷方式 2：最新成员左侧的所有子树的根可以缓存而不必重新计算

    考虑我们添加第二个存款的情况。我们已经计算了第一个存款的哈希。该哈希被缓存在 Tornado Cash 称为 `filledSubtrees` 的映射中。`filledSubtree` 就是我们已经计算过的梅克尔树的子树，其中所有叶子都是非零的。动画展示了这一点：

    \[incremental merkle tree animation for filled subtrees]

    重点是，任何时候您需要一个左侧的中间哈希，它已经为您计算好了。

    这种漂亮的特性是由于不能更改或移除叶子的限制。一旦一个子树充满了承诺而不是零值，它就不再需要重新计算。

    现在让我们进行概括。对于任何子树的左侧，都是一个充满承诺的子树（即使只是一个叶子），而右侧的所有内容始终是零叶子或全零叶子的子树。**由于左侧的根是缓存的，而右侧的根是从深度为零的子树的预计算结果中获得的，我们可以有效地生成一个深度为 32 的梅克尔树，仅需 32 次迭代。**&#x8FD9;虽然并不便宜，但可行。相比于 400 万次计算，这要好得多！

    #### 左哈希还是右哈希？

    但在我们“哈希到根”的过程中，我们如何知道将子树的哈希按什么顺序连接？

    例如，我们添加一个新的承诺哈希作为叶子。在我们上面的节点中，我们应该将其连接为 `new_commitment | other_value` 还是 `other_value | new_commitment`？

    这里有一个技巧：每个偶数索引的节点是左子节点，而每个奇数索引的节点是右子节点。这在叶子和每一层的节点上都是如此。下面的图展示了这一模式。

    \[numbered leaves for an incremental merkle tree]

    让我们对这个模式有一个直观的理解。如果第零个叶子被插入，那么我们将始终向右进行哈希（0 ÷ 2 将保持为零，并且零是偶数）。因为零是偶数，所以在我们到达根的过程中，我们将始终向右哈希。

    现在让我们看另一个极端情况。当插入最后一个叶子时，我们必须在到达根的过程中始终向左哈希。每个上层节点都是奇数。这一模式可以推广到树中的每个节点；从叶子的索引开始，重复除以二，直到我们到达根，就能告诉我们是左子节点还是右子节点。下面的动画展示了在奇数节点上我们向左哈希，而在偶数节点上我们向右哈希的过程。

    \[incremental merkle tree algorithm to hash left or right]

    因此，在任何层级，我们知道我们的哈希在与其兄弟节点的关系中该放在哪。

    因此，我们需要两个信息：

    * 我们正在插入的节点的索引
    * 当前索引是偶数还是奇数

    下面是 Tornado Cash 源代码的截图，使用这些信息。for 循环遍历各层，基于刚插入的叶子重新生成梅克尔根。

    \[screenshot of Tornado Cash \_insert() function]

    总之，为了在链上更新梅克尔根，我们

    * 在新索引处添加一个叶子，将其设置为 `currentIndex`
    * 向上移动一个层级并设置

    ```
    currentIndex
    ```

    为

    ```
    currentIndex
    ```

    除以

    <pre><code><strong>二
    </strong></code></pre>

    然后，

    * 如果 `currentIndex` 是奇数，则与 `filledSubtree` 进行左哈希
    * 如果 `currentIndex` 是偶数，则与 `precomputed` 零树进行右哈希。

    很酷的是，这样一个非平凡的算法可以压缩为小的 Solidity 表达。

    #### Tornado Cash 存储最后 30 个根，因为根会随着每个存款而变化

    每当插入一个项目时，梅克尔根必然会变化。如果取款人创建一个梅克尔证明以验证最新根（记住，叶子是公开可用的），但存款交易首先进入并更改了根；那么梅克尔证明将不再有效。zk 验证算法确保梅克尔证明的有效性与根匹配，因此如果根发生变化，证明将无法通过。

    为了给取款人时间进行提取，他们可以参考最近的 30 个根。

    变量 `roots` 是一个从 `uint256` 到 `bytes32` 的映射。当梅克尔证明达到根时（循环完成），它会以上述方式存储在 `roots` 中。`currentRootIndex` 增加到 `ROOT_HISTORY_SIZE`，但一旦达到最大值（30），就会覆盖索引为零的根。因此，它表现得像一个固定大小的队列。以下是 Tornado Cash 的梅克尔树代码中 `_insert` 函数的片段。在重新计算根后，它以上述方式存储。

    \[merkle tree with history lookback]

    #### 增量梅克尔树所需的存储变量

    以下是梅克尔树与历史一起工作的存储变量。

    ```
    mapping(uint256 => bytes32) public filledSubtrees;
    mapping(uint256 => bytes32) public roots;
    uint32 public constant ROOT_HISTORY_SIZE = 30;
    uint32 public currentRootIndex = 0;
    uint32 public nextIndex = 0;
    ```

    * `filledSubtrees` 是我们已经计算出的子树（即所有叶子都是非零的）
    * `roots` 是最近的 30 个根
    * `currentRootIndex` 是一个从 0 到 29 的数字，用于索引 `roots`
    * `nextIndex` 是当前将被填充的叶子

    #### 公共 deposit() 函数如何更新增量梅克尔树

    当用户调用 [deposit](https://github.com/tornadocash/tornado-core/blob/master/contracts/Tornado.sol#L55) 向 Tornado Cash 存款时，会调用 `_insert()` 更新梅克尔树，然后调用 `_processDeposit()`。

    ```
    function deposit(bytes32 _commitment) external payable nonReentrant {
        require(!commitments[_commitment], "承诺已提交");

        uint32 insertedIndex = _insert(_commitment);
        commitments[_commitment] = true;
        _processDeposit();

        emit Deposit(_commitment, insertedIndex, block.timestamp);
    }
    ```

    `_processDeposit()` 只是确保存款金额是准确的（您只能存入 0.1、1 或 10 以太币，具体取决于您交互的 Tornado Cash 实例）。该简单操作的代码如下。

    ```
    function _processDeposit() internal override {
        require(msg.value == denomination, "请发送与交易相同的 `mixDenomination` ETH");
    }
    ```

    ### 超优化的 MiMC 哈希

    为了在链上计算梅克尔根，必须使用哈希算法（显然），但 Tornado Cash 并不使用传统的 `keccak256`；而是使用 MiMC。

    这是因为某些哈希在零知识证明生成中比其他哈希更具计算优势。MiMC 被设计为“适合零知识”，但 `keccak256` 并不适合。“适合零知识”意味着该算法能够自然映射到零知识证明算法表示的计算方式。

    但这造成了一个有趣的难题：MiMC 必须在链上计算，以便在添加新节点时重新计算根，而以太坊并没有为 zk 友好的哈希提供预编译合约。（也许您可以为此编写一个 EIP？）

    因此，Tornado Cash 团队在原始字节码中编写了它。如果您查看 Tornado Cash 的 Etherscan 合约验证，您会看到一个警告：

    \[etherscan screenshot contains unverified code]

    Etherscan 无法验证原始字节码与 Solidity，因为 MiMC 哈希并未以 Solidity 编写。

    Tornado Cash 团队将 MiMC 哈希器部署为一个单独的 [智能合约](https://etherscan.io/address/0x83584f83f26af4edda9cbe8c730bc87c364b28fe)。为了使用 MiMC 哈希，梅克尔树代码需要对该合约进行跨合约调用。如您所见，这些是 \[静态调用]，因为接口将其定义为纯，因此 Etherscan 将其显示为没有交易历史。

    ```
    interface IHasher {
        function MiMCSponge(uint256 in_xL, uint256 in_xR) external pure returns (uint256 xL, uint256 xR);
    }
    ```

    我们知道它是“接口”的原因是基于上述 Tornado Cash 代码中的内容（[github link](https://github.com/tornadocash/tornado-core/blob/master/contracts/MerkleTreeWithHistory.sol#L15)）。

    在 circom 库的一个 [GitHub 议题](https://github.com/iden3/circomlib/issues/32) 中，您可以看到为什么代码没有 Solidity 版本，即使使用汇编块：直接堆栈操作是不可能的。

    （顺便提一下，非常底层的密码算法是 Huff 语言的一个绝佳用例，您可以通过我们的 [Huff 语言难题](https://github.com/RareSkills/huff-puzzles) 了解它）。

    #### 将自己的哈希函数作为原始字节码部署

    circomlib js 存储库包含用于创建原始字节码哈希的 JavaScript 工具。以下是生成 [MiMC](https://github.com/iden3/circomlibjs/blob/main/src/mimcsponge_gencontract.js) 和 [Poseidon 哈希](https://github.com/iden3/circomlibjs/blob/main/src/poseidon_gencontract.js) 的代码。

    ### 从 Tornado Cash 提取

    首先，用户必须使用 [updateTree 脚本](https://github.com/tornadocash/tornado-classic-ui/blob/master/scripts/updateTree.js) 本地重建梅克尔树。该脚本将下载所有相关的 [Solidity 事件](https://rareskills.io/post/ethereum-events) 并重建梅克尔树。然后，用户将生成梅克尔证明和叶子承诺前像的零知识证明。如前所述，Tornado Cash 存储最近 30 个梅克尔根，因此这应该给予用户足够的时间提交其证明。如果用户生成证明后等待太久，他们将不得不重新生成证明。

    Tornado Cash 合约将检查：

    1. 提交的 `nullifierHash` 未被使用过
    2. 根在根历史中（最近的 30 个根）
    3. 零知识证明是否有效：a. 隐藏的哈希前像生成叶子 b. 用户确实知道 `nullifierHash` 的前像 c. 用户使用该叶子创建了梅克尔证明，该证明结果为提议的根 d. 提议的根是最近 30 个根之一（这一点在 Solidity 代码中是公开检查的）

    下面是上述步骤的可视化：

    \[tornado cash withdraw workflow diagram]

    理解了这些背景后，Tornado Cash 的提取函数代码如下：

    ```
    function withdraw(bytes calldata _proof,
        bytes32 _root,
        bytes32 _nullifierHash,
        address payable _recipient,
        address payable _relayer,
        uint256 _fee,
        uint256 _refund
      ) external payable nonReentrant {
        require(_fee <= denomination, "费用超过转账值");
        require(!nullifierHashes[_nullifierHash], "该票据已被消费");
        require(isKnownRoot(_root), "无法找到您的梅克尔根"); // 确保使用的是最近的根
        require(
          verifier.verifyProof(
            _proof,
            [uint256(_root), uint256(_nullifierHash), uint256(_recipient), uint256(_relayer), _fee, _refund]
          ),
          "无效的提取证明"
        );

        nullifierHashes[_nullifierHash] = true;
        _processWithdraw(_recipient, _relayer, _fee, _refund);
        emit Withdrawal(_recipient, _nullifierHash, _relayer, _fee);
    }
    ```

    `_relayer`、`_fee` 和 `_refund` 与支付其他用户的 gas 费用的可选交易中介有关。Tornado Cash 的取款者通常希望使用完全新地址以增强隐私，但完全新地址没有 gas 来支付提取费用。

    为了解决这个问题，取款者可以要求中介进行交易，以换取从 Tornado Cash 提取的一部分。

    中介的提取地址也必须使用上述防止抢先攻击的保护措施，这可以在上面的代码截图中看到。

    ### 存取款的总结

    当调用存款时：

    * 用户提交 `hash(concat(nullifier, secret))` 以及他们存入的加密货币。
    * Tornado Cash 验证存款金额是否是接受的面值。
    * Tornado Cash 将承诺添加到下一个叶子。叶子从不移除。

    当调用提取时：

    * 用户根据 Tornado Cash 发布的事件重建梅克尔树。
    * 用户必须提供无效化器的哈希（公开）、他们正在验证的梅克尔根以及零知识证明，证明他们知道无效化器、秘密和梅克尔证明。
    * Tornado Cash 验证无效化器未被使用过。
    * Tornado Cash 验证提议的根是最近 30 个根之一。
    * Tornado Cash 验证零知识证明。

    用户提交一个没有已知前像的无效叶子并没有任何阻止措施。在这种情况下，提交的加密货币将永远停留在合约中。

    ### Tornado Cash 智能合约架构的快速概述

    构成 Tornado Cash 的智能合约如下：

    \[tornado cash github source code screenshot]

    `Tornado.sol` 是一个抽象合约，实际上由 `ERC20Tornado.sol` 或 `ETHTornado.sol` 实现，具体取决于部署是否旨在混合 ERC20 或特定面值的 ETH。不同的 ETH 面值和 ERC20 代币有各自的 Tornado Cash 实例。

    `MerkleTreeWithHistory.sol` 包含我们之前讨论的 `_insert()` 和 `isKnownRoot()` 功能。

    `Verifier.sol` 是 Circom 电路的 Solidity 转换输出。

    `cTornado.sol` 是治理的 ERC20 代币，不是核心协议的一部分。

    ### Tornado Cash 可以改善 gas 效率的地方

    Tornado Cash 的整体架构非常好，但仍然有几个机会可以进行 \[gas 优化]。

    * Tornado Cash 对预计算的全零叶子梅克尔子树进行线性查找，但可以通过硬编码的二分查找减少操作次数。
    * Tornado Cash 经常使用 uint32 作为堆栈变量；使用 uint256 会更好，以避免 EVM 进行隐式转换。
    * Tornado Cash 有一些常量不必要地修改为公共。公共常量仅在智能合约要读取时才是必要的，它们增加了智能合约的大小。
    * 前缀操作符 (++i) 比后缀操作符（i++）更节省 gas，Tornado Cash 可以在不影响逻辑的情况下更改此操作。
    * `nullifierHashes` 是一个公共映射，但它也被包装在公共视图函数 `isSpent()` 中。这是冗余的。

    ### 结论

    以上就是我们对 Tornado Cash _整个_ 代码库的概述，深入理解每个变量和函数的作用。Tornado Cash 将令人印象深刻的复杂性打包到相对较小的代码库中。我们探讨了几种非平凡的技术，包括：

    * 使用零知识证明来证明对哈希前像的知识，而不泄露前像本身。
    * 如何在链上使用增量梅克尔树。
    * 如何使用从原始字节码编写的自定义哈希函数。
    * 无效化器方案如何工作。
    * 如何匿名提取。
    * 如何防止零知识证明的 DApp 被抢先执行。

    ### RareSkills

    这些材料是我们 \[零知识课程] 的一部分。有关一般智能合约开发，请参见我们的 \[Solidity 启动营]，为专业 Solidity 开发人员提供最严格和最新的智能合约培训项目。

    ### 说明

    \[1] 任何熟悉 ZK 电路的人都知道，我们无法创建任意长度的循环。电路必须在创建时能够容纳非常大的数组。然而，说“遍历那么多哈希是计算昂贵的”仍然是准确的，因为这等同于创建一个不可行的巨大电路。

    _最初发布于 2023 年 6 月 27 日_
