---
layout: post
title:  "A Blockchain example in Typescript"
date:   2018-04-22 10:16:01 -0600
categories: Crypto
---

This is a basic example of how the blockchain works. It uses a crypto currency (simpleCoin) as a use case to demonstrate:

1. A Transaction
2. A Block
3. How Blocks are chained
4. Proof of Work
5. Mining Transactions and Rewards

Limitations:

As already stated this is just a basic example. A fully working and operational blockchain would require much more functionality such as:

1. A P2P Network with Transaction Verification
2. Blockchain Data Storage to Disk
3. Public / Private Keys to secure transactions
4. A Wallet

## A Transaction

Each Block on the Blockchain can contain numerous transactions. We therefore need a transaction object which contains all of the data we are going to need to describe a transaction.

**transaction.ts** 

As a minimum we are going to need a From Address, To Address and an Amount:
```typescript
export class Transaction {
    public fromAddress:string;
    public toAddress:string;
    public amount:number;

    constructor(fromAddress:string, toAddress:string, amount:number) {
        this.fromAddress = fromAddress;
        this.toAddress = toAddress;
        this.amount = amount;
    }
}
```

## A Block
A block simply contains some data and this data is hashed using cryptography.
A Block typically contains the following data:

1. Timestamp
2. Transaction data
3. The hash of the previous block

**block.ts**

First off, we import the crypto-js module which will be used to hash blocks using the SHA-256 cryptographic hash function (Bitcoin uses the SHA-256 algo):
```typescript
import SHA256 = require("crypto-js/sha256");
import { Transaction } from "./transaction";
```
We then create our block class which has a constructor. Within this constructor we setup our data and call a function to calculate the hash of the block:
```typescript    
export class Block {
    private timestamp:Date;
    public transactions:Transaction[];
    public previousHash:string;
    public hash:string;
    public nonce:number;

    //////////////////////////////////////
    /// Creating a new block will      ///
    /// construct the required values. ///
    //////////////////////////////////////
    constructor(timestamp:Date, transactions:Transaction[], previousHash?:string) {
        this.timestamp = timestamp;
        this.transactions = transactions;
        this.previousHash = previousHash;
        this.hash = this.calculateHash();
        this.nonce = 0;
    }
```
Finally we create a Function to calculate the SHA-256 hash:
```typescript 
    ///////////////////////////////////
    /// Calculate the hash from the ///
    /// set values.                 ///
    ///////////////////////////////////
    calculateHash() {
        return SHA256(this.previousHash + this.timestamp + JSON.stringify(this.transactions) + this.nonce).toString();
    }
}
```
Thats it, we have a Block object!

## How Blocks are chained
Now for the most important part of all, lets create our chain of blocks A.K.A The Blockchain.

**blockchain.ts**

Import the block and transaction object:

```typescript
import { Block } from './block';
import { Transaction } from "./transaction";
```

We can now create our blockchain class and declare an array of blocks. When we create our blockchain we need to generate the first block. This is known as the 'Genesis Block'.

```typescript
export class BlockChain {
    public chain:Block[];
    private difficulty:number;
    private pendingTransactions:Transaction[];
    private miningReward:number;

    ///////////////////
    /// Constructor ///
    ///////////////////
    constructor() {
        this.chain = [this.createGenesisBlock()];
        this.difficulty = 4;
        this.miningReward = 100;
        this.pendingTransactions = [];
    }

    //////////////////////////////////////////////
    /// Ahoy! Welcome to the very first block. ///
    /// This is known as the Genesis Block     ///
    //////////////////////////////////////////////
    createGenesisBlock() {
        let blockDate:Date = new Date();
        blockDate.getUTCDate();
        let genesisTransaction:Transaction[] = [
            new Transaction(null, "Genesis Block", 0)
        ];
        return new Block(blockDate, genesisTransaction);
    }
```

Now that we have our new Blockchain we can add new blocks.
The next block is simply 'chained' by including the hash of the previous Block. 
We therefore need a way of getting the hash of the lastest Block on the chain:

```typescript
    ///////////////////////////////////////
    /// Get the last block in the chain ///
    ///////////////////////////////////////
    getLatestBlock() {
        return this.chain[this.chain.length - 1];
    }
```
To create a new block we are first going to need some transactions. New trasactions are added to a list of Pending Transactions.

```typescript
    ////////////////////////////////////////////////////////////
    /// Create a new transaction.                            ///
    /// This will be added to the pending transactions array ///
    ////////////////////////////////////////////////////////////
    createTransaction(transaction:Transaction) {
        this.pendingTransactions.push(transaction);
    }

```

### Proof of Work
Proof of Work is a technique that prevents abuse by requiring a certain amount of computing work to create a new block on the chain. This is key to prevent anyone from tampering and breaking the integrity of our Blockchain. Any attempt to do this is no longer worth it if it requires a lot of computing power.

Bitcoin does this by requiring that the hash of a block starts with a specific number of zero's. This is known as the difficulty. However, as the data in our new block is always the same we cant gurantee that the hash of this data will require the right amount of leading zeros. In fact its very unlikely.

We therefore need some data which we are able to change. we can then keep hashing the data until it meets the rules set by the difficulty. The solution to this is to use a nonce, and the process of finding our winning hash is know as mining. 

So lets add a new function to our Block class for mining the new block:

```typescript
    ///////////////////////////////////////////////
    /// Mine Block. This will add all           ///
    /// of the pending transactions once mined. ///
    ///////////////////////////////////////////////
    mineBlock(difficulty) {
        while (this.hash.substring(0, difficulty) !== Array(difficulty + 1).join("0")) {
            this.nonce++;
            this.hash = this.calculateHash();
        }
        console.log("Block mined: " + this.hash + "\n");
    }
```

### Mining Transactions and Rewards
We have our new Blockchain Class where we are able to add new transactions, a Block Class where we can construct a new Block object for a given array of transactions, and a function to mine the next block with a set difficulty.

To bring this all together we need one more function inside our Blockchain class:

```typescript
    ////////////////////////////////////////////
    /// Mine all of the Pending Transactions ///
    ////////////////////////////////////////////
    minePendingTransactions(minerRewardAddress) {
        // Create a New Block
        let blockDate:Date = new Date();
        blockDate.getUTCDate();
        let block = new Block(blockDate, this.pendingTransactions);
        // Set the Hash of the previous block
        block.previousHash = this.getLatestBlock().hash;
        // Mine the Block
        block.mineBlock(this.difficulty);
        // Add the new Block to the Chain
        this.chain.push(block);
        // Reset the pending transaction array and reward the miner
        this.pendingTransactions = [
            new Transaction(null, minerRewardAddress, this.miningReward)
        ];
    }
```
The above code broken down:
1. First of all a new Block object is created with a timestamp and our array of pending transactions
2. The Hash of the previous block is set by retrieving the hash of the latest block
3. The mining process is then started with a set difficulty
4. When the block is found it is added to the Blockchain
5. Finally, as the process of mining new blocks requires a lot of compute power the miner is rewarded for their efforts with cryptocurrency. This is a new Trasaction which is used to create a new pending transactions array


## Testing our Blockchain

We can now test this by creating a new blockchain and a new transaction:

```typescript
import { BlockChain } from './blockchain';
import { Transaction } from "./transaction";
// Construct a new Blockchain called simpleCoin
let simpleCoin = new BlockChain();
// Create a transaction
simpleCoin.createTransaction(new Transaction(null, "1kGd6q2f6Fx4D63nRbxF7o7JTjJna7kc6Bkme1WrCmDMPKoXL7", 10));
// Print out the blockchain
console.log(JSON.stringify(simpleCoin, null, 4) + "\n");
```
This gives the following output:

```json
{
    "chain": [
        {
            "timestamp": "2018-04-22T18:15:58.179Z",
            "transactions": [
                {
                    "fromAddress": null,
                    "toAddress": "Genesis Block",
                    "amount": 0
                }
            ],
            "hash": "cf9a4d97bddc3ec0880415c129ecacc46d6e487b728de077986bed2607ea3fb0",
            "nonce": 0
        }
    ],
    "difficulty": 4,
    "miningReward": 100,
    "pendingTransactions": [
        {
            "fromAddress": null,
            "toAddress": "1kGd6q2f6Fx4D63nRbxF7o7JTjJna7kc6Bkme1WrCmDMPKoXL7",
            "amount": 10
        }
    ]
}
```
As you can see we only have one block in our chain. This is the Genesis Block. The transaction we created is in the pending transactions array. To add this to the Blockchain we need to mine the block:

```typescript
// Mine Pending Transactions
simpleCoin.minePendingTransactions("2kGd6q2f6Fx4D63nRbxF7o7JTjJna7kc6Bkme1WrCmDMPKoXL7");
// Print out the blockchain
console.log(JSON.stringify(simpleCoin, null, 4) + "\n");
```
This gives the following output:

```json
{
    "chain": [
        {
            "timestamp": "2018-04-22T18:15:58.179Z",
            "transactions": [
                {
                    "fromAddress": null,
                    "toAddress": "Genesis Block",
                    "amount": 0
                }
            ],
            "hash": "cf9a4d97bddc3ec0880415c129ecacc46d6e487b728de077986bed2607ea3fb0",
            "nonce": 0
        },
        {
            "timestamp": "2018-04-22T18:15:58.181Z",
            "transactions": [
                {
                    "fromAddress": null,
                    "toAddress": "1kGd6q2f6Fx4D63nRbxF7o7JTjJna7kc6Bkme1WrCmDMPKoXL7",
                    "amount": 10
                }
            ],
            "previousHash": "cf9a4d97bddc3ec0880415c129ecacc46d6e487b728de077986bed2607ea3fb0",
            "hash": "0000b7e9da3161fd17267f8bc484bd501950a14ab519e611d73650ae80d4b51d",
            "nonce": 69089
        }
    ],
    "difficulty": 4,
    "miningReward": 100,
    "pendingTransactions": [
        {
            "fromAddress": null,
            "toAddress": "2kGd6q2f6Fx4D63nRbxF7o7JTjJna7kc6Bkme1WrCmDMPKoXL7",
            "amount": 100
        }
    ]
}
```
We now have a new block. The previous hash set in the block was included in the data to generate this hash. As you can see the new block has a hash with 4 leading zeros which meets the set difficulty level. From the nonce value you can also see that it took 69,089 attempts to find a hash which meets the difficulty criteria. I advise that you reduce this to 1 for your testing!

Thanks for reading and following this working example. I hope that you have found it useful.

For a fully working version you can clone [this](https://github.com/jafuller88/blockchain) project on GitHub.

