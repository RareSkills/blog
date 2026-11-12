# Smart Contract Foundry Upgrades with the OpenZeppelin Plugin

Upgrading a smart contract is a multistep and error-prone process, so to minimize the chances of human error, it is desirable to use a tool that automates the procedure as much as possible.
 
Therefore, the [OpenZeppelin Upgrade Plugin](https://docs.openzeppelin.com/upgrades-plugins/) streamlines deploying, upgrading and managing smart contracts built with Foundry or Hardhat.

In this article, we'll learn how to use Upgrade Plugin with [Foundry](https://rareskills.io/post/foundry-testing-solidity) to manage contract upgrades, both locally and on the Sepolia Testnet. We'll also cover how these plugins safeguard against common upgrade issues.

## Prerequisites

To get the most out of this guide, the reader should be familiar with:

- The [delegatecall](https://rareskills.io/post/delegatecall) operation in Solidity.
- Standards like [ERC1967](https://rareskills.io/post/erc1967) and the role of [initializers](https://rareskills.io/post/initializable-solidity) in setting up state in upgradeable contracts.
- Common upgrade failures, including [function selector](https://rareskills.io/post/function-selector) and [storage slot](https://rareskills.io/post/evm-solidity-storage-layout) collisions.
- Knowledge of common proxy patterns such as the [Transparent Upgradeable Proxy](https://rareskills.io/post/transparent-upgradeable-proxy), [UUPS Proxy](https://rareskills.io/post/uups-proxy), or [Beacon Proxy](https://rareskills.io/post/beacon-proxy)
- The Namespace storage layout, or [EIP-7201](https://rareskills.io/post/erc-7201).

### Upgrades plugin for Foundry

The OpenZeppelin plugin for Foundry is a utility that can be imported into Foundry Solidity scripts or unit tests and can be imported as follows:

```solidity
import { Upgrades } from "openzeppelin-foundry-upgrades/Upgrades.sol";
```

This library exposes functions for deploying proxies, implementation contracts, and other related contracts. In the next section we provide a high level overview of its capabilities, and later show how to write unit tests for upgrades, and create a script that assists with deploying and upgrading smart contracts using this plugin.

## Features of the Upgrade Plugins

- Given a reference to the previous smart contract implementation, the plugin compares the previous implementation to the new one to check for potential problems like storage slot collisions and other issues we will discuss later.  
- The plugins support deploying and upgrading UUPS, Transparent and Beacon Proxy Patterns. It does not support the Diamond Proxy Pattern.  
- When an upgradeable contract is deployed for the first time using this plugin, up to three components are automatically created (depending on which whether the upgrade pattern is UUPS, Transparent, or Beacon Proxy):
    - **The implementation contract**: This contains the actual logic of the contract.
    - **Proxy**: If deploying a new proxy, the plugin handles its creation and links it to the specified implementation contract. However, if a proxy already exists, the plugin facilitates the upgrade process by linking the existing proxy to a new implementation.
    - **ProxyAdmin**: This administrative component manages who can upgrade the proxy exclusively for **transparent proxies** (only Transparent Upgradeable Proxies us a Proxy Admin).
    - **Beacon Proxies:** The Beacon Proxy Pattern does not assign individual admin addresses to proxies. Instead, a single beacon has an owner who updates the implementation for all linked proxies. The plugin automates the beacon and proxies' setup and updates the beacon's implementation.  
- The plugins are designed to work with both Hardhat and Foundry. While the Hardhat environment keeps a detailed log of deployments, Foundry focuses on the use of _reference contracts_ to ensure the safety of upgrades.

## Defining Reference Contracts

Unlike the OpenZeppelin Upgrade plugin for Hardhat, which automatically tracks implementation contracts and their versions via JSON, the Foundry plugin requires developers to explicitly define **Reference Contracts**.

Consider the NatSpec in the contract below:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

/// @custom:oz-upgrades-from MyContractV1
contract MyContractV2 {
  ...
}
```

or

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

/// @custom:oz-upgrades-from contracts/MyContract.sol:MyContractV1
contract MyContractV2 {
  ...
}
```

The natspec is referring to the _reference contract_, and it is what the Foundry plugin uses to get a reference to the reference contract.

**The reference contract is the contract to be upgraded and serves as the baseline to ensure that the new implementation is compatible with the original's state and layout.**

In general, you can think of the "reference contract" as the "previous implementation.

The `validateUpgrade` function from the plugin checks that a new implementation contract is compatible with a reference contract. It requires the `referenceContract` option to be set, or the `@custom:oz-upgrades-from <reference>` annotation to be present on the new contract.

Details on how the `validateUpgrade` function from the Upgrades tool is utilized will be discussed later in this guide.

## Testing Smart Contract Upgrades Locally in Foundry

We now show the steps to deploy and upgrade a Transparent Upgradeable Proxy locally. Later we will show how to do this on a testnet. Here are the steps we will take at a high level:

1. **Deploy the proxy and implementation `ContractA`:**
    - Deploy `ContractA` and initialize it.
    - This step automatically sets up `ContractA`, a `TransparentUpgradeableProxy`, and a `Proxy Admin` Contract.
    - `ContractA` acts as the reference contract for this upgrade process.
2. **Upgrade to `ContractB`:**
    - The tool uses the `upgradeAndCall` method on the proxy to update the contract to `ContractB`.

### Step 1: Setting Up the Environment

Start by creating a new project directory and initializing Foundry. Open your terminal and execute the following commands:

```bash
mkdir rareskills-foundry-upgrades && cd rareskills-foundry-upgrades
forge init
```

Next, we need to prepare the essential files for the project. These include smart contract files, a test file, and a file for dependency mappings.

Execute the command below to create these files in your project directory:

```bash
touch src/ContractA.sol && touch src/ContractB.sol && touch test/Upgrades.t.sol && touch remappings.txt
```

### Step 2: Configure the Project

Once the project is initialized, the next task is to set up the OpenZeppelin libraries necessary for handling upgradeable contracts.

Update the remappings file as follows:

```bash
forge remappings > remappings.txt
```

Run the following commands in the terminal to install the required libraries.

```bash
forge install OpenZeppelin/openzeppelin-contracts-upgradeable --no-commit
forge install OpenZeppelin/openzeppelin-foundry-upgrades --no-commit
```

The `no-commit` flag avoids the hassle of committing the current state of the repo before using the commands above.

Next, we need to ensure that the project knows where to find the OpenZeppelin files. This is done by configuring the `remappings.txt` file, which we previously created.

Open `remappings.txt` and insert the following lines:

```bash
@openzeppelin/contracts/=lib/openzeppelin-contracts-upgradeable/lib/openzeppelin-contracts/contracts/
@openzeppelin/contracts-upgradeable/=lib/openzeppelin-contracts-upgradeable/contracts/
```

If OpenZeppelin remappings are already in the `remapping.txt` file, replace them with the ones above.

Next, delete the `test/Counter.t.sol` test and `src/Counter.sol` that are automatically created:

```bash
rm test/Counter.t.sol
rm src/Counter.sol
```

Finally, open `foundry.toml` and add the following configuration:

```toml
[profile.default]
src = "src"
out = "out"
libs = ["node_modules", "lib"]
build_info = true
extra_output = ["storageLayout"]
ffi = true
ast = true
```

### Step 3: Create the Upgradeable Contracts

Now we will create two smart contracts, `ContractA` and `ContractB`, to demonstrate the upgrade process.

Start with `ContractA.sol`. This contract contains a single public variable, `value`, and a method, `initialize`, which replaces the constructor.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";

contract ContractA is Initializable{
    uint256 public value;

    function initialize(uint256 _setValue) public initializer {
        value = _setValue;
    }
}
```

Next, we will create `ContractB.sol` to show the upgrade path from `ContractA`.

`ContractB` extends the functionality of `ContractA` by adding a method to increment `value`.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";

/// @custom:oz-upgrades-from ContractA
contract ContractB is Initializable {
    uint256 public value;

    function initialize(uint256 _setValue) public initializer {
        value = _setValue;
    }

    function increaseValue() public {
        value += 10;
    }
}
```

By using the `@custom:oz-upgrades-from ContractA` annotation, we specify that `ContractB` is the upgraded version of `ContractA`.

This annotation does not require that `ContractA` and `ContractB` be in the same directory. It identifies `ContractA` by its name, provided it's uniquely defined within the project, otherwise, a fully qualified name is required.

Without this annotation, the plugin will not proceed and will display an error like the following:

```bash
`The contract ${sourceContract.fullyQualifiedName} does not specify what contract it upgrades from. Add the \`@custom:oz-upgrades-from <REFERENCE_CONTRACT>\` annotation to the contract, or include the reference contract name when running the validate command or function.`
```

Although we are preparing both `ContractA` and `ContractB` for this example, this does not mean we need to "anticipate" future upgrades of `ContractA`. We are simply creating `ContractB` in advance for convenience purposes of this example.

### Step 4: Testing Upgradeable Functionality

This step involves compiling the contracts, deploying them as a Transparent Proxy Pattern, performing an upgrade, and verifying if the contract's state updates as expected.

#### Preparing the Test Environment

First, navigate to the **test** folder and insert the following code into the **Upgrades.t.sol** file.

This setup tests the upgradeability from `ContractA` to `ContractB`.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "forge-std/Test.sol";
import "openzeppelin-foundry-upgrades/Upgrades.sol";
import "../src/ContractA.sol";
import "../src/ContractB.sol";

contract UpgradesTest is Test {
    // future code will go here
}
```

The initial test involves two main actions:
1. Deploying `ContractA` with an initial value using a transparent proxy.
2. Upgrading to `ContractB`.
3. Lastly, invoking `increaseValue` to modify the state.

Here's the code for the test. Please read the comments in the code below to understand the workflow:

```solidity
function testTransparent() public {
    // Deploy a transparent proxy with ContractA as the implementation and initialize it with 10
    address proxy = Upgrades.deployTransparentProxy(
        "ContractA.sol",
        msg.sender,
        abi.encodeCall(ContractA.initialize, (10))
    );

    // Get the instance of the contract
    ContractA instance = ContractA(proxy);

    // Get the implementation address of the proxy
    address implAddrV1 = Upgrades.getImplementationAddress(proxy);

    // Get the admin address of the proxy
    address adminAddr = Upgrades.getAdminAddress(proxy);

    // Ensure the admin address is valid
    assertFalse(adminAddr == address(0));

    // Log the initial value
    console.log("----------------------------------");
    console.log("Value before upgrade --> ", instance.value());
    console.log("----------------------------------");

    // Verify initial value is as expected
    assertEq(instance.value(), 10);

    // Upgrade the proxy to ContractB
    Upgrades.upgradeProxy(proxy, "ContractB.sol", "", msg.sender);

    // Get the new implementation address after upgrade
    address implAddrV2 = Upgrades.getImplementationAddress(proxy);

    // Verify admin address remains unchanged
    assertEq(Upgrades.getAdminAddress(proxy), adminAddr);

    // Verify implementation address has changed
    assertFalse(implAddrV1 == implAddrV2);

    // Invoke the increaseValue function separately
    ContractB(address(instance)).increaseValue();

    // Log and verify the updated value
    console.log("----------------------------------");
    console.log("Value after upgrade --> ", instance.value());
    console.log("----------------------------------");
    assertEq(instance.value(), 20);
}
```

As a summary, the tool is doing the following:
- Deploy `ContractA` via a transparent proxy using `Upgrades.deployTransparentProxy("ContractA.sol", msg.sender, abi.encodeCall(ContractA.initialize, (10)));` and `initialize` `ContractA` with a specific value.
- Upgrade the proxy to use `ContractB` using `Upgrades.upgradeProxy(proxy, "ContractB.sol", "", msg.sender);`
- Verify the updated value and consistency of the admin address post-upgrade.

#### Executing the Test

To run the tests, enter the following command in your terminal:

```bash
forge clean && forge test -vvv --ffi
```

You'll see an output similar to the following:

```bash
pari@MacBook-Air rareskills-foundry-upgrades % forge clean && forge test --mt testTransparent -vvv
[⠢] Compiling...
[⠊] Compiling 54 files with 0.8.24
[⠆] Solc 0.8.24 finished in 4.94s
Compiler run successful!

Ran 1 test for test/Upgrades.t.sol:UpgradesTest
[PASS] testTransparent() (gas: 1355057)
Logs:
  ----------------------------------
  Value before upgrade -->  10
  ----------------------------------
  ----------------------------------
  Value after upgrade -->  20
  ----------------------------------

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.43s (1.43s CPU time)

Ran 1 test suite in 5.47s (1.43s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

You may need to run `forge cache clean` and `forge clean` before running the test a second time.

## An example with a Beacon Proxy

We now demonstrate how to use this plugin with the Beacon Proxy Pattern. We will use the same implementation contracts `ContractA` and `ContractB` from the previous example.

### Test Overview
1. Deploy `ContractA` as the initial implementation for a Beacon.
2. **Create two beacon proxies**, each initialized with different values. Remember, in the beacon proxy pattern, multiple proxies point to a single implementation, but they have their own separate state.
3. **Validate the new implementation (`ContractB`)** against the original implementation (`ContractA`) using the **`validateUpgrade`** function.
4. **Upgrade the beacon's implementation to** `ContractB`, upgrading both proxies simultaneously.
5. **Test new functionalities** on both proxies to ensure the upgrade applies the changes as expected.

Add this function to the `Upgrades.t.sol` file to conduct the test. The comments explain the workflow:

```solidity
import {IBeacon} from "@openzeppelin/contracts/proxy/beacon/IBeacon.sol";
import {Options} from "openzeppelin-foundry-upgrades/Upgrades.sol";

function testBeacon() public {
    // Deploy a beacon with ContractA as the initial implementation
    address beacon = Upgrades.deployBeacon("ContractA.sol", msg.sender);

    // Get the initial implementation address of the beacon
    address implAddrV1 = IBeacon(beacon).implementation();

    // Deploy the first beacon proxy and initialize it
    address proxy1 = Upgrades.deployBeaconProxy(beacon, abi.encodeCall(ContractA.initialize, 15));
    ContractA instance1 = ContractA(proxy1);

    // Deploy the second beacon proxy and initialize it
    address proxy2 = Upgrades.deployBeaconProxy(beacon, abi.encodeCall(ContractA.initialize, 20));
    ContractA instance2 = ContractA(proxy2);

    // Check if both proxies point to the same beacon
    assertEq(Upgrades.getBeaconAddress(proxy1), beacon);
    assertEq(Upgrades.getBeaconAddress(proxy2), beacon);

    console.log("----------------------------------");
    console.log("Value before upgrade in Proxy 1 --> ", instance1.value());
    console.log("Value before upgrade in Proxy 2 --> ", instance2.value());
    console.log("----------------------------------");

    // Validate the new implementation before upgrading
    Options memory opts;
    opts.referenceContract = "ContractA.sol";
    Upgrades.validateUpgrade("ContractB.sol", opts);

    // Upgrade the beacon to use ContractB
    Upgrades.upgradeBeacon(beacon, "ContractB.sol", msg.sender);

    // Get the new implementation address of the beacon after upgrade
    address implAddrV2 = IBeacon(beacon).implementation();

    // Activate the increaseValue function in both proxies
    ContractB(address(instance1)).increaseValue();
    ContractB(address(instance2)).increaseValue();

    console.log("----------------------------------");
    console.log("Value after upgrade in Proxy 1 --> ", instance1.value());
    console.log("Value after upgrade in Proxy 2 --> ", instance2.value());
    console.log("----------------------------------");

    // Check if the values have been correctly increased
    assertEq(instance1.value(), 25);
    assertEq(instance2.value(), 30);

    // Check if the implementation address has changed
    assertFalse(implAddrV1 == implAddrV2);
}
```

## Supported functionality of the Upgrades plugin

The plugin supports more functionality than what we used in the two examples above. We provide an overview of the other functions in `Upgrades` below:

### **Initial Deployment Functions:**

These functions are primarily used for the deployment of the initial versions of contracts and for setting up their proxy structures:
- **`deployUUPSProxy(*string contractName, bytes initializerData, struct Options opts*)`:** Deploys a UUPS proxy using the given contract as the implementation.
- **`deployTransparentProxy(*string contractName, address initialOwner, bytes initializerData, struct Options opts*)`:** Deploys a transparent proxy using the given contract as the implementation.
- **`deployBeaconProxy(*address beacon, bytes data, struct Options opts*)`:** Deploys a beacon proxy using the given beacon and call data.
- **`deployBeacon(*string contractName, address initialOwner, struct Options opts*)`:** Deploys an upgradeable beacon using the given contract as the implementation.

### **Validation and Implementation Functions:**

These functions are used to ensure the compatibility and safety of implementations during upgrades:
- **`validateImplementation(*string contractName, struct Options opts*)`:** Validates an implementation contract, but does not deploy it.
- **`deployImplementation(*string contractName, struct Options opts*)`:** Validates and deploys an implementation contract, and returns its address.
- **`validateUpgrade(*string contractName, struct Options opts*)`:** Validates a new implementation contract in comparison with a reference contract, but does not deploy it.
- **`prepareUpgrade(*string contractName, struct Options opts*)`:** Validates a new implementation contract in comparison with a reference contract, deploys the new implementation contract and returns its address.

### **Upgrade Functions:**

These functions are used for managing and implementing upgrades to already deployed contracts:
- **`upgradeProxy(*address proxy, string contractName, bytes data, struct Options opts*)`:** Upgrades a proxy to a new implementation contract. This function calls `validateUpgrade()` under the hood, so it will fail if the validation doesn't succeed. If the user wishes to bypass some of the validations, this can be configured in the opts argument.
- **`upgradeBeacon(*address beacon, string contractName, struct Options opts*)`:** Upgrades a beacon to a new implementation contract.

### **Other Functions:**

- **`getAdminAddress(*address proxy*)`:** Gets the admin address of a transparent proxy from its ERC1967 admin storage slot.

## Deploying and Verifying on Sepolia Testnet

This section guides you through deploying `ContractA`, upgrading it to `ContractB` using a Transparent Proxy and verifying the contracts on Sepolia Explorer.

### Step 1: Add Environment Variables

First, set up the necessary configurations for deployment to the testnet:
- Create an `.env` file to store sensitive data.
- Include the `.env` file in your `.gitignore` to prevent exposure of private keys or other sensitive information on version-controlled platforms like Github.
- Populate the `.env` file with your specific data:

```bash
SEPOLIA_RPC_URL=your-sepolia-endpoint
PRIVATE_KEY=your-private-key
ETHERSCAN_API_KEY=your-etherscan-api
SENDER=your-EOA-address
```

Configuration details:
- **`SEPOLIA_RPC_URL`**: The RPC URL for connecting to the Sepolia network.
- **`PRIVATE_KEY`**: Your private key, used for signing transactions. Make sure this wallet has Ether to pay the gas cost for deployment.
- **`ETHERSCAN_API_KEY`**: Your Etherscan API key, needed for contract verification.
- **`SENDER`**: The Ethereum address from which the deployment transactions will be initiated.

### Step 2: Configure the `foundry.toml` file

Update the `foundry.toml` file to include settings for the Sepolia Testnet and Etherscan verification:

```toml
[etherscan]
sepolia = { key = "${ETHERSCAN_API_KEY}" }

[rpc_endpoints]
sepolia = "${SEPOLIA_RPC_URL}"
```

This configuration accomplishes two tasks:
- **Etherscan Configuration:** Links the Etherscan API Key with the Sepolia network for post-deployment contract verification.
- **RPC Endpoints:** Specifies the RPC URL for the Sepolia network.

### Step 3: Scripting Deployment and Upgrade

Navigate to the **script** folder and open the **Upgrades.s.sol** file to insert the necessary code for deployment and upgrade tasks.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import {Script} from "forge-std/Script.sol";

import {ContractA} from '../src/ContractA.sol';
import {ContractB} from '../src/ContractB.sol';

import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";

contract UpgradesScript is Script {
	  function setUp() public {}

    function run() public {
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
        vm.startBroadcast(deployerPrivateKey);

        // Deploy `ContractA` as a transparent proxy using the Upgrades Plugin
        address transparentProxy = Upgrades.deployTransparentProxy(
            "ContractA.sol",
            msg.sender,
            abi.encodeCall(ContractA.initialize, 10)
        );

    }
}
```

Behind the scenes, this will deploy the Proxy, the ProxyAdmin, and `ContractA` for us.

The `deployTransparentProxy` function takes 3 parameters:
- `contractName`(`string`): Name of the contract for implementation, such as "**MyContract.sol**", "**MyContract.sol:MyContract**", or a relative artifact path.
- `initialOwner(address)`: Address set as the owner of the ProxyAdmin contract, which is deployed automatically with the proxy.
- `initializerData(bytes)`: Encoded call data for the initializer function to be executed during proxy creation; leave empty if no initialization is needed.

The function returns the address of the deployed proxy.

#### Executing the Script

Use the following command to deploy and verify the upgrade on the Sepolia network.

```bash
forge clean && forge script script/UpgradesScript.s.sol --rpc-url sepolia --private-key $PRIVATE_KEY --broadcast --verify --sender $SENDER
```

This command cleans previous builds, executes the script, and verifies the contracts on [Sepolia Explorer](https://sepolia.etherscan.io/).

Upon execution, you should expect output indicating the successful deployment and verification of the contracts:

```bash
ONCHAIN EXECUTION COMPLETE & SUCCESSFUL.
Total Paid: 0.00356888630073128 ETH (862220 gas * avg 4.139182924 gwei)
##
Start verification for (3) contracts
Start verifying contract `0x427186c574B5fA11cB9d796871861EF87c74Ad37` deployed on sepolia

Submitting verification for [src/ContractA.sol:ContractA] 0x427186c574B5fA11cB9d796871861EF87c74Ad37.
Submitted contract for verification:
        Response: `OK`
        GUID: `hf2dplvhjmjpj3nixun3kupamtsgmbacngfeygpm9p34mbzb3g`
        URL: https://sepolia.etherscan.io/address/0x427186c574b5fa11cb9d796871861ef87c74ad37
Contract verification status:
Response: `NOTOK`
Details: `Pending in queue`
Contract verification status:
Response: `OK`
Details: `Pass - Verified`
Contract successfully verified
Start verifying contract `0xbA58580452Bc758C9a044584F6CEa468e5569a13` deployed on sepolia

Submitting verification for [lib/openzeppelin-contracts-upgradeable/lib/openzeppelin-contracts/contracts/proxy/transparent/TransparentUpgradeableProxy.sol:TransparentUpgradeableProxy] 0xbA58580452Bc758C9a044584F6CEa468e5569a13.
Submitted contract for verification:
        Response: `OK`
        GUID: `biaqcdgrhwjfu8d7b3n9jg6btmsjpktgd61rdx42n7ptttuvre`
        URL: https://sepolia.etherscan.io/address/0xba58580452bc758c9a044584f6cea468e5569a13
Contract verification status:
Response: `NOTOK`
Details: `Pending in queue`
Contract verification status:
Response: `OK`
Details: `Pass - Verified`
Contract successfully verified
Start verifying contract `0x8bB6A51C24ad9b6bA276c2bf0380e5E8Ce31E866` deployed on sepolia

Submitting verification for [lib/openzeppelin-contracts-upgradeable/lib/openzeppelin-contracts/contracts/proxy/transparent/ProxyAdmin.sol:ProxyAdmin] 0x8bB6A51C24ad9b6bA276c2bf0380e5E8Ce31E866.
Submitted contract for verification:
        Response: `OK`
        GUID: `rfnvjnxa8j2rqxxgwtnx9if17rdlyky2nk9eixrmzjbp5pn4gy`
        URL: https://sepolia.etherscan.io/address/0x8bb6a51c24ad9b6ba276c2bf0380e5e8ce31e866
Contract verification status:
Response: `NOTOK`
Details: `Pending in queue`
Contract verification status:
Response: `NOTOK`
Details: `Pending in queue`
Contract verification status:
Response: `NOTOK`
Details: `Already Verified`
Contract source code already verified
All (3) contracts were verified!

Transactions saved to: /Users/nest/rareskills/rareskills-foundry-upgrades/broadcast/UpgradesScript.s.sol/11155111/run-latest.json

Sensitive values saved to: /Users/nest/rareskills/rareskills-foundry-upgrades/cache/UpgradesScript.s.sol/11155111/run-latest.json
```

The `deployTransparentProxy` command executed during this script deployed `ContractA` and a `Transparent Upgradeable Proxy` along with a `Proxy Admin Contract`.

These transactions are viewable on Sepolia Explorer:
- **ContractA Deployment**: [https://sepolia.etherscan.io/tx/0x9dba6d8293629cb9557500d8645659de3127e75abcfa705b06e6cf379092a10e](https://sepolia.etherscan.io/tx/0x9dba6d8293629cb9557500d8645659de3127e75abcfa705b06e6cf379092a10e)
- **Transparent upgradeable Proxy**: [https://sepolia.etherscan.io/tx/0x9dba6d8293629cb9557500d8645659de3127e75abcfa705b06e6cf379092a10e](https://sepolia.etherscan.io/tx/0x9dba6d8293629cb9557500d8645659de3127e75abcfa705b06e6cf379092a10e)
- **Proxy Admin Contract**: [https://sepolia.etherscan.io/address/0xba58580452bc758c9a044584f6cea468e5569a13#code#F6#L1](https://sepolia.etherscan.io/address/0xba58580452bc758c9a044584f6cea468e5569a13#code)

#### Upgrading the contract

Before upgrading to `ContractB`, we'll validate the new implementation against the reference contract, `ContractA`, using the plugin's `validateUpgrade` function.

Once the validation is confirmed, next we will proceed with the upgrade using `upgradeProxy` function. Please read the comments in the code below:

```solidity
import {Options} from "openzeppelin-foundry-upgrades/Upgrades.sol";

function run() public {
    uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
    vm.startBroadcast(deployerPrivateKey);

    // Specifying the address of the existing transparent proxy
    address transparentProxy = 'your-transparent-proxy-address';

     // Setting options for validating the upgrade
    Options memory opts;
    opts.referenceContract = "ContractA.sol";

    // Validating the compatibility of the upgrade
    Upgrades.validateUpgrade("ContractB.sol", opts);

    // Upgrading to ContractB and attempting to increase the value
    Upgrades.upgradeProxy(transparentProxy, "ContractB.sol", abi.encodeCall(ContractB.increaseValue, ()));
}
```

If the new contract implementation isn't compatible with the reference contract, it throws the following error:

```bash
revert: Upgrade safety validation failed:
```

Again. Use the same script to deploy and upgrade `ContractB` and verify it on the Explorer.

![Etherscan foundry upgrade transaction](https://static.wixstatic.com/media/706568_48fef342bf7d49afa2d4c259975b9ef4~mv2.png/v1/fill/w_740,h_57,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_48fef342bf7d49afa2d4c259975b9ef4~mv2.png)

The upgrade transaction can be seen [here](https://sepolia.etherscan.io/tx/0x6409cb621386b776e75833c98fed6da9261e5dc52fcdddaed7142f751380f41c).

## How OpenZeppelin Foundry Plugin Helps Against Upgrade Issues

This section lists the potential security issues that are specify to proxy upgrades and how the tool guards against them.

### 1. Immutable Variables

In Solidity, the `immutable` keyword allows variables to be permanently set during contract creation by embedding their values directly into the bytecode. This forces future deployments to use the exact same arguments for the constructor, so that future deployments have the same bytecode. Since this is hard to track, the plugin discourages the use of immutable variables.

To allow immutable variables in upgradeable contracts, developers may bypass this safety check using the `@custom:oz-upgrades-unsafe-allow` annotation, as shown below.

```solidity
contract ImmutableVar {
    /// @custom:oz-upgrades-unsafe-allow state-variable-immutable
    uint256 public immutable a;

    /// @custom:oz-upgrades-unsafe-allow constructor
    constructor(uint256 _a) {
        a = _a;
    }
}
```

This allows for the use of immutable variables but demands careful management to maintain deployment consistency.

### 2. Storage Layout

Maintaining a consistent storage layout is essential when updating smart contracts. Any changes in the order or type of state variables can lead to data corruption.

This means that if you have an initial contract that looks like this:

```solidity
contract MyContract {
    uint256 private x;
    string private y;
}
```

Then you cannot change the type of a variable:

```solidity
contract MyContract {
    string private x; // uint256 became a string
    string private y;
}
```

Or change the order in which they are declared:

```solidity
contract MyContract {
    string private y; // x and y switched places
    uint256 private x;
}
```

If you need to introduce a new variable, make sure you always do so at the end:

```solidity
contract MyContract {
    uint256 private x;
    string private y;
    bytes private z;
}
```

Using our `ContractA` and `ContractB` examples from the beginning of this article, suppose we had incorrectly inserted a storage variable into `ContractB` as follows:

![Incorrectly inserted a storage variable at upgrade at ContractB](https://static.wixstatic.com/media/706568_a341fd7ad44e4c1b824821ccc8373ee8~mv2.png/v1/fill/w_740,h_205,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_a341fd7ad44e4c1b824821ccc8373ee8~mv2.png)

We would get the following error from the OpenZeppelin plugin (we are using the same test as the first example with the Transparent Upgradeable Proxy):

![error from the OpenZeppelin plugin for incorrect storage variable](https://static.wixstatic.com/media/706568_9b35832c0fcc44b48c0e204410fc8a16~mv2.png/v1/fill/w_740,h_176,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_9b35832c0fcc44b48c0e204410fc8a16~mv2.png)

### 3. Validating `__gap` are used properly

As discussed in our article in [Storage Namespaces](https://rareskills.io/post/erc-7201), using the **`__gap` variable is a strategy to prevent parent contracts from shifting the storage variables of child contracts when a new storage variable is inserted. To use the `__gap`** variable properly, the size of the gap must be reduced when a new variable is inserted. In the following example, the dev inserted a new storage variable and forgot to reduce the size of the **`__gap`**:

![Gap variable not reduced in  the storage NameSpaces](https://static.wixstatic.com/media/706568_19b8fa14d316465991c50b2b1fc13e20~mv2.png/v1/fill/w_740,h_191,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_19b8fa14d316465991c50b2b1fc13e20~mv2.png)

and the tool catches this:

![Error from OpenZeppelin for Gap Variable not reduced](https://static.wixstatic.com/media/706568_5319ac3af1a145ba88a00790f54eed06~mv2.png/v1/fill/w_740,h_174,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_5319ac3af1a145ba88a00790f54eed06~mv2.png)

The OpenZeppelin tool does not enforce the use of **`__gap`**. Instead, _if_ a **gap variable is present, the tool checks that the upgrade respects how the `__gap`** variable is intended to be used, i.e. maintaining storage variable alignment.

A more robust alternative to the **`__gap`** strategy is the use of Named Storage Layouts, as discussed in our tutorial on [Storage Namespaces](https://rareskills.io/post/erc-7201), and we provide an example of that in the next section.

### 4. Validating ERC-7201 is followed properly

Using our running example of `ContractA` and `ContractB` from the beginning, let's alter the contract to use ERC-7201.

Note the following changes:
- Instead of being a public variable, `value` is now a public function
- The underlying storage for `value` has been moved inside the `MyStorage` struct
- The setter functions must now interact with the `MyStorage` struct (note the struct is used for grouping the variables, it is never actually initialized).

Here is `ContractA` adjusted to follow ERC-7201:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";

contract ContractA is Initializable {

    /// @custom:storage-location erc7201:ContractA.storage.MyStorage
    struct MyStorage {
        uint256 value;
    }

    // keccak256(abi.encode(uint256(keccak256("ContractA.storage.MyStorage")) - 1)) & ~bytes32(uint256(0xff));
    bytes32 private constant MyStorageLocation = 0xd255ccbed1486709ef10c220c9b584c9ad5cacd00961bdfc2156c2c7f2e4fc00;

    function _getMyStorage() private pure returns (MyStorage storage $) {
        assembly {
            $.slot := MyStorageLocation
        }
    }

    function value() public view returns (uint256) {
        MyStorage storage $ = _getMyStorage();
        return $.value;
    }

    function initialize(uint256 _setValue) public initializer {
        MyStorage storage $ = _getMyStorage();
        $.value = _setValue;
    }
}
```

and `ContractB`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;
import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";

/// @custom:oz-upgrades-from ContractA
contract ContractB is Initializable {

    /// @custom:storage-location erc7201:ContractA.storage.MyStorage
    struct MyStorage {
        uint256 value;
    }

    // keccak256(abi.encode(uint256(keccak256("ContractA.storage.MyStorage")) - 1)) & ~bytes32(uint256(0xff));
    bytes32 private constant MyStorageLocation = 0xd255ccbed1486709ef10c220c9b584c9ad5cacd00961bdfc2156c2c7f2e4fc00;

    function _getMyStorage() private pure returns (MyStorage storage $) {
        assembly {
            $.slot := MyStorageLocation
        }
    }

    function value() public view returns (uint256) {
        MyStorage storage $ = _getMyStorage();
        return $.value;
    }

    function initialize(uint256 _setValue) public initializer {
        MyStorage storage $ = _getMyStorage();
        $.value = _setValue;
    }

    function increaseValue() public {
        MyStorage storage $ = _getMyStorage();
        $.value += 10;
    }
}
```

Let's say we mess up the `MyStorage` struct in `ContractB` by inserting a variable into the struct somewhere other than at the end:

```solidity
/// @custom:storage-location erc7201:ContractA.storage.MyStorage
struct MyStorage {
    uint256 badInsert; // this should not be here, it should be at the end
    uint256 value;
}
```

With that change, the tool will present the following error:

![Error from OpenZeppelin for incorrect struct MyStorage](https://static.wixstatic.com/media/706568_762c6c0de16a4bd8963aab1ae5e6c9b6~mv2.png/v1/fill/w_740,h_163,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_762c6c0de16a4bd8963aab1ae5e6c9b6~mv2.png)

Another issue the tool will prevent is renaming the namespace slot. Suppose we change the annotation above the struct as follows:

```solidity
/// @custom:storage-location erc7201:ContractA.storage.MyStorage
/// @custom:storage-location erc7201:ContractA.storage.MyStorage2
```

We now get the following error:

![OpenZeppelin Error for renaming  the namespace slot](https://static.wixstatic.com/media/706568_25fdb15533374f5fa4c166c02598244d~mv2.png/v1/fill/w_740,h_114,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_25fdb15533374f5fa4c166c02598244d~mv2.png)

Namespaces are not supposed to be removed between upgrades.

## Failure to call parent initializers cannot be automatically caught

In the following example, both `ContractA` and `ContractB` inherit from `Base`. However, neither of them call the parent's initializer, which is a bug. Since understanding if a function serves as an initializer requires a semantic interpretation, the tool cannot automatically catch this issue. The dev or auditor must manually check that all initializers are properly called (usually this sort of issue can easily be caught with unit tests if variables are initialized to a non-zero value).

The following code will not trigger any issues with the tool even though the `__Base_init` function is not called.

![Invalid contract code to call parent initializers](https://static.wixstatic.com/media/706568_0935df96a8ee4559a68a722f4e398632~mv2.png/v1/fill/w_740,h_783,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/706568_0935df96a8ee4559a68a722f4e398632~mv2.png)

## Conclusion

The article detailed how to use the OpenZeppelin upgrade Foundry plugin, how the plugin can be used for unit tests and Foundry scripts, and how it simplifies the multistep process of deployment and upgrades. We showed how to set up the environment for deployment, and gave some examples of how the tool can automatically catch various mistakes.

### Authorship and Acknowledgements

This article was written by [Pari Tomar](https://www.linkedin.com/in/tomarpari90/) in collaboration with RareSkills.

We would like to thank [Eric Lau](https://www.linkedin.com/in/ericglau/?originalSubdomain=ca) from OpenZeppelin for his helpful comments on a draft of this article.

*Originally Published August, 28, 2024*
