# Exception 异常

在 Solidity 中，异常（也叫 错误、回滚）是指在合约执行过程中出现的导致整个事务（transaction）状态回滚的情况。异常会撤销本次调用期间对 状态变量、余额、日志（event）等所做的所有修改，只保留到异常触发前的区块链状态。

## 1. 异常的种类与触发方式

| 类型 | 触发方式 | 语义 | Gas 费用 | 备注 |
|------|----------|------|----------|------|
| **`require(condition, "msg")`** | 手动调用或 Solidity 编译器在 **输入校验**、**外部调用返回值** 检查时自动插入 | **前置条件不满足** → 回滚并返回错误信息 | 消耗到 **错误点** 的 gas（不再继续执行） | 常用于函数参数、权限、外部调用成功性检查。 |
| **`revert("msg")`** | 手动调用或在 `if`/`else` 分支中显式 `revert` | **业务逻辑错误** → 回滚并返回错误信息 | 同 `require`，消耗到错误点的 gas | 适合需要在多个条件分支中统一返回错误的场景。 |
| **`assert(condition)`** | 手动调用或编译器在 **内部错误**（如算术溢出、数组越界）时自动插入（自 0.8 起算术溢出已改为 `revert`） | **不可能出现的内部错误** → 回滚并 **消耗全部剩余 gas**（即使 `assert` 触发后仍会消耗完剩余 gas） | **全部剩余 gas**（最贵） | 用于检查 **不应被破坏的内部不变式**，如 `assert(totalSupply == sum(balances))`。 |
| **自定义错误（`error MyError(uint256 code, address who);`）** | 手动 `revert MyError(...);` | **自定义错误类型** → 回滚并返回 **错误 selector + 编码参数**（比字符串更省 gas） | 消耗到错误点的 gas | Solidity 0.8.4+ 支持，推荐在需要返回结构化错误信息时使用。 |
| **`try/catch` 捕获** | 对 **外部合约调用**（`external` 函数）或 **`new` 创建合约** 使用 `try` 包裹 | 捕获 **`revert`/`require`/`assert`** 或 **自定义错误**，防止异常向上传递 | 捕获后仍会消耗到错误点的 gas（已消耗的部分），但不会导致调用者回滚 | 只能捕获 **外部调用**（`external`）的异常，不能捕获同合约内部的 `require/assert`。 |


## 2. 异常是如何回滚的？

- EVM 事务模型
  - 每笔交易在执行时会创建一个 临时的状态快照（memory）。
  - 当执行过程中出现 REVERT（opcode 0xfd）或 INVALID（opcode 0xfe）时，EVM 会 撤销 所有对 持久化状态（storage、balance、nonce）的写入，只保留 日志（event） 中已经写入的 topic/data（这些日志在 REVERT 时也会被撤销）。
  - 剩余 gas 会被 退回 给交易发起者（REVERT），但已经消耗的 gas 不会返还。
 
- require / revert 实现
  - 编译器把 require(condition, "msg") 编译成：
    ```
    if iszero(condition) { 
        // 把错误字符串放入 memory
        revert(offset, length)   // opcode REVERT
    }
    ```
  - revert(offset, length) 会把 错误数据（错误 selector + 可选字符串）返回给调用者。

- assert 实现
  - 编译器把 assert(condition) 编译成：
    ```
    if iszero(condition) { 
        invalid   // opcode INVALID (0xfe) → 消耗全部剩余 gas
    }
    ```
  - invalid 触发 异常，并且 不返回错误数据，只消耗全部 gas。
 
- 自定义错误
  - 定义：error MyError(uint256 code, address who);
  - 编译器为每个错误生成 selector（前 4 字节的 keccak256 哈希），在 revert 时返回 selector + abi.encodePacked(arguments)。
  - 这比普通字符串更紧凑（4 bytes + 编码参数），大幅降低 gas（尤其在错误信息较长时）。
 
- try/catch 捕获
  - try 包裹外部调用后，EVM 会把 错误数据（selector + encoded args）返回给调用者。
  - catch 分支可以匹配：
    - catch Error(string memory reason) → 捕获普通 require/revert 带的字符串。
    - catch Panic(uint256 errorCode) → 捕获 assert、算术溢出、数组越界等内部错误（errorCode 为 0x01‑0x12）。
    - catch (bytes memory lowLevelData) → 捕获自定义错误或任何低层错误（返回原始 bytes）。

## 3. 代码示例

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/* ---------- 自定义错误 ---------- */
error Unauthorized(address caller);
error InsufficientBalance(address account, uint256 requested, uint256 available);
error TransferFailed(address token, address to, uint256 amount);
error ZeroAddress();

/* ---------- ERC20 合约 ---------- */
contract MyToken {
    string public name = "MyToken";
    string public symbol = "MTK";
    uint8  public decimals = 18;
    uint256 public totalSupply;

    mapping(address => uint256) private _balances;
    mapping(address => mapping(address => uint256)) private _allowances;

    address public immutable owner;

    constructor(uint256 _initialSupply) {
        owner = msg.sender;
        _mint(msg.sender, _initialSupply);
    }

    /* ---------- 事件 ---------- */
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);

    /* ---------- 修饰器 ---------- */
    modifier onlyOwner() {
        if (msg.sender != owner) revert Unauthorized(msg.sender);
        _;
    }

    /* ---------- ERC20 基础功能 ---------- */
    function balanceOf(address account) external view returns (uint256) {
        return _balances[account];
    }

    function allowance(address _owner, address spender) external view returns (uint256) {
        return _allowances[_owner][spender];
    }

    function approve(address spender, uint256 amount) external returns (bool) {
        if (spender == address(0)) revert ZeroAddress();
        _allowances[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }

    function transfer(address to, uint256 amount) external returns (bool) {
        _transfer(msg.sender, to, amount);
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        uint256 current = _allowances[from][msg.sender];
        if (current < amount) revert InsufficientBalance(msg.sender, amount, current);
        unchecked { _allowances[from][msg.sender] = current - amount; }
        _transfer(from, to, amount);
        return true;
    }

    /* ---------- 内部转账实现 ---------- */
    function _transfer(address from, address to, uint256 amount) internal {
        if (to == address(0)) revert ZeroAddress();

        uint256 fromBal = _balances[from];
        if (fromBal < amount) revert InsufficientBalance(from, amount, fromBal);
        unchecked {
            _balances[from] = fromBal - amount;
            _balances[to] += amount;
        }
        emit Transfer(from, to, amount);
    }

    /* ---------- 铸币（仅 owner） ---------- */
    function mint(address to, uint256 amount) external onlyOwner {
        _mint(to, amount);
    }

    function _mint(address to, uint256 amount) internal {
        if (to == address(0)) revert ZeroAddress();
        totalSupply += amount;
        _balances[to] += amount;
        emit Transfer(address(0), to, amount);
    }

    /* ---------- 示例：外部调用并捕获错误 ---------- */
    function safeTransferERC20(address token, address to, uint256 amount) external {
        // 调用外部 ERC20 合约的 transfer
        try IERC20(token).transfer(to, amount) returns (bool success) {
            if (!success) revert TransferFailed(token, to, amount);
        } catch Error(string memory reason) {
            // 捕获普通 revert 带的字符串
            revert TransferFailed(token, to, amount);
        } catch (bytes memory lowLevel) {
            // 捕获自定义错误或低层错误
            revert TransferFailed(token, to, amount);
        }
    }
}

/* ---------- ERC20 接口（用于外部调用） ---------- */
interface IERC20 {
    function transfer(address to, uint256 amount) external returns (bool);
}
```
