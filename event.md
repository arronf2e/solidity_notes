# 事件

## 1. 定义

事件是 Solidity 合约向链下发送日志（log）的机制。它们被写入区块的 日志（log）字段，可以被以太坊节点、区块浏览器、去中心化应用（dApp）以及各种监控工具实时捕获。

## 2. 用途

- 状态变更通知：告诉前端或其他系统合约内部发生了什么（如转账、订单创建、投票等），dapp可以通过RPC接口订阅和监听这些事件，并在前端做响应。
- 审计/追踪：提供不可篡改的历史记录，便于审计。
- 高效查询：相较于在合约存储中读取数据，日志的检索成本更低，且可以通过 索引（indexed） 参数进行过滤。
- 触发外部业务：如预言机、链下服务、自动化脚本（The Graph、Chainlink Keepers）等。

## 3. 基本用法

```
event EventName(
    type indexed param1,   // 可选 indexed，最多 3 个参数可以标记为 indexed，它们会被放入 topics（日志的索引字段），从而可以在链上使用 eth_getLogs 或 eth_filterLog 按值过滤。
    type param2,          // 普通参数，存储在 data 部分，不能直接过滤，只能在获取日志后解析。
    ...
);
```

> indexed 参数的细节
- 数量限制：最多 3 个 indexed 参数（外加第 0 个 topic 为事件签名）。
- 过滤方式：查询时可以提供 topic（哈希值）或 null（表示不过滤该位置）。
- 哈希化：indexed 参数在日志中存储的是 Keccak‑256 哈希，而不是原始值（除 address、uint256、bytes32 这类 32‑byte 固定长度类型会直接存储，不再哈希）。因此，过滤时需要提供相同的原始值（节点会自行哈希）。
- 不支持动态数组：indexed 参数不能是 string、bytes、dynamic array，因为它们需要哈希后存储。若想过滤字符串，必须先把它转换为 bytes32（如 keccak256(abi.encodePacked(str))）再作为 indexed 参数。

## 4. 示例

```
pragma solidity ^0.8.0;

contract Token {
    // 事件声明
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);

    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    function transfer(address to, uint256 amount) external {
        require(balanceOf[msg.sender] >= amount, "Insufficient");
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;

        // 触发事件
        emit Transfer(msg.sender, to, amount);
    }

    function approve(address spender, uint256 amount) external {
        allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
    }
}
```
## 5. 事件的最佳实践

| 场景 | 推荐做法 |
|------|----------|
| **标准化** | 对于 ERC‑20、ERC‑721、ERC‑1155 等已有标准的合约，务必实现对应的标准事件（`Transfer`、`Approval`、`TransferSingle`、`URI` 等）。 |
| **命名** | 事件名使用 **PascalCase**，参数名使用 **camelCase**，保持与函数命名风格一致。 |
| **索引** | 对经常用于查询的字段（如 `from`、`to`、`owner`、`tokenId`）标记为 `indexed`，但不要超过 3 个。 |
| **最小化 data** | 只在 `data` 中放置必要的非索引信息，避免过大日志导致 gas 成本上升。 |
| **避免敏感信息** | 事件是公开的，**不要**在事件中泄露私钥、密码或任何敏感数据。 |
| **使用 `emit`** | 从 Solidity 0.4.21 起必须使用 `emit` 关键字触发事件，提升可读性。 |
| **结构化** | 对于复杂业务（如订单、拍卖），可以定义多个事件或使用结构体的字段拆分为多个参数，以便更好地索引。 |
| **防止重放** | 对于跨链或跨合约的业务，考虑在事件中加入唯一的 `nonce` 或 `txHash`，帮助链下系统防止重复处理。 |
| **日志成本** | 每条日志的基本 gas 消耗约 **375** + **8 × 数据字节数**（每 32‑byte 词 8 gas），因此尽量 **压缩** 参数（如使用 `uint128` 而非 `uint256`，如果数值范围足够）。 |

## 6. 读取事件

### 6.1 ethers/web3.js

```js
// ethers.js 示例：获取最近 1000 条 Transfer 事件
const abi = [
  "event Transfer(address indexed from, address indexed to, uint256 value)"
];
const contract = new ethers.Contract(tokenAddress, abi, provider);

// 过滤条件：只查询 from = 某地址的转账
const filter = contract.filters.Transfer(myAddress, null);
const logs = await contract.queryFilter(filter, startBlock, endBlock);

logs.forEach(log => {
  console.log(`From: ${log.args.from}, To: ${log.args.to}, Value: ${log.args.value}`);
});
```

### 6.2 通过 RPC eth_getLogs

```json
{
  "jsonrpc":"2.0",
  "method":"eth_getLogs",
  "params":[{
    "fromBlock":"0x0",
    "toBlock":"latest",
    "address":"0xTokenAddress",
    "topics":[
      "0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef", // Transfer 事件签名
      "0x000000000000000000000000aabbccddeeff00112233445566778899aabbccdd" // from 地址（已左填0）
    ]
  }],
  "id":1
}
```

