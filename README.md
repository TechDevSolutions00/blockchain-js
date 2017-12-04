# Build your own Blockchain in Javascript

With all the hype about blockchains and cryptocurrencies, I decided to learn a bit more about it. And what better way to learn than to try to build it? Here you will find my attempts to build blockchains from their basic principles, and hopefully in the process it helps someone else to learn something from this as well. Let's get started

## To run

This project is based upon `create-react-app` so a simple `yarn start` will pretty much start everything. You'll only need to start a simple `socket.io` server with `node src/server.js`. You can also run it with `docker-compose` if you prefer.

## Step 1: A chain of blocks?

To understand how blockchains work, let's start with the name. Blockchain? A chain of blocks?

A common misconception is that a blockchain is a single chain of blocks, when in reality, it's more like a tree of blocks. So at any given time, there are multiple chains for blocks by pointing to their respective parent. The pointing happens via hashes which are calculated based upon the data inside the block (i.e. hash of the parent, transaction data and other important stuff)

By pointing via hashes of blocks, we can enforce a specific order of blocks. I.e given a chain of blocks, you can't just take a block in the middle of the chain and change its data, since that would change its hash and subsequently also all blocks that are descendents of the block in question.

```javascript
class Block {
  constructor(blockchain, parentHash, nonce = sha256(new Date().getTime().toString()).toString()) {
    this.blockchain = blockchain;
    this.nonce = nonce;
    this.parentHash = parentHash;
    this.hash = sha256(this.nonce + this.parentHash).toString()
  }
```

If you look at the code, you can see how the P2P aspect of blockchains comes into play. Once a node decided it "mined" a block, it can broadcast that block to all other nodes, you can verify it and then add it to their tree, as well.

![blockbroadcast](https://user-images.githubusercontent.com/571810/32704273-37b7b07a-c7d0-11e7-900c-851031c81ad4.gif)


## Step 2: So what is THE blockchain?

In step 1, we saw that in a chain of blocks, the last block basically validates all data in the chain of its ascendents, as any change in the data up the chain would inevitably change the hash of the last block. That is all great, but what do people mean by THE blockchain?

By definition, THE blockchain is just the longest chain available in the tree. So at one point, a chain can be the lonest one, but then get superseeded by another. Let's visualize the longest chain in the tree.

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

![longestchain](https://user-images.githubusercontent.com/571810/33043509-b40cb21c-ce13-11e7-8fb2-20f3932e85d1.gif)

So given a tree, the longest chain represents our current view of which history of blocks, and thus which representation of data is the one we deem valid.

## Step 3: Not a free-for-all

If real blockchains worked like Step 2, then it would be a chaotic free-for-all where nodes just can abitrarily fork a chain of blocks and add basically infinitely many blocks to it, to make it the longest chain and thus THE blockchain (as you have seen in the above GIF). That would mean that anyone could just change history and effectively mutate past data. How do we avoid that situation?

By making it difficult to add a block with a computational puzzle. Instead of accepting any arbitrary block, part of the concensus rules of a blockchain mandate what blocks are valid and which ones aren't. In this case, we want to make adding blocks resource-intensive. The most common way of doing so, and probably the most admirable piece of the original Bitcoin whitepaper), is to pair this with proof-of-work (POW). POW allows us to ensure that nodes who want to add blocks to the tree to proof that they had to expend considerable effort. Since the SHA256 hash of a block is (hopefully) truly random, we can mandate that the hash ends in a certain number of '0's (in Bitcoin the requirement is for it to start with a certain number of '0's).

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

So the question is why miners would expend all this effort to add a block? Unless it is a fun game for them, usually we are talking about economic incentives now. In order for the blockchain to be secured by miners, the protocol gives miners a mining reward, currently amounting to 12.5 Bitcoin. Other nodes will accept the miners block with the reward to itself as long as it passes the other rules of the protocol we discussed above. Let's talk about the specific mechanic of how a miner gives itself the reward, which requires a concept of ownership and a way to include such ownership in a block.

To understand ownership, you'll need a high-level understanding of public-key encryption which is beyond the scope of this tutorial. ([https://www.youtube.com/watch?v=3QnD2c4Xovk](https://www.youtube.com/watch?v=3QnD2c4Xovk) looks like a good non-technical explanation). All you need to know for this is that the following possible:

1. There is a way to generate two things, a public key and a private key. Keep the private key secret.
2. Per the name, the public key is something that you can publish publically to other parties.
3. In order to proof that you were the one that generated the public key, you can sign a specific message (or arbitrary data) with your private key. Others can take your signature (that is specific to the message), the message as well as your public key and verify that the signature must have indeed come from someone who has control of the private key (as there is no way to satisfactorally sign the message without the private key).
(4. With a public key, you can encrypt a message (data) so that only the owner of the private key can decrypt it)

In short, ownership is the concept of being in control of something, in this case, you "own" the public key, and you can prove such ownership by signing data with your private key. Thus, in order to receive the mining reward, i.e. claim ownership over it, all the miner has to do is to include their public key in the block. That public key is also known as the wallet address in Bitcoin.

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

You should know enough of about blockchains and the way they enable to us record data as a ledger in order to understand how this can allows us to keep track of ownership of "coins". The next few steps will be about actually using the coins and making them useful in transactions.
