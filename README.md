# 用 Javascript 创建我们的区块链

随着区块链和加密货币的大热，我决定深入了解它。还有什么比自己构建一个更好的学习方法呢？接下来你将通过一步又一步的实验来理解区块链是怎么运行的，深入了解其中原理。下面的每一个步骤，你都可以通过 [blockchain-step${N}.nambrot.com](https://blockchain.nambrot.com) 找到对应版本，以及通过 [blockchain.nambrot.com](https://blockchain.nambrot.com) 可以看到最终版本。
免责声明：因为出于教学目的，在某些方面我的实现会不太标准，该区块链会偏离现实。


## Step 1: 区块里的链？

了解区块链的原理之前，我们先从区块里的一条链说起。

区块链常常被误以为是一条单独的链，实际上，区块链更像一棵区块树。因此在任何时间节点中，区块都有很多条链指向它们的父级。而这个指向是通过区块中的数据（例如: 父节点的哈希值，交易数据和其他重要的东西）所计算出来的。

因为区块哈希值的存在，区块链将强制按照指定的规则运行。例如，给定一个区块链，你不能只修改中间某一个节点的数据，因为这将改变这个节点的哈希值，从而导致它关联的所有子区块的哈希值都发生变化。

```javascript
class Block {
  constructor(blockchain, parentHash, nonce = sha256(new Date().getTime().toString()).toString()) {
    this.blockchain = blockchain;
    // 当前节点数据
    this.nonce = nonce;
    this.parentHash = parentHash;
    // 哈希值是根据 当前节点的数据以及父节点的哈希值进行计算的
    this.hash = sha256(this.nonce + this.parentHash).toString()
  }
}
```

从上面的代码，你可以看到区块链的 P2P 是怎样运作的。当一个节点决定“挖掘”一个区块的时候，它就可以将该区块广播给所有其他节点，其他节点可以对其进行验证，然后添加到各自的树中。

![blockbroadcast](https://user-images.githubusercontent.com/571810/32704273-37b7b07a-c7d0-11e7-900c-851031c81ad4.gif)


## Step 2: So what is THE blockchain? 所以什么是区块链呢

[第二步 Demo 的链接](https://blockchain-step2.nambrot.com/)

在第一步我们看到了一条链上的所有区块，最后一个区块会验证整条链上的所有信息，因为上层链路任何修改都会改变最后一个区块的哈希值。OK，那大家都用区块链干什么呢？

根据定义，区块链就是树结构上的最长的链。因此，这条最长链是可以被其他更长的链取代的。让我们来看看这条最长的链是怎么实现的。

```javascript
class Blockchain {
  longestChain() {
    const blocks = values(this.blocks)
    const maxByHeight = maxBy(prop('height'))
    const maxHeightBlock = reduce(maxByHeight, blocks[0], blocks)
    const getParent = (x) => {
      if (x === undefined) {
        return false
      }

      return [x, this.blocks[x.parentHash]]
    }
    return reverse(unfold(getParent, maxHeightBlock))
  }
}
```

![最长链](https://user-images.githubusercontent.com/571810/33043509-b40cb21c-ce13-11e7-8fb2-20f3932e85d1.gif)

在一个树中，这个最长链可以代表当前时间节点这个树的历史，因此我们可以它来确定哪一块的数据是有效的。

## Step 3: Not a free-for-all 不是谁都能改的

[Link to Step 3 Demo](https://blockchain-step3.nambrot.com/)

如果区块链真的像第二步那么运作，就乱套了。任何节点分出来的分支都可以随便通过添加上无限个区块成为最长链，从而成为区块链。这意味着任何人都可以改变历史并有效地改变过去的数据。那我们如何避免这种情况呢？

这个解决方案就是通过添加复杂的数学运算来提高插入节点的难度。区块链共识规定了哪些块有效的，哪些块是无效的。在这种情况下，我们希望通过密集计算提高它的成本。最常见的做法就是像比特币白皮书那样（也是它最令人佩服的部分），将节点和工作量证明（POW）配对。通过 POW 可以确保了每一个区块节点都要证明它们付出了巨大的努力才能添加到树中。由于 区块哈希是真随机（希望是）的，我们强制要求其结尾有一定数量的 '0'（在比特币中，要求它以一定数量的“0”开头）

```javascript
class Block {
  isValid() {
    return this.parentHash === 'root' ||
      (this.hash.substr(-DIFFICULTY) === "0".repeat(DIFFICULTY) &&
      this.hash === sha256(this.nonce + this.parentHash).toString())
  }

  setNonce(nonce) {
    this.nonce = nonce
    this._setHash()
  }

  _setHash() {
    this.hash = sha256(this.nonce + this.parentHash).toString()
  }
}
```
在真实的区块链中，0 的数量是根据最近一次添加的节点计算出来的。挖矿节点不得不尝试许多不同的随机数，才能得到最终以 {DIFFICULTY} 个'0'结尾的哈希值。

![proofofwork](https://user-images.githubusercontent.com/571810/33279514-cdae5fd2-d36c-11e7-97c5-94e61d4e9bce.gif)

工作量证明（Proof-of-work）是区块链可以实现去中心化的奥妙所在，同时也是遭到臭名昭著的 51% [双花难题](https://www.zhihu.com/question/407239242/answer/1344042223)的由来。一旦一个区块进入区块链，攻击者不得不为其后面的所有块重做工作量证明（ proof-of-work）。这里有个双花难题的例子：添加一笔交易到一个区块，但随后通过从父区块中挖出另一条链使其无效。然而，如果没有 51% 的算量，它将总是落后于其他合法的节点。因此，区块链的安全保障全依赖于算力不被集中在单一节点。

## Step 4: What do I mine? 我要怎么挖矿呢？

[Link to Step 4 Demo](https://blockchain-step4.nambrot.com/)

那么问题来了，为什么矿工要花费这么大的精力来添加一个块呢？要么这对于他们来说是一场有趣的游戏，要么我们需要给予他们经济上的奖励。为了让矿工保护区块链，该协议为矿工提供了挖矿奖励，目前是 12.5 比特币。只要它通过我们上面讨论的其他规则，其他节点就会接受这个挖矿节点并允许它奖励自己。让我们来谈谈矿工如何给自己奖励的具体机制，这需要一个所有权的概念和将这种所有权纳入区块的方法。

为了理解所有权，你需要对公钥有更高级别的理解，这超出了本教程的范围。[https://www.youtube.com/watch?v=3QnD2c4Xovk](https://www.youtube.com/watch?v=3QnD2c4Xovk) 这是一个看起来很好的非技术性教程。为此，你需要知道的是以下情况可能发生：

1. 有一种方法可以生成两样东西，一个公钥和一个私钥。保持私钥的秘密。

2. 根据名称，公钥是你可以公开发布给其他各方的东西。

3. 为了证明你是产生公钥的人，你可以用你的私钥签署一个特定的信息（或任意的数据）。其他人可以用你的签名（专门针对该信息）、该信息以及你的公钥，并验证该签名确实来自于控制私钥的人（因为没有私钥，就没有办法令人满意地签署该信息）。

4. 使用公钥，你可以对信息（数据）进行加密，以便只有私钥的拥有者可以解密。

简而言之，所有权是控制某物的概念，在这种情况下，你 "拥有 "公钥，你可以通过用你的私钥签署数据来证明这种所有权。因此，为了获得采矿奖励，即要求对其拥有所有权，矿工所要做的就是在区块中包括他们的公钥。该公钥也被称为比特币的钱包地址（过于简单）。

因此，我们只需在区块中添加一个名为 "coinbaseBeneficiary "的字段，包含矿工的公钥，并将其添加到哈希计算的有效载荷中。

```javascript
class Block {
  isValid() {
    return this.parentHash === 'root' ||
      (this.hash.substr(-DIFFICULTY) === "0".repeat(DIFFICULTY) &&
      this.hash === this._calculateHash())
  }

  createChild(coinbaseBeneficiary) {
    return new Block({
      blockchain: this.blockchain,
      parentHash: this.hash,
      height: this.height + 1,
      coinbaseBeneficiary
    })
  }

  _calculateHash() {
    return sha256(this.nonce + this.parentHash + this.coinbaseBeneficiary).toString()
  }
}
```

Thus, a coin is just ownership over a public key with a private key. By walking down a chain of blocks, you can add up which public keys own how many coins. In reality, these are called Unspent Transaction Outputs (UTXOs), the transaction part will come shortly). To avoid having to traverse chains of blocks everytime we want to find out how many coins an address controls, we "cache" that knowledge with each block into a UTXO pool. Whenever we add a block to a parent, we just take the parents UTXO pool and add the coins of the coinbase beneficiary.

```javascript
class UTXOPool {
  constructor(utxos = {}) {
    this.utxos = utxos
  }

  addUTXO(publicKey, amount) {
    if (this.utxos[publicKey]) {
      this.utxos[publicKey].amount += amount
    } else {
      const utxo = new UTXO(publicKey, amount)
      this.utxos[publicKey] = utxo
    }
  }

  clone() {
    return new UTXOPool(clone(this.utxos))
  }
}

class Blockchain {
  _addBlock(block) {
    if (!block.isValid())
      return
    if (this.containsBlock(block))
      return

    // check that the parent is actually existent and the advertised height is correct
    const parent = this.blocks[block.parentHash];
    if (parent === undefined && parent.height + 1 !== block.height )
      return

    // Add coinbase coin to the pool of the parent
    const newUtxoPool = parent.utxoPool.clone();
    newUtxoPool.addUTXO(block.coinbaseBeneficiary, 12.5)
    block.utxoPool = newUtxoPool;

    this.blocks[block.hash] = block;
    rerender()
  }
}
```

正如你看到，如果我们继续挖更多的区块，我们就能计算出更多的币。

![utxopool](https://user-images.githubusercontent.com/571810/33613173-ec3851de-d9a1-11e7-8a2d-90adce353ac5.gif)

因此你还应该开始了解区块链是如何饰演账本的角色，同时需要了解它是如何变现的。正如你在下面的 GIF 中看到的那样，区块链中的分叉产生了不同的 UTXOPools，从而产生不同的币（这就是确保共识如此重要的原因）。 这就是为什么通常建议等待块的数量达到一定数目，再考虑进行结算交易，否则分叉可能使您的账本状态无效。

![51attack](https://user-images.githubusercontent.com/571810/33613179-f1c861c0-d9a1-11e7-8366-4064cec2e95b.gif)

### Step 5: 你得到一个币了！

[Link to Step 5 Demo](https://blockchain-step5.nambrot.com/)

我们已经非常接近让它成为一个可用的区块链，唯一真正缺乏的是向某人发送硬币的能力。加上这个功能，我们终于开始交易了。它实际上非常简单：

```javascript
class Transaction {
  constructor(inputPublicKey, outputPublicKey, amount) {
    this.inputPublicKey = inputPublicKey
    this.outputPublicKey = outputPublicKey
    this.amount = amount
    this._setHash()
  }

  _setHash() {
    this.hash = this._calculateHash()
  }

  _calculateHash() {
    return sha256(this.inputPublicKey + this.outputPublicKey + this.amount).toString()
  }
}
```

交易只是一个把公钥的所有权转移给另一个节点的声明，因此我们在交易中只需要记录当前公钥和目标公钥以及想要转多少个币的信息。（在真正的比特币中，UTXO 必须被完全消耗，并且可以有多个输入和输出）我们显然需要确保人们只能花费存在的币。我们通过跟踪“余额”的 UTXOPool 来做到这一点。

```javascript
class UTXOPool {
  handleTransaction(transaction) {
    if (!this.isValidTransaction(transaction.inputPublicKey, transaction.amount))
      return
    const inputUTXO = this.utxos[transaction.inputPublicKey];
    inputUTXO.amount -= transaction.amount
    if (inputUTXO.amount === 0)
      delete this.utxos[transaction.inputPublicKey]
    this.addUTXO(transaction.outputPublicKey, transaction.amount)
  }

  isValidTransaction(inputPublicKey, amount) {
    const utxo = this.utxos[inputPublicKey]
    return utxo !== undefined && utxo.amount >= amount && amount > 0
  }
}
```

由于我们将交易的哈希信息放到区块中运算了，其他节点能轻易验证出 1. 这个区块的父节点是否有效 2. 这个交易必须来自已经进行了 POW.
```javascript
class Blockchain {
  _addBlock(block) {
    // ...
    const newUtxoPool = parent.utxoPool.clone();
    block.utxoPool = newUtxoPool;

    // Add coinbase coin to the pool
    block.utxoPool.addUTXO(block.coinbaseBeneficiary, 12.5)

    // Reapply transactions to validate them
    const transactions = block.transactions
    block.transactions = {}
    let containsInvalidTransactions = false;

    Object.values(transactions).forEach(transaction => {
      if (block.isValidTransaction(transaction.inputPublicKey, transaction.amount)) {
        block.addTransaction(transaction.inputPublicKey, transaction.outputPublicKey, transaction.amount)
      } else {
        containsInvalidTransactions = true
      }
    })

    // If we found any invalid transactions, dont add the block
    if (containsInvalidTransactions)
      return
    // ...
  }
}
```

你应该意识到这是一个保持矿工“真诚”的方法。如果矿工包含了无效的教育，其他节点将拒绝这个区块，认为它不是最长链的一部分。我们要确保有效交易的共识。看看下图：

![addingtx](https://user-images.githubusercontent.com/571810/33800311-3746ef66-dd0b-11e7-9427-64c0053a4d5e.gif)

# Step 6: 不一定要计算

[Link to Step 6 Demo](https://blockchain-step6.nambrot.com/)

如果你数学不好（准确来说是你电脑算力不足），是否就以为这你无法将交易添加到区块链中？那也太可怕了吧！相反，作为非挖矿节点，我们也可以订阅广播，来实现类似交易的能力：

```javascript
class Blockchain {
  constructor() {
    // ...
    subscribeTo("TRANSACTION_BROADCAST", ({ transaction, blockchainName }) => {
      if (blockchainName === this.name) {
        this.pendingTransactions[transaction.hash] = new Transaction(
          transaction.inputPublicKey,
          transaction.outputPublicKey,
          transaction.amount
        );
      }
    });
  }
}
```

![txbroadcast](https://user-images.githubusercontent.com/571810/33802204-a2f0ed8c-dd3f-11e7-8fa7-3ba84f01e97d.gif)


# Step 7: 天底下没有免费的午餐

[Link to Step 7 Demo](https://blockchain-step7.nambrot.com/)

除非你赞同 “爱你的邻居” 这种做慈善的说法，人们通常不喜欢免费为其他人做事。那么为什么挖矿节点要为非挖矿节点添加交易呢？是的，他们不会这么做。因此，让我们通过交易费用为他们添加一些激励措施，我们可以指定一些矿点作为交易作者，他们的区块可以在我们的交易中获取小费。

```javascript
class Block {
  addTransaction(inputPublicKey, outputPublicKey, amount, fee) {
    if (!this.isValidTransaction(inputPublicKey, amount, fee))
      return
    const transaction = new Transaction(inputPublicKey, outputPublicKey, amount, fee)
    this.transactions[transaction.hash] = transaction
    this.utxoPool.handleTransaction(transaction, this.coinbaseBeneficiary)
    this._setHash();
  }
}

class UTXOPool {
  handleTransaction(transaction, feeReceiver) {
    if (!this.isValidTransaction(transaction.inputPublicKey, transaction.amount, transaction.fee))
      return
    const inputUTXO = this.utxos[transaction.inputPublicKey];
    inputUTXO.amount -= transaction.amount
    inputUTXO.amount -= transaction.fee
    if (inputUTXO.amount === 0)
      delete this.utxos[transaction.inputPublicKey]
    this.addUTXO(transaction.outputPublicKey, transaction.amount)
    this.addUTXO(feeReceiver, transaction.fee)
  }
}
```

# Step 8: Don't touch my money 不要碰我的钱

[Link to Final Demo](https://blockchain.nambrot.com/)

你已经注意到任何节点都有可能消费可用的 UTXO。如果是这样也太疯狂了吧！让我们通过所有权来修复这个问题。正如我们上面所说的，所有权实际上就是你可以生成私钥的能力。为了确认所有者的意图，我们需要私钥和一个哈希签名。然后当节点接收到交易的区块时，可以验证签名对交易来说确实有效。
```javascript
class Transaction {
  constructor(inputPublicKey, outputPublicKey, amount, fee, signature) {
    this.inputPublicKey = inputPublicKey;
    this.outputPublicKey = outputPublicKey;
    this.amount = amount;
    this.fee = fee;
    this.signature = signature;
    this._setHash();
  }

  hasValidSignature() {
    return (
      this.signature !== undefined &&
      verifySignature(this.hash, this.signature, this.inputPublicKey)
    );
  }
}

class Block {
  isValidTransaction(transaction) {
    return (
      this.utxoPool.isValidTransaction(transaction) &&
      transaction.hasValidSignature()
    );
  }
}
```

如下图所示，这将通过来自私钥的签名将 UTXO 的控制权与公钥的相应所有者联系起来，从而完成我们的区块链

![transactionsinging](https://user-images.githubusercontent.com/571810/33810869-ab96ec7c-ddd8-11e7-81bd-2435de149d83.gif)

就是这样！！！你应该也觉得区块链非常简单。以至于[比特币白皮书](https://bitcoin.org/bitcoin.pdf)也就简单的 8 页。正如演示所见，您真正需要学习的只是一些公钥加密知识和一些比较绕的哈希函数。

# 更多

这就结束了？将来我会添加 merkle trees 和 segwit 相关的功能进来，但就目前来说，我希望可以让你更好的理解区块链，比如比特币是怎么工作的。

## 怎么运行

这个项目是基于 `create-react-app` 构建的，所以一条简单的 `yarn start` 即可完美运行一切。你只需要通过 `node src/server.js` 打开一个 `socket.io` 服务。根据你的喜好，你也可以执行 `docker-compose`。或者你可以在 [blockchain.nambrot.com](https://blockchain.nambrot.com) 上直接运行。
