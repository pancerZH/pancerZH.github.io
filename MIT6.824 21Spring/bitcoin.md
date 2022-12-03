---
sort: 14
---

# Bitcoin

This blog is based on the [paper]( https://pdos.csail.mit.edu/6.824/papers/bitcoin.pdf) required in MIT 6.824, which introduced a new payment system, and I believe Bitcoin is now known widely.

## Introduction

Bitcoin is based on cryptographic proof instead of trust, without the need for a trusted third party. The system is secure as long as honest nodes collectively control more CPU power than any cooperating group of attacker nodes. This means online payments to be sent directly from one party to another without going through a financial institution.

We define an electronic coin as a chain of digital signatures. Each owner transfers the coin to the next by digitally signing a hash of the previous transaction and the public key of the next owner and adding these to the end of the coin. A payee can verify the signatures to verify the chain of ownership.

To avoid double-spending, the payee need to know that the previous owners did not sign any earlier transactions. The only way to confirm the absence of a transaction is to be aware of all transactions, so that transactions must be publicly announce, and participants should agree on a single history of the order in which they were received.

The solution we propose begins with a timestamp server.  A timestamp server works by taking a hash of a block of items to be timestamped and widely publishing the hash. Each timestamp includes the previous timestamp in its hash, forming a chain, with each additional timestamp reinforcing the ones before it.

To implement a distributed timestamp server on a peer-to-peer basis, a proof-of-work system is needed. The proof-of-work involves scanning for a value that when hashed, such as with SHA-256, the hash begins with a number of zero bits, implemented by incrementing a nonce in the block until a value is found that gives the block's hash the required zero bits. Proof-of-work is essentially one-CPU-one-vote. The majority decision is represented by the longest chain.

## Network

The steps to run the network are as follows:

1. New transactions are broadcast to all nodes.
2. Each node collects new transactions into a block.
3. Each node works on finding a difficult proof-of-work for its block.
4. When a node finds a proof-of-work, it broadcasts the block to all nodes.
5. Nodes accept the block only if all transactions in it are valid and not already spent.
6. Nodes express their acceptance of the block by working on creating the next block in the chain, using the hash of the accepted block as the previous hash. 

Nodes always consider the longest chain to be the correct one and will keep working on extending it. When conflicts happen, they keep both nodes and form two chains, but discard one later according to the length.

New transaction broadcasts do not necessarily need to reach all nodes.  As long as they reach many nodes, they will get into a block before long.

## Incentives

The first transaction in a block is a special transaction that starts a new coin owned by the creator of the block, which is an incentive. Also, the incentive can also be funded with transaction fees.

## Reclaiming Disk Space

Once the latest transaction in a coin is buried under enough blocks, the spent transactions before it can be discarded to save disk space. To facilitate this without breaking the block's hash, transactions are hashed in a Merkle Tree, with only the root included in the block's hash. Old blocks can then be compacted by stubbing off branches of the tree.

## Simplified Payment Verification

A user only needs to keep a copy of the block  headers of the longest proof-of-work chain. He can see that a network node has accepted it, and blocks added after it further confirm the network has accepted it. The verification is reliable as long as honest nodes control the network.

## Combining and Splitting Value

To allow value to be split and combined, transactions contain multiple inputs and outputs. It should be noted that fan-out, where a transaction depends on several transactions, and those transactions depend on many more, is not a problem here. There is never the need to extract a complete standalone copy of a transaction's history.

## Summary

We have proposed a system for electronic transactions without relying on trust.  We started with the usual framework of coins made from digital signatures, which provides strong control of ownership, but is incomplete without a way to prevent double-spending. To solve this, we proposed a peer-to-peer network  using  proof-of-work to  record a public history  of  transactions that quickly becomes computationally impractical for an attacker to change if honest nodes control a majority of CPU power. The network is robust in its unstructured simplicity. Nodes work all at once with little coordination.