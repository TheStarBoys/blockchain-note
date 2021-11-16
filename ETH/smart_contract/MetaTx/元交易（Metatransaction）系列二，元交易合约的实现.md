# 引言
[上文](https://blog.csdn.net/shengzang1998/article/details/121251057)中提到，普通的 ETH 交易并不能够做到让用户无需 gas 费，需要交易中嵌套一个交易，即元交易，来实现免 gas 费。

本文将分析开源库 [OpenZeppelin/openzeppelin-contracts](https://github.com/OpenZeppelin/openzeppelin-contracts) 中的元交易合约的实现，让你能够快速入门元交易实现细节，从而能够自己对后续更多的相关技术深入探索。

# 前置知识概述
元交易会涉及到 ECDSA 与 EIP712 等知识，如果你是熟手，可以跳过此节内容，直接浏览具体实现分析部分。

## Hash
也称哈希、散列、数字摘要。通过哈希函数，可以将长短不一的信息转化为一段长度任意但可预测的（确定性的）结果。这是一类神奇的函数，可以将一大堆信息转变成一串短的，可作为摘要的数据 “指纹”。对于一个给定的输入而言，生成的 “指纹” 始终一致。如果你的原始数据中有任何细微的改动，生成的哈希值将大不相同。以太坊中采用的是 `Keccak-256` 算法。

## ECDSA
在密码学中，ECDSA（Elliptic Curve Digital Signature Algorithm，椭圆曲线数字签名算法）是使用椭圆曲线密码学的数字签名算法（DSA）的一个变种。

主要用于对数据（比如一个文件）创建数字签名，以便于你在不破坏它的安全性的前提下对它的真实性进行验证。可以将它想象成一个实际的签名，你可以识别部分人的签名，但是你无法在别人不知道的情况下伪造它。

你不应该将ECDSA与用来对数据进行加密的AES（高级加密标准）相混淆。ECDSA不会对数据进行加密、或阻止别人看到或访问你的数据，它可以防止的是确保数据没有被篡改。

如图所示，在以太坊中，ECDSA 用于对原始数据的 hash 值进行签名及恢复。
![ECDSA 在 ETH 中的应用](https://img-blog.csdnimg.cn/ab62f999126d45178ca2c3ee90702332.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5LiA5YmR5a-S6Zyc,size_19,color_FFFFFF,t_70,g_se,x_16)
将原始数据通过 hash 函数得到它的 hash 值后，用户 A 用自己的私钥对该 hash 值进行签名，得到 Signature（签名）。有了该签名与 hash 值，任何人都能够从中恢复出签名人的钱包地址，在这里用户 B 则恢复得到了用户 A 的钱包地址。

## EIP712
Ethereum Improvement Proposals (EIPs)，你可以在这里查看所有的 [EIPs](https://eips.ethereum.org/)。[EIP712](https://eips.ethereum.org/EIPS/eip-712) （Ethereum typed structured data hashing and signing）以太坊类型的结构化数据哈希与签名。

如果我们只关心字节字符串的话，签名数据是一个已经解决了的问题。但不幸的是，在现实世界中，我们关心的是复杂而有意义的信息，对结构化数据进行哈希是非常重要的，错误会导致系统安全属性的丢失。

此 EIP 旨在提高链上使用的链下消息签名的可用性。我们看到越来越多的人采用链下消息签名，因为它节省了 gas 费，减少了区块链上的交易数量。当前签名消息是一个不透明的十六进制字符串，显示给用户，关于组成消息的项目的上下文很少。

![Sign the message](https://img-blog.csdnimg.cn/8f29018cd9264a59ae38f38b9acb5b9e.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5LiA5YmR5a-S6Zyc,size_20,color_FFFFFF,t_70,g_se,x_16)
EIP712 概述了一个编码数据及其结构的方案，该方案允许在签名时将数据显示给用户进行验证。下面是一个用户在签署 EIP712 消息时显示的示例。

![Sign the typed structured message](https://img-blog.csdnimg.cn/468b7560f44642688a75b1d8750e57c2.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5LiA5YmR5a-S6Zyc,size_20,color_FFFFFF,t_70,g_se,x_16)
# 元交易合约的实现
此分析针对 [openzeppelin-contracts v4.3.2 版本](https://github.com/OpenZeppelin/openzeppelin-contracts/tree/v4.3.2)。

```solidity
contract MinimalForwarder is EIP712 {
    using ECDSA for bytes32;

    struct ForwardRequest {
        address from;
        address to;
        uint256 value;
        uint256 gas;
        uint256 nonce;
        bytes data;
    }

	constructor() EIP712("MinimalForwarder", "0.0.1") {}
}
```

ECDSA 是 openzeppelin 实现的一个 solidity 库，它实现了从 hash 值中恢复钱包地址的方法，将它应用在 bytes32 上，就可以直接在 bytes32 上调用 `recover` 方法。`recover` 函数签名：`function recover(bytes32 hash, bytes memory signature) internal pure returns (address)` 。

ForwardRequest 结构体定义了一个交易中用于签名的基本组成成分。与以太坊交易不同的是没有 `gasPrice`，因为智能合约的执行只关心 gas 的消耗。ForwardRequest 中 的 `nonce` 概念与以太坊类似，都是为了避免双花攻击，但这里的 `nonce` 仅由智能合约维护，跟普通的以太坊交易中的 `nonce` 无关。

构造函数中直接使用 EIP712 的构造函数进行初始化，EIP712 的构造函数签名为：`constructor(string memory name, string memory version)` ，其中 name 是合约名称，version 是合约版本，这将作为 EIP712 签名验证的一部分，它在部署时，将自动获取合约的地址、chainId 等信息。意味着，即便有相同的 ForwardRequest 结构体数据，但合约地址或区块链网络不同，也会导致签名无效。

```solidity
mapping(address => uint256) private _nonces;

function getNonce(address from) public view returns (uint256) {
	return _nonces[from];
}
```

为了避免双花攻击，在智能合约中维护 `nonce` 是必要的。

```solidity
bytes32 private constant _TYPEHASH =
	keccak256("ForwardRequest(address from,address to,uint256 value,uint256 gas,uint256 nonce,bytes data)");

function verify(ForwardRequest calldata req, bytes calldata signature) public view returns (bool) {
	address signer = _hashTypedDataV4(
            keccak256(abi.encode(_TYPEHASH, req.from, req.to, req.value, req.gas, req.nonce, keccak256(req.data)))
    ).recover(signature);
	return _nonces[req.from] == req.nonce && signer == req.from;
}
```

看到 `verify` 函数，我们知道，要将钱包地址恢复，至少需要经过 ECDSA 的签名以及用于签名的原始数据，而此处，ECDSA 签名的原始数据就是经过 `abi` 编码的 `keccak256(abi.encode(_TYPEHASH, req.from, req.to, req.value, req.gas, req.nonce, keccak256(req.data)))`  ForwardRequest 结构体数据的哈希值。再通过调用 ECDSA 库中的 recover 函数，传入签名，就能够恢复得到签名者的钱包地址。

通过 ` _nonces[req.from] == req.nonce` 来确保交易的调用是顺序的，且不会遭受双花攻击。` signer == req.from` 避免签名者与实际元交易发送者不匹配。

接下来看，如何执行元交易。

```solidity
function execute(ForwardRequest calldata req, bytes calldata signature)
	public
	payable
	returns (bool, bytes memory)
{
	require(verify(req, signature), "MinimalForwarder: signature does not match request");
	_nonces[req.from] = req.nonce + 1;

	(bool success, bytes memory returndata) = req.to.call{gas: req.gas, value: req.value}(
		abi.encodePacked(req.data, req.from)
	);
	// Validate that the relayer has sent enough gas for the call.
	// See https://ronan.eth.link/blog/ethereum-gas-dangers/
	assert(gasleft() > req.gas / 63);

	return (success, returndata);
}
```

在使用 `Address.call` 方法的时候，根据元交易参数，指定了 `call` 的 `gas` 与 `value` 值。需要注意的是，这里并不直接将元交易的 `data` 字段当作 `call` 操作的 `data`，而是将 `data` 与 `from` 进行 `abi` 编码后一起作为 `call` 操作的参数，这在目标合约（也就是 `req.to`）中会被解析，从而得到交易的发送者，在下面会详细讲解。

`assert(gasleft() > req.gas / 63)` 简单理解为避免中继器（代为执行元交易的人）恶意地或无意地使用足够低的 gas 使得交易执行成功，而元交易执行失败。详情可以在 [ethereum gas dangers](https://ronan.eth.link/blog/ethereum-gas-dangers/) 中学习。

# ERC2771
要支持元交易，仅实现元交易智能合约是不够的，因为目标合约无法知道实际的元交易 `from` 是谁。如果没有额外的措施，它将只能够从 `msg.sender` 中获取，由于在元交易合约实现中，是通过 `Address.call` 调用的，因此将得到的发送者是元交易合约的地址。ERC2771 则解决了该问题。

```solidity
abstract contract ERC2771Context is Context
```

ERC2771Context 继承了 Context，而 Context 中简单封装了从 `msg.sender` 与 `msg.data` ，以便规范这两个功能的使用，且能够让其在子合约中修改其行为。要求使用 Context 合约获取 `msg` 相关的数据，而不是直接使用 `msg.sender` 等。

```solidity
abstract contract Context {
    function _msgSender() internal view virtual returns (address) {
        return msg.sender;
    }

    function _msgData() internal view virtual returns (bytes calldata) {
        return msg.data;
    }
}
```

ERC2771Context 就修改了 Context 合约的方法。

```solidity
function _msgSender() internal view virtual override returns (address sender) {
	if (isTrustedForwarder(msg.sender)) {
		// The assembly code is more direct than the Solidity version using `abi.decode`.
		assembly {
		sender := shr(96, calldataload(sub(calldatasize(), 20)))
		}
	} else {
		return super._msgSender();
	}
}
```

先通过 `isTrustedForwarder(msg.sender)` 验证元交易的调用方是期望的元交易合约地址。assembly 代码将上文的元交易合约中 `req.to.call{...}(abi.encodePacked(req.data, req.from))` 编码进的 `data` 部分内容的 `req.from` 获取到，然后再返回该值。

# 元交易使用概览
让我们来尝试简单使用元交易合约，要支持元交易，你所编写的合约必须继承 ERC2771Context。在这里简单实现一个 NFT 合约，在部署它之前，你必须先部署元交易合约，将元交易合约地址作为参数传递给 NFT 合约构造函数。

```solidity
// SPDX-License-Identifier: GPL3.0
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/utils/math/SafeMath.sol";
import "@openzeppelin/contracts/metatx/ERC2771Context.sol";
import "@openzeppelin/contracts/token/ERC721/ERC721.sol";

contract NFT is ERC2771Context, ERC721 {
    using SafeMath for uint256;

    uint256 private _currentTokenId = 0;

    constructor(
        string memory name,
        string memory symbol,
        address trustedForwarder
    ) ERC721(name, symbol) ERC2771Context(trustedForwarder) {}

    function safeMint() public virtual {
        safeMint("");
    }

    function safeMint(bytes memory _data) internal virtual {
        uint256 tokenId = _getNextTokenId();
        _incrementTokenId();
        _safeMint(_msgSender(), tokenId, _data);
    }
    
    function getCurrTokenId() public virtual view returns (uint256) {
        return _currentTokenId;
    }

    /**
     * @dev calculates the next token ID based on value of _currentTokenId
     * @return uint256 for the next token ID
     */
    function _getNextTokenId() internal virtual view returns (uint256) {
        return _currentTokenId.add(1);
    }

    /**
     * @dev increments the value of _currentTokenId
     */
    function _incrementTokenId() internal virtual {
        _currentTokenId++;
    }

    function _msgSender() internal view virtual override(Context, ERC2771Context) returns (address) {
        return ERC2771Context._msgSender();
    }

    function _msgData() internal view virtual override(Context, ERC2771Context) returns (bytes calldata) {
        return ERC2771Context._msgData();
    }
}
```

在这个示例中，如果 Alice 没有足够的 ETH 支付 gas 费，来铸造一个 NFT，她可以签署一个元交易，元交易的 `data` 是由 `abi.encodeWithSignature(functionSelector, parmas...)` 得到的，将该元交易递交给具有足够 ETH 的 Bob，Bob 调用元交易合约 `MinimalForwarder.execute(req, signature)`，从而让 Alice 的元交易成功执行。

[OpenZeppelin 的完整代码实现](https://github.com/OpenZeppelin/openzeppelin-contracts/tree/v4.3.2/contracts/metatx)

# 延伸阅读
- [智能合约开发实战——元交易（Metatransaction）系列一，什么是元交易？](https://blog.csdn.net/shengzang1998/article/details/121251057)
- [智能合约开发实战——元交易（Metatransaction）系列三，前端如何发起元交易的代码讲解](https://blog.csdn.net/shengzang1998/article/details/121279061)

# 引用
- [以太坊元交易](https://ethfans.org/posts/ethereum-meta-transactions)
- [维基百科](https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm)
- [EIPs](https://eips.ethereum.org/)
- [OpenZeppelin 的完整代码实现](https://github.com/OpenZeppelin/openzeppelin-contracts/tree/v4.3.2/contracts/metatx)