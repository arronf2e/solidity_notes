# solidity_notes

## 1. 值类型

### 1.1 bool

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.21;

contract DataType {
    bool public _bool = true;

    // 未初始化的bool类型变量默认值为false
    bool public isOk; 

    // 运算
    bool public bool1 = !_bool;
    bool public bool2 = _bool && isOk;
    bool public bool3 = _bool || isOk;
    bool public bool4 = _bool == isOk;
    bool public bool5 = _bool != isOk;
}
```
