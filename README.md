# 数据类型

```solidity
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

```

# 函数

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.21;

// function <function name>([parameter types[, ...]]) {internal|external|public|private} [pure|view|payable] [virtual|override] [<modifiers>]
// [returns (<return types>)]{ <function body> }


contract Function {

    uint256 public num = 1;


    // 1. pure vs view
    // 函数既不能读取也不能写入链上的状态变量，
    function add() external  {
        num = num + 1;
    }
    // 纯函数
    function pureFun(uint256 number) external pure returns(uint256 newNumber) {
        newNumber =  number + 1; 
    }

    // view能查看不能修改
    function viewFun() external view returns(uint256 newNumber) {
        newNumber = num + 1;
    }

    // 2. internal vs external
    // internal: 内部函数，无法直接调用，一般用来隐藏敏感函数，减少攻击面
    function minus() internal {
        num = num - 1;
    }

    // 合约内的函数可以调用内部函数
    function minusCall() external {
        minus();
    }

    // payable, 递钱，能给合约支付eth的函数
    function minusPayable() external payable returns(uint256 balance) {
        minus();    
        // this 表示当前合约
        balance = address(this).balance;
    }

}
```

# 函数返回值

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract FunctionReturn {
    uint256 public number;
    bool public isOk;

    // 显式返回多个变量
    function returnMultiple() public pure returns(uint256, bool, uint256[3] memory) {
        return (1, true, [uint256(1), 2, 5]);
    }

    // 命名式返回（避免遮蔽状态变量）
    function returnNamed() public pure returns(uint256 retNumber, bool retBool) {
        retNumber = 2;
        retBool = false;
    }

    // 解构赋值必须在函数内
    function testDestruction() public {
        (number, isOk) = returnNamed();
    }
}
```

# 变量数据存储和作用域

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.21;

contract Storage {
    // 引用类型: 数组，结构体

    // 存储位置
    // 1. storag，存链上，类似硬盘，gas多，合约里的状态变量默认都是存 storage
    // 2. memory，存内存，少量gas
    // 3. calldata，存内存，gas更少，不能修改 immutable，一般用在函数的参数

    function fCalldata(uint[] calldata _x) public pure returns(uint[] calldata){
        //参数为calldata数组，不能被修改
        // _x[0] = 0; //这样修改会报错
        return(_x);
    }

    uint[] x = [1,2,3];

    function testRefer() public {
        // storage（合约的状态变量）赋值给本地storage（函数里的）时候，会创建引用，改变新变量会影响原变量
        uint[] storage xStorage = x; // 引用 
        xStorage[0] = 100;
    }

    // 变量作用域
    // 1. 状态变量，存储链上，所有函数都可以使用,gas高
    uint public e = 1;
    uint public y;
    string public z;

    // 2. 局部变量
    // 部变量是仅在函数执行过程中有效的变量，函数退出后，变量无效。局部变量的数据存储在内存里，不上链，gas低
    function foo() pure public {
        uint a = 1;
        uint b = 2;
        uint c = a + b;
    }

    // 3. 全局变量
    // 不用声明就可以使用的全局变量
    // 详细：https://learnblockchain.cn/docs/solidity/units-and-global-variables.html#special-variables-and-functions
    address user = msg.sender;
    uint blockNum = block.number;
    uint timeStamp = block.timestamp;

    // 4. 以太单位与时间单位
    // wei: 1
    // gwei: 1e9 = 1000000000
    // ether: 1e18 = 1000000000000000000

    function weiUnit() external pure returns(uint) {
        assert(1 wei == 1e0);
        assert(1 wei == 1);
        return 1 wei;
    }

    function gweiUnit() external pure returns(uint) {
        assert(1 gwei == 1e9);
        assert(1 gwei == 1000000000);
        return 1 gwei;
    }

    function etherUnit() external pure returns(uint) {
        assert(1 ether == 1e18);
        assert(1 ether == 1000000000000000000);
        return 1 ether;
    }

    // 5. 时间单位
    // seconds: 1
    // minutes: 60 seconds = 60
    // hours: 60 minutes = 3600
    // days: 24 hours = 86400
    // weeks: 7 days = 604800
    function secondsUnit() external pure returns(uint) {
        assert(1 seconds == 1);
        return 1 seconds;
    }

    function minutesUnit() external pure returns(uint) {
        assert(1 minutes == 60);
        assert(1 minutes == 60 seconds);
        return 1 minutes;
    }

    function hoursUnit() external pure returns(uint) {
        assert(1 hours == 3600);
        assert(1 hours == 60 minutes);
        return 1 hours;
    }

    function daysUnit() external pure returns(uint) {
        assert(1 days == 86400);
        assert(1 days == 24 hours);
        return 1 days;
    }

    function weeksUnit() external pure returns(uint) {
        assert(1 weeks == 604800);
        assert(1 weeks == 7 days);
        return 1 weeks;
    }
}
```

# 引用类型

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.21;

contract ReferType {
    // 引用类型

    // 1. 数组 array
    // 固定长度数组
    uint8[5] array1;
    address[100] array2;

    // 不定长数组
    uint8[] array3;
    address[] array4;
    bytes array5;

    // 创建数组的规则
    // 对于memory修饰的动态数组，可以用new操作符来创建，但是必须声明长度，并且声明后长度不能改变。
    function test() public pure {
        uint[] memory array6 = new uint[](5);
        bytes memory array7 = new bytes(9);
    }

    // 数组方法, length, push, pop
    uint256 len = array1.length;
    function test2() public {
        array3.push(2);
        array4.pop();
    }

    // 结构体 struct
    struct Student {
        uint256 id;
        uint256 score;
    }

    Student student;

    //  给结构体赋值
    // 方法1:在函数中创建一个storage的struct引用
    function initStudent1() external{
        Student storage _student = student; // 多余引用，浪费 gas
        _student.id = 11;
        _student.score = 100;
    }

    // 方法2:直接引用状态变量的struct，字段少可以这么做
    function initStudent2() external{
        student.id = 1;
        student.score = 80;
    }

    // 方法3:构造函数式，不推荐，必须按顺序
    function initStudent3() external {
        student = Student(3, 90);
    }

    // 方法4:key value， 强烈推荐
    function initStudent4() external {
        student = Student({id: 4, score: 60});
    }

}
```

# mapping

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.21;

contract MappingDemo {

    // 一个实体用 struct，多个实体用 mapping。
    // 一个学生 struct，一千个学生 mapping。

    // 用 mapping 的场景：
    // - "用户的余额"
    // - "ID 对应的订单"
    // - "地址是否有权限"
    // - "tokenId 的拥有者"
    // - "提案编号的内容"

    // mapping，键值对，哈希表
    mapping(uint => address) public idToAddress; // id映射到地址
    mapping(address => address) public swapPair; // 币对的映射，地址到地址

    // 规则
    // 1. key 必须为内置类型，不能使用 struct
    // 错误示例：
    // struct Student{
    //     uint256 id;
    //     uint256 score; 
    // }
    // mapping(Student => uint) public testVar;

    // 2. mapping 的存储位置必须是storage
    // 3. 如果映射声明为public，那么Solidity会自动给你创建一个getter函数，可以通过Key来查询对应的Value

    // demo
    function writeMapping(uint _key, address _value) public {
        idToAddress[_key] = _value;
    }
}
```

# 变量默认值

| 类型 | 默认初始值 |
|------|-----------|
| `bool` | `false` |
| `uint`, `uint8` … `uint256` | `0` |
| `int`, `int8` … `int256` | `0` |
| `address` | `0x0000000000000000000000000000000000000000` / `address(0)` |
| `bytesN`（固定长度字节数组） | 全部为 `0` 的字节 |
| `bytes`、`string`（动态字节数组） | 空（长度为 0） |
| `enum` | 第一个枚举成员的值（对应 `0`） |
| `struct` | 其所有成员均使用各自类型的默认值进行递归初始化 |
| `mapping` | 所有键的值均为对应类型的默认值（映射本身不占用存储） |
| `array`（固定长度） | 每个元素均使用其类型的默认值进行初始化 |
| `array`（动态长度） | 初始为空，长度为 0 |


# 常数

在 Solidity 中，constant 和 immutable 都是用于声明状态变量为不可修改的修饰符，主要目的是优化 gas 消耗并提升合约安全性。它们都支持值类型（如 uint、address）和字符串（constant 额外支持），但不能用于数组或映射等复杂类型。两者都不会在部署后被修改，但关键区别在于值设置时机、存储方式和 gas 影响。 

constant：编译时即写死的常量，不占 storage，读取免费，适用于永远不变的数值。
immutable：部署时可设、之后不可改的变量，占用 1 个 storage 槽，读取有常规成本，适用于需要在构造函数中动态决定但随后保持不变的配置。

在实际开发中，优先使用 constant 来表示完全固定的值；需要在部署时才确定的参数 则使用 immutable。这样既能节约 gas，又能保持灵活性。

1. const
```
// constant变量必须在声明的时候初始化，之后不能改变
uint256 constant CONSTANT_NUM = 10;
string constant CONSTANT_STRING = "hello world";
address constant CONSTANT_ADDRESS = 0x0000000000000000000000000000000000000000;
```

2. immutable
```
address immutable USER = msg.sender;
```
