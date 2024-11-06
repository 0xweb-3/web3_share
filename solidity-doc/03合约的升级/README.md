# 合约升级

## 代理合约

````solidity
// SPDX-License-Identifier: GPL-3.0

pragma solidity >=0.8.2 <0.9.0;

contract Proxy {
    // ⚠️ 注意数据槽位
    uint256 public data;

    address public logicContract; // 存储逻辑合约的地址

    // 可选的事件记录：用于记录资金的接收
    event Received(address indexed sender, uint256 amount);

    constructor (address _logicContract) {
        logicContract = _logicContract;
    }

    // bytes memory calldata = abi.encodeWithSignature("setData(uint256)", 42);
    fallback() external payable { 
        (bool success, bytes memory returnData) = logicContract.delegatecall(msg.data);
        returnData;
        require(success, "delegatecall failed");
        assembly {
            return(add(returnData, 0x20), mload(returnData))
        }
    }

    // 接收ETH时触发Received事件
    receive() external payable {}

    // 允许升级逻辑合约
    function upgrade(address _newLogicContract) public {
        logicContract = _newLogicContract;
    }
}


contract LogicV1 {
    uint256 public data;

    function setData(uint256 _data) public {
        // 为了演示，修改传入的数据（加1）
        data = _data + 1;
    }
}

contract LogicV2 {
    uint256 public data;

    function setData(uint256 _data) public  {
        data = _data * 2;  // 更新的逻辑：将数据乘以 2
    }
}


contract Caller {
    address public proxy; //代理合约地址

    event CallResult(bool success, bytes data);

    constructor(address _proxy){
        proxy = _proxy;
    }

    function callSetData() external returns (bool success, bytes memory data) {
        // 使用 call 并检查返回的成功状态
        (success, data) = proxy.call(abi.encodeWithSignature("setData(uint256)", 3));
        require(success, "call to setData failed");

        // 触发事件，记录调用结果
        emit CallResult(success, data);

        return (success, data);
    }
}
````

## 透明代理

透明代理模式通过确保只有特定角色（例如合约所有者或管理员）可以升级逻辑合约，并避免可能的逻辑合约调用冲突。透明代理通常由三个合约组成：

1. **Proxy**：负责路由用户调用到逻辑合约。
2. **LogicV1 和 LogicV2**：提供不同版本的业务逻辑合约。
3. **Caller**：用户通过这个合约与代理交互。

要实现透明代理，代理合约会分离存储数据与逻辑合约，这样在升级逻辑合约时不影响存储数据的安全。此外，透明代理模式限制了逻辑合约的调用者，防止直接通过代理调用逻辑合约函数。

```solidity
// SPDX-License-Identifier: GPL-3.0

pragma solidity >=0.8.2 <0.9.0;

contract Proxy {
    address public admin;          // 管理员地址
    address public logicContract;   // 逻辑合约地址
    uint256 public data;            // 数据存储

    // 可选的事件记录：用于记录资金的接收
    event Received(address indexed sender, uint256 amount);
    event Upgraded(address indexed newLogicContract);

    modifier onlyAdmin() {
        require(msg.sender == admin, "Proxy: Not authorized");
        _;
    }

    constructor(address _logicContract) {
        admin = msg.sender;
        logicContract = _logicContract;
    }

    // 仅管理员可以升级逻辑合约
    function upgrade(address _newLogicContract) external onlyAdmin {
        logicContract = _newLogicContract;
        emit Upgraded(_newLogicContract);
    }

    // fallback函数，用于将调用转发至逻辑合约
    fallback() external payable {
        require(msg.sender != admin, "Proxy: Admin cannot fallback to logic"); // 防止管理员自己调用逻辑合约
        (bool success, bytes memory returnData) = logicContract.delegatecall(msg.data);
        require(success, "delegatecall failed");
        assembly {
            return(add(returnData, 0x20), mload(returnData))
        }
    }

    // 接收ETH时触发Received事件
    receive() external payable {
        emit Received(msg.sender, msg.value);
    }
}

// 逻辑合约V1
contract LogicV1 {
    address internal _admin;
    address internal _logicContract;
    uint256 public data;

    function setData(uint256 _data) public {
        data = _data + 1;
    }
}

// 逻辑合约V2
contract LogicV2 {
    address internal _admin;
    address internal _logicContract;
    uint256 public data;

    function setData(uint256 _data) public {
        data = _data * 2;
    }
}

// Caller合约用于与Proxy交互
contract Caller {
    address public proxy; //代理合约地址

    event CallResult(bool success, bytes data);

    constructor(address _proxy) {
        proxy = _proxy;
    }

    function callSetData() external returns (bool success, bytes memory data) {
        // 使用 call 并检查返回的成功状态
        (success, data) = proxy.call(abi.encodeWithSignature("setData(uint256)", 3));
        require(success, "call to setData failed");

        // 触发事件，记录调用结果
        emit CallResult(success, data);

        return (success, data);
    }
}

```

## 通用可升级合约(UUPS)

UUPS（可升级代理模式）是一种升级合约逻辑的模式，与传统的透明代理不同。UUPS 模式使用一个独立的合约作为实现合约，并通过 `delegatecall` 代理调用逻辑，同时通过 `upgradeTo` 函数升级合约。

UUPS 的特点是实现了升级逻辑在实现合约中，而非代理合约中，这样可以使代理合约更轻量化。UUPS 代理模式依赖 `ERC1967` 标准管理升级实现，包括设置逻辑合约地址和安全地执行升级。

```solidity
// SPDX-License-Identifier: GPL-3.0

pragma solidity >=0.8.2 <0.9.0;

// UUPS的Proxy，跟普通的proxy像。
// 升级函数在逻辑函数中，管理员可以通过升级函数更改逻辑合约地址，从而改变合约的逻辑。
// 教学演示用，不要用在生产环境
contract UUPSProxy {
    address public implementation; // 逻辑合约地址
    address public admin; // admin地址
    string public words; // 字符串，可以通过逻辑合约的函数改变

    event CallFunResultEvent(bool, bytes);


    // 构造函数，初始化admin和逻辑合约地址
    constructor(address _implementation){
        admin = msg.sender;
        implementation = _implementation;
    }

    // fallback函数，将调用委托给逻辑合约
    fallback() external payable {
        (bool success, bytes memory data) = implementation.delegatecall(msg.data);
         emit CallFunResultEvent(success, data);
    }

    // 接收ETH时触发Received事件
    receive() external payable {}
}

// 逻辑合约1
contract Logic1 {
    // 状态变量和proxy合约一致，防止插槽冲突
    address public implementation; 
    address public admin; 
    string public words; // 字符串，可以通过逻辑合约的函数改变

    // 改变proxy中状态变量 选择器： 0xc2985578
    function foo() public {
        words = "old";
    }

    // 升级函数，改变逻辑合约地址，只能由admin调用。选择器：0x0900f010
    // UUPS中，逻辑函数中必须包含升级函数，不然就不能再升级了。
    function upgrade(address newImplementation) external {
        require(msg.sender == admin);
        implementation = newImplementation;
    }
}

// 逻辑合约2
contract Logic2 {
    // 状态变量和proxy合约一致，防止插槽冲突
    address public implementation; 
    address public admin; 
    string public words; // 字符串，可以通过逻辑合约的函数改变

    // 改变proxy中状态变量
    function foo() public {
        words = "xin";
    }

    // 升级函数，改变逻辑合约地址，只能由admin调用。选择器：0x0900f010
    // UUPS中，逻辑函数中必须包含升级函数，不然就不能再升级了。
    function upgrade(address newImplementation) external {
        require(msg.sender == admin);
        implementation = newImplementation;
    }
}
```

### UUPS 模式说明

1. **UUPS（可升级代理）模式**：
   - UUPS 模式实现了升级逻辑在实现合约中，而非在代理合约中。
   - 使用 `ERC1967` 标准槽位 `_IMPLEMENTATION_SLOT` 来存储逻辑合约的地址。
2. **代理合约的轻量化**：
   - 在 UUPS 模式中，代理合约主要负责将调用代理到逻辑合约。
   - 升级功能被推到了逻辑合约内部的 `upgrade` 函数中，因此代理合约不需要复杂的升级逻辑。
3. **安全性**：
   - UUPS 代理由 `onlyAdmin` 修饰符保护升级函数，确保只有管理员可以执行升级操作。
   - 使用 `ERC1967` 标准槽位来安全地存储逻辑合约的地址，防止存储槽被意外覆盖。
4. **升级流程**：
   - 管理员可以通过调用 `LogicV1` 中的 `upgrade` 函数，将逻辑合约升级为 `LogicV2` 或其他实现合约。





