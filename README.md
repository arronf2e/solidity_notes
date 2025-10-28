// SPDX-License-Identifier: MIT
pragma solidity ^0.8.21;

contract DataType {
    bool public _bool = true;

    // 未初始化的bool类型变量默认值为false
    bool public isOk; 

    // 运算，注意：&& 和 || 运算符遵循短路规则
    bool public bool1 = !_bool;  // false
    bool public bool2 = _bool && isOk; // false
    bool public bool3 = _bool || isOk; // true
    bool public bool4 = _bool == isOk; // false
    bool public bool5 = _bool != isOk; // true

    int public num1 = 8;// 整数，包括负数
    int public num2 = -8;
    uint256 public num3 = 300000; // 无符号整数，256位

    // address 
    address public myadd = 0x7A58c0Be72BE218B41C608b7Fe7C5bB630736C71;

    // 可以转账、查余额的 address
    address payable public payableAddress = payable(myadd); // 可支付地址

    uint256 public balnace = payableAddress.balance;  

    // 定长数组
    // 属于值类型，数组长度在声明之后不能改变。根据字节数组的长度分为 bytes1, bytes8, bytes32 等类型。定长字节数组最多存储 32 bytes 数据，即bytes32
    bytes32 public _byte32 = "MiniSolidity"; 
    bytes1 public m = _byte32[0];

    // enum 比较冷门的数据类型
    // 名称来代替从 0 开始的 uint
    enum Fruit {
        Apple,
        Banana
    }

    Fruit apple = Fruit.Apple;

}
