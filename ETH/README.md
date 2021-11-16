

# Ethereum Note

[official website](https://ethereum.org/en/)



## Note

- Core
  - Consensus
    - Clique
- Smart Contract
  - Precompiled Contract
  - Solidity
    - ERC20
    - ERC721
  - DApp
    - Stack

## Resource

- Ethereum
  - Doc
    - [面向 Go 开发者的以太坊资源](https://ethereum.org/zh/golang/)
    - [Developer resource](https://ethereum.org/zh/developers/)
    - [Ethereum development with Go](https://goethereumbook.org/zh/)
    - [Go Ethereum](https://geth.ethereum.org/)
  - Core
    - Consensus
  - Smart Contract
    - Solidity
      - Doc
        - [en doc](https://solidity.readthedocs.io/en/v0.7.0/introduction-to-smart-contracts.html)
        - [Tutorial](https://github.com/ethereum/go-ethereum/wiki/Contract-Tutorial)
      - IDE
        - [Ethereum Studio](https://studio.ethereum.org/)
        - [Remix](https://remix.ethereum.org/)
    - DApp
      - DEX
      - TheGraph



## Example

### Use Solidity to develop

> solidity based 0.7.0 version

**1. First you should write the code**

```solidity
pragma solidity >=0.6.0 <0.8.0;

/**
 * @title Storage
 * @dev Store & retrieve value in a variable
 */
contract Storage {

    uint256 number;

    /**
     * @dev Store value in variable
     * @param num value to store
     */
    function store(uint256 num) public {
        number = num;
    }

    /**
     * @dev Return value 
     * @return value of 'number'
     */
    function retrieve() public view returns (uint256){
        return number;
    }
}
```



**2. Generate ABI file and bin file from solidity file**

```bash
# solc -o . --abi xxx.sol
solc -o . --abi Storage.sol
# solc -o . --bin xxx.sol
solc -o . --bin Storage.sol
```



**3. Generate Go file**

If you want to interact by Golang with Smart Contract.

```bash
# example is a example package, you need to replace.
# abigen --abi=xxx.abi --pkg=example --out=xxx.go
abigen --abi=Storage.abi --pkg=Storage --out=Storage.go
```

If you want to interact with Smart Contract and deploy smart contract by Golang.

```bash
# abigen --bin=xxx.bin --abi=xxx.abi --pkg=example --out=xxx.go
abigen --bin=Storage.bin --abi=Storage.abi --pkg=example --out=Storage.go
```



## Development Framework

### truffle suit

[Document](https://www.trufflesuite.com/)



## Stack

[The Ethereum Stack](https://ethereum.org/en/developers/docs/#the-ethereum-stack)



## Layer2

[Optimistic Rollup最全指南：这里有你该知道的一切](https://www.8btc.com/article/6590587)

