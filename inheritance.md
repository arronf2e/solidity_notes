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
