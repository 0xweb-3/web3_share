````solidity
// SPDX-License-Identifier: GPL-3.0

pragma solidity >=0.8.2 <0.9.0;


// 抽象合约 Upgradeable，定义了一个基础的 setData 函数和存储变量
abstract contract Upgradeable {
    uint256 internal data;

    // 抽象的 setData 函数，子合约必须实现它
    function setData(uint256 _data) public virtual {
        data = _data;
    }

    function getData() public view  returns (uint256) {
        return data;
    }
}

contract Proxy {
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
    }

    receive() external payable { 
        emit Received(msg.sender, msg.value);
    }

    // 允许升级逻辑合约
    function upgrade(address _newLogicContract) public {
        logicContract = _newLogicContract;
    }
}


contract LogicV1 is Upgradeable {
    // 重写父合约中的 setData 函数，添加自定义的逻辑
    function setData(uint256 _data) public override {
        // 为了演示，修改传入的数据（加1）
        data = _data + 1;
    }
}

contract LogicV2 is Upgradeable{
    function setData(uint256 _data) public override  {
        data = _data * 2;  // 更新的逻辑：将数据乘以 2
    }
}

````