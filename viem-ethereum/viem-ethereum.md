# Viem React Example: Transfer, Mint, and View Blockchain State

In this tutorial, we’ll build a fully functional Dapp with the Viem TypeScript library + React (Next.js). We’ll cover the necessary steps to connect your wallet, transfer crypto, interact with smart contracts (e.g. mint NFTs) and query the blockchain.

Viem is a TypeScript alternative to existing low-level Ethereum interfaces like web3.js and Ethers.js. It supports browser native [BigInt](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/BigInt) and automatically [infer types](https://viem.sh/docs/typescript.html) from ABIs and EIP-712. It has a 35kb bundle size, tree-shakable design to minimize the final bundle, 99.8% test coverage. Check out their [benchmarks and full documentation](https://viem.sh/docs/introduction.html).

We’ve structured this tutorial with concise explanations delivered via in-line code comments. Simply copy-paste the codes and read the explanations.

Here’s an outline of the tutorial:

-   Viem Clarifying Terminologies: Client | Transport | Chain
-   Getting Started: Set up Client & Transport with React + Viem
-   Part 1: Connect to Web3 Wallet with React + Viem
-   Part 2: Transferring Crypto with React + Viem
-   Part 3: Minting an NFT with React + Viem

A showcase of what you’ll build:

![viem TypeScript demo dapp](https://static.wixstatic.com/media/935a00_f58f30a2b40a4d31aab773b095694c38~mv2.png/v1/fill/w_666,h_439,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_f58f30a2b40a4d31aab773b095694c38~mv2.png)

[Full code in Git Repository](https://github.com/AymericRT/viemtutorial.git)

## Viem Clarifying Terminologies: Client | Transport | Chain

Viem has three fundamental concepts: **Client**, **Transport** and **Chain**

-   **Client** in viem, is similar to Ethers.js Provider. It provides the TypeScript functions for doing common actions on Ethereum. Depending on the action, it will fall under one of three types of clients.
    -   **Public Client** is an interface to “public” JSON RPC API methods, e.g., retrieving block numbers, querying account balances, accessing “view” functions on smart contracts, and other read-only, non-state-changing operations. These functions are referred to as Public Actions.
    -   **Wallet Client** is an interface to interact with Ethereum Accounts, e.g., sending transactions, signing messages, requesting addresses, switching chains, and operations requiring the user’s permission. For example, minting an NFT is state-changing, so this would be done under the wallet client. These functions are referred to as Wallet Actions.
    -   **Test Client** is used to create simulated transactions for testing. This would typically be used in unit tests.
-   **Transport** is instantiated with the **Client**, it represents the intermediary layer for executing requests. There are three types of Transport:
    -   **HTTP** Transport, utilizing HTTP JSON-RPC;
    -   **WebSocket** Transport for real-time connections via WebSocket JSON-RPC;
    -   **Custom Transport**, which handle requests via EIP-1193 request method;
    -   **Fallback** allows you to specify multiple transports in a list. If one fails, it moves down the list to find a functional one. An example will be provided later.
-   **Chain** refers to the EVM-compatible chain for establishing a connection, they are identified through the chain object (identified by chain id). Only one chain can be instantiated along with a client. We can use the provided viem chain library, e.g., polygon, eth mainnet or build your own manually.

### Public Client

This is how to declare a **Public Client**.

```js
import { createPublicClient, http } from 'viem'
import { mainnet } from 'viem/chains'

const publicClient = createPublicClient({
  chain: mainnet,
  transport: http()
})
```

This is how you utilize Public actions.

```js
const balance = await publicClient.getBalance({
  address: '0xA0Cf798816D4b9b9866b5330EEa46a18382f251e',
})

const block = await publicClient.getBlockNumber()
```

These are the available Public Actions:

-   getChainId
-   getGasPrice
-   signMessage
-   verifyMessage
-   getTransactionReceipt
-   More on [Public Action Documentation](https://viem.sh/docs/actions/public/introduction.html)

Viem has a feature called Optimization Public Client, supports [eth\_call Aggregation](https://viem.sh/docs/clients/public.html) for improved performance by sending batch requests. This is very well documented.

### Wallet Client

This is how to establish a Wallet Client:

```js
import { createWalletClient, custom } from 'viem'
import { mainnet } from 'viem/chains'

const walletClient = createWalletClient({
    chain: mainnet,transport: custom(window.ethereum)
})
```

This is how to utilize Wallet Actions:

```js
// Gets the user address
const [address] = await walletClient.getAddresses()

// Sends a transaction
const hash = await walletClient.sendTransaction({
    account: address,
    to: '0xa5cc3c03994DB5b0d9A5eEdD10CabaB0813678AC',
    value: parseEther('0.001')
})
```

Available wallet functions:

-   requestAddresses ( Wallets like MetaMask may need a user’s to requestAddresses first )
-   switchChain
-   signMessage
-   getPermissions
-   sendTransaction
-   More on [Wallet Action Documentation](https://viem.sh/docs/actions/wallet/introduction.html)

We’ll demonstrate how to use **sendTransaction** and **getAddresses** in this tutorial.

**Optional**

You are able to extend Wallet Client with [**Public Actions**](https://viem.sh/docs/actions/public/introduction.html)**.** This helps you avoid handling several clients. The following code snippet extends the wallet client with public actions.

```js
import { createWalletClient, http, publicActions } from 'viem'
import { mainnet } from 'viem/chains'

const extendedClient = createWalletClient({
  chain: mainnet,
  transport: http()
}).extend(publicActions)

// Public Action
const block = await extendedClient.getBlockNumber()
// Wallet Action
const [address] = await extendedClient.getAddresses();
```

### Test Client

Test Client provides an interface to shadow accounts, mining blocks and impersonate transactions through a local test Node.js such as Anvil or Hardhat. We won’t be discussing this in detail but you can read more on [Test Client Documentation](https://viem.sh/docs/clients/test.html).

### Transport

We can only pass one means of transport (the protocol we will use to connect to the blockchain) at a time, here’s an example of how each of the transport can be used.

**HTTP**

```js
import { createPublicClient, http } from 'viem'
import { mainnet } from 'viem/chains'
const transport = http('https://eth-mainnet.g.alchemy.com/v2/...')

const client = createPublicClient({
  chain: mainnet,
  transport,
})
```

The transport will fallback to a public RPC URL if no **url** is provided. It is recommended to pass an authenticated RPC URL to minimize issues with rate-limiting.

**WebSocket**

```js
import { createPublicClient, webSocket } from 'viem'
import { mainnet } from 'viem/chains'

const transport = webSocket('wss://eth-mainnet.g.alchemy.com/v2/...')
const client = createPublicClient({
  chain: mainnet,
  transport,
})
```

Transport will fallback to public RPC URL for the same reasons above.

**Note** how in both examples above, the transport is defined by the kind of URL specified for transport. The first url is an HTTPS url, the second is a WSS url.

**Custom (EIP-1193) (We will be using this)**

This transport is used to integrate with injected wallets that provide an EIP-1193 provider such as WalletConnect, Coinbase SDK and MetaMask.

```js
import { createWalletClient, custom } from 'viem'
import { mainnet } from 'viem/chains'

const client = createWalletClient({
  chain: mainnet,
  transport: custom(window.ethereum)
})
```

**Fallback**

This transport takes in multiple Transports. If a transport fails, it will resort to the next transport method given. In the following example, if Alchemy fails, it will fallback to Infura.

```js
import { createPublicClient, fallback, http } from 'viem'
import { mainnet } from 'viem/chains'

const alchemy = http('https://eth-mainnet.g.alchemy.com/v2/...')

const infura = http('https://mainnet.infura.io/v3/...')

const client = createPublicClient({
  chain: mainnet,
  transport: fallback([alchemy, infura]),
})
```

### Chains

Viem provides popular EVM-Compatible chains through the viem/chains library such as: Polygon, Optimism, Avalanche, and more on [Viem Chains Documentation](https://viem.sh/docs/clients/chains.html).

You can switch chains by passing them as the argument, eg. polygonMumbai.

```js
import { createPublicClient, http } from 'viem'
import { polygonMumbai } from 'viem/chains'

const client = createPublicClient({
    chain: polygonMumbai,
    transport: http(),
})
```

You can also build your own chain object that inherits the Chain type (Credits: [Viem.sh](https://viem.sh/docs/clients/chains.html))

```js
import { Chain } from 'viem'

export const avalanche = {
    id: 43_114,
    name: 'Avalanche',
    network: 'avalanche',
    nativeCurrency: {
        decimals: 18,
        name: 'Avalanche',
        symbol: 'AVAX',
    },
    rpcUrls: {
        public: {
            https: ['https://api.avax.network/ext/bc/C/rpc']
        },
        default: {
            https: ['https://api.avax.network/ext/bc/C/rpc']
        },
    },
    blockExplorers: {
        etherscan: {
            name: 'SnowTrace',
            url: 'https://snowtrace.io'
        },
            default: {
                name: 'SnowTrace',
                url: 'https://snowtrace.io'
            },
        },
        contracts: {multicall3: {
            address: '0xca11bde05977b3631167028862be2a173976ca11',
            blockCreated: 11_907_934,
        },
    },
} as const satisfies Chain
```

Remember, only one chain can be assigned to a client at a time.

## Getting Started: Set up Client & Transport with React + Viem

For simplicity we recommend using **polygonMumbai** or **Sepolia** as your test network.

### Step 1: Create a Next.js Project & Install Viem

First create your **Next.js** project with

```bash
npx create-next-app@latest myapp
```

Check \[yes\] for the following:

-   **TypeScript**
-   **ESLint**
-   **Tailwind**
-   **App Router** (preferably)

Open your project in **vscode**.

Install viem with one the following command:

```bash
npm i viem
pnpm i viem
yarn add viem
```

### Step 2: Set up Client and Transport

In the app directory, create two new files:

-   **client.ts**
-   **walletButton.tsx**

Your app directory should look like this:

app
├── **client.ts**
├── globals.css
├── layout.tsx
├── page.tsx
└── **walletButton.tsx**

### Client.ts

We’ll initialize the Client & Transport in a separate TypeScript file. Go ahead and copy paste the following codes into **client.ts**.

```ts
// client.ts
import { createWalletClient, createPublicClient, custom, http } from "viem";
import { polygonMumbai, mainnet } from "viem/chains";
import "viem/window";

// Instantiate Public Client
const publicClient = createPublicClient({
  chain: mainnet,
  transport: http(),
});

// Instantiate Wallet Clientconst walletClient = createWalletClient({
    chain: polygonMumbai,
    transport: custom(window.ethereum),
});
```

This will inevitably throw a type error, **window.ethereum** might be **undefined** since some browsers like Safari does not support the window.ethereum object.

We can handle the error by checking if window.ethereum is present or undefined.

```ts!
// client.ts

import { createWalletClient, createPublicClient, custom, http } from "viem";
import { polygonMumbai } from "viem/chains";
import "viem/window";

export function ConnectWalletClient() {
    // Check for window.ethereum
    let transport;
    if (window.ethereum) {
        transport = custom(window.ethereum);
    } else {
        const errorMessage ="MetaMask or another web3 wallet is not installed. Please install one to proceed.";
        throw new Error(errorMessage);
    }

    // Declare a Wallet Client
    const walletClient = createWalletClient({
        chain: polygonMumbai,
        transport: transport,
    });

    return walletClient;
}

export function ConnectPublicClient() {
    // Check for window.ethereum
    let transport;
    if (window.ethereum) {
        transport = custom(window.ethereum);
    } else {
        const errorMessage ="MetaMask or another web3 wallet is not installed. Please install one to proceed.";
        throw new Error(errorMessage);
    }

    // Declare a Public Client
    const publicClient = createPublicClient({
        chain: polygonMumbai,
        transport: transport,
    });

    return publicClient;
}
```

This will still allow you to open the website in a browser that does not support window.ethereum.

It is recommended to keep the chains for the **walletClient** and **publicClient** consistent or else you might run into incompatible chain errors.

## Part 1: Connect to Web3 Wallet with React + Viem

This section demonstrates how the viem Client connects to your web3 wallet.

### Step 2: Create a Button that Connects to Web3 Wallet

### walletButton.tsx

We’ll now create a client component, that handles the connection logic to your web3 wallet.

The button instantiates a walletClient and requests the user’s wallet address, if the wallet isn’t already connected it will prompt it to, and finally output its address.

There is a lot of code here, but focus on the handleClick() function.

```tsx!
// walletButton.tsx
"use client";
import { useState } from "react";
import { ConnectWalletClient, ConnectPublicClient } from "./client";

export default function WalletButton() {
    //State variables for address & balance
    const [address, setAddress] = useState<string | null>(null);
    const [balance, setBalance] = useState<BigInt>(BigInt(0));
    // Function requests connection and retrieves the address of wallet
    // Then it retrieves the balance of the address
    // Finally it updates the value for address & balance variable
    async function handleClick() {
        try {
            // Instantiate a Wallet & Public Client
            const walletClient = ConnectWalletClient();
            const publicClient = ConnectPublicClient();

            // Performs Wallet Action to retrieve wallet address
            const [address] = await walletClient.getAddresses();

            // Performs Public Action to retrieve address balance
            const balance = await publicClient.getBalance({ address });
            // Update values for address & balance state variable
            setAddress(address);
            setBalance(balance);
        } catch (error) {
            // Error handling
            alert(`Transaction failed: ${error}`);
        }
    }
// Unimportant Section Below / Nice to Have UI
    return (
        <>
            <Status address={address} balance={balance} />
            <button className="px-8 py-2 rounded-md bg-[#1e2124] flex flex-row items-center justify-center border border-[#1e2124] hover:border hover:border-indigo-600 shadow-md shadow-indigo-500/10"
             onClick={handleClick}
            >
            <img     src="https://upload.wikimedia.org/wikipedia/commons/3/36/MetaMask_Fox.svg" alt="MetaMask Fox" style={{ width: "25px", height: "25px" }} />
            <h1 className="mx-auto">Connect Wallet</h1>
            </button></>);}

// Displays the wallet address once it’s successfuly connected
// You do not have to read it, it's just frontend stuff

function Status({
  address,
  balance,}: {
  address: string | null;
  balance: BigInt;
}) {
    if (!address) {
        return (
            <div className="flex items-center">
                <div className="border bg-red-600 border-red-600 rounded-full w-1.5 h-1.5 mr-2">
                </div>
                <div>Disconnected</div>
            </div>);
    }
    return (
        <div className="flex items-center w-full">
            <div className="border bg-green-500 border-green-500 rounded-full w-1.5 h-1.5 mr-2"></div>
            <div className="text-xs md:text-xs">{address} <br /> Balance: {balance.toString()}</div>
            </div>
    );
}
```

### Step 3: Insert walletButton component

###

### page.tsx

What’s left is to design the main page and import the WalletButton component. We’ve commented out some code you will add later.

```tsx!
import WalletButton from "./walletButton";
// import MintButton from "./mintButton";
// import SendButton from "./sendButton";

export default function Home() {
      return (
            <main className="min-h-screen">
                  <div className="flex flex-col items-center justify-center h-screen ">
                        <a href="https://rareskills.io" target="_blank" className="text-white font-bold text-3xl hover:text-[#0044CC]" > Viem.sh </a>
                        <div className="h-[300px] min-w-[150px] flex flex-col justify-between  backdrop-blur-2xl bg-[#290330]/30 rounded-lg mx-auto p-7 text-white border border-purple-950">
                              <WalletButton />
                              {/* <SendButton />
                              <MintButton /> */}
                        </div>
                        <a href="https://rareskills.io" target="_blank" className="text-white font-bold text-3xl hover:text-[#0044CC]" > Rareskills.io </a>
                 </div>
            </main>
      );
}
```

### globals.css

Some nice background UI, replace **globals.css** with the codes below.

```css!
@tailwind base;@tailwind components;@tailwind utilities;

body {
  background-color: #0c002e;
  background-image: radial-gradient(
      at 100% 100%,rgb(84, 2, 103) 0px,
      transparent 50%),
    radial-gradient(at 0% 0%, rgb(97, 0, 118) 0px, transparent 50%);}
```

### Step 4: Run the website and test it

When you click the button, it should initiate a connection to your wallet. Once you’ve approved, it should look like this:

```bash
npm run dev
```

![viem connect wallet](https://static.wixstatic.com/media/935a00_87d0b6e917ab4b6d9aa0a5758d39fdd9~mv2.png/v1/fill/w_666,h_454,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_87d0b6e917ab4b6d9aa0a5758d39fdd9~mv2.png)

After the button is clicked, the following will show:

![viem showing connected address](https://static.wixstatic.com/media/935a00_4a990af10c7342349ab08f07a72c0e2d~mv2.png/v1/fill/w_666,h_423,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_4a990af10c7342349ab08f07a72c0e2d~mv2.png)

## Part 2: Transferring Crypto with React + Viem

Now that our wallet is connected, we can start transfering cryptocurrencies. We will utilize the **sendTransaction** wallet action.

-   Get some Matic from [Matic Faucet](https://faucet.polygon.technology/)

### Step 5: Add feature to transfer cryptocurrency

Create a new tsx file **sendButton.tsx** in the app directory

app
├── client.ts
├── globals.css
├── layout.tsx
├── page.tsx
├── **sendButton.tsx**
└── walletButton.tsx

### sendButton.tsx

We’ll create a button that initiates the **sendTransaction** action. Viem make’s it very simple for us to do so. The logic flow should be similar to **walletButton.tsx**, instantiate the walletClient and perform Wallet Client actions.

```tsx!
"use client";
import { parseEther } from "viem";
import { ConnectWalletClient} from "./client";

export default function SendButton() {
     //Send Transaction Function
     async function handleClick() {
        try {
           // Declare wallet client
           const walletClient = ConnectWalletClient();
           // Get the main wallet address
           const [address] = await walletClient.getAddresses();
           // sendTransaction is a Wallet action.
           // It returns the transaction hash
           // requires 3 parameters  to transfer cryptocurrency,
           // account, to and value
           const hash = await walletClient.sendTransaction({
              account: address,
              to: "Account_Address",
              value: parseEther("0.001"), // send 0.001 matic
            });
            // Display the transaction hash in an alert
            alert(`Transaction successful. Transaction Hash: ${hash}`);
         } catch (error) {
              // Handle Error
              alert(`Transaction failed: ${error}`);
         }
     }

     return (
        <button
            className="py-2.5 px-2 rounded-md bg-[#1e2124] flex flex-row items-center justify-center border border-[#1e2124] hover:border hover:border-indigo-600 shadow-md shadow-indigo-500/10"
            onClick={handleClick}>
            Send Transaction
       </button>
    );
}
```

### Step 6: Insert sendButton Component

### page.tsx

Uncomment the lines relating to the sendButton component.

```tsx!
import WalletButton from "./walletButton";
import SendButton from "./sendButton";
// import MintButton from "./mintButton";

export default function Home() {

    return (
        <main className="min-h-screen">
            <div className="flex flex-col items-center justify-center h-screen ">
                <a href="https://rareskills.io" target="_blank" className="text-white font-bold text-3xl hover:text-[#0044CC]" > Viem.sh </a>
                <div className="h-[300px] min-w-[150px] flex flex-col justify-between  backdrop-blur-2xl bg-[#290330]/30 rounded-lg mx-auto p-7 text-white border border-purple-950">
                    <WalletButton />
                    <SendButton />
                    {/* <MintButton /> */}
                </div>
                <a href="https://rareskills.io" target="_blank" className="text-white font-bold text-3xl hover:text-[#0044CC]" > Rareskills.io </a>
            </div>
        </main>
    );
}
```

Your browser should look like this now!

![viem send transaction](https://static.wixstatic.com/media/935a00_7a03df653a83485ebeccac4b0eded604~mv2.png/v1/fill/w_666,h_493,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_7a03df653a83485ebeccac4b0eded604~mv2.png)

## Part 3: Minting an NFT with React + Viem

This section will discuss how to interact with a Smart Contract and give an example through minting an NFT.

To interact with a smart contract, we need two things:

-   Contract Address
-   Contract ABI

For this example, we’ll demonstrate it with Rareskill’s Contract that has a Mint function that doesn’t rlly do anything but track the number of times you Mint.

-   Rareskill’s Contract Address: **0x7E6Ddd9dC419ee2F10eeAa8cBB72C215B9Eb5E23**
-   [Rareskill’s Contract ABI](https://mumbai.polygonscan.com/address/0x7e6ddd9dc419ee2f10eeaa8cbb72c215b9eb5e23#code)

Feel free to use your own contract.

### Step 7: Add functionality to interact with Smart Contract

Create two new files **abi.ts** and **mintButton.tsx**

app
├── **abi.ts**
├── client.ts
├── globals.css
├── layout.tsx
├── **mintButton.tsx**
├── page.tsx
├── sendButton.tsx
└── walletButton.tsx

### abi.ts

Copy Paste the Rareskill’s Contract ABI or your own.

```ts
// abi.ts
export const wagmiAbi = [...contract abi...] as const;
```

Remember to follow this exact format and do not forget the “**as const;”** at the end.

### Contract Instance & Contract Action Method

**Contract Instance method**

```ts
//Contract Instance
const contract = getContract({
  address: "0x7E6Ddd9dC419ee2F10eeAa8cBB72C215B9Eb5E23",
  abi: wagmiAbi,
  publicClient,
  walletClient,
});
```

The **getContract** function creates our contract instance **contract**. Upon creation we are able to invoke contract methods, listen to events, etc. This is a simpler method as we do not have to repeatedly pass the **address** and **abi** properties to perform a contract action.

**Parameters:**

-   address
-   abi
-   publicClient (optional)
-   walletClient (optional)

We are required to pass the address and abi parameters. Passing publicClient and walletClient is optional, but it allows us access to a set of contract methods depending on the type of client.

Available contract methods for **publicClient**:

-   [**createEventFilter**](https://viem.sh/docs/contract/createContractEventFilter.html)
-   [**estimateGas**](https://viem.sh/docs/contract/estimateContractGas.html)
-   [**read**](https://viem.sh/docs/contract/readContract.html)
-   [**simulate**](https://viem.sh/docs/contract/simulateContract.html)
-   [**watchEvent**](https://viem.sh/docs/contract/watchContractEvent.html)

Available contract methods for **walletClient**:

-   [**estimateGas**](https://viem.sh/docs/contract/estimateContractGas.html)
-   [**write**](https://viem.sh/docs/contract/writeContract.html)

In general, calling a contract Instance method follows format below:

```ts
// function
contract.(estimateGas|read|simulate|write).(functionName)(args, options)

// event
contract.(createEventFilter|watchEvent).(eventName)(args, options)
```

Calling Contract Methods using Contract Instance

```ts
// Read Contract symbol
const symbol = await contract.read.symbol();

// Read Contract name
const name = await contract.read.name();

// Call mint method
const result = await contract.write.mint({account: address});
```

The examples above invokes the read and write contract method through the **contract** instance.If you’re using Type-script it would auto-complete suggestions of the available contract methods.

The **read.symbol()** and **read.name()** is straight forward. On the other hand, the write function

```ts
const result = await contract.write.mint({account: address});
```

takes {account: address} as it’s required parameter and all other’s optional. If you find yourself troubled with what parameters to add, hover your mouse on the “mint()” keyword, VS Code should hint you.

**Contract Action Method — the Tedious way**

The code in the above section is syntactic sugar for the following. We include this section to show you what is happening under the hood.

This code gets the totalSupply of the contract:

```ts
const totalSupply = await publicClient.readContract({
    address: '0x7E6Ddd9dC419ee2F10eeAa8cBB72C215B9Eb5E23',
    abi: wagmiAbi,
    functionName: 'totalSupply',
})
```

Tedious right? You have to recurringly pass the address and abi.

This is the equivalent of calling the **mint** function from the example above using the Contract Action method. Upon success it returns the transaction hash.

```ts
const hash = await walletClient.writeContract({
    address: "0x7E6Ddd9dC419ee2F10eeAa8cBB72C215B9Eb5E23",
    abi: wagmiAbi,
    functionName: "mint",
    account,
});
```

To keep bundle size minimal, use Contract Action; while Contract instance provides more functions, it increases memory use.

### mintButton.tsx

To demonstrate a state-chaning transaction, we’ll create a button that invokes the Smart Contract’s **mint** function as well as querying it’s **name, symbol** and **totalSupply**.

We utilized both the Contract Instance and Contract Action method to showcase how it’s implemented.

```tsx!
"use client";
import { formatEther, getContract } from "viem";
import { wagmiAbi } from "./abi";
import { ConnectWalletClient, ConnectPublicClient } from "./client";

export default function MintButton() {

  // Function to Interact With Smart Contract
  async function handleClick() {

    // Declare Client
    const walletClient = ConnectWalletClient();
    const publicClient = ConnectPublicClient();

    // Create a Contract Instance
    // Pass publicClient to perform Public Client Contract Methods
    // Pass walletClient to perform Wallet Client Contract Methods
    const contract = getContract({
      address: "0x7E6Ddd9dC419ee2F10eeAa8cBB72C215B9Eb5E23",
      abi: wagmiAbi,
      publicClient,
      walletClient,
    });

    // Reads the view state function symbol via Contract Instance method
    const symbol = await contract.read.symbol();

    // Reads the view state function name via Contract Instance method
    const name = await contract.read.name();

    // Reads the view state function symbol via Contract Action method
    const totalSupply = await publicClient.readContract({
      address: '0x7E6Ddd9dC419ee2F10eeAa8cBB72C215B9Eb5E23',
      abi: wagmiAbi,
      functionName: 'totalSupply',
    })

    // Format ether converts BigInt(Wei) to String(Ether)
    const totalSupplyInEther = formatEther(totalSupply);

    alert(`Symbol: ${symbol}\nName: ${name}\ntotalSupply: ${totalSupplyInEther}`);

    try {
      // Declare Wallet Client and Retrieve wallet address
      const client = walletClient;
      const [address] = await client.getAddresses();

      // Writes the state-changin function mint via Contract Instance method.
      const result = await contract.write.mint({
        account: address
      });

      alert(`${result} ${name}`);
    } catch (error) {
      // Handle any errors that occur during the transaction
      alert(`Transaction failed: ${error}`);
    }}

    return (
      <>
        <button
          className="py-2.5 px-2 rounded-md bg-[#1e2124] flex flex-row items-center justify-center border border-[#1e2124] hover:border hover:border-indigo-600 shadow-md shadow-indigo-500/10"
          onClick={handleClick}>
          <svg
            className="w-4 h-4 mr-2 -ml-1 text-[#626890]"
            aria-hidden="true"
            focusable="false"
            data-prefix="fab"
            data-icon="ethereum"
            role="img"
            xmlns="https://www.w3.org/2000/svg"
            viewBox="0 0 320 512">
            <path
              fill="currentColor"
              d="M311.9 260.8L160 353.6 8 260.8 160 0l151.9 260.8zM160 383.4L8 290.6 160 512l152-221.4-152 92.8z">
            </path>
          </svg>
          <h1 className="text-center">Mint</h1>
        </button>
      </>
    );
}
```

Congratulations on finishing this tutorial! Your final product should look like this:

![viem mint NFT](https://static.wixstatic.com/media/935a00_897b17a5855c4b0c8951b673e3e4d8e3~mv2.png/v1/fill/w_666,h_459,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_897b17a5855c4b0c8951b673e3e4d8e3~mv2.png)

### Authorship

This article was co-authored by Aymeric Taylor ([LinkedIn](https://www.linkedin.com/in/aymeric-russel-taylor/), [X](https://twitter.com/TaylorAymeric)), a research intern at RareSkills.

*Originally Published August 10, 2023*
