# Consensus



## Attacks

### long-range attacks

In a naively implemented proof of stake, suppose that there is an attacker with 1% of all coins at or shortly after the genesis block. That attacker then starts their own chain, and starts mining it. Although the attacker will find themselves selected for producing a block only 1% of the time, they can easily produce 100 times as many blocks, and simply create a longer blockchain in that way. 

https://blog.ethereum.org/2014/05/15/long-range-attacks-the-serious-problem-with-adaptive-proof-of-work/

### nothing at stake attacks

You don't lose anything from behaving badly, you lose nothing by signing each and every fork, your incentive is to sign everywhere because it doesn't cost you anything.

So as it doesn't cost you anything, it's a good strategy to work on each and every chain should a fork occur and double spend a digital good.

https://ethereum.stackexchange.com/questions/2402/what-exactly-is-the-nothing-at-stake-problem



### Docs

- [Exploring the Attack Surface of Blockchain: A Systematic Overview](https://arxiv.org/abs/1904.03487)

