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
