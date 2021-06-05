# Ethereum

[official website](https://ethereum.org/en/)



## Development environment setup

Before starting setup, we need setup go env.

```bash
export GO111MODULE=on
export GOPROXY=https://goproxy.cn
```



### For Windows user

**1. Download MinGW**

[download](https://netactuate.dl.sourceforge.net/project/mingw-w64/Toolchains%20targetting%20Win64/Personal%20Builds/mingw-builds/4.8.2/threads-posix/seh/x86_64-4.8.2-release-posix-seh-rt_v3-rev2.7z)

setup environment variable: path.



**2. Get source code of go-ethereum**

```bash
go get -u github.com/ethereum/go-ethereum
```



**3. Download Geth**

To download Geth, go to the [Downloads page](https://geth.ethereum.org/downloads) and select the latest stable release matching your platform.



**4. install solidity compiler**

We use directly binary package, [click here](https://github.com/ethereum/solidity/releases)

In this page, select a version that you need.

I suggest to select newest version now,  which is 0.7.0.

You can [click here](https://github.com/ethereum/solidity/releases/download/v0.7.0/solidity-windows.zip) to download directly.

After the download, you need to setup environment which is path.



**5. Get abigen tool**

```bash
cd $GOPATH/src/github.com/ethereum/go-ethereum/
make
make devtools
```



### For Mac user





### For Linux user



## Resource

- Ethereum
  - Doc
    - [面向 Go 开发者的以太坊资源](https://ethereum.org/zh/golang/)
    - [Developer resource](https://ethereum.org/zh/developers/)
    - [Ethereum development with Go](https://goethereumbook.org/zh/)
    - [Go Ethereum](https://geth.ethereum.org/)
- Solidity
  - Doc
    - [en doc](https://solidity.readthedocs.io/en/v0.7.0/introduction-to-smart-contracts.html)
    - [Tutorial](https://github.com/ethereum/go-ethereum/wiki/Contract-Tutorial)
  - IDE
    - [Ethereum Studio](https://studio.ethereum.org/)
    - [Remix](https://remix.ethereum.org/)



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



## Q&A

**Q1. geth attach error: Unable to attach to remote geth: no known transport for URL scheme "c"**

A1.

```bash
geth attach ipc:\\.\pipe\geth.ipc
```

