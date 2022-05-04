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

By making it difficult to add a block with a computational puzzle. Instead of accepting any arbitrary block, part of the concensus rules of a blockchain mandate what blocks are valid and which ones aren't. In this case, we want to make adding blocks resource-intensive. The most common way of doing so, and probably the most admirable piece of the original Bitcoin whitepaper), is to pair this with proof-of-work (POW). POW allows us to ensure that nodes who want to add blocks to the tree to proof that they had to expend considerable effort. Since the SHA256 hash of a block is (hopefully) truly random, we can mandate that the hash ends in a certain number of '0's (in Bitcoin the requirement is for it to start with a certain number of '0's).
这个解决方案就是通过添加复杂的数学运算来提高插入节点的难度。区块链共识规定了哪些块有效的，哪些块是无效的。在这种情况下，我们希望通过密集计算提高它的成本。最常见的做法就是像比特币白皮书那样（也是它最令人佩服的部分），将节点和工作量证明（POW）配对。通过 POW 可以确保了每一个区块节点都要证明它们付出了巨大的努力才能添加到树中。我们可以要求散列以一定数量的“0”结尾，来实现一个块的 SHA256 散列（希望）是真正随机的。（在比特币中，要求它以一定数量的“0”开头）

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

The actual number of '0's in real world blockchains is calculated dynamically based upon the speed at which recent blocks have been added. A miner would then have to try many different nonces to hope that evetually it yields in a hash that ends with {DIFFICULTY} '0's.

![proofofwork](https://user-images.githubusercontent.com/571810/33279514-cdae5fd2-d36c-11e7-97c5-94e61d4e9bce.gif)

Proof-of-work is what "secures" the blockchain, makes it decentralized and the reason where the infamous 51% double-spend attack comes from. Once a block makes it onto the blockchain (the longest chain of blocks), an attacker would have to redo the proof-of-work for that block and all blocks following it. The example would be a double-spend: Add a transaction to a block, but then make it "invalid" by mining an alternate chain from the parent. However, without having 51% of the computation power of the network, it would be always lagging behind all the other nodes in the network trying to add blocks from the currently legitimate blockchain. Thus the security of the blockchain relies on computational power to not be centralized within single parties.

## Step 4: What do I mine?

[Link to Step 4 Demo](https://blockchain-step4.nambrot.com/)

So the question is why miners would expend all this effort to add a block? Unless it is a fun game for them, usually we are talking about economic incentives now. In order for the blockchain to be secured by miners, the protocol gives miners a mining reward, currently amounting to 12.5 Bitcoin. Other nodes will accept the miners block with the reward to itself as long as it passes the other rules of the protocol we discussed above. Let's talk about the specific mechanic of how a miner gives itself the reward, which requires a concept of ownership and a way to include such ownership in a block.

To understand ownership, you'll need a high-level understanding of public-key encryption which is beyond the scope of this tutorial. ([https://www.youtube.com/watch?v=3QnD2c4Xovk](https://www.youtube.com/watch?v=3QnD2c4Xovk) looks like a good non-technical explanation). All you need to know for this is that the following possible:

1. There is a way to generate two things, a public key and a private key. Keep the private key secret.
2. Per the name, the public key is something that you can publish publically to other parties.
3. In order to proof that you were the one that generated the public key, you can sign a specific message (or arbitrary data) with your private key. Others can take your signature (that is specific to the message), the message as well as your public key and verify that the signature must have indeed come from someone who has control of the private key (as there is no way to satisfactorally sign the message without the private key).
4. (With a public key, you can encrypt a message (data) so that only the owner of the private key can decrypt it)

In short, ownership is the concept of being in control of something, in this case, you "own" the public key, and you can prove such ownership by signing data with your private key. Thus, in order to receive the mining reward, i.e. claim ownership over it, all the miner has to do is to include their public key in the block. That public key is also known as the wallet address in Bitcoin (oversimplication).

So let's just simply add a field in the block called `coinbaseBeneficiary` that contains the public key of the miner and add it to the payload for the hash calculation:

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

As you can see, if we keep mining more blocks, we keep accumulating coins.

![utxopool](https://user-images.githubusercontent.com/571810/33613173-ec3851de-d9a1-11e7-8a2d-90adce353ac5.gif)

This way, you should also start the see of how the blockchain acts as a ledger, but also how it is inherently volatile. A fork in the blockchain will yield different UTXOPools and thus different assertions over coins (which is why securing consensus is so important) as you can see in the GIF below. That's why it is generally recommended to wait a certain number of blocks until you can consider a transaction to be settled, otherwise a fork can invalidate what you assumed to be the state of the ledger.

![51attack](https://user-images.githubusercontent.com/571810/33613179-f1c861c0-d9a1-11e7-8366-4064cec2e95b.gif)

### Step 5: You get a coin! You get a coin! 你得到一个币了！

[Link to Step 5 Demo](https://blockchain-step5.nambrot.com/)

We are getting very close to have this be a usable blockchain. The only thing we are really lacking is the ability to send someone a coin. We are finally getting to transactions. And it's actually pretty simple:


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

A transaction is simply a declaration to move ownership from one public key to the other, thus all we have to record in a transaction is the public key that has any coins, the target public key and the amount of coins we want to transfer. (In real Bitcoin, UTXOs have to be fully consumed and can have multiple inputs and outputs.) We obviously need to make sure that people only spend coins that exist. We do that with the UTXOPool which keeps track of "balances".

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

Since we add the hashes of the transactions to the computation of the block hash, other nodes in the network can easily verify that the transactions are 1. valid given a block's ascendants and 2. came from a node that had to do the "proof-of-work".

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

You should recognize an additional method of keeping miners "honest". If miners include transactions that are invalid, peer nodes will reject the block and thus not consider it to be part of the longest chain. Thus we maintain concensus over valid transactions. Here is everything in GIF-form:

![addingtx](https://user-images.githubusercontent.com/571810/33800311-3746ef66-dd0b-11e7-9427-64c0053a4d5e.gif)

# Step 6: 不一定要计算

[Link to Step 6 Demo](https://blockchain-step6.nambrot.com/)

What if you (or more specifically your computers) are bad at math, does that mean you don't get to have your transactions added to the blockchain? That would be terrible! Instead, as a non-mining node, let's add the ability to broadcast a transaction, that a different mining node can then add to their block:
如果你数学不好（准确来说是你电脑算力不足），是否就以为这你无法将交易添加到区块链中？那也太可怕了吧！相反，作为非挖掘节点，我们也可以订阅广播，来实现类似挖掘节点的能力：

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


# Step 7: No free lunches 天下没有免费的午餐

[Link to Step 7 Demo](https://blockchain-step7.nambrot.com/)

Unless you subscribe to the charitable interpretations of "love thy neighbor", people generally don't like to do things free for others. So why would a mining node add a transaction for a non-mining node? You are right, they wouldn't. So let's add some incentives for them with a transaction fee that we can specify as a transaction author to increase the chances of some mining node adding our transaction to their block.

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

If you paid attention, you have noticed that it was possible for any node to spend any UTXO available. If that were the case in reality, it would be madness! Let's fix this by completing the ownership story. As we said above, ownership is really just the ability to prove that you have generated the private key. So to know if a transaction was truly the intention of the owner, all we have to do is request a signature of the transaction hash with the private key. Nodes can then verify that the signature is indeed valid for the transaction when they validate transaction of blocks they are receiving.
如果你注意了，你已经注意到任何节点

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

As seen in the GIF below, this will complete our blockchain by tieing control of UTXOs to the corresponding owner of the public key via signatures from their private key.

![transactionsinging](https://user-images.githubusercontent.com/571810/33810869-ab96ec7c-ddd8-11e7-81bd-2435de149d83.gif)

AND THATS IT!!! As you'll hopefully agree with me, blockchains are actually quite simple. So simple that [Bitcoin's original whitepaper](https://bitcoin.org/bitcoin.pdf) is only 8 pages. As we learned in this walkthrough, all you really need to know is some public key encryption knowledge and the fact that some hash functions are very hard to reverse.

# Wait, there is more

JK, there isn't as of yet. I might add merkle trees and segwit in the future, but for now, I hope this gives you a good overview of how blockchains such as Bitcoin work.

这就结束了？将来我会添加 merkle trees 和 segwit 相关的功能进来，但就目前来说，我希望可以让你更好的理解区块链，比如比特币是怎么工作的。


## 怎么运行

这个项目是基于 `create-react-app` 构建的，所以一条简单的 `yarn start` 即可完美运行一切。你只需要通过 `node src/server.js` 打开一个 `socket.io` 服务。根据你的喜好，你也可以执行 `docker-compose`。或者你可以在 [blockchain.nambrot.com](https://blockchain.nambrot.com) 上直接运行。
