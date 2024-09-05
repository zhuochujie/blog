---
title: Create2方法的使用以及合约地址的计算
date: 2024-09-05 13:23:07
tags: [Solidity,Web3,区块链,智能合约,create2]
---

## create2方法简介
简单来说就是通过合约的字节码加一个salt创建一个合约。通过create2方法创建合约地址是确定性的，只要保证创建者、字节码以及salt保持不变，计算出来的地址就是确定的。而通过new关键字和直接部署合约的方式创建出来的合约地址跟创建者的nonce有关联，当创建者的nonce发生改变就无法计算出准确的合约地址。
## 使用方法
```js
bytes memory bytecode = type(UniswapV2Pair).creationCode;
bytes32 salt = keccak256(abi.encodePacked(token0, token1));
assembly {
    pair := create2(0, add(bytecode, 32), mload(bytecode), salt)
}
```
Solidity0.8以后可以直接使用new关键字替代create2,并且还可以传入构造参数

```js
bytes32 salt = keccak256(abi.encodePacked(token0, token1));
address pair = new UniswapV2Pair{salt:salt}(token0, token1);
```
## 计算合约地址
```js
function calculateAddr(address tokenA, address tokenB) public returns (address contractAddress) {
    (address token0, address token1) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA);
    bytes32 salt = keccak256(abi.encodePacked(token0, token1));
    contractAddress = address(uint160(uint256(keccak256(abi.encodePacked(
        bytes1(0xff),
        address(this), // 这个地址是调用create2方法创建合约的地址
        salt
    )))));
}
```
## 总结
当需要提前计算合约地址或者离线计算合约地址的情况下使用create2方法或者0.8版本以上添加salt参数。