# CREATE2

## CREATE2 的原理和实现机制

`CREATE2` 是 Solidity 中一个用于创建新合约的操作码，它在以太坊中允许用户通过指定合约地址来部署合约。这与传统的 `CREATE` 操作码不同，因为 `CREATE2` 允许开发者控制合约部署后的地址，而不是由部署交易的发送者地址和合约代码决定。

`CREATE2` 的一个关键特性是，它使用了合约创建时的 **salt** 和部署时的代码来计算目标地址。通过 `CREATE2`，开发者可以预先计算出合约的地址，并且确保在之后的某些操作中，这个地址是唯一的。这对于实现合约的自动部署、合约工厂模式、以及某些需要提前知道合约地址的场景非常有用。

### **CREATE2 操作原理**

`CREATE2` 计算新合约地址的公式如下：

```
address = keccak256(0xff ++ sender_address ++ salt ++ keccak256(bytecode)) [12 bytes]
```

- **0xff**：是一个固定的前缀字节，表示合约创建操作。
- **sender_address**：是合约创建者的地址。
- **salt**：是一个由用户提供的唯一字节数据，用来确保创建的合约地址的唯一性。
- **keccak256(bytecode)**：是合约的字节码。
- 结果的哈希值的最后 20 字节就是合约的地址。

### **使用场景**

1. **工厂模式**：通过 `CREATE2` 创建一系列具有相同结构和代码的合约，每个合约的地址是不同的，可以在多个部署之间复用相同的代码。
2. **预先计算合约地址**：某些情况下，开发者希望知道合约的地址，例如在合约部署之前。
3. **合约地址的可预测性**：如果知道 `salt` 和合约的代码，你可以计算出一个合约的地址，在部署时确保地址唯一。

### **CREATE2 示例**

下面是一个使用 `CREATE2` 部署新合约的简单示例。这个示例中，我们通过工厂合约来部署新合约，并使用 `CREATE2` 预先确定合约的地址。

#### 1. `SimpleStorage` 合约（目标合约）

这是我们要部署的合约，它只有一个 `set` 和 `get` 函数来存储一个 `uint` 值。

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.8.2 <0.9.0;

contract SimpleStorage {
    uint256 public storedData;

    function set(uint256 x) public {
        storedData = x;
    }

    function get() public view returns (uint256) {
        return storedData;
    }
}
```

#### 2. `Factory` 合约（工厂合约）

`Factory` 合约将使用 `CREATE2` 来部署多个 `SimpleStorage` 合约，并且每个合约的地址可以通过指定不同的 `salt` 来控制。

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.8.2 <0.9.0;

import "./SimpleStorage.sol";

contract Factory {
    
    // 部署 SimpleStorage 合约并计算合约地址
    function deploySimpleStorage(bytes32 salt) public returns (address) {
        bytes memory bytecode = type(SimpleStorage).creationCode;
        
        // 使用 CREATE2 部署合约
        address simpleStorageAddress;
        assembly {
            simpleStorageAddress := create2(0, add(bytecode, 0x20), mload(bytecode), salt)
            if iszero(simpleStorageAddress) {
                revert(0, 0)
            }
        }
        return simpleStorageAddress;
    }

    // 计算通过 CREATE2 部署的合约地址
    function predictAddress(bytes32 salt) public view returns (address) {
        bytes memory bytecode = type(SimpleStorage).creationCode;
        bytes32 hash = keccak256(abi.encodePacked(
            bytes1(0xff),
            address(this),
            salt,
            keccak256(bytecode)
        ));
        return address(uint160(uint256(hash)));
    }
}
```

### **代码解释**

- **`SimpleStorage` 合约**：一个简单的存储合约，包含 `set` 和 `get` 函数，用于存储和返回一个 `uint256` 类型的数据。
- **`Factory` 合约**：
  - `deploySimpleStorage`：通过 `CREATE2` 创建一个新的 `SimpleStorage` 合约。它接收一个 `salt`，并使用 `CREATE2` 操作码部署合约。
  - `predictAddress`：计算通过 `CREATE2` 部署的 `SimpleStorage` 合约的地址。使用与合约创建过程相同的哈希公式，预先计算出合约地址。

### **使用 `CREATE2` 部署合约**

1. **部署 `Factory` 合约**：首先，部署工厂合约 `Factory`。
2. **预测合约地址**：调用 `Factory.predictAddress(salt)`，传入一个盐值（`salt`），它会返回预期的合约地址。你可以用这个地址来做进一步的操作。
3. **部署合约**：调用 `Factory.deploySimpleStorage(salt)`，传入相同的盐值，工厂合约将部署一个新的 `SimpleStorage` 合约，并返回该合约的实际地址。
4. **与新合约交互**：你可以使用返回的地址来与新部署的 `SimpleStorage` 合约进行交互，例如调用 `set` 或 `get` 函数。

### **例子**

假设我们希望通过 `CREATE2` 部署两个 `SimpleStorage` 合约，并确保它们的地址不同。我们可以使用不同的 `salt` 值来实现这一点。

```js
const salt1 = web3.utils.asciiToHex("salt1");
const salt2 = web3.utils.asciiToHex("salt2");

// 使用 Factory 合约预测合约地址
const predictedAddress1 = await factory.methods.predictAddress(salt1).call();
console.log("Predicted Address 1:", predictedAddress1);

const predictedAddress2 = await factory.methods.predictAddress(salt2).call();
console.log("Predicted Address 2:", predictedAddress2);

// 部署合约
const deployedAddress1 = await factory.methods.deploySimpleStorage(salt1).send({ from: deployerAddress });
console.log("Deployed Address 1:", deployedAddress1);

const deployedAddress2 = await factory.methods.deploySimpleStorage(salt2).send({ from: deployerAddress });
console.log("Deployed Address 2:", deployedAddress2);
```

### **总结**

- `CREATE2` 允许你在部署合约时控制目标合约的地址，通过盐值（`salt`）和合约字节码来计算合约地址。这使得你可以在合约部署之前，预测或计算出合约的地址。
- 它在工厂模式、合约地址预知和某些优化操作中非常有用。
- 通过 `CREATE2` 部署的合约地址是可以预测的，这对某些用例（如合约升级、合约工厂等）非常有用。

### 如何保证部署合约地址相同

在 Solidity 中，要保证通过 `CREATE2` 部署的合约地址相同，需要满足以下几个条件：

1. **创建者合约地址**：同一个部署合约的地址。
2. **Salt**：一个一致的、固定的随机数（`bytes32`类型），在部署时传入。
3. **合约字节码**：必须保持完全一致，包括合约的编译器版本和所有代码内容。

## 如何在不同以太坊同源链上部署相同合约，保证地址一致

在以太坊和其他同源链（如 Binance Smart Chain、Polygon 等）上，若要部署相同的合约并确保生成的合约地址相同，可以通过 `CREATE2` 操作来实现。以下步骤将帮助你在多条链上获得相同的合约地址：

### 1. 确保合约部署者的地址相同
由于 `CREATE2` 的合约地址生成公式依赖于**部署合约的地址**，必须在每条链上确保**同一地址**作为部署者合约。如果有多签或去中心化部署的需求，也可以将多个用户的地址合并在一起，在链上生成一个唯一的部署者合约。

### 2. 保持 `salt` 一致
在 `CREATE2` 计算的合约地址中，`salt` 是一个由你指定的 32 字节值。在每条链上部署时必须使用相同的 `salt` 值，以确保合约地址生成的一致性。

### 3. 保持待部署合约的字节码完全一致
任何字节码上的差异（如合约内容、编译器版本等）都会导致不同的 `keccak256(bytecode)` 值，进而生成不同的地址。因此需要注意：
   - 使用相同的 Solidity 源代码（保持一致的函数、逻辑等）。
   - 使用相同的 Solidity 编译器版本和优化设置，以确保生成的字节码完全一致。

### 4. 合约示例：多链上部署同一合约
可以实现一个简单的 Factory 合约，通过 `CREATE2` 部署目标合约，并预测地址。以下是示例代码：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract MultiChainFactory {
    // 部署 SimpleStorage 合约
    function deploy(bytes32 salt) external returns (address) {
        bytes memory bytecode = type(SimpleStorage).creationCode;
        address addr;

        assembly {
            addr := create2(0, add(bytecode, 0x20), mload(bytecode), salt)
            if iszero(extcodesize(addr)) { revert(0, 0) }
        }

        return addr;
    }

    // 预测 SimpleStorage 的部署地址
    function predictAddress(bytes32 salt) external view returns (address) {
        bytes memory bytecode = type(SimpleStorage).creationCode;
        bytes32 hash = keccak256(abi.encodePacked(
            bytes1(0xff),
            address(this),
            salt,
            keccak256(bytecode)
        ));
        return address(uint160(uint256(hash)));
    }
}

// 示例的待部署合约
contract SimpleStorage {
    uint public data;

    function setData(uint _data) external {
        data = _data;
    }
}
```

### 部署步骤
1. **在每条链上部署 MultiChainFactory 合约**：确保每条链上由同一地址部署该合约，保持 `Factory` 合约地址一致。
   
2. **调用 `deploy` 方法**：在每条链上使用相同的 `salt` 值和相同的 `SimpleStorage` 字节码，部署目标合约 `SimpleStorage`。

3. **验证地址一致性**：调用 `predictAddress` 方法，查看地址是否一致。
