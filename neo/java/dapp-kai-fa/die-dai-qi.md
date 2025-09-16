---
layout:
  width: default
  title:
    visible: true
  description:
    visible: true
  tableOfContents:
    visible: false
  outline:
    visible: true
  pagination:
    visible: true
  metadata:
    visible: true
---

# 迭代器

**简要说明**：您可以使用`SmartContract.callFunctionReturningIterator(...)`来调用返回迭代器的方法，并使用结果`Iterator`对象及其实用方法遍历迭代器。

智能合约方法可以返回迭代器来简化读取具有许多条目的存储。使用RPC（例如`invokefunction`）调用此类方法时，您连接的节点会打开一个会话并返回`sessionId`和`iteratorId`。然后您可以使用这些值与RPC `traverseIterator`一起使用，指定每次调用要迭代通过的条目数`n`。最初，此RPC返回迭代器的前`n`个条目。如果有更多条目，对同一RPC的后续调用将返回迭代器的下`n`个条目，以此类推。

**注意1：**&#x60A8;无法遍历作为写交易结果返回的迭代器。

**注意2：**&#x67D0;些节点（尤其是公共节点）由于DoS担心已禁用会话。如果是这种情况，Neow3j提供了一个自定义实用方法来在NeoVM级别解包迭代器并使用`SmartContract.callFunctionAndUnwrapIterator(...)`方法返回条目数组。
