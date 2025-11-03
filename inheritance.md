# 继承

为什么需要继承？
- 代码复用：把通用功能（如 Ownable、ERC20、ERC721）抽取到基类，子合约直接复用。
- 模块化：把权限、数学库、接口实现等拆分成独立合约，便于维护和升级。
- 多态：子合约可以覆盖（override）父合约的函数，实现不同的业务逻辑。

## 1. 规则

- virtual: 父合约中的函数，如果希望子合约重写，需要加上virtual关键字。
- override：子合约重写了父合约中的函数，需要加上override关键字。

## 2. 基本语法 

```solidity
pragma solidity ^0.8.0;

// 父合约（基类）
contract A {
    uint256 public x;

    constructor(uint256 _x) {
        x = _x;
    }

    function foo() public virtual returns (string memory) {
        return "A";
    }
}

// 子合约（派生类）
contract B is A {
    // 必须在子合约构造函数里调用父构造函数
    constructor(uint256 _x) A(_x) {}

    // 覆盖父合约的 foo，必须加上 `override`
    function foo() public virtual override returns (string memory) {
        return "B";
    }

    // 调用父合约的实现
    function callParentFoo() public view returns (string memory) {
        return super.foo();   // 等价于 A.foo()
    }
}
```

- is 关键字用于声明继承关系。
- virtual：父函数若希望子合约能够覆盖，需要标记为 virtual。
- override：子合约覆盖父函数时必须标记 override（可以写多个父类，用逗号分隔）。
- super：在多重继承中，super 按 线性化顺序（C3 线性化）调用最近的父实现。

## 3. 多重继承与线性化（C3 Linearization）

Solidity 允许一个合约 继承多个父合约，但必须遵守 C3 线性化 规则，确保调用顺序唯一且确定。

```solidity
contract A {
    function foo() public virtual returns (string memory) { return "A"; }
}
contract B is A {
    function foo() public virtual override returns (string memory) { return "B"; }
}
contract C is A {
    function foo() public virtual override returns (string memory) { return "C"; }
}
contract D is B, C {
    // D 的线性化顺序： D → B → C → A
    function foo() public override(B, C) returns (string memory) {
        return super.foo(); // 调用 B.foo() → C.foo() → A.foo()
    }
}
```

## 4. 修饰器（modifier）继承

### 4.1 单继承

```solidity
contract A {
    address public owner;

    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
}

contract B is A {
    // 直接使用父合约的修饰器
    function setOwner(address newOwner) external onlyOwner {
        owner = newOwner;
    }
}
```

### 4.2 多重继承

```solidity
contract A {
    modifier onlyOwner() virtual {
        require(msg.sender == owner, "A: Not owner");
        _;
    }
}

contract B {
    modifier onlyOwner() virtual {
        require(msg.sender == owner, "B: Not owner");
        _;
    }
}

contract C is A, B {
    // 必须覆盖同名修饰器，并指明覆盖了哪些父实现
    modifier onlyOwner() override(A, B) {
        // 可以选择保留父实现的逻辑（使用 super）或自行实现
        super.onlyOwner();   // 调用最近的父实现（线性化顺序决定）
        // 这里可以再加自己的检查
        _;
    }

    function foo() external onlyOwner {
        // 只有通过所有父级的 onlyOwner 检查后才能执行
    }
}
```

## 5. 抽象合约（Abstract Contract）

- 定义：包含未实现（virtual 且没有 override）的函数的合约。不能直接部署，只能被子合约继承并实现缺失的函数。
- 用途：定义接口、模板或公共逻辑框架。

```solidity
abstract contract ERC20Base {
    function totalSupply() public view virtual returns (uint256);
    function balanceOf(address account) public view virtual returns (uint256);
    // 其它公共函数可以在这里实现
}
contract MyToken is ERC20Base {
    uint256 private _total;
    mapping(address => uint256) private _balances;

    function totalSupply() public view override returns (uint256) {
        return _total;
    }
    function balanceOf(address account) public view override returns (uint256) {
        return _balances[account];
    }
}
```

## 6. 接口


- 接口是一种 合约抽象类型，只定义 函数签名（函数名称、参数、返回值、可见性）和 事件，不包含实现代码、状态变量或 构造函数。
- 它的作用类似于 API 规范：告诉外部调用者（或其他合约）该合约提供哪些函数、如何调用，但不关心内部实现细节。
- 在编译时，接口会被 链接 到实际实现合约的字节码中，调用时会直接跳转到实现合约的对应函数。

### 6.1 用法 

```solidity
pragma solidity ^0.8.0;

/* ---------- 接口声明 ---------- */
interface IToken {
    // 只能声明 external 函数（默认 external）
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address to, uint256 amount) external returns (bool);
    function approve(address spender, uint256 amount) external returns (bool);
    function allowance(address owner, address spender) external view returns (uint256);

    // 事件也可以在接口里声明
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
}
```

### 6.2 接口 vs 抽象合约（Abstract Contract）

| 特性 | Interface | Abstract Contract |
|------|-----------|-------------------|
| **函数实现** | **不允许**（只能声明） | 可以有实现，也可以有未实现的 `virtual` 函数 |
| **状态变量** | 不允许 | 允许（可以声明存储变量） |
| **构造函数** | 不允许 | 允许（子合约需要调用） |
| **继承方式** | `is` 关键字（只能继承） | `is` 关键字（可以继承并实现） |
| **用途** | 定义外部调用的 **API**，常用于 **ERC 标准**、跨合约交互 | 用于 **模板/基类**，提供部分实现或公共工具函数 |

> **简言之**：如果你只想描述 **“这个合约必须实现哪些函数”**，用 **interface**；如果你想提供 **部分实现 + 需要子合约补全的抽象函数**，用 **abstract contract**。

### 6.3 实现接口的合约

```solidity
contract MyToken is IToken {
    // 状态变量（接口本身不能有，但实现合约可以有）
    uint256 private _totalSupply;
    mapping(address => uint256) private _balances;
    mapping(address => mapping(address => uint256)) private _allowances;

    // 实现接口函数
    function totalSupply() external view override returns (uint256) {
        return _totalSupply;
    }

    function balanceOf(address account) external view override returns (uint256) {
        return _balances[account];
    }

    function transfer(address to, uint256 amount) external override returns (bool) {
        require(_balances[msg.sender] >= amount, "Insufficient");
        _balances[msg.sender] -= amount;
        _balances[to] += amount;
        emit Transfer(msg.sender, to, amount);
        return true;
    }

    function approve(address spender, uint256 amount) external override returns (bool) {
        _allowances[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }

    function allowance(address owner, address spender) external view override returns (uint256) {
        return _allowances[owner][spender];
    }
}
```

- **`override`** 必须写在每个实现的函数上，告诉编译器这是对接口（或父合约）函数的实现。  
- **`emit`** 事件同样需要在实现合约里触发。

### 6.4 使用接口进行跨合约调用

```solidity
contract TokenInteractor {
    // 通过接口与任意符合 IToken 标准的合约交互
    function getBalance(address token, address user) external view returns (uint256) {
        return IToken(token).balanceOf(user);
    }

    function transferTokens(address token, address to, uint256 amount) external returns (bool) {
        // 调用外部合约的 transfer
        return IToken(token).transfer(to, amount);
    }
}
```

- 只要 `token` 地址对应的合约实现了 `IToken` 接口（即拥有相同函数签名），上述调用就会成功。  
- **不需要**在编译时知道具体实现合约的完整代码，只要接口匹配即可。

### 6.5 接口的局限与注意事项

| 限制 | 说明 |
|------|------|
| **不能有实现** | 只能声明函数签名，若需要共享实现，请使用抽象合约或库（library）。 |
| **不能有状态变量** | 所有存储必须在实现合约里声明。 |
| **只能是 `external`** | 接口函数默认 `external`，不能声明 `public`、`internal`、`private`。 |
| **不支持构造函数** | 接口只能描述行为，不能约束部署时的初始化。 |
| **事件只能声明** | 只能声明事件，实际 `emit` 必须在实现合约里完成。 |
| **多继承冲突** | 若两个父接口声明了同名函数且签名不完全相同，编译会报错。需要在子合约中统一实现。 |
