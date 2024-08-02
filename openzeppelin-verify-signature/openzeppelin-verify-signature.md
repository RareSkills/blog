# Verify Signature Solidity in Foundry

Here is a minimal (copy and paste) example of how to safely create and verify ECDSA signatures with OpenZeppelin in the [Foundry](https://book.getfoundry.sh/) environment.

## Contract: Verifier.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";

contract Verifier {
    using ECDSA for bytes32;

    address public verifyingAddress;

    constructor(address _verifyingAddress) {
        verifyingAddress = _verifyingAddress;
    }

    function verifyV1(
        string calldata message,
        bytes32 r,
        bytes32 s,
        uint8 v
    ) public view {
        bytes32 signedMessageHash = keccak256(abi.encode(message))
            .toEthSignedMessageHash();
        require(
            signedMessageHash.recover(v, r, s) == verifyingAddress,
            "signature not valid v1"
        );
    }

    function verifyV2(
        string calldata message,
        bytes calldata signature
    ) public view {
        bytes32 signedMessageHash = keccak256(abi.encode(message))
            .toEthSignedMessageHash();
        require(
            signedMessageHash.recover(signature) == verifyingAddress,
            "signature not valid v2"
        );
    }
}
```

## Test (Verifier.t.sol)
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "../src/Verify.sol";

import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";
import "forge-std/console.sol";

contract TestSigs1 is Test {
    using ECDSA for bytes32;
    Verifier verifier;

    address owner;
    uint256 privateKey =
        0x1010101010101010101010101010101010101010101010101010101010101010;

    function setUp() public {
        owner = vm.addr(privateKey);
        verifier = new Verifier(owner);
    }

    function testVerifyV1andV2() public {
        string memory message = "attack at dawn";

        bytes32 msgHash = keccak256(abi.encode(message))
            .toEthSignedMessageHash();

        (uint8 v, bytes32 r, bytes32 s) = vm.sign(privateKey, msgHash);

        bytes memory signature = abi.encodePacked(r, s, v);
        assertEq(signature.length, 65);

        console.logBytes(signature);
        verifier.verifyV1(message, r, s, v);
        verifier.verifyV2(message, signature);
    }
}
```

This will work even if you change the data type from "string" to something else.

Note that OpenZeppelin supports two ways to represent the signature. It's generally more convenient to use the bytes version because this is only one extra piece of data to pass around. Note however that [ERC20-Permit](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/extensions/ERC20Permit.sol) uses the three part signature (r, s, v).

## Learn More

This article is used as reference material in our [Solidity Bootcamp](https://www.rareskills.io/solidity-bootcamp).

We also have a free [learn solidity](https://www.rareskills.io/learn-solidity) tutorial.

*Originally Published March 8, 2023*